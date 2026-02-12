---
title: "견고한 API 설계를 위한 예외 처리와 유효성 검사: @ControllerAdvice와 Validation 활용법"
series_order: 10
description: "사용자의 잘못된 입력과 예상치 못한 서버 에러를 우아하게 처리하여 서비스의 신뢰도를 높이는 방법을 알아봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Spring, SpringBoot, ExceptionHandling, Validation, ControllerAdvice, 백엔드, API설계]
---

# 견고한 API 설계를 위한 예외 처리와 유효성 검사: @ControllerAdvice와 Validation 활용법

## 서론
열심히 API를 개발해서 배포했는데, 사용자가 나이를 입력하는 칸에 "스물다섯 살"이라고 한글을 적어 보냈습니다. 서버는 당황해서 `Internal Server Error (500)`와 함께 정체 모를 영어 문장(Stack Trace)들을 사용자에게 쏟아냅니다. 사용자는 겁에 질려 앱을 삭제하고, 개발자는 밤새 로그를 뒤지며 한숨을 쉽니다.

![API 예외 처리 아키텍처](/images/02_Back-end/Exception_Handling_and_Validation/exception_handling_architecture.png)
*컨트롤러 계층에서 발생하는 다양한 예외를 가로채어 공통된 형식으로 변환하는 예외 처리 아키텍처입니다.*

좋은 API는 단순히 '성공'할 때 잘 작동하는 것만이 아닙니다. **'실패'할 때 얼마나 친절하고 우아하게 안내해 주느냐**가 그 서비스의 품격을 결정하죠. 오늘은 스프링이 제공하는 마법 같은 도구들을 이용해, 에러를 우리 편으로 만드는 법을 알아보겠습니다.

## 본론

### 1. Bean Validation: "입구 컷"의 정석
잘못된 데이터는 아예 비즈니스 로직 근처에도 오지 못하게 막아야 합니다. 자바에서는 `Hibernate Validator`라는 아주 훌륭한 도구를 통해 어노테이션 몇 개로 유효성 검사를 끝낼 수 있습니다.

- **핵심 어노테이션**: `@NotBlank`, `@Min`, `@Email`, `@Size` 등.
- **인사이트**: 컨트롤러 파라미터 앞에 `@Valid`만 붙여주세요. 스프링이 알아서 데이터의 형식을 검사하고, 문제가 있다면 즉시 거절합니다.

```java
public class MemberRequest {
    @NotBlank(message = "이름은 필수입니다.")
    private String name;

    @Min(value = 1, message = "나이는 1살 이상이어야 합니다.")
    private int age;
}
```

---

### 2. @ControllerAdvice: 전역 에러 컨트롤 타워
에러가 발생할 때마다 컨트롤러마다 `try-catch`를 도배하고 계신가요? 이는 유지보수의 지옥으로 가는 지름길입니다. 스프링은 **@ControllerAdvice**라는 중앙 통제실을 제공합니다.

- **역할**: 모든 컨트롤러에서 발생하는 예외를 한곳에서 잡아 처리합니다.
- **장점**: 비즈니스 로직에 `try-catch`가 사라져 코드가 깨끗해지고, 모든 에러 응답 형식을 통일할 수 있습니다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MemberNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(MemberNotFoundException e) {
        ErrorResponse response = new ErrorResponse("NOT_FOUND", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }
}
```
> **컴퓨터의 평**: "주인님, 이제 어디서든 에러만 던지세요(throw). 제가 여기서 다 받아서 예쁘게 포장해 보낼게요!"

---

### 3. 일관된 에러 응답 형식 (Error Response)
에러가 날 때마다 응답 구조가 달라지면 프론트엔드 개발자 동료가 매우 힘들어합니다. 에러 코드와 메시지를 담은 고정된 객체를 정의해서 사용합시다.

| 필드명 | 의미 | 예시 |
| :--- | :--- | :--- |
| **code** | 서비스 고유 에러 코드 | "COMMON_001" |
| **message** | 사용자에게 보여줄 메시지 | "잘못된 입력값입니다." |
| **errors** | 필드별 유효성 검사 실패 목록 | name: "이름은 필수입니다." |

---

### 4. 실무력을 높이는 결정적 인사이트
실무에서 더 견고한 시스템을 만들기 위해 다음 세 가지를 꼭 기억하세요.

1.  **비즈니스 예외(Custom Exception)를 정의하세요**: 자바가 제공하는 `RuntimeException`만 쓰기보다는 `OrderCanceledException`처럼 도메인의 의미를 담은 커스텀 예외를 만들어 사용하세요. 에러 추적이 훨씬 쉬워집니다.
2.  **스택 트래픽(Stack Trace)을 숨기세요**: 운영 환경에서 에러가 났을 때 서버 내부의 경로와 코드 라인 번호를 그대로 노출하는 것은 보안상 매우 위험합니다. 로그에는 남기되, 사용자에게는 정제된 메시지만 전달하세요.
3.  **로그를 남길 때는 '왜'를 담으세요**: 단순히 `e.getMessage()`만 남기기보다, 어떤 입력값 때문에 에러가 났는지 컨텍스트를 함께 남겨야 나중에 '삽질'을 줄일 수 있습니다.

## 결론 및 요약 / 회고
예외 처리는 단순히 버그를 막는 행위가 아니라, **사용자와 소통하는 방식**입니다.
- **Validation**으로 잘못된 데이터의 침입을 막으세요.
- **@ControllerAdvice**로 에러 처리 로직을 한데 모으세요.
- **일관된 응답**으로 팀 동료와 사용자의 신뢰를 얻으세요.

예외 처리를 잘 설계해두면 개발자는 밤에 발 뻗고 잘 수 있고, 사용자는 서비스에 믿음을 갖게 됩니다. 완벽한 코드만큼이나 완벽한 에러 처리에 공을 들이는 개발자가 됩시다!

오늘도 여러분의 서버가 예외 없이 평온하길, 설령 예외가 나더라도 우아하게 대처하길 응원합니다!

## 참고 자료
- Spring Framework Documentation: Exceptions
- [Baeldung - Error Handling for REST with Spring](https://www.baeldung.com/exception-handling-for-rest-with-spring)
- Hibernate Validator Official Guide
---
