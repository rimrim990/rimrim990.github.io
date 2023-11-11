---
layout: post
title: 오버레이 파일 시스템
author: rimrim990
categories: [infra]
tags: [docker]
toc: true
---
도커에서 사용하는 오버레이 파일 시스템에 대해 알아보자!
<!--break-->

## 도커 스토리지 드라이버

### 스토리지 드라이버란 무엇인가?
> Docker supports several storage drivers, using a pluggable architecture. The storage driver controls how images and containers are stored and managed on your Docker host.

스토리지 드라이버는 이미지와 컨테이너 레이어를 어떻게 저장하고 관리할지에 대한 매커니즘을 제공한다. 도커는 여러 가지 스토리지 드라이버를 제공하는데 이를 플러그인처럼 간편하게 사용할 수 있다.

**스토리지 드라이버 확인하기**
```shell
$ docker info | grep "Storage Driver"
Storage Driver: overlay2
```
- 리눅스 계열의 운영체제는 기본적으로 `overlay2` 스토리지 드라이버를 사용한다.

```shell
$ ls /var/lib/docker/overlay2/

0ad4eab8f323829d2e83243657092304815f32965ed2a32316ca87db837aa9ee
1687ca7b349c20f0abbf4a2796d3a0823fe439fa4383dc985cf5286690a57bbb
b1730d66bd175233edb2d7ca7ecea25666ade282d404b6e7bc6f046bf4b2ab7c
```
- 특정 스토리지 드라이버를 사용해 생성한 파일들은 `/var/lib/docker/{드라이버_이름}` 경로 하에 저장된다

**스토리지 드라이버 변경하기**
```shell
# docker 중단
$ sudo systemctl stop docker

# /etc/docker/daemon.json 에 스토리지 드라이버 설정
$ sudo vi /etc/docker/daemon.json
{
  "storage-driver": "vfs"
}

# docker 시작
$ sudo systemctl start docker
```
- `/etc/docker/daemon.json`에 사용하고자 하는 스토리지 드라이버를 설정해주면, 도커 데몬이 재시작될 때 해당 설정 값을 읽어 스토리지 드라이버를 선택한다

```shell
$ docker info | grep "Storage Driver"
Storage Driver: vfs
```
- `docker info`를 다시 수행해보니 스토리지 드라이버가 변경되었다

**주의할 점**
```shell
$ docker image ls
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE

$ docker container ls
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
- 스토리지 드라이버를 `vfs`로 변경했기 때문에 이전에 `overlay2`에서 생성한 이미지와 컨테이너에 접근할 수 없게 되었다

```shell
$ sudo ls /var/lib/docker/

overlay ... vfs
```
- 이는 도커가 스토리지 드라이버 마다 컨테이너와 이미지 레이어를 별도로 관리하기 때문이다
- `overlay2`에서는 `/var/lib/docker/overlay`디렉터리를 사용했지만, `vfs` 는 `/var/lib/docker/vfs`를 사용하게 된다

```shell
$ sudo vi /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```
- 스토리지 드라이버 설정을 `overlay2`로 변경하고 도커를 재시작한다면, 이전에 생성한 이미지와 컨테이너들에 다시 접근 가능해진다

## OverlayFS

### OverlayFS 무엇인가?

`오버레이 파일시스템`인 OverlayFS는 `유니온 파일 시스템`의 최신 구현체 중 하나로, 이를 향상시킨 버전이 `Overlay2` 파일 시스템이다
- 도커는 `Overlay2` 파일 시스템을 기본 스토리지 드라이버로 사용한다

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/a285c676-92f1-41c0-acb1-d7b4b27c3097">
<figcaption style="text-align: center">[이미지 출처] alice_k106 블로그 </figcaption>
</figure>

오버레이 파일 시스템은 여러 개의 파일 시스템을 밑에서부터 층층히 쌓아 올려 레이어를 생성하고, 최상위 층에서는 하위 레이어들이 하나로 통합된 뷰를 제공한다.
이렇게 하위 파일 시스템들을 하나의 파일 시스템으로 마운트 하는 기능을 `유니온 마운트`라고 한다.

### 오버레이 파일시스템 살펴보기
오버레이 파일시스템은 호스트 상의 여러 디렉터리들을 하나의 디렉터리로 통합된 뷰를 제공한다. 오버레이 파일시스템에서는 각 디렉터리들을 `레이어`, 통합된 뷰를 생성하는 작업을 `유니온 마운트`라고 일컫는다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/283c8fdf-64eb-4c57-88f1-33e91c113c34">
<figcaption style="text-align: center">[이미지 출처] alice_k106 블로그 </figcaption>
</figure>

오버레이 파일 시스템을 구성하는 레이어의 종류에는 `lowerdir`, `upperdir`, `merged`가 존재한다
- `lowerdir`는 읽기 전용의 하위 레이어이다
- `upperdir`는 쓰기 가능한 레이어로, `lowerdir`의 상위에 위치한다
- `merged`는 하위 레이어들의 통합된 뷰를 제공하는 레이어이다

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/958b153d-3731-43c4-83f3-8d0d0f3c8f01">
<figcaption style="text-align: center">[이미지 출처] 카카오 엔터프라이즈 Tech & </figcaption>
</figure>

오버레이 파일 시스템에서 레이어에는 순서가 존재한다.

예를 들어,  파일 `a`와 `b`를 갖는 레이어1과 `b`와 `c`를 갖는 레이어2를 유니온 마운트한다고 가정해보자.
- 만약 레이어1 위에 레이어2가 쌓인다면, `merged` 레이어는 레이어2의 파일 `b`를 보게 된다
- 만약 레이어2 위에 레이어1이 쌓인다면, `merged` 레이어는 레이어1의 파일 `b`를 보게 된다

정리하면 하위 레이어와 상위 레이어에 동일한 파일이 존재할 때, 하위 레이어의 파일은 상위 레이어에 의해 가려진다.

### Cow
만약 읽기 전용인 `lowerdir`에 쓰기 작업이 발생하면 어떻게 처리될까?

오버레이 파일시스템은 우선 `lowerdir`에 있는 원본 파일을 쓰기 전용 레이어인 `upperdir`로 복사해온다.
이후 `upperdir`에 복사된 파일에 쓰기 작업을 수행한다.
`upperdir` 레이어는 `lowerdir`보다 상위에 존재하기 때문에, `merged`에는 `upperdir`의 수정본만 보이게 된다.

오버레이 파일 시스템처럼 복사된 원본 파일에 쓰기 요청을 반영하는 전략을 `Copy-on-Write`, `CoW`라고 한다.

오버레이 파일 시스템에서는 `CoW` 방식을 사용하기 때문에 `lowerdir` 레이어를 공유할 수 있어, 저장 공간을 절약할 수 있다. 그러나 쓰기 작업을 수행할 때마다 원본 파일을 복사해야 한다는 오버헤드가 존재한다.

### 오버레이 파일시스템 사용해보기
**레이어 생성하기**
```shell
$ mkdir lower1 lower2 upper merge work

$ ls
lower1 lower2 merge upper work
```
- 오버레이 파일 시스템을 사용해보기 앞서, 레이어가 될 디렉터리들을 생성하였다

```shell
$ echo docker >> lower1/a
$ echo overlay >> lower1/a
$ echo filesystem >> lower2/b

$ ls lower1/
a b
$ ls lower2/
b
```
- `lowerdir` 레이어가 될`lower1`과 `lower2` 디렉터리에 파일을 추가하였다
- `lower1`과 `lower2`에는 동일한 이름의 `b`라는 파일이 존재한다

**유니온 마운트하기**
```shell
sudo mount -t overlay overlay -o lowerdir=lower2:lower1,upperdir=upper,workdir=work merge
```
- `mount` 명령어로 오버레이 파일 시스템을 마운트하였다
  - 마운트 타입 (`-t`) 으로 오버레이 파일 시스템을 선택하였고, 옵션 (`-o`) 에서 레이어를 지정해주었다
  - `lowerdir`에 여러 레이어를 쌓을 때 `:`을 사용하여 레이어를 구분하였는데, 먼저 기입된 레이어가 상위 레이어가 된다

```shell
$ ls merge/
a b

$ cat merge/b
filesystem
```
- `merge` 디렉터리에 오버레이 파일시스템을 마운트함으로써 통합된 뷰가 생성되었다
- `lower1`의 파일 `b`는 상위 레이어인 `lower2`의 파일 `b`에 의해 가려졌다

**디렉터리 파일 수정하기**
```shell
$ rm merge/a

$ ls merge/
b
```
- 오버레이 파일시스템의 동작을 이해하기 위해 `merge` 레이어에서 파일 `a`를 삭제해보았다
- `merge` 레이어에는 더이상 파일 `a`가 존재하지 않는다

```
$ ls upper/
a (삭제 마킹)

$ ls lower1/
a
```
- 그러나 파일 `a`의 원본을 갖고 있던 `lower1` 레이어어는 여전히 파일 `a`가 존재한다
- 또한 `upper` 레이어에 파일 `a`가 삭제 마킹된 상태로 추가되었다
- 쓰기 작업이 발생했을 때 `CoW`가 제대로 적용되고 있음을 확인할 수 있었다

**주의할 점**
```shell
$ echo edited >> lower2/b

$ cat lower2/b
edited
```
- 원활한 실습을 위해서는 직접 하위 레이어를 수정하지 않도록 주의해야 한다

```shell
$ ls upper/
a

$ cat merge/b
filesystem
```
- `CoW`는 오버레이 파일 시스템의 마운트 지점인 `merge/`에 한해서만 동작하기 때문이다
- 하위 레이어인 `lower1`과 `lower2`는 다른 파일 시스템에 속해있기 때문에, 파일을 수정하여도 `CoW` 방식으로 동작하지 않는다
  - `upper` 레이어로 원본이 복사되지 않았다
  - `merge/b`와`lower2/b`의 출력이 일치하지 않게 되었다

## 도커 컨테이너와 Overlay2

### 도커에 적용된 Overlay2 파일시스템
<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/Algotirhm/assets/62409503/e354d86c-2283-438a-bb1b-696c40a76dfc">
<figcaption style="text-align: center">[이미지 출처] Docker</figcaption>
</figure>

지난 포스팅에서 도커 컨테이너는 읽기 전용인 `이미지 레이어`와 쓰기 가능한 `컨테이너 레이어`로 구성됨을 알아보았다. 도커 컨테이너의 이미지 레이어가 `OverlayFS`의 `lowerdir`, 컨테이너 레이어가 `upperdir`에 해당된다.

컨테이너 레이어는 도커 컨테이너가 시작할 때마다 새로 생성되는데, 컨테이너가 종료되면 제거되는 휘발성 데이터이다. 따라서 컨테이너가 종료되어도 데이터를 영구적으로 보존하고 싶다면 볼륨을 사용해야 한다.

### 컨테이너에서 파일 읽기
`overlay2`를 사용해 생성된 도커 컨테이너의 읽기와 쓰기 동작은 앞서 살펴본 `OverlayFS`와 동일한다.

파일 읽기 요청이 들어오면, 최상위에 존재하는 컨테이너 레이어의 파일을 읽는다. 만약 컨테이너 레이어에 파일이 없으면 이미지 레이어의 파일을 읽어올 것이다.

컨테이너 레이어와 이미지 레이어에 동일한 이름의 파일이 존재하면, 상위 레이어인 컨테이너 레이어의 파일이 우선시된다.

### 컨테이너에서 파일 수정하기
컨테이너에서 파일을 처음 수정하면, 해당 파일은 아직 컨테이너 레이어에 존재하지 않을 것이다. 따라서 이미지 레이어에서 원본을 복사해온다. 이후 컨테이너 레이어에서 복사된 파일에 쓰기 작업을 수행한다.
- 오버레이 파일시스템에서는 파일을 복사할 때 블락 단위가 아닌 파일 전체를 복사해온다. 따라서 파일의 크기가 크다면 쓰기 작업이 더 오래 소요된다
- 하위 레이어의 수가 많다면 파일을 찾는데 더 오랜 시간이 소요되기 때문에 쓰기 작업에 영향을 줄 수 있다

컨테이너에서 파일을 삭제해도 원본 파일은 그대로 유지되며, 컨테이너 레이어에는 삭제 마킹된 (`whiteout`) 파일이 생성된다.

## 참고자료
- 용찬호, 『시작하세요! 도커 / 쿠버네티스』, 위키북스(2020)
- alice_k106, https://blog.naver.com/alice_k106/221530340759
- 카카오 엔터프라이즈 Tech&, https://tech.kakaoenterprise.com/171
- docker, https://docs.docker.com/storage/storagedriver/overlayfs-driver/

