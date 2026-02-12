---
title: "Spring AOP의 동작 원리: 관점 지향 프로그래밍과 프록시 메커니즘"
series_order: 6
description: "로깅, 트랜잭션, 보안 등 코드 곳곳에 흩어진 중복 로직을 한 방에 해결하는 AOP의 원리와 프록시의 비밀을 알아봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Spring, SpringBoot, AOP, Proxy, 백엔드, 프레임워크, 관점지향프로그래밍]
---

# Spring AOP의 동작 원리: 관점 지향 프로그래밍과 프록시 메커니즘

## 서론
프로젝트를 진행하다 보면 "아, 이 코드 또 쓰고 있네"라는 자괴감에 빠질 때가 있습니다. 모든 메서드 시작과 끝에 로그를 남기거나, 성능을 측정하기 위해 실행 시간을 계산하는 코드를 넣을 때가 대표적이죠. 비즈니스 로직은 단 3줄인데, 앞뒤로 붙는 부가적인 코드가 10줄이라면 코드의 본질을 파악하기 힘든 상황이 됩니다.

![횡단 관심사의 분리 (AOP)](/images/02_Back-end/Spring_AOP_Proxy/cross_cutting_concerns.png)
*핵심 비즈니스 로직과 로깅, 보안 등 공통 관심사를 분리하여 모듈화하는 AOP의 핵심 개념도입니다.*

이런 반복적인 작업을 멈추기 위해 등장한 것이 바로 **AOP(Aspect-Oriented Programming)**입니다. 오늘은 코드 곳곳에 흩어진 공통 관심사를 한데 모아 관리하는 이 우아한 기술과, 그 뒤에서 묵묵히 일하는 **프록시(Proxy)**의 정체를 밝혀보겠습니다.

## 본론

### 1. AOP: 횡단 관심사의 분리
AOP는 프로그램을 짤 때 **핵심 관심사(Business Logic)**와 **공통 관심사(Cross-cutting Concerns)**를 분리해서 생각하자는 패러다임입니다.

- **핵심 관심사**: 주문하기, 회원 가입하기, 상품 조회하기 등 서비스의 본질적인 기능.
- **공통 관심사**: 로깅, 보안, 트랜잭션 관리, 성능 측정 등 여러 기능에 공통적으로 들어가는 기능.

**인사이트**: 이전에는 각 메서드가 스스로 보안 검사도 하고 로깅도 해야 했다면, AOP를 적용하면 메서드는 오직 자신의 비즈니스 로직에만 집중하고 나머지 번거로운 일들은 외부의 전문 관리자(Aspect)에게 맡기게 됩니다.

---

### 2. 프록시(Proxy): 내 대신 일해주는 비서
스프링이 어떻게 기존 코드를 건드리지 않고 새로운 기능을 추가할 수 있을까요? 바로 **프록시 패턴** 덕분입니다. 스프링은 원본 객체(Target)를 직접 호출하는 대신, 그 객체를 감싸고 있는 '대역 배우'인 프록시를 호출하게 만듭니다.

- **원본 객체**: 실제로 요리를 하는 요리사.
- **프록시 객체**: 요리사 앞에 있는 비서. 손님의 주문을 기록하고(Logging), 요리 시간을 측정하며(Profiling), 요리가 끝나면 서빙까지 함.

![AOP 프록시 동작 원리 도식](/images/02_Back-end/Spring_AOP_Proxy/aop_proxy_mechanism_diagram.png)
*클라이언트의 요청을 프록시가 가로채어 공통 로직(Advice)을 실행한 뒤 실제 객체(Target)를 호출하는 프록시 메커니즘입니다.*

---

### 3. AOP를 이해하는 3가지 핵심 용어
AOP는 용어가 조금 생소합니다. 하지만 다음 세 가지만 기억하면 끝입니다.

| 용어 | 의미 | 비유 |
| :--- | :--- | :--- |
| **Aspect** | 공통 로직을 모듈화한 것 | 보안팀, 로깅팀 등 전문 부서 |
| **Advice** | 실질적으로 수행할 기능 (언제 할 것인가?) | "입장하기 전에 신분증 검사해!" |
| **Pointcut** | 어디에 적용할 것인가? | "클럽 정문(특정 메서드)에서만 해!" |

---

### 4. 실무력을 높이는 결정적 인사이트
AOP를 잘 쓰면 코드가 예술적으로 변하지만, 남용하면 디버깅 지옥을 맛볼 수 있습니다.

1.  **트랜잭션(`@Transactional`)의 원리를 이해하세요**: 우리가 흔히 쓰는 이 어노테이션도 사실 스프링 AOP의 산물입니다. 프록시가 메서드 호출 전후로 `commit`과 `rollback`을 자동으로 처리해 주는 것이죠.
2.  **내부 호출(Self-Invocation) 문제를 주의하세요**: 프록시는 외부에서 호출될 때만 작동합니다. 클래스 내부에서 `this.method()` 식으로 호출하면 프록시를 거치지 않기 때문에 AOP가 적용되지 않는 대참사가 벌어집니다. **비서를 거치지 않고 사장님을 직접 만나면 기록이 안 남는 것**과 같습니다.

```java
// AOP 적용 예시: 실행 시간 측정
@Aspect
@Component
public class PerformanceAspect {
    @Around("execution(* com.example.service.*.*(..))") // 포인트컷: 서비스 패키지의 모든 메서드
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        Object result = joinPoint.proceed(); // 실제 메서드 실행 (Target 호출)
        
        long end = System.currentTimeMillis();
        System.out.println("실행 시간: " + (end - start) + "ms");
        return result;
    }
}
```
> **컴퓨터의 평**: "주인님, 이제 로직만 짜세요. 번거로운 로깅이나 시간 체크는 제가 뒤에서 몰래(Proxy) 다 해둘게요!"

## 결론 및 요약 / 회고
AOP는 단순히 중복 코드를 제거하는 기술을 넘어, 객체지향의 원칙인 **단일 책임 원칙(SRP)**을 완성하게 해주는 도구입니다.
- **핵심 로직**과 **공통 로직**을 분리하세요.
- 그 마법의 중심에는 **프록시**라는 비서가 있습니다.
- 하지만 내부 호출 문제처럼 프록시의 한계도 명확히 이해하고 있어야 합니다.

이제 여러분의 코드가 핵심에만 집중하여 더욱 빛나길 응원합니다. 다음 시간에는 데이터베이스를 객체처럼 다루는 JPA의 마법에 대해 알아보겠습니다!

## 참고 자료
- Spring Framework Documentation: Aspect Oriented Programming with Spring
- 토비의 스프링 3.1 (이일민 저)
- [Baeldung - Introduction to Spring AOP](https://www.baeldung.com/spring-aop)
---
