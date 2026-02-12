---
title: "CI/CD와 배포 자동화: Docker와 Github Actions로 구축하는 현대적 개발 환경"
series_order: 14
description: ""내 컴에선 되는데?"라는 고전적인 문제부터, 배포 버튼 하나로 서버가 업데이트되는 마법 같은 자동화 여정을 다룹니다."
date: 2026-02-10
update: 2026-02-10
tags: [Docker, GithubActions, CICD, DevOps, 배포, 클라우드, 백엔드]
---

# CI/CD와 배포 자동화: Docker와 Github Actions로 구축하는 현대적 개발 환경

## 서론
과거의 배포는 한 편의 전쟁과 같았습니다. 개발자가 코드를 빌드하고, FTP로 서버에 접속해서 파일을 올리고, 기존 서버를 껐다가 켜는 이 모든 과정이 수동이었죠. 이 과정에서 파일 하나라도 빠뜨리면 서버는 비명을 지르고, 사용자들은 분노했습니다.

"내 컴퓨터에선 분명히 잘 됐는데, 서버만 가면 왜 이럴까?" 

![CI/CD 파이프라인 개요도](/images/02_Back-end/CI_CD_and_Deployment/cicd_pipeline_overview.png)
*코드 푸시부터 빌드, 테스트, 이미지 생성 및 서버 반영까지 이어지는 전체 배포 자동화 파이프라인의 구성도입니다.*

이 지긋지긋한 문제를 해결하기 위해 등장한 것이 바로 **컨테이너(Docker)**와 **자동화(CI/CD)**입니다. 오늘은 배포 버튼 하나로 테스트부터 배포까지 한 방에 끝내는 마법 같은 여정을 떠나보겠습니다.

## 본론

### 1. Docker: 어디서든 똑같이 돌아가는 마법의 상자
도커는 애플리케이션과 그 실행 환경(Java 버전, 설정 등)을 하나의 **이미지**라는 상자에 통째로 담아버립니다.

- **핵심 가치**: "어디서 실행하든 100% 똑같은 환경을 보장한다."
- **인사이트**: 더 이상 서버에 자바를 깔고 환경 변수를 맞추느라 고생할 필요가 없습니다. 도커 이미지 하나면 내 컴퓨터에서도, AWS에서도, 구글 클라우드에서도 똑같이 작동하니까요.

```dockerfile
# Dockerfile 예시
FROM openjdk:17-jdk-slim
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```
> **컴퓨터의 평**: "주인님, 이제 서버 환경 걱정하지 마세요. 이 상자(Container)만 있으면 저는 어디서든 행복하게 일할 수 있어요!"

---

### 2. CI/CD: 멈추지 않는 개발 공장
- **CI (Continuous Integration)**: 지속적 통합. 코드 변경 사항을 합칠 때마다 자동으로 빌드하고 테스트를 돌려 결함을 찾습니다.
- **CD (Continuous Deployment)**: 지속적 배포. 검증된 코드를 자동으로 운영 서버에 반영합니다.

**인사이트**: CI/CD가 잘 구축된 팀은 배포가 두렵지 않습니다. 기계가 완벽하게 검증하고 배포해 주기 때문에, 개발자는 오직 '좋은 코드'를 만드는 데만 집중할 수 있습니다.

---

### 3. Github Actions: 깃허브에서 일어나는 자동화 마법
가장 대중적인 CI/CD 도구 중 하나입니다. 코드를 푸시(push)하는 순간 미리 정의한 시나리오대로 움직입니다.

![Github Actions 배포 파이프라인](/images/02_Back-end/CI_CD_and_Deployment/github_actions_pipeline.png)
*코드 변경 시 Github Actions가 감지하여 빌드 및 테스트를 수행하고 결과물을 배포 환경으로 전달하는 자동화 흐름도입니다.*

```yaml
# .github/workflows/deploy.yml
name: Deploy to Prod
on:
  push:
    branches: [ main ]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Build with Gradle
        run: ./gradlew build
      - name: Docker build and Push
        run: docker build -t my-app . && docker push my-app
```

---

### 4. 실무력을 높이는 결정적 인사이트
자동화 환경을 구축할 때 반드시 챙겨야 할 포인트입니다.

1.  **테스트 코드가 CI의 핵심입니다**: 자동 빌드를 아무리 잘 만들어도 테스트 코드가 없다면 "자동으로 망가진 코드를 배포하는 기계"일 뿐입니다. CI 단계에서 반드시 모든 테스트가 통과해야 다음 단계로 넘어가게 설계하세요.
2.  **보안 정보를 암호화하세요**: DB 비밀번호나 API 키를 코드나 설정 파일에 직접 쓰면 절대 안 됩니다. Github Secrets 같은 도구를 이용해 민감한 정보는 철저히 숨겨야 합니다.
3.  **무중단 배포를 지향하세요**: 새로운 버전이 올라가는 동안 서버가 멈춘다면 사용자는 불편함을 느낍니다. **Blue-Green**이나 **Rolling Update** 같은 전략을 통해 사용자가 눈치채지 못하게 조용히 교체하는 것이 프로의 자세입니다.

## 결론 및 요약 / 회고
CI/CD와 Docker는 이제 선택이 아닌 필수입니다. 
- **Docker**로 환경의 일관성을 확보하세요.
- **CI**로 코드의 품질을 자동으로 검증하세요.
- **CD**로 배포의 스트레스를 날려버리세요.

배포 자동화 시스템을 처음 구축할 때는 조금 막막할 수 있습니다. 하지만 한 번 제대로 만들어둔 파이프라인은 여러분의 개발 인생 수백 시간을 아껴줄 것입니다. 이제 배포 버튼을 누르고 여유롭게 커피 한 잔을 즐기는 개발자가 되어보세요!

오늘도 여러분의 메인 브랜치가 초록색 체크 표시(Success)로 빛나길 응원합니다!

## 참고 자료
- Docker Documentation: Getting Started
- [Github Actions 공식 문서](https://docs.github.com/en/actions)
- 인프라 엔지니어의 길 (가사이 사토시 저)
---
