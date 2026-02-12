---
title: "외부 API 연동의 정석: RestTemplate부터 WebClient까지 우아하게 통신하기"
series_order: 13
description: "다른 서비스와 데이터를 주고받는 다양한 방법과, 장애 전파를 막는 안정적인 연동 전략을 알아봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Java, Spring, RestTemplate, WebClient, 외부API, 백엔드, HTTP통신]
---

# 외부 API 연동의 정석: RestTemplate부터 WebClient까지 우아하게 통신하기

## 서론
현대의 백엔드 시스템은 결코 혼자서 살아갈 수 없습니다. 결제를 위해 토스페이먼츠를 호출하고, 알림을 보내기 위해 카카오 메시지 API를 찌르며, 날씨 정보를 가져오기 위해 공공데이터포털을 방문하죠. 

하지만 외부 서비스는 언제든 느려질 수 있고, 가끔은 점검이라는 명목으로 우리를 외면하기도 합니다. 이때 우리가 적절한 대비 없이 외부 API를 호출했다가는, 외부 서비스의 지연이 우리 시스템 전체의 장애로 번지는 '장애 전파' 문제가 발생할 수 있습니다.

![외부 API 장애 전파 방지 아키텍처](/images/02_Back-end/External_API_Integration/circuit_breaker_concept.png)
*외부 시스템의 응답 지연이 전체 서버의 스레드 고갈로 이어지지 않도록 타임아웃과 서킷 브레이커를 적용한 안정적인 연동 구조입니다.*

오늘은 스프링이 제공하는 두 가지 강력한 도구, **RestTemplate**과 **WebClient**를 통해 외부와 우아하고 안전하게 소통하는 법을 파헤쳐 보겠습니다.

## 본론

### 1. RestTemplate: 믿음직한 고참 통신원
오랫동안 스프링의 기본 HTTP 클라이언트로 군림해온 친구입니다. **동기(Blocking)** 방식으로 작동하며, 사용법이 매우 직관적입니다.

- **특징**: 요청을 보내면 응답이 올 때까지 스레드가 딱 멈춰서 기다립니다.
- **인사이트**: 단순하고 직관적이라 트래픽이 아주 많지 않은 환경에서는 여전히 매력적인 선택지입니다. 하지만 응답이 늦어지면 내 서버의 스레드들도 같이 줄줄이 대기 상태에 빠진다는 점을 주의해야 합니다.

```java
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject("https://api.weather.com/today", String.class);
```
> **컴퓨터의 평**: "주인님, 답장 올 때까지 여기서 가만히 기다릴게요. 다른 일은 못 해요!"

---

### 2. WebClient: 빠르고 유연한 신세대 통신원
스프링 5부터 도입된 **비동기(Non-blocking)** 방식의 클라이언트입니다.

- **특징**: 요청을 던져두고 응답이 오기 전까지 다른 일을 하러 갑니다. 응답이 오면 그때 리액티브하게 처리하죠.
- **인사이트**: 적은 수의 스레드로도 엄청난 양의 외부 API 호출을 처리할 수 있습니다. 앞으로 스프링의 표준은 WebClient로 넘어가고 있으니, 지금부터 익숙해지는 것이 좋습니다.

```java
WebClient webClient = WebClient.create();
Mono<String> result = webClient.get()
    .uri("https://api.weather.com/today")
    .retrieve()
    .bodyToMono(String.class);
```
> **컴퓨터의 평**: "주인님, 편지 부쳤어요! 답장 올 때까지 저는 다른 손님 주문받고 있을게요!"

---

### 3. 장애를 막는 방패: Timeout과 Retry
외부 API 연동에서 가장 중요한 것은 **"남 때문에 내가 죽지 않는 것"**입니다.

- **Connect Timeout / Read Timeout**: "답장 안 오면 3초 뒤에 포기해!" (무한 대기 방지)
- **Retry**: "잠시 네트워크가 불안정할 수 있으니 3번까지만 더 시도해봐." (일시적 오류 극복)

| 구분 | 전략 | 효과 |
| :--- | :--- | :--- |
| **Timeout** | 인내심의 한계 설정 | 시스템 자원 고갈 방지 |
| **Retry** | 재시도 정책 | 네트워크 순단 현상 극복 |
| **Circuit Breaker** | 선로 차단 | 외부 서비스 장애 시 아예 호출 차단 (서킷 브레이커) |

---

### 4. 실무력을 높이는 결정적 인사이트
실무에서 외부 API를 다룰 때 반드시 챙겨야 할 포인트입니다.

1.  **로깅은 생명입니다**: 외부 API는 우리가 제어할 수 없습니다. 나중에 문제가 생겼을 때 "나는 제대로 보냈는데 저쪽에서 에러가 났다"는 것을 증명하기 위해 요청과 응답 로그를 꼼꼼히 남겨야 합니다.
2.  **DTO로 철저히 캡슐화하세요**: 외부 API의 응답 구조가 언제든 바뀔 수 있습니다. 외부 응답 객체를 우리 서비스의 핵심 로직까지 끌고 들어오지 말고, 연동 레이어에서 우리만의 DTO로 즉시 변환하세요.
3.  **에러 핸들링을 세분화하세요**: `4xx`(우리 잘못)와 `5xx`(저쪽 잘못)에 대한 대처는 달라야 합니다. `4xx`는 우리가 데이터를 고쳐야 하고, `5xx`는 재시도를 하거나 사용자에게 점검 중임을 알려야 하죠.

## 결론 및 요약 / 회고
외부 API 연동은 기술적인 구현보다 **예외 상황에 대한 설계**가 훨씬 중요합니다.
- **RestTemplate**은 단순한 연동에, **WebClient**는 고성능/비동기 연동에 사용하세요.
- **Timeout**은 필수입니다. 절대 무한정 기다리지 마세요.
- **DTO 분리**와 **철저한 로깅**으로 변화에 강한 코드를 만드세요.

세상은 넓고 우리가 연동해야 할 API는 많습니다. 오늘 배운 도구들을 잘 활용해서, 남의 장애에 휘말리지 않는 강인한 백엔드 시스템을 설계하시길 바랍니다.

오늘도 여러분의 API 호출이 200 OK와 함께 빛의 속도로 돌아오길 응원합니다!

## 참고 자료
- Spring Framework Documentation: RestTemplate
- [Spring WebFlux: WebClient Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-client)
- [Baeldung - RestTemplate vs WebClient](https://www.baeldung.com/spring-webclient-resttemplate)
---
