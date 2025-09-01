# 북적북적 DB 제약조건(FK, UNIQUE) 추가

## api-spec.md 수정
## requirements.md 수정
## project_overview.md 수정
## erd.md 수정
## schema.sql 수정

docs폴더에 들어갈 개발문서 작성및 정리



# 0.(선행) 기본 설정 결정

* **엔진/문자셋**: InnoDB, `utf8mb4` + `utf8mb4_0900_ai_ci`(MySQL) / `utf8mb4_general_ci`(MariaDB)
* **시간대/타임스탬프 규약**: DB는 UTC, 앱은 KST 표시
* **ID 전략**: `BIGINT AUTO_INCREMENT` 또는 UUID(서로 일관 되게)

---

# 1. 데이터베이스 & 계정 만들기

```sql
CREATE DATABASE bookjuk
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_general_ci; -- MariaDB 예시

CREATE USER 'bookjuk'@'%' IDENTIFIED BY 'strong-password';
GRANT ALL PRIVILEGES ON bookjuk.* TO 'bookjuk'@'%';
FLUSH PRIVILEGES;
```

---

# 2. “제약 포함” 스키마를 최초 마이그레이션(V1)로 적용

> 빈 DB라면 **처음부터 FK/UNIQUE를 켠 상태로 생성**하는 게 최선입니다.
> Flyway 기준 예: `V1__init_schema.sql`

## 생성 순서 (부모 → 자식)

1. `user`
2. `meeting`
3. `post`
4. `meeting_participant`
5. `meeting_review`
6. `comment`

## 예시 DDL (요지)

```sql
-- 1) user
CREATE TABLE user (
  user_id       BIGINT PRIMARY KEY AUTO_INCREMENT,
  email         VARCHAR(255) NOT NULL,
  username      VARCHAR(100) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  is_deleted    TINYINT(1) NOT NULL DEFAULT 0,
  created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT uq_user_email UNIQUE(email)
) ENGINE=InnoDB;

-- 2) meeting
CREATE TABLE meeting (
  meeting_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
  host_id         BIGINT NOT NULL,
  title           VARCHAR(255) NOT NULL,
  description     VARCHAR(2000),
  book_title      VARCHAR(255) NOT NULL,
  book_author     VARCHAR(100) NOT NULL,
  genre           VARCHAR(100),
  meeting_time    DATETIME NOT NULL,
  region          VARCHAR(20) NOT NULL,
  city            VARCHAR(30) NOT NULL,
  district        VARCHAR(30),
  detail_address  VARCHAR(255),
  max_participants INT NOT NULL,
  status          ENUM('RECRUITING','FULL','COMPLETED','CANCELLED') NOT NULL DEFAULT 'RECRUITING',
  created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_meeting_host_id(host_id),
  CONSTRAINT fk_meeting_host
    FOREIGN KEY (host_id) REFERENCES user(user_id)
    ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 3) post
CREATE TABLE post (
  post_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
  meeting_id  BIGINT NOT NULL,
  user_id     BIGINT NOT NULL,
  title       VARCHAR(255) NOT NULL,
  content     TEXT NOT NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_post_meeting_id(meeting_id),
  INDEX idx_post_user_id(user_id),
  CONSTRAINT fk_post_meeting
    FOREIGN KEY (meeting_id) REFERENCES meeting(meeting_id)
    ON DELETE CASCADE,
  CONSTRAINT fk_post_user
    FOREIGN KEY (user_id) REFERENCES user(user_id)
    ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 4) meeting_participant
CREATE TABLE meeting_participant (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  meeting_id  BIGINT NOT NULL,
  user_id     BIGINT NOT NULL,
  role        ENUM('HOST','MEMBER','PENDING') NOT NULL DEFAULT 'PENDING',
  joined_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_mp_meeting_id(meeting_id),
  INDEX idx_mp_user_id(user_id),
  CONSTRAINT uq_mp_meeting_user UNIQUE(meeting_id, user_id),
  CONSTRAINT fk_mp_meeting
    FOREIGN KEY (meeting_id) REFERENCES meeting(meeting_id)
    ON DELETE CASCADE,
  CONSTRAINT fk_mp_user
    FOREIGN KEY (user_id) REFERENCES user(user_id)
    ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 5) meeting_review (또는 like/평가 테이블)
CREATE TABLE meeting_review (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  meeting_id   BIGINT NOT NULL,
  reviewer_id  BIGINT NOT NULL,
  reviewee_id  BIGINT NOT NULL,
  score        INT,              -- 정책에 따라 1~5 등
  comment      VARCHAR(1000),
  created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_review_meeting(meeting_id),
  INDEX idx_review_reviewer(reviewer_id),
  INDEX idx_review_reviewee(reviewee_id),
  CONSTRAINT uq_mr_meeting_reviewer_reviewee UNIQUE(meeting_id, reviewer_id, reviewee_id),
  CONSTRAINT fk_mr_meeting
    FOREIGN KEY (meeting_id) REFERENCES meeting(meeting_id)
    ON DELETE CASCADE,
  CONSTRAINT fk_mr_reviewer
    FOREIGN KEY (reviewer_id) REFERENCES user(user_id)
    ON DELETE RESTRICT,
  CONSTRAINT fk_mr_reviewee
    FOREIGN KEY (reviewee_id) REFERENCES user(user_id)
    ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 6) comment
CREATE TABLE comment (
  comment_id  BIGINT PRIMARY KEY AUTO_INCREMENT,
  post_id     BIGINT NOT NULL,
  user_id     BIGINT NOT NULL,
  content     VARCHAR(1000) NOT NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_comment_post_id(post_id),
  INDEX idx_comment_user_id(user_id),
  CONSTRAINT fk_comment_post
    FOREIGN KEY (post_id) REFERENCES post(post_id)
    ON DELETE CASCADE,
  CONSTRAINT fk_comment_user
    FOREIGN KEY (user_id) REFERENCES user(user_id)
    ON DELETE RESTRICT
) ENGINE=InnoDB;
```

> 포인트
>
> * **UNIQUE**(이메일, (meeting\_id,user\_id), (meeting\_id,reviewer\_id,reviewee\_id))는 **테이블 생성 시** 바로 반영
> * FK는 전부 **즉시 적용**(빈 DB라 락/검사 비용이 사실상 0)
> * 삭제 정책: 모임 삭제 시 관련 자료 CASCADE, 사용자 삭제는 RESTRICT(운영에서 soft delete로 처리)

---

# 3. 시드 데이터(V2, 선택)

* 실제 운영 전에는 **시드 최소화** 권장(참여자/리뷰는 FK 때문에 반드시 부모 먼저)
* Flyway: `V2__seed_minimal.sql` 예시

```sql
INSERT INTO user (email, username, password_hash)
VALUES ('host@example.com','호스트','{bcrypt}...') -- 비번은 실제 bcrypt 해시

, ('member1@example.com','멤버1','{bcrypt}...')
, ('member2@example.com','멤버2','{bcrypt}...');

INSERT INTO meeting (host_id, title, book_title, book_author, genre, meeting_time, region, city, district, max_participants)
VALUES (1, '9월 독서모임', '달까지 가자', '장류진', '소설', '2025-09-15 19:30:00', '서울특별시', '강남구', '삼성동', 8);

-- 참가자(중복 방지 UNIQUE가 있으니 중복 시 실패)
INSERT INTO meeting_participant (meeting_id, user_id, role)
VALUES (1, 1, 'HOST'), (1, 2, 'MEMBER');
```

> **idempotent** 하게 만들고 싶다면 `INSERT IGNORE`(MySQL/MariaDB) 또는 `INSERT ... ON DUPLICATE KEY UPDATE` 사용.

---

# 4. 애플리케이션 설정(스키마 보호)

> JPA/Hibernate를 쓰면 **DDL 자동 생성을 끄고 검증만** 하세요.

```properties
# application-prod.properties
spring.jpa.hibernate.ddl-auto=validate   # create/update 금지, 스키마와 엔티티 불일치 시 실패
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.jdbc.time_zone=UTC
spring.datasource.url=jdbc:mariadb://host:3306/bookjuk?useUnicode=true&characterEncoding=utf8
```

---

# 5. 스모크 테스트 & 가드레일

* **생성 ⇒ 조회 ⇒ 삭제(CASCADE 확인)** 간단한 e2e 테스트
* 동시성(참가 신청/좋아요) 테스트로 UNIQUE 위반이 앱 레벨에서 잘 처리되는지 확인
* 관측성: `X-Request-Id` 로깅, FK/UNIQUE 위반 시 에러 변환 메시지

---

# 6. 한눈 요약

* 빈 DB면 **처음부터 FK/UNIQUE 켠 상태로 V1 마이그레이션**으로 생성
* **부모→자식** 테이블 순서로 만들고, 삭제 정책을 명확히
* `ddl-auto=validate`로 스키마를 보호
* (선택) 최소 시드 → e2e 스모크 → 동시성 확인


