# 데이터베이스

## 1. 트랜잭션과 ACID

> **한 줄 요약:** 다 되거나 안 되거나 → 규칙 지키고 → 서로 안 보고 → 저장되면 끝

**정의.** 트랜잭션은 여러 작업을 하나로 묶은 논리적 작업 단위입니다. 커밋하면 전부 반영되고, 롤백하면 전부 취소됩니다. 트랜잭션이 보장해야 하는 네 가지 성질이 ACID입니다.

| 성질 | 뜻 | 계좌 이체 예시 (A → B로 10만 원) | DB가 지키는 방법 |
|---|---|---|---|
| **A**tomicity 원자성 | 전부 반영되거나 전부 취소된다 | 출금만 되고 입금 전에 장애가 나면 출금도 취소 | 언두 로그, 롤백 |
| **C**onsistency 일관성 | 실행 전후로 제약 조건이 지켜진다 | 잔액이 음수가 될 수 없다는 규칙 유지 | PK·UNIQUE·FK·CHECK 제약 |
| **I**solation 격리성 | 동시에 실행되는 트랜잭션끼리 중간 상태를 보지 않는다 | 출금만 된 상태를 다른 조회가 보지 못함 | 락, MVCC, 격리 수준 |
| **D**urability 지속성 | 커밋된 결과는 장애가 나도 남는다 | 커밋 직후 정전이 나도 이체 결과 유지 | 리두 로그, WAL |

```python
# Django: 두 단계를 한 트랜잭션으로 묶는다
from django.db import transaction

with transaction.atomic():
    DailyReport.objects.filter(day=today).delete()
    DailyReport.objects.bulk_create(build_reports(today))
# 중간에 예외가 나면 delete도 롤백된다 → "지운 상태"로 남지 않는다
```

### 꼬리 질문: 장애 뒤 원자성과 지속성은 무엇으로 지키나?

- DB는 데이터 파일을 바꾸기 **전에** 변경 내용을 로그에 먼저 디스크에 기록합니다. 이것이 **WAL(Write-Ahead Logging)**입니다. 커밋은 로그가 디스크에 써졌을 때 성공으로 칩니다.
- 재시작하면 **리두 로그**로 커밋됐지만 데이터 파일에 아직 반영되지 않은 변경을 **다시 실행**합니다 → 지속성
- **언두 로그**로 커밋되지 않은 변경을 바꾸기 전 값으로 **되돌립니다** → 원자성
- 로그는 끝에 이어 쓰기만 하면 돼서 빠릅니다. 그래서 커밋 때는 로그만 확실히 쓰고, 흩어진 데이터 파일 반영은 나중에 몰아서 합니다.
- MySQL InnoDB에서는 redo log와 undo log가 이 역할을 하고, undo log는 MVCC의 이전 버전 읽기에도 쓰입니다.

---

## 2. 트랜잭션 격리 수준

> **한 줄 요약:** 이름이 곧 막는 문제 (Committed → Dirty, Repeatable → Non-Repeatable, Serializable → Phantom)

**정의.** 격리 수준은 동시에 실행되는 트랜잭션이 서로의 변경을 얼마나 볼 수 있는지 정하는 단계입니다. 엄격할수록 정합성은 좋아지고 동시 처리량은 떨어집니다.

**먼저 문제 세 가지**

| 문제 | 뜻 | 예시 |
|---|---|---|
| Dirty Read | **커밋 안 된** 다른 트랜잭션의 변경을 읽음 | B가 잔액을 0으로 바꾸고 커밋 전인데 A가 0을 읽음 → B가 롤백하면 A는 없던 값을 본 셈 |
| Non-Repeatable Read | 같은 **행**을 두 번 읽었는데 값이 다름 | A가 100을 읽음 → B가 50으로 바꾸고 커밋 → A가 다시 읽으니 50 |
| Phantom Read | 같은 **조건**으로 두 번 조회했는데 행 개수가 다름 | A가 "오늘 주문" 3건 조회 → B가 1건 추가·커밋 → A가 다시 조회하니 4건 |

**격리 수준 네 가지** (아래로 갈수록 엄격)

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | 생김 | 생김 | 생김 |
| Read Committed | 막음 | 생김 | 생김 |
| Repeatable Read | 막음 | 막음 | 생김 (SQL 표준 기준) |
| Serializable | 막음 | 막음 | 막음 |

- 기본값: MySQL InnoDB는 **Repeatable Read**, PostgreSQL과 Oracle은 **Read Committed**
- InnoDB의 Repeatable Read는 일반 조회는 MVCC 스냅샷으로, 잠금 조회(`FOR UPDATE` 등)는 넥스트 키 락으로 처리해 표준에서 허용하는 팬텀 리드도 대부분 막습니다.

### 꼬리 질문: Read Committed에서 같은 행을 두 번 읽었는데 값이 달랐다면?

- **Non-Repeatable Read**입니다. **Repeatable Read**로 올리면 막힙니다.
- 어떻게 막나: 트랜잭션이 처음 읽은 시점의 스냅샷(MVCC)을 끝날 때까지 계속 읽기 때문에, 다른 트랜잭션이 커밋해도 같은 행은 같은 값으로 보입니다.

---

## 3. 낙관적 락과 비관적 락

> **한 줄 요약:** 비관적은 먼저 잠그고, 낙관적은 나중에 확인한다

| | 비관적 락 (Pessimistic) | 낙관적 락 (Optimistic) |
|---|---|---|
| 가정 | 충돌이 **자주** 난다 | 충돌은 **드물다** |
| 방법 | 읽을 때 `SELECT … FOR UPDATE`로 **먼저 잠근다** | 잠그지 않고 읽고, **저장할 때 버전을 확인**한다 |
| 충돌 시 | 다른 트랜잭션이 기다린다 | 저장이 실패하면 다시 읽고 **재시도** |
| 대가 | 대기로 처리량 감소, 데드락 위험 | 충돌이 잦으면 재시도가 폭증 |
| 어울리는 곳 | 재고 차감, 좌석 예약처럼 같은 행에 몰릴 때 | 게시글 수정처럼 동시 수정이 드물 때 |

```sql
-- 비관적 락: 트랜잭션이 끝날 때까지 이 행을 다른 트랜잭션이 못 바꾼다
SELECT stock FROM items WHERE id = 10 FOR UPDATE;

-- 낙관적 락: 읽을 때 version = 3이었다면
UPDATE items SET stock = stock - 1, version = 4
WHERE id = 10 AND version = 3;
-- 영향받은 행이 0이면 그 사이 누군가 먼저 바꾼 것 → 다시 읽고 재시도
```

- 낙관적 락은 이름만 락이고 DB에 실제 잠금을 걸지 않습니다. 저장 시점에 충돌을 **검사**하는 방식입니다.
- 함께 나오는 용어: 공유 락(S, 읽기끼리는 같이 잡을 수 있음), 배타 락(X, 혼자만 잡음). `FOR UPDATE`는 배타 락입니다.

### 꼬리 질문: 중복 주문을 락 없이 막을 수 있나?

- 막을 수 있습니다. 외부 주문번호 같은 **자연키에 UNIQUE 제약**을 걸고 **UPSERT**로 저장하면, 같은 주문이 동시에 두 번 들어와도 DB가 하나만 남기고 두 번째는 갱신으로 흡수합니다(아래 4절).
- 정확히는 "애플리케이션이 락을 잡지 않는다"입니다. DB는 UNIQUE 인덱스에 행을 넣을 때 내부적으로 짧은 잠금을 씁니다.

---

## 4. 멱등성과 중복 방지

> **한 줄 요약:** 자연키 + UNIQUE + UPSERT, 그리고 "조회 후 INSERT는 동시 요청이 둘 다 통과한다"

**정의.** 멱등성은 같은 요청을 여러 번 처리해도 한 번 처리한 것과 결과가 같은 성질입니다. 재시도, 중복 실행, 중복 수신이 있어도 데이터가 한 번만 반영되게 하는 기반입니다.

**방법.** 같은 요청인지 판별할 키를 정하고, 그 키가 유일하다는 것을 DB가 강제하게 합니다.

```sql
CREATE TABLE orders (
  order_no     VARCHAR(40) PRIMARY KEY,  -- 외부 시스템의 주문번호 (자연키)
  status       VARCHAR(20) NOT NULL,
  confirmed_at DATETIME NULL             -- 내부에서 확정한 값 (외부 데이터로 덮어쓰면 안 됨)
);

-- 같은 주문이 다시 들어오면 새 행이 아니라 갱신으로 흡수된다
INSERT INTO orders (order_no, status) VALUES ('A-1001', 'PAID')
ON DUPLICATE KEY UPDATE status = VALUES(status);
-- MySQL 8.0.20부터 VALUES()는 권장되지 않음 → 행 별칭 문법: ... VALUES (...) AS new ON DUPLICATE KEY UPDATE status = new.status
```

**왜 DB 제약이어야 하나 (check-then-act 경합)**

```
요청 1: SELECT … 'A-1001' → 없음
요청 2: SELECT … 'A-1001' → 없음      ← 요청 1이 아직 INSERT 전
요청 1: INSERT 'A-1001'
요청 2: INSERT 'A-1001'               ← 제약이 없으면 중복 행
```

- 코드에서 "조회해 보고 없으면 INSERT"하면 동시 요청이 둘 다 통과합니다. UNIQUE 제약이 **마지막 방어선**입니다.
- auto-increment id를 PK로 쓰면 매번 다른 값이라 충돌하지 않습니다. 반드시 **요청을 식별하는 자연키**에 제약을 겁니다.
- 요청에 자연키가 없으면 클라이언트가 **멱등 키**(UUID 등)를 붙여 보내고, 서버가 같은 키를 일정 시간 무시합니다.
- UPSERT가 막는 것은 DB 행 중복뿐입니다. 메시지 발송, 외부 API 호출 같은 부수 효과는 따로 막아야 합니다.

### 꼬리 질문: UPSERT가 덮어쓰면 안 되는 필드는 어떻게 지키나?

> 행을 잠그지 말고, 컬럼마다 주인을 정한다

| 방법 | 예시 | 언제 |
|---|---|---|
| 갱신 목록에서 뺀다 | 위 예시처럼 `confirmed_at`을 `UPDATE` 뒤에 적지 않음 | 외부 데이터가 주인인 컬럼만 갱신할 때 |
| 조건부로 갱신한다 | `status = IF(status = 'CONFIRMED', status, VALUES(status))` | 확정된 행은 기존 값을 유지할 때 |
| 아무것도 안 바꾼다 (no-op) | `ON DUPLICATE KEY UPDATE order_no = order_no` | 처음 들어온 값이 이겨야 할 때 |

- 행 전체를 "수정 불가"로 잠그면 확정 뒤에 들어오는 취소 같은 변경을 놓칩니다. 그래서 행이 아니라 **컬럼 단위로 소유권**을 나누고, 상태는 한 방향으로만 바뀌게 합니다.
