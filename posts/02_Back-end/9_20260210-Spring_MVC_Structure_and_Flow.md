---
title: "Spring MVC 구조와 Flow: 요청부터 응답까지의 내부 여정 완벽 가이드"
series_order: 9
description: "스프링 웹 애플리케이션의 심장부, Spring MVC의 동작 원리와 DispatcherServlet의 역할을 상세히 파헤쳐 봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Spring, SpringBoot, SpringMVC, DispatcherServlet, 백엔드, 웹아키텍처]
---

# Spring MVC 구조와 Flow: 요청부터 응답까지의 내부 여정 완벽 가이드

## 서론
웹 브라우저 주소창에 URL을 입력하고 '엔터'를 치는 순간, 우리 스프링 서버 내부에서는 거대한 축제가 벌어집니다. 수많은 객체가 일사불란하게 움직이며 사용자가 원하는 데이터를 찾아내고, 예쁘게 포장해서 다시 브라우저로 보내주죠.

우리는 보통 `@Controller` 하나 만들고 `@GetMapping` 붙이면 끝이라고 생각하지만, 그 이면에서는 스프링 MVC라는 정교한 기계 장치가 돌아가고 있습니다. 

![Spring MVC의 복잡한 흐름 짤](/images/02_Back-end/spring_mvc/spring_mvc_complexity.png)
*(여기에 '정교하게 맞물린 톱니바퀴'나 '거대한 컨트롤 타워' 짤을 추천합니다!)*

오늘은 우리가 무심코 사용하던 스프링 MVC가 내부적으로 어떻게 구성되어 있는지, 특히 그 중심에 있는 **DispatcherServlet**이라는 '전설의 지배자'가 어떤 일을 하는지 아주 쉽게 파헤쳐 보겠습니다.

## 본론

### 1. DispatcherServlet: 모든 길은 이곳으로 통한다
과거에는 모든 서블릿이 각자의 경로를 직접 관리해야 했습니다. 하지만 스프링 MVC는 **프론트 컨트롤러(Front Controller) 패턴**을 도입했습니다. 호텔의 '컨시어지 데스크'나 식당의 '지배인'처럼, 모든 요청을 일단 한곳에서 받아서 적절한 사람에게 나눠주는 방식이죠.

그 지배인의 이름이 바로 **DispatcherServlet**입니다.

- **핵심 역할**: 중앙 집중식 요청 처리.
- **인사이트**: 모든 HTTP 요청의 입구와 출구를 단 한 곳으로 통일함으로써 공통 로직(보안, 로깅, 인코딩 등)을 한꺼번에 처리할 수 있는 우아함을 얻었습니다.

---

### 2. 스프링 MVC의 7단계 릴레이 레이스
요청이 들어와서 응답이 나갈 때까지, 데이터는 다음의 과정을 거칩니다.

![Spring MVC 동작 흐름도](/images/02_Back-end/spring_mvc/spring_mvc_flow_diagram.png)
*(Client -> DispatcherServlet -> HandlerMapping -> Adapter -> Controller -> ViewResolver -> View로 이어지는 기술 도식을 넣어주세요!)*

1.  **클라이언트 요청**: 사용자가 브라우저에서 요청을 보냅니다.
2.  **DispatcherServlet 접수**: "어서 오세요, 제가 처리해 드릴게요."
3.  **Handler Mapping**: "이 요청은 어느 컨트롤러가 잘 처리할까?" (지도를 보고 담당자 찾기)
4.  **Handler Adapter**: "담당 컨트롤러야, 이 형식에 맞춰서 일해줘!" (서로 다른 컨트롤러 규격을 맞춰주는 어댑터)
5.  **Controller 실행**: 우리가 짠 비즈니스 로직이 수행됩니다. 결과값(Model)과 보낼 곳(View)을 반환합니다.
6.  **View Resolver**: "응답으로 보낼 HTML 파일이 어디 있더라?" (논리적인 이름을 실제 파일 경로로 변환)
7.  **View 렌더링 및 응답**: 최종 결과물을 브라우저로 쏴줍니다.

---

### 3. 한눈에 보는 MVC 핵심 컴포넌트

| 컴포넌트 | 비유 | 주요 역할 |
| :--- | :--- | :--- |
| **DispatcherServlet** | 총괄 지배인 | 전체 흐름을 제어하는 프론트 컨트롤러 |
| **HandlerMapping** | 안내 데스크의 지도 | URL과 컨트롤러를 매핑해주는 정보원 |
| **HandlerAdapter** | 통역사/변환기 | 다양한 형태의 핸들러를 실행할 수 있게 해주는 도구 |
| **ViewResolver** | 내비게이션 | 뷰 이름을 실제 뷰(JSP, Thymeleaf 등)로 연결 |

```java
// 우리가 작성하는 컨트롤러는 전체 과정의 아주 일부분일 뿐입니다!
@Controller
public class MemberController {
    @GetMapping("/members")
    public String list(Model model) {
        model.addAttribute("members", memberService.findAll());
        return "members/memberList"; // ViewResolver가 이를 찾아냅니다.
    }
}
```
> **컴퓨터의 평**: "주인님은 비즈니스 로직만 생각하세요. 경로 찾고, 변환하고, 화면 띄우는 귀찮은 일은 제가 뒤에서 팀원(컴포넌트)들과 다 해둘게요!"

---

### 4. 실무력을 높이는 결정적 인사이트
스프링 MVC 구조를 이해했다면 실무에서 다음 두 가지를 더 깊게 고민해볼 수 있습니다.

1.  **핸들러 어댑터의 확장성**: 왜 굳이 어댑터를 둘까요? 직접 호출하면 안 될까요? 그 이유는 스프링이 확장성을 지향하기 때문입니다. `@Controller`뿐만 아니라 과거의 `HttpRequestHandler`나 인터페이스 방식의 컨트롤러 등 어떤 형태의 핸들러라도 어댑터만 있으면 유연하게 수용할 수 있습니다.
2.  **REST API와 @RestController**: 현대의 백엔드는 HTML을 반환하기보다 데이터를 반환(JSON)하는 경우가 많습니다. 이때는 **ViewResolver**를 거치지 않고, **HttpMessageConverter**가 직접 데이터를 응답 바디에 써줍니다. 우리가 흔히 쓰는 `@ResponseBody`의 원리가 바로 이것입니다.

## 결론 및 요약 / 회고
Spring MVC는 '분할과 정복'의 정수를 보여줍니다. 각자의 역할이 명확하게 나누어져 있어 유지보수가 쉽고 확장이 용이하죠.
- **DispatcherServlet**은 모든 요청의 중심입니다.
- **HandlerMapping**과 **Adapter**가 담당자를 찾아 실행합니다.
- **ViewResolver**나 **MessageConverter**가 최종 응답을 예쁘게 포장합니다.

그동안 단순히 컨트롤러 메서드만 짜왔다면, 이제는 내 요청이 어떤 컴포넌트들을 거쳐 여행하고 있는지 상상해 보세요. 스프링이라는 거대한 기계가 얼마나 우아하게 돌아가고 있는지 느끼게 될 것입니다.

오늘도 여러분의 요청이 404 에러 없이 목적지에 무사히 도착하길 응원합니다!

## 참고 자료
- Spring Framework Documentation: Web Servlet Config
- 토비의 스프링 3.1 (이일민 저)
- [Baeldung - Spring MVC Handler Adapters](https://www.baeldung.com/spring-mvc-handler-adapters)
---
*(여기에 '정교한 지휘를 하는 마에스트로'나 '일사불란한 공장의 공정' 짤을 넣어 마무리하면 좋습니다!)*
