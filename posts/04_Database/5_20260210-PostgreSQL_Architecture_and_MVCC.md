---
title: "PostgreSQL 깊이 파헤치기: 객체-관계형 DB의 유연함과 MVCC의 원리"
series_order: 5
description: "개발자들이 사랑하는 '가장 진보된 오픈 소스 DB', PostgreSQL의 독특한 철학과 아키텍처, 그리고 성능의 핵심인 MVCC를 알아봅니다."
date: 2026-02-10
update: 2026-02-11
tags: [Database, PostgreSQL, MVCC, Vacuum, JSONB, 백엔드, 데이터베이스아키텍처]
---

# PostgreSQL 깊이 파헤치기: 객체-관계형 DB의 유연함과 MVCC의 원리

## 서론
지난 시간에 MySQL이라는 대중적인 세단을 타봤다면, 오늘은 조금 더 묵직하고 다재다능한 **특수 목적 차량**에 가까운 **PostgreSQL**에 올라탈 시간입니다.

개발자들 사이에서 "고급 기능을 원하면 PostgreSQL을 쓴다"는 말이 있을 정도로, 이 DB는 강력하고 풍부한 기능을 자랑합니다. 단순히 데이터를 저장하는 것을 넘어, 복잡한 쿼리와 특수한 데이터 타입까지 유연하게 처리하는 이 DB의 정체는 무엇일까요?

![PostgreSQL 시스템 아키텍처](/images/04_Database/PostgreSQL_Architecture_and_MVCC/postgresql_architecture_diagram.png)
*강력한 확장성과 다양한 데이터 타입을 지원하는 PostgreSQL의 핵심 프로세스 및 메모리 구조도입니다.*

오늘은 PostgreSQL이 왜 **'객체-관계형 데이터베이스(ORDBMS)'**라고 불리는지, 그리고 MySQL과는 다른 방식의 **MVCC**와 **Vacuum**에 대해 상세히 파헤쳐 보겠습니다.

## 본론

### 1. ORDBMS: 데이터 타입의 한계를 넘다
PostgreSQL은 단순히 테이블만 제공하지 않습니다. 개발자가 직접 데이터 타입을 정의할 수 있고, DB 내부에서 복잡한 로직을 수행하는 함수를 자유롭게 작성할 수 있습니다.

- **JSONB의 활용**: NoSQL처럼 JSON 데이터를 저장하면서도 인덱스를 걸어 빠르게 조회할 수 있습니다. RDB의 안정성과 NoSQL의 유연함을 동시에 챙길 수 있죠.
- **다양한 인덱스 지원**: B-Tree는 기본이고 지리 정보를 위한 **GiST**, 전문 검색을 위한 **GIN** 등 거의 모든 형태의 데이터를 효율적으로 탐색할 수 있는 무기를 갖추고 있습니다.

---

### 2. PostgreSQL식 MVCC: "기존 데이터는 보존합니다"
MySQL이 'Undo 로그'를 사용한다면, PostgreSQL은 방식이 완전히 다릅니다. 데이터를 수정하면 기존 데이터를 덮어쓰는 게 아니라, **새로운 버전의 데이터 행(Tuple)**을 하나 더 생성합니다.

- **동작 방식**: 읽는 사용자에게는 예전 버전을, 쓰는 사용자에게는 새 버전을 보여줍니다. 읽기와 쓰기가 서로를 기다릴 필요가 전혀 없습니다.
- **장점**: 읽기 작업이 아무리 많아도 쓰기 성능에 영향을 주지 않아, 대규모 읽기 요청이 발생하는 환경에서 매우 안정적입니다.

---

### 3. 필연적인 관리 작업: Vacuum
새로운 버전을 계속 생성하다 보면, 이전에 사용하던 예전 데이터(Dead Tuple)는 쓰레기가 됩니다. 이 쓰레기들이 디스크 공간을 차지하고 성능을 떨어뜨리는데, 이를 정리하는 작업이 바로 **Vacuum**입니다.

![PostgreSQL Vacuum 동작 원리](/images/04_Database/PostgreSQL_Architecture_and_MVCC/vacuum_mechanism_diagram.png)
*데이터 수정/삭제로 인해 발생하는 Dead Tuple을 정리하고 디스크 공간을 회수하는 Vacuum 프로세스의 동작 원리입니다.*

- **Auto Vacuum**: 다행히 PostgreSQL은 백그라운드에서 자동으로 청소를 해줍니다. 하지만 대규모 수정/삭제 작업이 일어난 후에는 이 청소부가 제대로 작동하는지 모니터링하는 것이 필수입니다.

| 비교 항목 | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **MVCC 구현** | Undo 로그 (기존 데이터 수정) | 데이터 다중 버전화 (새로운 행 생성) |
| **공간 관리** | 기존 공간 재사용 위주 | Vacuum을 통한 공간 회수 필요 |
| **확장 기능** | 제한적임 | 플러그인과 확장이 매우 강력함 |

---

### 4. 실무 운영 인사이트
1.  **커넥션 관리가 핵심**: PostgreSQL은 커넥션 하나당 별도의 프로세스를 생성합니다. 커넥션이 너무 많아지면 메모리 부족으로 시스템이 멈출 수 있으니, 반드시 **PgBouncer** 같은 커넥션 풀러를 사용하세요.
2.  **데이터 타입의 유연함**: `UUID`를 기본 지원하며, 배열(`Array`) 타입도 컬럼에 저장할 수 있습니다. 무리하게 테이블을 쪼개기보다 배열이나 JSONB를 활용하는 것이 성능상 유리할 때가 있습니다.
3.  **CTE (WITH 절) 활용**: 복잡한 쿼리를 작성할 때 `WITH` 절을 사용하면 가독성뿐만 아니라 쿼리 최적화에도 도움을 줍니다. 가독성 좋은 코드가 튜닝도 쉽습니다.

```sql
-- CTE와 JSONB를 활용한 우아한 쿼리 예시
WITH target_users AS (
    SELECT id FROM users WHERE metadata @> '{"role": "admin"}'
)
SELECT * FROM orders WHERE user_id IN (SELECT id FROM target_users);
```
**컴퓨터의 평**: "어떤 복잡한 데이터라도 가져오세요. 제가 가장 효율적인 경로로 찾아내겠습니다!"

## 결론 및 요약 / 회고
PostgreSQL은 알면 알수록 그 깊이와 유연함에 놀라게 되는 DB입니다.
- **ORDBMS**로서 데이터 모델링의 자유도를 높여줍니다.
- **MVCC** 방식 덕분에 읽기/쓰기 간 간섭이 적어 안정적입니다.
- **Vacuum**의 특성을 이해해야 장기적인 성능 저하를 방지할 수 있습니다.

데이터의 **정확성, 복잡성, 확장성**이 모두 중요한 비즈니스라면 PostgreSQL은 최고의 선택지가 될 것입니다.

## 참고 자료
- PostgreSQL 내부 구조 (스즈키 사토시 저)
- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/current/index.html)
- Mastering PostgreSQL (한스-위르겐 쇤니히 저)
---
