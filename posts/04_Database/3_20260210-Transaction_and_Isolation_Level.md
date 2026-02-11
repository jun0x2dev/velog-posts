---
title: "트랜잭션과 격리 수준: 데이터 무결성을 지키는 ACID 원칙과 격리 수준의 내부 동작"
series_order: 3
description: "금융 사고를 방지하는 데이터 정합성의 근간, 트랜잭션의 ACID 원칙과 데이터베이스 격리 수준(Isolation Level)을 상세히 알아봅니다."
date: 2026-02-10
update: 2026-02-11
tags: [Database, Transaction, ACID, IsolationLevel, 데이터베이스, 백엔드, 데이터무결성]
---

# 트랜잭션과 격리 수준: 데이터 무결성을 지키는 ACID 원칙과 격리 수준의 내부 동작

## 서론
여러분이 친구에게 식사비를 송금한다고 상상해 봅시다. 내 계좌에서 돈은 빠져나갔는데, 갑자기 서버에 장애가 발생해 친구 계좌에는 돈이 들어오지 않았습니다. 내 돈은 공중으로 분해된 걸까요? 이런 문제를 방지하기 위해 데이터베이스에는 **'전부가 아니면 전무(All or Nothing)'**라는 철학이 존재합니다. 

![트랜잭션 송금 프로세스 다이어그램](/images/04_Database/Transaction_and_Isolation_Level/transaction_transfer_flow.png)
*송금 과정에서 '원자성(Atomicity)'이 보장되지 않았을 때 발생할 수 있는 데이터 불일치 상황을 묘사한 프로세스 다이어그램입니다.*

오늘은 데이터베이스의 신뢰성을 지탱하는 **ACID 원칙**과, 여러 사용자가 동시에 데이터를 수정할 때 발생하는 혼란을 잠재우는 **격리 수준(Isolation Level)**에 대해 파헤쳐 보겠습니다.

## 본론

### 1. 트랜잭션(Transaction)과 ACID 원칙
트랜잭션은 하나의 논리적인 작업 단위입니다. 이 단위가 안전하게 수행되기 위해서는 다음 네 가지 성질을 반드시 만족해야 합니다.

- **Atomicity (원자성)**: "전부 성공하거나, 전부 실패하거나." 중간 단계는 없습니다. 실패하면 처음부터 없었던 일처럼 되돌립니다(Rollback).
- **Consistency (일관성)**: 트랜잭션이 완료된 후에도 DB의 무결성 제약 조건은 항상 유지되어야 합니다.
- **Isolation (격리성)**: 실행 중인 트랜잭션은 다른 트랜잭션의 방해를 받지 않습니다.
- **Durability (지속성)**: 성공적으로 완료된 트랜잭션은 시스템 장애가 발생해도 영구적으로 보존됩니다. 
- **Under the hood**: 이를 위해 DB는 **WAL(Write Ahead Log)** 방식을 사용합니다. 실제 데이터를 쓰기 전에 로그 파일에 먼저 기록하여, 장애 발생 시 로그를 보고 복구하는 것이죠.

---

### 2. 격리 수준(Isolation Level): 안전과 성능의 상충 관계
모든 트랜잭션을 완벽하게 순차 실행하면 데이터는 가장 안전하겠지만, 서비스 속도가 기하급수적으로 느려질 것입니다. 그래서 우리는 상황에 맞춰 격리 수준을 타협합니다.

| 격리 수준 | 특징 | 발생 가능한 정합성 이슈 |
| :--- | :--- | :--- |
| **Read Uncommitted** | 커밋되지 않은 데이터도 읽음 | Dirty Read |
| **Read Committed** | 커밋된 데이터만 읽음 | Non-repeatable Read |
| **Repeatable Read** | 트랜잭션 내내 일관된 조회 결과 보장 | Phantom Read (MySQL에서는 대부분 방지) |
| **Serializable** | 완벽한 직렬화 실행 | 없음 (하지만 성능 저하 심각) |

---

### 3. 동시성 제어 실패 시 발생하는 현상들
격리 수준이 낮으면 다음과 같은 데이터 정합성 문제가 발생합니다.

- **Dirty Read**: 다른 트랜잭션이 수정 중인 확정되지 않은 데이터를 읽었다가, 그 트랜잭션이 롤백되어 잘못된 정보를 얻게 되는 경우.
- **Non-repeatable Read**: 한 트랜잭션 내에서 같은 조회를 두 번 했는데, 중간에 다른 트랜잭션이 값을 수정해버려 결과가 달라지는 현상.
- **Phantom Read**: 조회 결과 집합에 없던 데이터가 갑자기 나타나는 현상.

```java
// Spring Framework에서의 트랜잭션 처리
@Transactional
public void transferMoney(Long fromId, Long toId, int amount) {
    Account from = repository.findById(fromId);
    Account to = repository.findById(toId);
    
    from.withdraw(amount);
    to.deposit(amount);
    // 중간에 예외가 발생하면 withdraw 작업도 자동으로 취소됩니다.
}
```
**컴퓨터의 평**: "주인님은 로직에만 집중하세요. 정합성이 깨질 것 같으면 제가 즉시 시간을 되돌려(Rollback) 드릴 테니까요!"

---

### 4. 실무 최적화 인사이트
1.  **트랜잭션 범위 최소화**: 트랜잭션 안에서 외부 API를 호출하거나 무거운 연산을 하지 마세요. DB 커넥션 점유 시간이 길어지면 전체 시스템의 스루풋(Throughput)이 급감합니다.
2.  **낙관적 락(Optimistic Lock) vs 비관적 락(Pessimistic Lock)**: 충돌이 적을 것으로 예상되면 버전 정보로 체크하는 **낙관적 락**을, 충돌이 잦고 정합성이 극도로 중요하다면 `SELECT FOR UPDATE` 같은 **비관적 락**을 사용하세요.
3.  **데드락(Deadlock) 방지**: 두 트랜잭션이 서로의 자원을 기다리며 멈추는 현상입니다. 항상 동일한 순서로 테이블을 업데이트하는 규칙만 세워도 많은 데드락을 예방할 수 있습니다.

## 결론 및 요약 / 회고
트랜잭션은 데이터의 신뢰성을 지키는 최후의 보루입니다.
- **ACID**는 데이터가 어떤 상황에서도 오염되지 않음을 약속합니다.
- **격리 수준**을 이해하고 서비스 특성에 맞는 설정을 선택하세요.
- **비즈니스 로직**과 **트랜잭션 범위**를 명확히 분리하여 성능 저하를 막으세요.

수천 명의 사용자가 동시에 데이터를 건드리는 복잡한 환경에서도 데이터의 무결성을 유지하는 것, 그것이 백엔드 개발자의 진정한 가치라고 생각합니다.

## 참고 자료
- 데이터베이스 시스템 (Abraham Silberschatz 저)
- [MySQL Documentation - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- Real MySQL 8.0 (백은빈, 이성욱 저)
---
