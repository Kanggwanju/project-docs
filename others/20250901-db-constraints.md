
# 북적북적 DB 제약조건(FK, UNIQUE) 추가
- 발표 피드백에 따라 DB에 제약 조건을 추가했습니다.
- 이 문서에는 그에 따른 변경 사항들을 정리했습니다.

## 변경사항
- schema.sql 수정
- api-spec.md 수정
- requirements.md 수정
- project_overview.md 수정
- erd.md 수정
- README.md 수정

---

## 1. schma.sql (DB 스키마 v1.1 업데이트 내역)
- 이번 업데이트는 데이터베이스의 데이터 무결성과 일관성을 강화하기 위해 여러 제약 조건을 추가했습니다.

---

### 1. `user` 테이블
* **`uq_user_email UNIQUE(email)`**
    * **역할**: **이메일의 중복을 방지**합니다.
    * **목적**: 모든 사용자가 유일한 이메일 주소를 갖도록 강제하여, 회원가입 시 중복된 계정이 생성되는 것을 막습니다.

---

### 2. `meeting` 테이블
* **`fk_meeting_host FOREIGN KEY (host_id) REFERENCES user(user_id) ON DELETE RESTRICT`**
    * **역할**: `meeting` 테이블의 `host_id`가 `user` 테이블의 `user_id`를 참조하는 **외래 키**입니다.
    * **목적**: 모임 주최자(`host_id`) 데이터가 **존재하는 한**, 해당 유저(`user_id`)를 삭제할 수 없도록 **제한**합니다.

---

### 3. `meeting_participant` 테이블
* **`uq_mp_meeting_user UNIQUE(meeting_id, user_id)`**
    * **역할**: **복합 유니크 제약 조건**으로, 한 모임에 한 사용자가 **두 번 이상 참여 신청**하는 것을 방지합니다.
* **`fk_mp_meeting FOREIGN KEY (meeting_id) REFERENCES meeting(meeting_id) ON DELETE CASCADE`**
    * **역할**: 모임(`meeting`)이 삭제될 때, 이 모임에 연결된 모든 참여자 데이터(`meeting_participant`)를 **연쇄적으로 함께 삭제**합니다.
    * **목적**: 모임이 사라지면 참여자 데이터도 의미가 없어지므로, 자동으로 데이터를 정리합니다.
* **`fk_mp_user FOREIGN KEY (user_id) REFERENCES user(user_id) ON DELETE RESTRICT`**
    * **역할**: 모임에 참여한 기록(`meeting_participant`)이 있는 사용자를 **삭제할 수 없도록 제한**합니다.
    * **목적**: 사용자 삭제 시 의도치 않게 참여 기록이 사라지는 것을 방지합니다.

---

### 4. `meeting_review` 테이블
* **`uq_mr_meeting_reviewer_reviewee UNIQUE(meeting_id, reviewer_id, reviewee_id)`**
    * **역할**: 한 모임에서 `리뷰어`가 특정 `리뷰이`에게 **한 번만 리뷰**할 수 있도록 합니다.
* **`chk_not_self_review CHECK (reviewer_id <> reviewee_id)`**
    * **역할**: `reviewer_id`와 `reviewee_id`가 **같지 않아야 한다**는 규칙을 설정합니다.
    * **목적**: 데이터베이스 차원에서 **자기 자신을 리뷰하는 것을 방지**합니다.
* **`fk_mr_meeting FOREIGN KEY (meeting_id) ... ON DELETE CASCADE`**
    * **역할**: 모임이 삭제될 때, 해당 모임에 대한 모든 리뷰 데이터도 **함께 삭제**합니다.
* **`fk_mr_reviewer FOREIGN KEY (reviewer_id) ... ON DELETE RESTRICT`**
    * **역할**: 리뷰를 남긴 사용자를 **삭제할 수 없도록 제한**합니다.
* **`fk_mr_reviewee FOREIGN KEY (reviewee_id) ... ON DELETE RESTRICT`**
    * **역할**: 리뷰를 받은 사용자를 **삭제할 수 없도록 제한**합니다.

---

### 5. `post` 및 `comment` 테이블
* **`fk_post_meeting ... ON DELETE CASCADE` (Post)**
* **`fk_comment_post ... ON DELETE CASCADE` (Comment)**
    * **역할**: 모임이 삭제되면 **모든 게시글이** 함께 삭제되고, 게시글이 삭제되면 **모든 댓글이** 함께 삭제됩니다.
    * **목적**: 논리적으로 종속된 데이터를 한 번에 정리하여 데이터 일관성을 유지합니다.
* **`fk_post_user ... ON DELETE RESTRICT` (Post)**
* **`fk_comment_user ... ON DELETE RESTRICT` (Comment)**
    * **역할**: 게시글이나 댓글을 작성한 사용자를 **삭제할 수 없도록 제한**합니다.
    * **목적**: 유저 삭제 시 관련 활동 기록이 사라지는 것을 방지합니다.

---

## 2. api-spec.md (API 명세서 v2.1 업데이트 내역)
- 이번 업데이트는 DB 제약 조건을 추가한 것을 토대로, API 명세서의 DB 참고사항을 수정했습니다.

### 수정사항
- 기존의 제약 조건이 없고, 이로인해 애플리케이션 레벨에서 무결성을 검증해야 한다는 문구 삭제
- 제약 조건을 통한 데이터 무결성이 보장됨을 언급
- 애플리케이션 레벨에서 제약 조건 위반에 대한 예외 처리가 필요함을 언급

---

## 3. requirements.md (요구사항 명세서 v1.5 업데이트 내역)
- 이번 업데이트는 DB 제약 조건을 추가한 것을 토대로, 요구사항 명세서를 수정했습니다.

### 수정사항
- DB 스키마 정책 변경, 그로인한 선택 사유 문구 변경
- 참여 신청/취소에서 `DB의 복합 UNIQUE 제약으로 방지` 문구 추가
- 데이터 모델 요약에서 제약 사항 변경

---

## 4. project_overview.md (프로젝트 개요 v1.2 업데이트 내역)
- 이번 업데이트는 DB 제약 조건을 추가한 것을 토대로, 프로젝트 개요를 수정했습니다.

### 수정사항
- 기술 스택(데이터베이스: MariaDB)
  - 제약 추가로 인한 장점 변경, 기존(유연성, 개발 속도, 확장성), 수정(MySQL 호환, 비용 효율적, 국내 운용사례 풍부)
  - 스키마 정책 추가
  - 이유 수정, 무결성과 동시성을 만족한다는 이야기로 변경
---
## 5. erd.md (ERD v1.8 업데이트 내역)
- 이번 업데이트는 DB 제약 조건을 추가한 것을 토대로, ERD 문서를 수정했습니다.

### 수정사항
* **설계 원칙 변경**: 기존의 "애플리케이션 레벨 무결성"에서 벗어나, "DB와 애플리케이션의 협업을 통한 무결성 관리"로 설계 원칙을 변경했습니다.
* **Mermaid ERD 수정**:
  * `UNIQUE`, `FK`, `CHECK` 등 **물리적 제약 조건**을 각 컬럼 옆에 명시하여 DB의 역할을 시각화했습니다.
  * 테이블 간의 논리적 관계 대신 **물리적 외래 키 제약**을 반영했습니다.
* **테이블 상세 정보 수정**: 각 테이블의 `제약조건` 컬럼에 `FK`, `UNIQUE` 등 DB 제약 조건을 명확하게 추가했습니다.
* **관계 설명 수정**: 모든 관계가 **물리적 외래 키** 제약 조건으로 정의되었음을 명시했습니다.
* **비즈니스 규칙 수정**: "애플리케이션 검증"이라는 문구에 "(DB의 제약 조건으로 보장)"을 추가하여, DB와 애플리케이션이 이중으로 무결성을 검증하고 있음을 강조했습니다.

---

## 6. README.md
- DB 제약 조건을 추가한 것을 토대로, README 문서를 수정했습니다.

### 수정사항
* **기존**: `FK 제약 조건 제거로 유연성 확보, 애플리케이션 레벨에서 무결성 보장`
* **변경**: `DB 제약 조건(FK, UNIQUE, CHECK)을 통해 데이터 무결성을 강력하게 보장, 애플리케이션은 사용자 친화적인 비즈니스 로직과 동시성 제어에 집중`

