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

![핵심 로직과 부가 기능이 깔끔하게 분리된 유연한 이벤트 기반 아키텍처 구조도](/images/02_Back-end/Spring_Transactional_Event_Listener/event_driven_decoupling.png)
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

> ### 아키텍트의 시선
> 트랜잭션과 이벤트를 결합할 때 가장 중요한 것은 '성공의 정의'를 어디까지로 볼 것인가입니다. 외부 시스템과의 연동이 필수적이라면, 단순한 리스너를 넘어 이벤트의 발행 자체를 보장하는 'Transactional Outbox' 패턴까지 고려해보는 것이 진정한 시니어의 자세입니다.

## 결론 및 요약 / 회고
