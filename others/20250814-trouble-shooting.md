# 📌 2025년 8월 14일 트러블 슈팅

# 🛠️ Troubleshooting: @Builder.Default와 enum 변환 시 디폴트 값이 먹지 않는 문제

## 🔑 키워드

`Lombok` · `@Builder` · `@Builder.Default` · `Enum 변환` · `DTO → Condition 매핑` · `Null 처리`

---

## 1) 증상(Symptom)

* `MeetingSearchCondition`에 아래와 같이 디폴트를 걸어둠:

  ```java
  @Builder.Default
  private MeetingStatus status = MeetingStatus.RECRUITING;
  ```
* 그런데 요청에서 `status`를 안 주거나 잘못 넘긴 경우 **기대: RECRUITING**, **현실: null**로 들어가 정렬/필터 로직이 깨짐.

---

## 2) 원인(Root Cause)

* `@Builder.Default`는 “**빌더가 해당 필드에 값을 설정하지 않았을 때만**” 디폴트를 적용함.
* **명시적으로 null을 세팅하면** “값을 설정한 것”으로 간주되어 **디폴트 무시**.
* 특히 `DTO(status: String)` → `enumStatus: MeetingStatus` 변환 과정에서

    * 원본 문자열이 `null`이 아니더라도 변환 실패 시 `enumStatus`가 `null`이 될 수 있음
    * 이 `enumStatus(null)`를 빌더에 넣으면 **디폴트가 작동하지 않음**.

> ✅ 결론: **디폴트 적용 여부는 “변환 후 enum 변수(enumStatus)가 null인지”로 결정해야 함.**

---

## 3) 해결(Resolution)

### 전략

* `DTO → Condition` 변환 시점에서 **“변환된 enum이 null인지”** 확인하고, 그때 디폴트를 적용.
* 빌더에 `null`을 절대 넣지 않도록 **사전 방어**.

### 적용 코드 (안전한 변환 + 디폴트)

```java
// 예시: MeetingListSearchRequest → MeetingSearchCondition
public MeetingSearchCondition toCondition() {
    // 1) 문자열 → Enum 변환 (대소문자/유효성 방어)
    MeetingStatus enumStatus = null;
    if (status != null && !status.isBlank()) {
        try {
            enumStatus = MeetingStatus.valueOf(status.trim().toUpperCase());
        } catch (IllegalArgumentException e) {
            // 무시: 잘못된 값은 필터에서 제외
            enumStatus = null; // 안전하게 null 처리
        }
    }

    // 2) 변환 결과 기준으로 디폴트 적용 (핵심!)
    MeetingStatus safeStatus = (enumStatus != null)
            ? enumStatus
            : MeetingStatus.RECRUITING;

    return MeetingSearchCondition.builder()
            .status(safeStatus)       // null 금지
            // .genre(…)
            // .region1(…)
            // .region2(…)
            // .sort(…)
            .build();
}
```

> 포인트:
>
> * `status != null ? enumStatus : RECRUITING` 가 아니라
> * **`enumStatus != null ? enumStatus : RECRUITING`** 로 판단해야 정확함. (변환 실패 케이스 커버)

---

## 4) 대안(Alternative) — 생성자/팩토리에서 강제 디폴트

```java
public static MeetingSearchCondition of(String status /*, … */) {
    MeetingStatus enumStatus = null;
    if (status != null) {
        try { enumStatus = MeetingStatus.valueOf(status.toUpperCase()); } catch (Exception ignored) {}
    }
    return MeetingSearchCondition.builder()
        .status(enumStatus != null ? enumStatus : MeetingStatus.RECRUITING)
        .build();
}
```

* 장점: 생성 경로를 통일하면 빌더 남용을 줄이고 **null 유입 차단**이 쉬움.

---


## 5) 한 줄 요약(What I Learned)

* **@Builder.Default는 null을 “명시적으로” 세팅하면 무력화된다.**
* 디폴트는 \*\*“변환 후 enum 값(null/아님)”\*\*을 기준으로 적용해야 안전하다.


