---
layout: post
title: Docker와 VSCode로 개발환경 구성하기
categories: [dev]
tags:
  - docker
  - vscode
author: 심동근
email: sdg@jinhakapply.com
date: 2022-01-28
---

# Docker 설치

도커를 설치한다. wsl2 버전으로 설치!

> [Docker](https://www.docker.com/get-started)

# Remote - Containers VSCode 확장프로그램 설치

VSCode 확장프로그램 `Remote - Containers`을 설치한다. 로컬에 있는 파일이나 외부 파일을 컨테이너로 만든다.

> [Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

# 컨테이너 생성

## 명령어

Ctrl + Shift + P를 눌러 command palette를 띄우고 `Remote-Containers`를 입력하면 다양한 방법으로 도커 컨테이너를 생성할 수 있따는 것을 알 수 있다. 예제로 Github에 있는 프로젝트를 클론해서 생성해보겠다.
![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-0.png)

## Github 클론 후 컨테이너 생성

- Remote-Containers: Clone Repository in Named Container Volume 선택
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-1.png)
- Git Clone할 URL 입력
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-2.png)
- Create a new volume 선택
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-3.png)
- Volume 이름 입력
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-4.png)
- 메인 폴더 이름 입력
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-5.png)
- 도커 설정 선택.. Show All Definitions (저는 이미 정의된 것 사용)
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-6.png)
- Jekyll 프로젝트이기에 Jekyll 선택 (여러분에게 맞는걸 선택하길)
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-7.png)
- OS버전 선택
  > ![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-8.png)

그럼 이제 프로젝트가 자동으로 만들어진다.
물론 여러가지 언어들을 사용할 수 있고 상황에 맞게 옵션이 알맞게 보인다.
정말 잘 만든 것 같다.

# VSCODE에서 소스코드 작업

![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-9.png)
터미널에서 기존에 쓰던 명령어 실행이 가능하다.

```bash
vscode ➜ /workspaces/sdg9670.github.io (main ✗) $ bundle exec jekyll serve --livereload
```

또한 VSCode 확장 프로그램은 기존에 쓰던 것과 분리된다. 그러므로 기존에 쓰던 확장 프로그램을 일괄로 옮기던지 새로 설치하면된다.

![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-10.png)

개방하는 포트도 자동으로 감지한다. 아니면 따로 설정이 가능하다.

![](/assets/img/posts/dev/2022-01-28-docker-with-vscode/2021-12-28-vscode-with-docker-11.png)

이제 신나게 소스코드를 작업하고 커밋 후 푸쉬만 하면된다.

# 이렇게 하면 뭐가 좋지?

기존 로컬에 설치해서 개발하면 개발 환경을 세팅하려면 시간이 오래걸렸다. 또한 다른 팀원들과 개발 환경이 달라서 발생하는 문제도 있을 것이다. 도커 이미지로 개발 환경을 세팅함으로써 개발 환경 구성 시간이 단축되고 일관된 환경에서 개발이 가능하다. **그리고 무엇보다 로컬 PC를 깨끗하게 관리할 수 있다**

# 주의 사항

웬만하면 Git Clone처럼 Docker Volume안에 파일을 보관하는 것이 좋다. 그 이유는 로컬 파일을 마운트시 속도가 상당히 느리다. 아니 `개발이 불가능할 정도`다. 로컬하고 파일 복사, 붙여넣기는 잘되니 파일들을 마운트말고 Volume에 보관해서 개발하는 것을 추천한다.