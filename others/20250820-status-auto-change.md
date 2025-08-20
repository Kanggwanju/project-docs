
# 1) 동작 목표 (요약)

* `meetingTime <= now` 이고 `status == RECRUITING` 인 모임들을 **5초마다** `COMPLETED`로 일괄 갱신
* 서버가 켜질 때 한 번 **밀린 것 보정** (기동 직후 즉시 정리)
* 다중 인스턴스면 선택적으로 **ShedLock**으로 중복 실행 방지

---

# 2) 코드 — “그대로 복붙 가능한 일반 버전”

## 2-1) 스케줄링 활성화

```java
// package 경로는 자유
package com.example.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableScheduling;

@Configuration
@EnableScheduling
public class SchedulingConfig {
}
```

## 2-2) 엔티티 가정 (참고용)

* 엔티티 이름: `Meeting`
* 상태 enum: `MeetingStatus { RECRUITING, COMPLETED, CANCELLED }`
* 모임 시작 시각: `meetingTime` (타입: `LocalDateTime`)

> 네 프로젝트의 실제 이름만 다르면 **레포지토리/스케줄러에서 필드·타입만 맞춰 바꾸면** 됩니다.

## 2-3) 벌크 업데이트용 레포지토리

```java
package com.example.meeting;

import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.time.LocalDateTime;

@Repository
public interface MeetingRepository extends JpaRepository<Meeting, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
        update Meeting m
           set m.status = :next
         where m.status = :curr
           and m.meetingTime <= :now
    """)
    int bulkCompleteMeetings(@Param("curr") MeetingStatus curr,
                             @Param("next") MeetingStatus next,
                             @Param("now")  LocalDateTime now);
}
```

> 만약 `meetingTime`이 `Instant`라면 `LocalDateTime now` → `Instant now` 로 바꾸고, 스케줄러에서도 `Instant.now()`를 쓰면 됩니다.

## 2-4) 스케줄러 서비스 (5초마다 + 기동 시 1회 보정)

```java
package com.example.meeting;

import lombok.RequiredArgsConstructor;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Clock;
import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class MeetingStatusScheduler {

    private final MeetingRepository meetingRepository;

    // 테스트 용이성과 일관된 now()를 위해 Clock 사용 (생략 가능)
    private final Clock clock = Clock.systemDefaultZone();

    // ✅ 5초마다 실행 (지연 기반: 이전 실행 끝난 후 5초 뒤)
    @Transactional
    @Scheduled(fixedDelay = 5000)
    public void sweep() {
        LocalDateTime now = LocalDateTime.now(clock);
        meetingRepository.bulkCompleteMeetings(
                MeetingStatus.RECRUITING, MeetingStatus.COMPLETED, now);
    }

    // ✅ 서버 기동 직후 한 번 밀린 것 정리
    @Transactional
    @EventListener(org.springframework.boot.context.event.ApplicationReadyEvent.class)
    public void catchupOnBoot() {
        LocalDateTime now = LocalDateTime.now(clock);
        meetingRepository.bulkCompleteMeetings(
                MeetingStatus.RECRUITING, MeetingStatus.COMPLETED, now);
    }
}
```

---

# 3) 인덱스 (권장 — 성능 안정화)

DB에 한 번만 적용하면 됩니다.

```sql
-- 테이블명이 meeting, 컬럼명이 meeting_time / status 라고 가정
CREATE INDEX idx_meeting_time_status ON meeting (meeting_time, status);
```

> 네 테이블/컬럼명이 다르면 거기에 맞춰 이름만 바꾸면 OK.

---

# 4) (선택) 다중 인스턴스일 때 중복 실행 방지 — ShedLock

단일 서버면 불필요합니다. 여러 대 띄울 때만 추가!

### 4-1) 의존성

* **Gradle**

```gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.15.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.15.0'
```

### 4-2) 설정 + 테이블 생성

```java
package com.example.config;

import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT30S") // 최대 잠금 유지 시간
public class ShedLockConfig {
}
```

```sql
-- ShedLock용 락 테이블 (JDBC 템플릿 프로바이더 기준)
CREATE TABLE shedlock(
    name VARCHAR(64) NOT NULL PRIMARY KEY,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL
);
```

### 4-3) 스케줄러 메서드에 락 적용

```java
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;

@Transactional
@Scheduled(fixedDelay = 5000)
@SchedulerLock(name = "MeetingStatusScheduler.sweep", lockAtMostFor = "PT30S", lockAtLeastFor = "PT5S")
public void sweep() { ... }
```

---

# 5) (선택) BDD 스타일 짧은 테스트

```java
package com.example.meeting;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
@Transactional
class MeetingStatusSchedulerTest {

    @Autowired MeetingRepository meetingRepository;
    @Autowired MeetingStatusScheduler scheduler;

    @Test
    void givenRecruitingPastMeeting_whenSweep_thenStatusBecomesCompleted() {
        // Given
        Meeting m = Meeting.builder()
                .status(MeetingStatus.RECRUITING)
                .meetingTime(LocalDateTime.now().minusMinutes(1))
                .title("테스트")
                .build();
        meetingRepository.save(m);

        // When
        scheduler.sweep();

        // Then
        Meeting reloaded = meetingRepository.findById(m.getId()).orElseThrow();
        assertEquals(MeetingStatus.COMPLETED, reloaded.getStatus());
    }
}
```

---

# 6) 네 프로젝트에 맞춰 **수정해야 할 부분 체크리스트**

꼭 확인해서 이름만 바꾸면 됩니다. (파일 구조/스키마는 그대로 유지)

1. **패키지/클래스 이름**

    * `com.example.meeting` → 너희 실제 패키지
    * `Meeting`, `MeetingRepository`, `MeetingStatusScheduler` 클래스명

2. **엔티티/필드명**

    * 엔티티: `Meeting`
    * 상태 필드: `status` (Enum: `RECRUITING, COMPLETED, CANCELLED`)
    * 시간 필드: `meetingTime`

        * 만약 `meetingStartAt/startedAt` 등 다른 이름 → 레포지토리 JPQL의 `m.meetingTime` 부분만 변경
        * 타입이 `Instant`면 레포지토리/스케줄러의 `LocalDateTime` → `Instant`로 통일

3. **DB 컬럼/테이블명** (인덱스 DDL)

    * 테이블: `meeting`
    * 컬럼: `meeting_time`, `status`
    * 실제 스키마 명칭에 맞춰 `CREATE INDEX` 문만 수정

4. **타임존**

    * `LocalDateTime`을 쓰면 서버 시간대(KST) 기준 비교
    * 다국가/표준화 필요하면 `Instant` 저장 + 화면에서 변환 권장

5. **권한/예외 상태**

    * `CANCELLED`는 자동 전환 대상 X (쿼리 WHERE에 `status = RECRUITING`이므로 이미 제외됨)
    * 추가 비즈니스 룰이 있다면 스케줄러 전/후크로 보완 가능

6. **다중 인스턴스**

    * 서버 여러 대면 ShedLock 섹션 적용(의존성 + 테이블 + @SchedulerLock)

---

# 7) 작동 확인 포인트

* 서버 기동 직후: **지난 모임**이 즉시 `COMPLETED`로 정리되는지 로그/DB 확인
* 시작 시간이 막 지난 모임: **최대 5초 내**에 `COMPLETED` 되는지 확인
* 목록 조회에서 상태 필터가 잘 걸리는지 점검
* 인덱스 적용 후 업데이트/조회 쿼리 성능 확인

---

## 한 줄 정리

* 제공한 코드 그대로 붙이고, **엔티티/필드/패키지 이름만** 네 프로젝트에 맞게 바꾸면 끝!
* 단일 서버면 이대로 충분, 다중 서버면 ShedLock만 얹어주면 안전해.
