---
title: "Spring Security와 OAuth2: 보안의 기초부터 소셜 로그인까지 완벽 가이드"
series_order: 11
description: "스프링 시큐리티의 핵심 동작 원리와 OAuth2를 이용한 소셜 로그인 구현 전략을 상세히 알아봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Spring, SpringBoot, SpringSecurity, OAuth2, 보안, JWT, 백엔드]
---

# Spring Security와 OAuth2: 보안의 기초부터 소셜 로그인까지 완벽 가이드

## 서론
서비스를 만들다 보면 가장 무서운 순간이 옵니다. 바로 "회원가입과 로그인"을 구현해야 하는 시점이죠. 잘못 만들었다가는 내 소중한 사용자들의 정보가 털리거나, 엉뚱한 사람이 남의 계정으로 로그인하는 대참사가 벌어질 수 있기 때문입니다. 

![보안이 뚫려 당황한 개발자 짤](/images/02_Back-end/spring_security/security_breach_horror.png)
*(여기에 '자물쇠가 다 부서진 문'이나 '해킹 시도에 당황한 캐릭터' 짤을 추천합니다!)*

스프링 시큐리티는 이런 우리를 위해 거대하고 튼튼한 성벽을 제공합니다. 하지만 그 성벽이 너무 거대해서 가끔은 개발자인 우리조차 성 안으로 못 들어가는 불상사가 생기기도 하죠. 오늘은 이 거대한 성벽의 구조와, 현대 웹의 필수품인 **OAuth2**를 이용한 소셜 로그인 마법을 파헤쳐 보겠습니다.

## 본론

### 1. Spring Security: 거대한 필터 체인의 성벽
스프링 시큐리티의 핵심은 **서블릿 필터(Filter)**입니다. 요청이 컨트롤러에 닿기도 전에 수십 개의 필터가 줄을 서서 "너 수상한 놈 아니지?"라고 검문하는 구조입니다.

- **인증(Authentication)**: "너 누구야?" (신분증 확인)
- **인가(Authorization)**: "너 여기 들어올 권한 있어?" (VIP 카드 확인)

**인사이트**: 시큐리티를 공부한다는 것은 이 수많은 필터 중 누가 어떤 검문을 담당하는지를 이해하는 과정입니다. 특히 `UsernamePasswordAuthenticationFilter`와 `SecurityContextHolder`의 관계를 아는 것이 핵심입니다.

---

### 2. OAuth2: "로그인은 구글한테 맡길게"
우리가 직접 비밀번호를 암호화해서 저장하고 관리하는 것은 위험 부담이 큽니다. 그래서 구글, 카카오 같은 대기업의 인증 시스템을 빌려 쓰는 것이 바로 **OAuth2**입니다.

- **Resource Owner**: 사용자 (나)
- **Client**: 우리 서비스
- **Authorization Server**: 구글/카카오 인증 서버
- **Resource Server**: 구글/카카오 데이터 서버

![OAuth2 흐름도](/images/02_Back-end/spring_security/oauth2_flow_diagram.png)
*(Client, User, Provider 간의 Access Token 발급 과정을 보여주는 기술 도식을 넣어주세요!)*

| 용어 | 의미 | 비유 |
| :--- | :--- | :--- |
| **Client ID/Secret** | 우리 서비스의 신분증 | "저 구글 서비스 쓰기로 한 그 앱입니다." |
| **Scope** | 가져올 정보의 범위 | "이메일이랑 프로필 사진만 볼게요." |
| **Access Token** | 데이터를 여는 열쇠 | "이거 보여주면 구글이 문 열어줄 거예요." |

---

### 3. JWT (JSON Web Token): 상태를 저장하지 않는 우아함
세션(Session) 방식은 서버 메모리에 로그인 정보를 담아두어야 해서 서버가 늘어날 때 관리가 힘듭니다. 반면 **JWT**는 로그인 정보를 암호화해서 사용자에게 줘버립니다. 

- **장점**: 서버가 로그인 상태를 기억할 필요가 없어 확장이 매우 쉽습니다(Stateless).
- **단점**: 한 번 발급된 토큰은 유효기간이 끝날 때까지 회수가 어렵습니다. 그래서 **Refresh Token**이라는 보조 열쇠가 필요하죠.

```java
// JWT 발급 예시 (개념적 코드)
String token = Jwts.builder()
    .setSubject(user.getEmail())
    .claim("role", "USER")
    .signWith(SignatureAlgorithm.HS512, secretKey)
    .compact();
```
> **컴퓨터의 평**: "주인님, 저는 이제 사용자가 누구인지 기억 안 할 거예요. 토큰만 가져오면 그때그때 확인해 드릴게요!"

---

### 4. 실무력을 높이는 결정적 인사이트
보안을 설계할 때 반드시 고려해야 할 실무 포인트입니다.

1.  **비밀번호는 절대 평문으로 저장하지 마세요**: `BCryptPasswordEncoder`는 선택이 아닌 필수입니다. 설령 DB가 털려도 비밀번호만은 지켜야 합니다.
2.  **CORS 설정을 잊지 마세요**: 프론트엔드와 백엔드의 도메인이 다르면 브라우저가 요청을 막습니다. 시큐리티 설정에서 `cors()`를 적절히 열어주어야 합니다.
3.  **인가 오류와 인증 오류를 구분하세요**: `401 Unauthorized`(너 누구야?)와 `403 Forbidden`(너 권한 없어!)을 명확히 구분해서 응답해야 프론트엔드에서 적절한 처리가 가능합니다.

## 결론 및 요약 / 회고
보안은 '한 번 설정하면 끝'인 기능이 아니라, 서비스가 끝날 때까지 관리해야 하는 생물과 같습니다.
- **Spring Security**는 필터 체인을 통해 성벽을 쌓습니다.
- **OAuth2**로 신뢰할 수 있는 대행사에게 인증을 맡기세요.
- **JWT**로 서버의 부담을 줄이고 확장성을 확보하세요.

처음 시큐리티 설정을 할 때는 수많은 클래스와 어노테이션에 압도당할 수 있습니다. 하지만 요청 하나가 필터를 하나씩 통과하는 과정을 머릿속에 그려본다면, 이 거대한 성벽이 우리 서비스를 얼마나 안전하게 지켜주고 있는지 감사하게 될 것입니다.

오늘도 여러분의 성벽이 무너지지 않고 견고하게 유지되길 응원합니다!

## 참고 자료
- Spring Security Reference Documentation
- [Baeldung - Spring Security OAuth2](https://www.baeldung.com/spring-security-oauth)
- OAuth 2.0 Simplified (Aaron Parecki 저)
---
*(여기에 '단단하게 잠긴 금고'나 '철통 보안을 자랑하는 요새' 짤을 넣어 마무리하면 좋습니다!)*
