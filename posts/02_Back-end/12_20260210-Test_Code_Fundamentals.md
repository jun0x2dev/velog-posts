---
title: "테스트 코드의 정석: JUnit5와 Mockito를 활용한 단위 및 통합 테스트 전략"
series_order: 12
description: ""내 컴에선 되는데?"라는 공포에서 벗어나, 코드의 안정성을 보장하는 테스트 코드 작성법을 알아봅니다."
date: 2026-02-10
update: 2026-02-10
tags: [Java, Spring, Test, JUnit5, Mockito, TDD, 백엔드]
---

# 테스트 코드의 정석: JUnit5와 Mockito를 활용한 단위 및 통합 테스트 전략

## 서론
새로운 기능을 추가하고 배포하기 직전, 여러분은 어떤 기분이 드나요? "제발 아무 일 없게 해주세요"라며 기도하고 계신가요? 아니면 "내가 짠 코드가 기존 기능을 망가뜨리지는 않았을까?" 하는 불안함에 밤잠을 설치시나요?

![테스트 피라미드 모델](/images/02_Back-end/Test_Code_Fundamentals/test_pyramid_model.png)
*비용이 저렴하고 빠른 단위 테스트를 기반으로 중요한 흐름을 검증하는 통합 테스트를 쌓아 올리는 테스트 자동화 전략입니다.*

이런 공포에서 우리를 구원해 줄 유일한 방법은 바로 **테스트 코드**입니다. 테스트 코드는 단순히 "잘 돌아가나?"를 확인하는 용도를 넘어, 미래의 나를 위한 가장 강력한 보험이자 설계 도구입니다. 오늘은 자바 진영의 표준인 **JUnit5**와 **Mockito**를 이용해 우아하게 테스트를 짜는 법을 알아보겠습니다.

## 본론

### 1. 테스트의 종류: 피라미드 구조 이해하기
모든 것을 똑같은 방식으로 테스트할 수는 없습니다. 효율적인 테스트를 위해 우리는 단계를 나눕니다.

- **단위 테스트 (Unit Test)**: 가장 작은 단위(메서드 하나)를 테스트합니다. 속도가 매우 빠르고 에러 위치를 정확히 알 수 있습니다.
- **통합 테스트 (Integration Test)**: 여러 컴포넌트(DB, 외부 API 등)가 함께 잘 작동하는지 확인합니다. 스프링 컨테이너를 띄워야 하므로 상대적으로 느립니다.

**인사이트**: 테스트 피라미드 이론에 따라, 비용이 저렴하고 빠른 **단위 테스트**를 최대한 많이 작성하고, 중요한 흐름만 **통합 테스트**로 검증하는 것이 효율적입니다.

---

### 2. JUnit5: 자바 테스트의 표준 엔진
JUnit5는 현대적인 자바 테스트를 위한 강력한 기능을 제공합니다.

- **@Test**: 이 메서드가 테스트임을 알립니다.
- **@BeforeEach / @AfterEach**: 각 테스트 전후에 실행할 로직을 정의합니다. (데이터 초기화 등)
- **Assertions**: `assertEquals`, `assertThrows` 등을 통해 예상값과 실제값을 비교합니다.

```java
@Test
void 계산기_더하기_테스트() {
    // given (준비)
    Calculator calc = new Calculator();
    
    // when (실행)
    int result = calc.add(2, 3);
    
    // then (검증)
    assertEquals(5, result);
}
```
> **컴퓨터의 평**: "주인님, 2 더하기 3은 5가 맞네요. 제가 매번 확인해 드릴 테니 안심하세요!"

---

### 3. Mockito: 가짜 객체의 마법
테스트하려는 클래스가 다른 클래스(예: DB 접근용 Repository)에 의존하고 있다면 어떻게 할까요? 실제 DB를 연결하는 순간 단위 테스트는 무거워집니다. 이때 등장하는 것이 바로 **Mockito**입니다.

- **Mock**: 진짜처럼 행동하는 '가짜 객체'입니다.
- **Stubbing**: "이 가짜 객체의 특정 메서드를 호출하면 이런 값을 돌려줘!"라고 미리 약속하는 것입니다.

```java
@ExtendWith(MockitoExtension.class)
class MemberServiceTest {
    @Mock
    private MemberRepository memberRepository; // 가짜 저장소

    @InjectMocks
    private MemberService memberService; // 가짜가 주입된 서비스

    @Test
    void 회원_조회_테스트() {
        // given
        given(memberRepository.findById(1L))
            .willReturn(Optional.of(new Member("준오"))); // 약속!

        // when
        Member result = memberService.findById(1L);

        // then
        assertThat(result.getName()).isEqualTo("준오");
    }
}
```

---

### 4. 실무력을 높이는 결정적 인사이트
좋은 테스트 코드를 짜기 위한 3가지 철칙입니다.

1.  **Given-When-Then 패턴을 사용하세요**: 테스트 코드의 가독성을 비약적으로 높여줍니다. 어떤 상황에서(Given), 무엇을 했을 때(When), 어떤 결과가 나와야 하는지(Then)가 명확해야 합니다.
2.  **테스트는 독립적이어야 합니다**: 1번 테스트의 결과가 2번 테스트에 영향을 주면 안 됩니다. 각 테스트는 언제 어디서 실행되어도 항상 같은 결과를 내야 합니다(Deterministic).
3.  **성공 케이스보다 실패 케이스에 집중하세요**: "값이 잘 들어갈 때"보다 "잘못된 값이 들어왔을 때 예외가 적절히 터지는지"를 테스트하는 것이 시스템을 훨씬 견고하게 만듭니다.

## 결론 및 요약 / 회고
테스트 코드를 짜는 시간은 낭비가 아니라, 나중에 발생할 거대한 버그를 수정하는 시간을 미리 땡겨쓰는 것입니다.
- **단위 테스트**로 로직의 정확성을 확보하세요.
- **Mockito**로 의존성을 끊고 가볍게 테스트하세요.
- ** Given-When-Then**으로 읽기 좋은 테스트를 만드세요.

처음에는 테스트 코드를 짜는 게 귀찮고 어렵게 느껴질 수 있습니다. 하지만 내가 짠 코드 옆에 든든한 '검수원'이 항상 붙어있다는 느낌을 한 번 맛보게 되면, 다시는 테스트 없는 코드를 배포하고 싶지 않을 것입니다.

오늘도 여러분의 테스트가 모두 초록불(Success)로 가득 차길 응원합니다!

## 참고 자료
- JUnit 5 User Guide
- [Baeldung - Mockito Series](https://www.baeldung.com/mockito-series)
- 단위 테스트 (블라디미르 코리코프 저)
---
