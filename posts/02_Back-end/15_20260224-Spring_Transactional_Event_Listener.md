---
title: "트랜잭션의 안전한 마침표: @TransactionalEventListener로 이벤트 기반 아키텍처 구축하기"
series_order: 15
description: "비즈니스 로직과 부가 기능을 분리하면서도 트랜잭션의 정합성을 완벽하게 지키는 방법을 알아봅니다."
date: 2026-02-24
update: 2026-02-24
tags: [Spring, SpringBoot, Transaction, Event, EventListener, TransactionalEventListener, 백엔드]
---

# 트랜잭션의 안전한 마침표: @TransactionalEventListener로 이벤트 기반 아키텍처 구축하기

## 서론
회원 가입이 성공하면 축하 이메일을 보내고, 가입 축하 쿠폰을 발급해야 하는 기능을 만든다고 가정해 봅시다. 가장 쉬운 방법은 `MemberService`의 `join` 메서드 안에 메일 발송 로직과 쿠폰 발급 로직을 모두 집어넣는 것입니다. 하지만 서비스가 커질수록 이 메서드는 '스파게티 코드'가 되어갈 것이고, 메일 서버가 느려지면 회원 가입 전체가 느려지는 불상사가 발생합니다.

![이벤트 기반 아키텍처를 통한 결합도 낮추기](/images/02_Back-end/Spring_Transactional_Event_Listener/event_driven_decoupling.png)
*핵심 비즈니스 로직과 부가 기능을 이벤트로 분리하여 서비스 간의 결합도를 낮추는 아키텍처 구조도입니다.*

이런 결합을 끊어내기 위해 우리는 '이벤트(Event)'를 사용합니다. 그런데 여기서 아주 무서운 함정이 하나 있습니다. 회원 가입 도중에 에러가 발생해서 DB는 **롤백(Rollback)**되었는데, 축하 이메일은 이미 발송되어 버린다면 어떻게 될까요? 사용자는 존재하지도 않는 계정의 가입 축하 메시지를 받는 황당한 경험을 하게 됩니다.

오늘은 트랜잭션의 성공 여부에 따라 이벤트를 정교하게 제어할 수 있게 해주는 **@TransactionalEventListener**의 마법을 파헤쳐 보겠습니다.

## 본론

### 1. 스프링 이벤트: 강한 결합의 사슬을 끊다
스프링은 `ApplicationEventPublisher`를 통해 아주 간편한 이벤트 시스템을 제공합니다. 

- **핵심 개념**: "나는 회원 가입이 끝났다는 사실만 알릴게(Publish), 그 뒤에 메일을 보내든 쿠폰을 주든 관심 없어!"
- **장점**: `MemberService`에서 메일 서비스나 쿠폰 서비스에 대한 의존성이 사라집니다. 코드가 간결해지고 각자의 역할이 명확해지죠.

---

### 2. @EventListener의 한계: 너무 성급한 이벤트 처리
기본적인 `@EventListener`는 이벤트를 발행하는 시점에 즉시 리스너가 실행됩니다. 즉, 발행자와 리스너가 **동일한 트랜잭션** 범위 안에서 묶여 있다는 뜻입니다.

- **문제 상황**: DB에 저장이 완료되기도 전에 외부 API(메일 발송 등)를 호출합니다. 만약 이후 DB 작업에서 예외가 발생해 롤백되더라도, 이미 날아간 메일은 주워 담을 수 없습니다.

---

### 3. @TransactionalEventListener: 트랜잭션의 눈치를 보는 영리한 리스너
이런 문제를 해결하기 위해 등장한 것이 바로 `@TransactionalEventListener`입니다. 이 리스너는 트랜잭션의 **상태(Phase)**를 지켜보고 있다가, 적절한 타이밍에 맞춰 행동합니다.

| Phase (단계) | 설명 | 비유 |
| :--- | :--- | :--- |
| **AFTER_COMMIT** | 트랜잭션이 성공적으로 커밋된 후 실행 (기본값) | "잔금 입금된 거 확인했으니 물건 보낼게요." |
| **AFTER_ROLLBACK** | 트랜잭션이 롤백된 경우에만 실행 | "주문 취소됐으니 준비하던 재료 치울게요." |
| **AFTER_COMPLETION** | 성공/실패 여부와 상관없이 트랜잭션 종료 후 실행 | "결과가 어떻든 작업 로그는 남겨야죠." |
| **BEFORE_COMMIT** | 트랜잭션이 커밋되기 직전에 실행 | "계약서 도장 찍기 전에 마지막으로 확인할게요." |

![TransactionalEventListener 동작 타이밍](/images/02_Back-end/Spring_Transactional_Event_Listener/transactional_event_listener_phases.png)
*트랜잭션의 커밋과 롤백 시점에 따라 이벤트 리스너가 실행되는 타이밍을 제어하는 개념도입니다.*

> **컴퓨터의 평**: "주인님, DB에 확실히 저장된 거 확인하고 메일 보낼게요. 괜히 롤백됐는데 축하 메일 보내서 민망해지는 일은 없을 거예요!"

---

### 4. 실무력을 높이는 결정적 인사이트
`@TransactionalEventListener`를 실무에 도입할 때 반드시 마주치게 되는 '삽질' 포인트 3가지입니다.

1.  **AFTER_COMMIT 단계에서는 새 트랜잭션이 필요합니다**: `AFTER_COMMIT` 시점에는 이미 기존 트랜잭션이 끝난 상태입니다. 따라서 리스너 안에서 다시 DB를 수정해야 한다면 `@Transactional(propagation = Propagation.REQUIRES_NEW)`을 사용하여 새로운 트랜잭션을 열어야 합니다. 그렇지 않으면 수정 사항이 DB에 반영되지 않습니다.
2.  **비동기(@Async)와 함께 사용하세요**: 아무리 커밋 후에 실행된다고 해도, 리스너가 무거운 작업을 수행하면 사용자의 응답 대기 시간이 길어집니다. `AFTER_COMMIT`과 `@Async`를 조합하면, 사용자는 빠른 응답을 받고 부가 기능은 백그라운드에서 안전하게 처리됩니다.
3.  **이벤트 저장소(Event Log)를 고려하세요**: 만약 트랜잭션은 성공했는데 이벤트 리스너 실행 중에 서버가 죽는다면 어떻게 될까요? 메일 발송이 누락될 수 있습니다. 정말 중요한 로직이라면 이벤트를 DB에 먼저 기록해두는 '아웃박스 패턴(Outbox Pattern)'을 고민해봐야 합니다.

## 결론 및 요약 / 회고
단순히 결합도를 낮추는 것을 넘어, 데이터의 정합성까지 고려하는 것이 진정한 이벤트 기반 설계의 시작입니다.
- **@EventListener**는 즉시 실행되지만 트랜잭션 안전성이 부족합니다.
- **@TransactionalEventListener**의 **AFTER_COMMIT**을 사용하여 DB와 외부 시스템 간의 정합성을 맞추세요.
- 리스너 내의 추가 DB 작업은 **REQUIRES_NEW**가 필요함을 잊지 마세요.

그동안 "왜 리스너에서 DB 수정이 안 되지?" 혹은 "롤백됐는데 왜 알림이 갔지?"라며 머리를 싸매셨던 분들에게 이 글이 작은 이정표가 되길 바랍니다.

오늘도 여러분의 시스템이 예외 상황에서도 흔들리지 않는 정합성을 유지하길 응원합니다!

## 참고 자료
- Spring Framework Documentation: Transactional Event Listeners
- [Baeldung - Spring Events](https://www.baeldung.com/spring-events)
- 실전! 스프링 부트와 JPA 활용 (김영한 저)
