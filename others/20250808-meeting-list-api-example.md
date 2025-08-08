# 모임 리스트 조회 API (검색/필터/정렬) - Spring JPA + QueryDSL 예제

## 0) 시나리오 가정
- 정렬 기준  
  - `latest`: 생성일 내림차순  
  - `closingSoon`: 모임시간(`meetingTime`) 오름차순  
  - `popular`: 현재참여인원(`currentParticipants`) 내림차순  
- 검색 필터는 **모두 optional** (파라미터 없으면 조건 미적용)

---

## 1) 엔티티 & DDL

```java
// 모임 상태
public enum MeetingStatus {
    RECRUITING, CLOSED, FINISHED
}
```

```java
// 호스트(간단히 사용자)
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    private String nickname;
    // ...
}
```

```java
@Entity
public class Meeting {
    @Id @GeneratedValue
    private Long id;

    private String title;
    private String imageUrl;

    private String region;        // ex) "강남구"
    private String genre;         // ex) "소설"

    private LocalDateTime meetingTime;

    @Enumerated(EnumType.STRING)
    private MeetingStatus status;

    private int currentParticipants;
    private int maxParticipants;

    @ManyToOne(fetch = FetchType.LAZY)
    private User host;

    private LocalDateTime createdAt;

    // 생성/수정일 자동화 하시면 좋습니다 (Auditing)
}
```

---

## 2) 요청/응답 DTO

```java
// 쿼리 파라미터 바인딩용
public record MeetingSearchCond(
        String region,
        String genre,
        MeetingStatus status,
        String sort // latest | closingSoon | popular
) {}
```

```java
// 리스트 아이템 응답
public record MeetingListItemDto(
        Long meetingId,
        String title,
        String imageUrl,
        String region,
        String genre,
        LocalDateTime meetingTime,
        MeetingStatus status,
        int currentParticipants,
        int maxParticipants,
        Long hostUserId,
        String hostNickname
) {}
```

```java
// 페이지 응답 래퍼
public record PageResponse<T>(
        List<T> content,
        int page,       // 0-based or 1-based는 컨벤션에 맞게
        int size,
        long totalElements
) {}
```

---

## 3) QueryDSL 리포지토리 (핵심)

```java
public interface MeetingQueryRepository {
    Page<MeetingListItemDto> search(MeetingSearchCond cond, Pageable pageable);
}
```

```java
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.Projections;
import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.jpa.impl.JPAQueryFactory;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;

import java.util.List;

import static com.example.domain.QMeeting.meeting;
import static com.example.domain.QUser.user;

@RequiredArgsConstructor
public class MeetingQueryRepositoryImpl implements MeetingQueryRepository {

    private final JPAQueryFactory query;

    @Override
    public Page<MeetingListItemDto> search(MeetingSearchCond cond, Pageable pageable) {
        BooleanBuilder where = new BooleanBuilder();

        if (cond.region() != null && !cond.region().isBlank()) {
            where.and(meeting.region.eq(cond.region()));
        }
        if (cond.genre() != null && !cond.genre().isBlank()) {
            where.and(meeting.genre.eq(cond.genre()));
        }
        if (cond.status() != null) {
            where.and(meeting.status.eq(cond.status()));
        }

        OrderSpecifier<?> order = toOrder(cond.sort());

        // 데이터 조회
        List<MeetingListItemDto> rows = query
                .select(Projections.constructor(MeetingListItemDto.class,
                        meeting.id,
                        meeting.title,
                        meeting.imageUrl,
                        meeting.region,
                        meeting.genre,
                        meeting.meetingTime,
                        meeting.status,
                        meeting.currentParticipants,
                        meeting.maxParticipants,
                        meeting.host.id,
                        meeting.host.nickname
                ))
                .from(meeting)
                .leftJoin(meeting.host, user) // host 정보 가져오기
                .where(where)
                .orderBy(order)               // 정렬
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();

        // 전체 개수
        long total = query
                .select(meeting.count())
                .from(meeting)
                .where(where)
                .fetchOne();

        return new PageImpl<>(rows, pageable, total);
    }

    private OrderSpecifier<?> toOrder(String sort) {
        if (sort == null) return meeting.createdAt.desc(); // 기본 latest
        return switch (sort) {
            case "closingSoon" -> meeting.meetingTime.asc();
            case "popular"     -> meeting.currentParticipants.desc();
            default            -> meeting.createdAt.desc(); // latest
        };
    }
}
```

---

## 4) 스프링 Data 리포지토리 + 서비스

```java
public interface MeetingRepository
        extends JpaRepository<Meeting, Long>, MeetingQueryRepository {
}
```

```java
@Service
@RequiredArgsConstructor
public class MeetingService {
    private final MeetingRepository meetingRepository;

    public PageResponse<MeetingListItemDto> search(MeetingSearchCond cond, Pageable pageable) {
        var page = meetingRepository.search(cond, pageable);
        return new PageResponse<>(
                page.getContent(),
                pageable.getPageNumber() + 1,
                pageable.getPageSize(),
                page.getTotalElements()
        );
    }
}
```

---

## 5) 컨트롤러

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/meetings")
public class MeetingController {

    private final MeetingService meetingService;

    @GetMapping
    public PageResponse<MeetingListItemDto> search(
            @RequestParam(required = false) String region,
            @RequestParam(required = false) String genre,
            @RequestParam(required = false) MeetingStatus status,
            @RequestParam(required = false, defaultValue = "latest") String sort,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "6") int size
    ) {
        MeetingSearchCond cond = new MeetingSearchCond(region, genre, status, sort);
        Pageable pageable = PageRequest.of(Math.max(0, page - 1), size);
        return meetingService.search(cond, pageable);
    }
}
```

---

## 6) 사용 예시

```
GET /api/meetings?region=강남구&genre=소설&status=RECRUITING&sort=latest&page=1&size=6
GET /api/meetings?genre=소설&status=RECRUITING&page=2&size=6
GET /api/meetings?sort=popular&page=1&size=6
```

---

## 7) 실무 팁
- 인덱스: `region, genre, status, meeting_time, created_at, current_participants`
- Enum은 `@Enumerated(EnumType.STRING)` 사용
- N+1: host join으로 해결
- 정렬키 널값 대비 필요시 coalesce
- 응답 포맷은 프로젝트 컨벤션에 맞춰 변경
