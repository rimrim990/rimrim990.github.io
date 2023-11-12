---
layout: post
title: 도커 이미지 레이어
author: rimrim990
categories: [infra]
tags: [docker]
toc: true
---
도커 이미지 레이어에 대해 알아보자!
<!--break-->

## 도커 이미지
### 이미지 레이어 살펴보기
알다시피, 도커 컨테이너는 이미지를 기반으로 생성된다.
이미지는 컨테이너 애플리케이션 실행을 위한 모든 것들로 구성되 있는데, 여기에는 애플리케이션 코드 뿐만 아니라 실행에 필요한 시스템 바이너리, 의존성 라이브러리, 설정들도 포함된다.

**이미지를 구성하는 레이어**
```
$ docker pull nginx:latest

# 콘솔 출력
latest: Pulling from library/nginx
31ce7ceb6d44: Already exists
f1359798dfe4: Pull complete
4de1e0313830: Pull complete
7745719004b6: Pull complete
0f17732d34d5: Pull complete
0eb0ed12e64c: Pull complete
5836f8c1cebc: Pull complete
Digest: sha256:86e53c4c16a6a276b204b0fd3a8143d86547c967dc8258b3d47c3a21bb68d3c6
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
```

또한 컨테이너 이미지는 여러 개의 <strong>레이어</strong>로 구성되어 있다.
도커 허브에서 `pull`로 이미지를 다운받았을 때 위와 같은 출력을 본 적이 있을 것이다.

```
0f17732d34d5: Pull complete
0eb0ed12e64c: Pull complete
5836f8c1cebc: Pull complete
```
콘솔에 출력된 해시 값 한 줄 한 줄이 컨테이너 <strong>이미지</strong>에 해당한다.
자세히 살펴보면 하나의 이미지를 구성하는 여러 레이어를 각각 별도로 다운받고 있음을 알 수 있다.

**레이어 디렉터리**

앞서 다운받은 각 레이어들은 호스트 파일 시스템에 디렉터리로 존재하게 되는데, 보통 `/var/lib/docker/` 경로 하에 존재한다.

```
$ pwd
/var/lib/docker/overlay2/df569387d47ead32fe84766138afe270e1f46d3456343264f476c26834945631/diff

$ ls -a
.  ..  bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
앞서 도커 허브에서 다운받은 `nginx`를 구성하는 레이어 중 하나의 경로로 이동해보니, 다양한 디렉터리와 파일들이 존재했다.

### 레이어는 어떻게 생성되지?

**Dockerfile 작성**

도커 허브에서 컨테이너 생성에 필요한 이미지를 다운 받을 수도 있지만, `Dockerfile`을 작성하여 직접 커스텀 이미지를 빌드할 수도 있다.

```dockerfile
# syntax=docker/dockerfile:1

FROM ubuntu
LABEL org.opencontainers.image.authors="org@example.com"
COPY . /app
RUN apt-get update && apt-get install make
RUN make /app
CMD python /app/app.py
```
위와 같이 `Dockerfile`을 작성한 후 이미지를 빌드하면, 파일에 명시된 각 명령어들이 각각 하나의 레이어가 된다.
정확히는 컨테이너의 파일 시스템을 변경하는 명령어만이 레이어로 생성된다.
따라서 `LABEL`과 같이 이미지의 메타데이터를 변경하는 명령어는 레이어로 생성되지 않는다.

각 레이어는 이전 레이어에서 변경된 파일들만을 포함하고 있다.
만약 현재 레이어에서 파일이 추가되거나 제거된다면 새로운 레이어가 생성될 것이다.

**명령어 살펴보기**
```dockerfile
FROM ubuntu
```
이미지 레이어 생성 과정을 이해하기 위해, `Dockerfile`에 명시된 명령어를 순차적으로 살펴보도록 하겠다.
우선, `FROM` 명령어는 이미지 빌드 시에 사용할 기반 이미지를 설정한다. 이로 인해 `ubuntu` 이미지로부터 첫 번째 레이어가 생성된다.

```dockerfile
COPY . /app
```
`COPY` 명령어는 호스트의 현재 위치에서 파일들을 복사해 컨테이너 내부로 붙여넣어준다.
이로 인해 컨테이너 파일시스템이 변경되면서 하나의 레이어가 추가되었다.

```dockerfile
RUN apt-get update && apt-get install make
RUN make /app
```
`RUN` 명령어는 현재 레이어 위에 새로운 레이어를 생성하고, 그 안에서 인자로 넘어온 명령어를 수행한다.
두 번의 `RUN` 명령오 수행으로 또 다시 두 개의 레이어가 추가되었다.

```dockerfile
CMD python /app/app.py
```
`CMD` 명령어는 빌드된 이미지로부터 컨테이너가 시작될 때 내부에서 실행할 명령어를 설정한다.
이미지로부터 컨테이너를 새로 시작한다면, 컨테이너 내부에서는 `python /app/app.py`가 자동으로 수행될 것이다.
`CMD` 명령어는 이미지의 메타데이터만 변경하기 때문에 레이어를 생성하지는 않는다.

**이미지 빌드하기**
```
$ docker build -t test .

# 콘솔 출력
=> [1/4] FROM docker.io/library/ubuntu:latest
=> [2/4] COPY . /app
=> [3/4] RUN apt-get update && apt-get install make
=> [4/4] RUN make /app
=> exporting to image
```
`build` 명령어를 사용하여 앞서 작성한 `Dockerfile`을 빌드하였다.
이미지의 레이어를 생성한 명령어들이 콘솔에 출력됨을 확인할 수 있었다.

```
$ docker inspect test

# 이미지 레이어 정보
"Layers": [
                "sha256:d2d3127fc3d362a0516c53fcb8fb983ebb9d1ea2dfbd63b69967a0084f0d45a3",
                "sha256:9bab4925ee5b967c5ad8bc245733e4191f77e575cc529d01b593d04619144ebf",
                "sha256:d4d988d4f9b4cdb15c9501abaff332127aaaa87b0b3eaf1b12ba8bf019b209ea",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
]
```
`inspect` 명령어로 이미지 정보를 출력해보니 실제로도 4개의 레이어가 존재했다.

### 이미지 레이어 중복의 문제

**문제점**

앞서 컨테이너 이미지에는 애플리케이션 코드 뿐만 아니라 시스템 바이너리, 의존성 라이브러리, 설정 등이 포함됨을 언급했었다.
컨테이너 애플리케이션 실행을 위해 필요한 각종 파일들이 하나의 이미지로 패키징되어 있어, 간편하게 실행 환경을 구축할 수 있다.
그렇지만 문제점도 존재하는데, 서로 다른 이미지라 하더라도 자주 사용되는 파일들은 중복될 가능성이 높다는 점이다.

```
# 이미지 A
nginx
ubuntu

# 이미지 B
mysql
nginx
ubuntu
```
만약 위와 같이 서로 다른 이미지 A와 B를 생성한다고 가정해보자.
생성된 두 이미지는 `nginx`와 `ubuntu`를 구성하는 이미지를 중복하여 포함할 것이다.
이러한 이미지 중복 문제로 인해 사용자는 불필요하게 저장 공간을 낭비하고, 배포 시 네트워크 자원을 더 많이 사용하게 된다.

## 컨테이너 레이어

도커에서는 이미지 중복 문제를 어떻게 해결하였을까?
앞서 이미지는 여러 레이어들의 조합으로 생성됨을 확인했는데, 도커는 이미지 간 중복을 최소화하기 위해 동일한 레이어는 공유하는 전략을 채택하였다.

### 컨테이너 레이어 구조 살펴보기

`Dockerfile`을 통해 이미지를 빌드하면 각 명령어들이 레이어가 되어 차곡차곡 쌓이게 된다.
이렇게 만들어진 레이어들은 다른 이미지와의 공유를 위해 읽기 전용으로만 사용된다.
그런데 만약 컨테이너 실행 중에 새로운 파일을 작성하거나 기존 파일을 제거하는 등, 파일 시스템에 변화가 생기면 이는 어떻게 처리 될까?

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/Algotirhm/assets/62409503/a52b6f8f-e447-4653-9785-414a296cea3c">
<figcaption style="text-align: center">[이미지 출처] Docker</figcaption>
</figure>

컨테이너가 시작되면 해당 컨테이너는 쓰기가 가능한 레이어를 새로 할당받는데, 이를 `컨테이너 레이어`라고 한다.
반대로 컨테이너 하위에 쌓여있는 읽기 전용 레이어들은 `이미지 레이어`라고 한다.
도커는 이미지 레이어에 쓰기 작업이 발생하면, 이를 컨테이너 레이어에서 처리한다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/Algotirhm/assets/62409503/e354d86c-2283-438a-bb1b-696c40a76dfc">
<figcaption style="text-align: center">[이미지 출처] Docker</figcaption>
</figure>

그림에서처럼 모든 컨테이너들은 이미지 레이어는 공유하고, 컨테이너 레이어는 독립적으로 보유하고 있다.
만약 이미지 레이어에 속한 파일에 쓰기 작업이 발생하면, 우선 해당 파일을 컨테이너 레이어로 복사온다.
이후 원본 파일이 아닌 컨테이너 레이어에 복사된 파일에 쓰기 작업을 수행한다.

~~도커 컨테이너의 이러한 동작 방식은 오버레이 파일 시스템에서 기인한 것인데, 이는 다음 시간에 살펴보도록 하겠다!~~

> Docker uses storage drivers to manage the contents of the image layers and the writable container layer. Each storage driver handles the implementation differently, but all drivers use stackable image layers and the copy-on-write (CoW) strategy.

어떤 스토리지 드라이버를 사용하더라도 내부 구현만 다를 뿐, 이미지와 컨테이너 레이어 구조를 사용하며 복사본에 쓰기 작업을 수행한다는 점은 동일하다.

## 참고자료
- 용찬호, 『시작하세요! 도커 / 쿠버네티스』, 위키북스(2020)
- 카카오 엔터프라이즈 Tech&, https://tech.kakaoenterprise.com/171
- docker, https://docs.docker.com/storage/storagedriver/
