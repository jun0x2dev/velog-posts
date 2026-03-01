---
title: "Spring WebMVC vs WebFlux: 스레드 모델부터 실전 마이그레이션까지"
series_order: 20
description: "Blocking I/O의 구조적 한계와 Reactive 모델의 동작 원리를 스레드 수준에서 파헤치고, 실제 파일 서버 마이그레이션 결과로 검증합니다."
date: 2026-03-01
update: 2026-03-01
tags: [Spring, SpringBoot, WebMVC, WebFlux, Reactor, Netty, Tomcat, 백엔드, 성능최적화]
---

# Spring WebMVC vs WebFlux: 스레드 모델부터 실전 마이그레이션까지

## 서론

로컬 디스크에서 이미지 파일을 읽어 반환하는 단순한 파일 서버가 있었습니다. 특별한 비즈니스 로직도 없고, DB 조회도 없고, 그저 파일을 읽어서 내려주는 I/O 작업이 전부인 서버였죠.

JMeter로 부하를 걸어보니 최대 응답시간이 160ms였습니다. 평균은 나쁘지 않았지만, 요청이 겹치는 순간 응답 시간이 불규칙하게 튀었습니다. 표준 편차도 2.93이나 됐죠. 이 서버는 클라이언트가 다수의 파일을 동시에 요청하는 구조였는데, 파일 하나라도 응답이 늦어지면 클라이언트 측 렌더링이 지연되는 문제가 있었습니다.

원인은 두 가지가 동시에 겹친 케이스였습니다. **Tomcat 스레드 풀 포화로 인한 대기**와 **FileInputStream의 1024바이트 반복 블로킹 읽기로 인한 시스템 콜 누적**이었습니다.

WebFlux로 전환 후 최대 응답시간은 17ms로 줄었고, 표준 편차는 0.73으로 안정됐습니다. 이번 글에서는 이 결과가 나온 이유를 스레드 모델과 I/O 동작 원리 수준에서 파헤쳐 보겠습니다.

## 본론

### 1. I/O 모델의 근본 차이: 스레드는 왜 기다리나

서버가 클라이언트 요청을 받아 파일을 읽는다고 해봅시다. 이때 실제로 일어나는 일은 생각보다 단순합니다. OS에 **read() 시스템 콜**을 날리고, 디스크 컨트롤러가 데이터를 메모리로 복사해 오면 그제서야 반환받는 거죠.

문제는 이 과정이 끝날 때까지 스레드가 아무것도 못 하고 멈춰 있다는 겁니다. 스레드는 CPU를 점유하지 않지만, 그렇다고 다른 일을 하지도 못합니다. 그저 "데이터 다 됐어요?" 하며 기다리는 상태, 이게 **Blocking I/O**입니다.

FileInputStream으로 파일을 읽으면 기본적으로 1024바이트씩 반복해서 read()를 호출합니다. 1MB 파일 하나를 읽으려면 약 1,000번의 블로킹 시스템 콜이 발생하는 셈입니다. 요청이 동시에 몰리면 이 시스템 콜들이 쌓이면서 응답 지연이 시작됩니다.

**Non-blocking I/O**는 다릅니다. 리눅스의 `epoll`, macOS의 `kqueue` 같은 이벤트 알림 메커니즘을 활용합니다. "파일 읽어줘"라고 OS에 요청만 해두고, 데이터가 준비되면 알림을 받는 방식입니다. 스레드는 그 사이에 다른 요청을 처리할 수 있죠.

---

### 2. 스레드 모델 비교: Tomcat vs Netty

Spring MVC의 기본 서버는 Tomcat, Spring WebFlux의 기본 서버는 Netty입니다. 둘의 스레드 모델은 근본적으로 다릅니다.

![Tomcat One-Thread-Per-Request와 Netty Event Loop 구조 비교](/images/02_Back-end/Spring_WebMVC_vs_WebFlux/thread_model_comparison.png)
*Tomcat은 요청마다 스레드를 할당하고, Netty는 소수의 이벤트 루프 스레드가 모든 요청을 처리합니다.*

**Tomcat: One-Request-One-Thread**

요청이 들어오면 스레드 풀에서 스레드 하나를 꺼내 그 요청의 전 생애주기를 담당하게 합니다. DB 조회를 기다리든, 파일을 읽든, 그 스레드는 응답이 나갈 때까지 묶여 있습니다. 스레드 풀이 200개라면 201번째 요청이 들어오는 순간부터 큐에서 대기해야 합니다. 이게 스레드 풀 포화입니다.

**Netty: Event Loop**

CPU 코어 수에 비례한 소수의 이벤트 루프 스레드가 모든 요청을 처리합니다. 4코어 서버라면 8개 내외의 스레드가 수천 개의 연결을 감당합니다. 이 스레드들은 I/O 대기 중에 절대 블로킹되지 않으며, 이벤트(파일 읽기 완료, 네트워크 응답 수신 등)가 발생할 때마다 콜백을 실행하는 방식으로 동작합니다.

| 항목 | Tomcat (WebMVC) | Netty (WebFlux) |
| :--- | :--- | :--- |
| **스레드 수** | 수백 개 (기본 200) | CPU 코어 × 2 내외 |
| **메모리 사용** | 스레드당 ~1MB 스택 | 적은 스레드, 낮은 메모리 |
| **I/O 대기 시 동작** | 스레드 블로킹 (대기) | 이벤트 루프로 다른 작업 처리 |
| **컨텍스트 스위칭 비용** | 스레드 수 비례로 증가 | 최소화 |
| **적합한 작업** | CPU-bound, 복잡한 비즈니스 로직 | I/O-bound, 고동시성 |

---

### 3. Spring MVC 내부: 동기 처리의 구조

Spring MVC는 서블릿 API 위에 올라가 있습니다. `DispatcherServlet`이 요청을 받아 각 컴포넌트로 분배하는 구조인데, 이 흐름 전체가 **하나의 스레드 안에서 순차적으로 실행**됩니다.

이 구조 덕분에 스프링의 여러 기능이 자연스럽게 동작합니다. 트랜잭션 컨텍스트, 시큐리티 컨텍스트, 로깅 MDC가 모두 **ThreadLocal**에 저장되기 때문입니다. 한 요청의 전 과정이 같은 스레드 위에 있으니 이 값들을 어디서든 꺼내 쓸 수 있죠.

하지만 이게 동시에 약점이기도 합니다. 외부 API 호출이나 파일 I/O처럼 대기 시간이 긴 작업이 있으면, 그 스레드는 응답이 올 때까지 아무것도 못 하고 자리를 차지합니다. 스레드가 비싼 자원임을 감안하면, 기다리는 데 쓰이는 스레드는 낭비입니다.

---

### 4. Spring WebFlux 내부: Reactive Streams 모델

WebFlux는 **Reactive Streams** 스펙을 기반으로 합니다. 핵심은 Publisher와 Subscriber의 계약입니다.

- **Publisher**: 데이터를 생산하는 쪽입니다.
- **Subscriber**: 데이터를 소비하는 쪽입니다.
- **Backpressure**: 소비자가 "나 지금 10개밖에 못 받아"라고 생산자에게 신호를 보낼 수 있습니다. 소비자가 감당할 수 있는 만큼만 데이터를 흘려보내는 흐름 제어 메커니즘입니다.

Spring WebFlux는 Project Reactor 라이브러리를 사용하며, 두 가지 타입을 제공합니다.

- **Mono\<T\>**: 0개 또는 1개의 값을 비동기로 반환합니다. 단건 조회에 적합합니다.
- **Flux\<T\>**: 0개 이상 N개의 값을 비동기 스트림으로 반환합니다. 목록 조회, SSE에 적합합니다.

![Reactive Streams의 Publisher-Subscriber-Backpressure 동작 모델](/images/02_Back-end/Spring_WebMVC_vs_WebFlux/reactive_streams_model.png)
*Subscriber가 request(n)으로 처리 가능한 데이터 수를 제어하며, Publisher는 그만큼만 데이터를 내려보냅니다.*


WebFlux에서는 ThreadLocal이 동작하지 않습니다. 요청 하나가 여러 스레드에 걸쳐 실행될 수 있기 때문입니다. 대신 **Reactor Context**를 사용해 컨텍스트 정보를 스트림에 싣고 다닙니다. Spring Security의 WebFlux 통합이 이 방식으로 구현되어 있습니다.

---

### 5. 코드 비교: 파일 반환 API

실제 문제였던 JPG 파일 반환 API를 두 예제 방식으로 구현해보겠습니다.

**WebMVC 방식 (Blocking)**

```java
@GetMapping("/tiles/{id}.jpg")
public ResponseEntity<byte[]> getTile(@PathVariable String id) throws IOException {
    File file = new File("/tiles/" + id + ".jpg");

    // FileInputStream은 기본적으로 1024바이트씩 반복 블로킹 읽기
    // 1MB 파일이면 약 1,000번의 시스템 콜이 발생합니다
    byte[] data = Files.readAllBytes(file.toPath());

    // 힙에 파일 전체를 올린 뒤 응답 바디에 복사 → 이중 복사로 GC 압박
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.IMAGE_JPEG);
    return ResponseEntity.ok().headers(headers).body(data);
}
```
> **컴퓨터의 평**: "파일 다 읽을 때까지 저 여기서 기다릴게요. 다른 요청이 밀려도 제 차례가 될 때까지 줄 서세요."

**WebFlux 방식 (Non-blocking)**

```java
@GetMapping("/tiles/{id}.jpg")
public Mono<ResponseEntity<Resource>> getTile(@PathVariable String id) {
    return Mono.fromSupplier(() -> {
        Resource resource = new FileSystemResource("/tiles/" + id + ".jpg");
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_JPEG)
                .body(resource);
    }).subscribeOn(Schedulers.boundedElastic()); // 블로킹 I/O는 전용 스케줄러에 위임
}
```
> **컴퓨터의 평**: "파일 읽기는 전담 팀(boundedElastic)에 맡겼습니다. 저는 그 사이에 다른 요청 처리할게요. 다 되면 알려줄 테니까요."

여기서 한 가지 중요한 포인트가 있습니다. **WebFlux를 쓴다고 자동으로 모든 I/O가 논블로킹이 되지는 않습니다.** `FileSystemResource`를 통한 파일 읽기 자체는 여전히 블로킹 작업이기 때문에 `Schedulers.boundedElastic()`처럼 블로킹 작업 전용 스케줄러에 위임해야 합니다. 이벤트 루프 스레드를 블로킹 작업으로 막아버리면 WebFlux의 장점이 사라집니다.

`Resource` 기반 응답은 Netty 내부에서 `FileRegion` 또는 `ChunkedFile`을 활용해 커널 레벨의 최적화된 파일 전송을 수행합니다. 기존의 `ByteArrayOutputStream`으로 파일을 힙에 통째로 올리던 방식과 달리 불필요한 메모리 복사가 없어 GC 압박도 함께 줄어듭니다.

---

### 6. 실전 마이그레이션 결과 분석

JMeter 테스트 구성은 동시 사용자 20명, Ramp-up 10초, Loop Count 1,000회로 총 20,000건을 요청했습니다. Ramp-up을 10초로 설정한 이유는 JVM 워밍업 영향을 배제하고 서버가 안정 상태에 도달한 이후의 수치를 보기 위해서입니다.

| 지표 | WebMVC | WebFlux | 개선율 |
| :--- | ---: | ---: | ---: |
| **평균 응답시간** | 18ms | 14ms | 22% 개선 |
| **최대 응답시간** | 160ms | 17ms | **89% 개선** |
| **표준 편차** | 2.93 | 0.73 | **75% 개선** |
| **처리량 (TPS)** | 기준치 | +24% | 24% 개선 |

**왜 최대 응답시간이 89%나 줄었나**

두 병목이 동시에 제거됐기 때문입니다. 첫째, Tomcat 스레드 풀 포화로 인한 대기가 사라졌습니다. Netty의 이벤트 루프는 스레드 수와 무관하게 연결을 수락하고, I/O 이벤트 발생 시 처리합니다. 스레드가 부족해서 큐에서 기다리는 상황이 원천적으로 없어집니다. 둘째, `FileInputStream`의 1024바이트 반복 읽기가 `Resource` 기반 전송으로 교체되면서 불필요한 시스템 콜이 줄었습니다.

**왜 표준 편차가 75% 줄었나**

기존 방식은 `ByteArrayOutputStream`으로 파일 전체를 힙에 올렸습니다. 파일 크기가 커질수록 힙 사용량이 불규칙하게 증가했고, GC가 발동하는 타이밍에 따라 응답 시간이 튀었습니다. 메모리 복사 단계를 제거하자 GC 압박이 완화됐고, 이벤트 루프의 균일한 처리 패턴이 더해지면서 응답 시간 분포가 안정됐습니다.

**왜 처리량은 24%밖에 안 늘었나**

동시 사용자 20명은 Tomcat 스레드 풀 기본값 200개 대비 충분히 여유가 있는 수치입니다. 스레드 풀이 포화 상태가 아니었으니 처리량 차이가 작게 나오는 건 당연한 결과입니다. 동시 사용자를 수백 명 이상으로 늘리면 WebMVC는 스레드 풀 포화로 처리량이 급락하는 반면, WebFlux는 이벤트 루프 특성상 처리량을 유지합니다.

이 테스트에서 핵심 지표는 처리량이 아니라 **최대 응답시간과 표준 편차**였습니다. 평균이 낮아도 일부 요청에서 예측 불가능한 지연이 발생하면, 다수의 파일을 동시 요청하는 클라이언트 입장에서는 하나라도 늦어지는 순간 전체 응답이 블로킹됩니다. 이런 구조적 지연은 서버 증설로는 근본적으로 해결되지 않습니다.

---

### 7. 트레이드오프: 언제 무엇을 선택해야 하나

WebFlux가 무조건 좋다는 결론은 틀렸습니다. 상황에 맞는 도구가 따로 있습니다.

**WebMVC가 유리한 상황**

- **JDBC를 써야 할 때**: 표준 JDBC 드라이버는 블로킹입니다. R2DBC(Reactive Relational Database Connectivity)가 있지만 생태계가 좁고 JPA를 바로 쓸 수 없습니다.
- **복잡한 트랜잭션 로직이 있을 때**: ThreadLocal 기반의 트랜잭션 전파는 WebMVC에서 직관적으로 동작합니다. WebFlux에서는 리액티브 트랜잭션 관리를 별도로 해야 합니다.
- **팀의 리액티브 경험이 없을 때**: Mono/Flux 연산자, 에러 처리 방식, 디버깅 방법은 학습 곡선이 상당합니다. 잘못 쓰면 이벤트 루프 스레드를 블로킹시켜 성능이 오히려 더 나빠질 수 있습니다.

**WebFlux가 유리한 상황**

- **I/O-bound 작업이 대부분일 때**: 파일 서빙, 외부 API 프록시처럼 대기 시간이 지배적인 서비스에서 가장 효과적입니다.
- **고동시성이 필요할 때**: 수천 개 이상의 동시 연결을 적은 리소스로 처리해야 할 때 Netty의 이벤트 루프가 빛납니다.
- **SSE나 WebSocket이 필요할 때**: 실시간 스트리밍에 Flux가 자연스럽게 맞아 떨어집니다.

| 선택 기준 | WebMVC | WebFlux |
| :--- | :---: | :---: |
| JDBC / JPA 사용 | 권장 | 비권장 |
| 복잡한 트랜잭션 로직 | 권장 | 가능하나 복잡 |
| I/O-bound 고동시성 | 한계 존재 | 권장 |
| SSE / WebSocket | 제한적 | 권장 |
| 팀 리액티브 경험 없음 | 권장 | 주의 필요 |
| 레거시 코드베이스 통합 | 권장 | 비권장 |

---

### 8. Java 21 Virtual Thread와 WebMVC의 반격

Java 21에서 정식 출시된 **Virtual Thread(가상 스레드)**는 이 구도를 흔들고 있습니다.

가상 스레드는 JVM이 관리하는 경량 스레드입니다. OS 스레드와 1:1 매핑되는 기존 플랫폼 스레드와 달리, 수백만 개를 만들어도 실제 OS 스레드는 소수만 사용합니다. JVM이 블로킹 I/O 발생 시 자동으로 OS 스레드 점유를 해제하고 다른 가상 스레드를 실행시킵니다.

즉, 기존 WebMVC 코드를 그대로 두고 설정 한 줄만 바꾸면 됩니다.

```yaml
# Spring Boot 3.2+ application.yml
spring:
  threads:
    virtual:
      enabled: true  # Tomcat이 가상 스레드를 사용합니다
```

블로킹 코드를 리팩토링하지 않고도 WebFlux와 유사한 고동시성 처리가 가능해집니다. 그렇다면 WebFlux가 사라질까요?

아직은 아닙니다. 가상 스레드는 **스레드 블로킹 비용 문제**를 해결하지만, WebFlux만의 영역은 따로 있습니다.

- **Backpressure**: 소비자가 처리 가능한 속도로 데이터 흐름을 제어하는 메커니즘은 Reactive Streams만의 영역입니다. 가상 스레드는 이를 제공하지 않습니다.
- **선언적 스트림 합성**: `flatMap`, `zip`, `merge` 같은 연산자로 복잡한 비동기 흐름을 조합하는 것은 WebFlux의 강점입니다.
- **메모리 효율**: 가상 스레드도 결국 스레드 객체이므로, Netty 이벤트 루프보다는 리소스를 더 씁니다.

Virtual Thread는 "기존 WebMVC 코드를 리팩토링 없이 개선"하는 도구이고, WebFlux는 "처음부터 리액티브 패러다임으로 설계"하는 도구입니다. 용도가 일부 겹치지만 대체 관계는 아닙니다.

## 결론 및 요약

이번 글에서 다룬 핵심을 정리합니다.

- **Blocking I/O**: 스레드가 I/O 완료까지 대기. 스레드 풀 포화 시 전체 처리가 지연됩니다.
- **Non-blocking I/O**: 이벤트 알림 기반. 소수의 스레드로 고동시성을 처리합니다.
- **Tomcat**: One-Request-One-Thread. 동시 요청 수가 스레드 수를 초과하면 큐 대기입니다.
- **Netty**: Event Loop. 소수의 스레드가 수천 연결을 이벤트 기반으로 처리합니다.
- **WebFlux 주의점**: 이벤트 루프 스레드를 블로킹 작업으로 막으면 역효과입니다. 파일 읽기 같은 블로킹 I/O는 반드시 `Schedulers.boundedElastic()`에 위임해야 합니다.

선택 기준은 명확합니다. **비즈니스 로직이 복잡하고 JDBC를 쓴다면 WebMVC**, **I/O-bound에 고동시성이 필요하다면 WebFlux**, **Java 21 이상이고 기존 코드 유지가 중요하다면 Virtual Thread + WebMVC**를 검토하세요.

"WebFlux가 좋다더라"는 말만 듣고 무작정 전환하면, 리액티브 디버깅의 벽에 부딪혀 오히려 개발 속도가 느려집니다. 병목이 어디 있는지 먼저 측정하고, 그에 맞는 도구를 고르는 것이 가장 확실한 방법입니다.

오늘도 여러분의 서버가 스레드 풀 포화 없이 균일하게 달리길 응원합니다!

## 번외: 이 글을 쓴 이유

사실 위에서 소개한 마이그레이션은 직접 진행했던 작업입니다. 결과 수치는 만족스러웠지만, 돌이켜보면 WebFlux를 완전히 이해한 상태에서 진행한 건 아니었습니다.

`Schedulers.boundedElastic()`을 쓴 것도, `Resource` 기반 응답으로 바꾼 것도 당시에는 "이렇게 하면 된다"는 수준으로 접근했습니다. 왜 이벤트 루프 스레드를 블로킹시키면 안 되는지, Reactive Streams의 Backpressure가 실제로 어떤 문제를 해결하는지, 그 원리를 제대로 설명할 수 있는 상태는 아니었습니다.

그게 좀 찜찜하게 남아서 이번 기회에 처음부터 다시 정리했습니다. 직접 써보면서 "아, 그래서 그랬구나" 싶은 부분들이 꽤 있었고, 글로 정리하고 나니 이제는 누가 물어봐도 스레드 모델부터 차분하게 설명할 수 있을 것 같습니다.

도구를 잘 쓰는 것과 도구를 이해하는 것은 다릅니다. 결과가 좋았더라도 원리를 모른 채 넘어가면 다음에 다른 상황을 만났을 때 같은 판단을 내리기 어렵습니다. 이 글이 저처럼 WebFlux를 써봤는데 뭔가 찜찜함이 남는 분들에게 조금이나마 도움이 됐으면 합니다.
