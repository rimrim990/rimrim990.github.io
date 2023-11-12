---
layout: post
title: 도커 컨테이너 데이터 저장하기
author: rimrim990
categories: [infra]
tags: [docker]
toc: true
---
컨테이너 데이터를 영구 저장하는 방법을 알아보자!
<!--break-->

## 컨테이너 데이터 관리하기

### 데이터가 저장되는 위치
컨테이너 안에서 생성되거나 수정된 모든 데이터는 쓰기 가능한 **컨테이너 레이어**에 저장된다. 컨테이너 레이어에 저장된다는 것은, 해당 컨테이너가 종료되면 데이터도 함께 삭제됨을 의미한다.

**문제점**

도커에서 이미지와 컨테이너 레이어는 스토리지 드라이버에 의해 관리된다. 도커의 모든 스토리지 드라이버는 레이어 구조와 `CoW` 전략을 사용하기 때문에, 데이터를 작성할 때 오버헤드가 발생한다.
호스트 파일 시스템에 파일을 작성한다면, 아래와 같은 복잡한 과정은 필요 없어진다.
- 레이어 스택을 탐색하여 일치하는 파일을 찾아내야 한다.
- 읽기 레이어에서 원본 파일을 복사해와야 한다

### 컨테이너 데이터를 외부로 가져오기
만약 컨테이너 생성한 데이터를 다른 프로세스에서 사용하고 싶다면 어떻게 해야할까?

**직접 가져와보기**
```shell
$ docker run -it ubuntu

$ root@3cdfae1e1642:/# echo this is container >> hello
$ root@3cdfae1e1642:/# ls
bin  boot  dev  etc  hello  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
- 테스트를 위해 `ubuntu` 컨테이너를 포그라운드 모드로 실행하였다
- `ubuntu` 컨테이너 내부에서 `hello` 라는 이름의 파일을 생성하였다

```shell
$ docker inspect crazy_raman

"GraphDriver": {
  "Data": {
    "UpperDir": "/var/lib/docker/overlay2/d88ad095f3e2f8b1849c6880407082e5608f460d47afc294608c1e0ff090f8d0/diff",
  },
  "Name": "overlay2"
},
```
- 컨테이너가 실행 중인 상태에서 다른 쉘을 열어 생성한 컨테이너에 대해 `inspect` 명령어를 실행하였다
- 출력된 정보를 통해 컨테이너 레이어 디렉터리인 `UpperDir`의 위치를 확인할 수 있었다

```shell
# 데이터 확인하기
$ sudo ls /var/lib/docker/overlay2/d88ad095f3e2f8b1849c6880407082e5608f460d47afc294608c1e0ff090f8d0/diff
hello

# 출력하기
$ sudo cat /var/lib/docker/overlay2/d88ad095f3e2f8b1849c6880407082e5608f460d47afc294608c1e0ff090f8d0/diff/hello
this is container
```
- `UpperDir` 디렉터리로 이동하자 앞서 컨테이너 내부에서 생성했던 `hello` 파일을 찾을 수 있었다
- `cp` 명령어를 사용하여 컨테이너 데이터를 호스트로 복사하는 것도 가능하다

**문제점**

컨테이너 레이어가 저장된 디렉터리를 찾아내 직접 파일을 복사해올 수 있었다. 그러나 매번 컨테이너 레이어가 저장된 위치에 접근하여 데이터를 가져오는 것은 상당히 귀찮은 일이다.

게다가 실행 중인 컨테이너가 종료되면 컨테이너 레이어가 제거되기 때문에 이마저도 불가능해진다.
```shell
# 디렉터리가 존재하지 않는다!
$ sudo ls /var/lib/docker/overlay2/d88ad095f3e2f8b1849c6880407082e5608f460d47afc294608c1e0ff090f8d0
No such file or directory
```
- 컨테이너를 종료한 후 호스트에서 동일 위치에 다시 접근하였으나, 디렉터리가 존재하지 않았다

한 가지 문제점이 더 있는데, 도커 컨테이너 간에는 데이터 공유가 불가능하다는 것이다.
컨테이너는 격리된 파일 시스템을 사용하기 때문에 호스트처럼 다른 컨테이너의 레이어에 접근할 수 없다.

```shell
# ㅠㅠ
$ root@948ca18bb0d8:/# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
- 컨테이너 내부에서는 자신에게 할당된 파일 시스템만 접근할 수 있다

## 호스트 머신에 데이터 저장하기
도커는 **호스트 저장 공간**에 데이터를 영구적으로 보관할 수 있는 기능을 제공한다.
- 컨테이너가 종료되어도 데이터는 유지된다
- 컨테이너 간에 데이터 공유가 가능해진다

호스트 머신에 데이터를 저장할 수 있는 방법으로는 **볼륨**과 **바인드 마운트**가 존재한다. 이러한 방식들은 호스트의 파일 혹은 디렉터리를 컨테이너 파일시스템의 특정 경로로 마운트해준다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://docs.docker.com/storage/images/types-of-mounts.webp?w=450&h=300">
<figcaption style="text-align: center">[이미지 출처] docker </figcaption>
</figure>

볼륨과 바인드 마운트 중에서 어떠한 방식을 사용하더라도, 컨테이너 내부에서는 데이터에 동일하게 접근할 수 있다.
사용자는 컨테이너 안에서 일반적인 파일 혹은 디렉터리처럼 사용 가능하다.

### 볼륨
**볼륨**은 도커에 의해 관리되는 위치에 생성된다.

**볼륨 생성하기**
```
$ docker volume create test
```
- 도커가 적절한 위치에 생성해주기 때문에 어느 위치에 생성할지 고민할 필요가 없다
- 컨테이너 안에서 데이터에 접근할 때도 파일이 실제 어느 위치에 존재하는지 신경쓸 필요가 없다

```shell
$ docker volume inspect test
[
    {
        "CreatedAt": "2023-11-12T10:20:12Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/test/_data",
        "Name": "test",
        "Options": null,
        "Scope": "local"
    }
]
```
- 볼륨이 생성된 위치는 `/var/lib/docker/` 디렉터리 하위였다
  - `/var/lib/docker`는 도커에서 사용하고 있는 루트 경로이다

**볼륨 사용하기**
```shell
# 볼륨을 사용하는 컨테이너 생성
$ docker run -it -v test:/root/ ubuntu

# 볼륨 위치에 파일 생성
$ root@f86fac5dec4a:~# cat test >> /root/hello
```
- 이전에 생성했던 `test` 볼륨을 사용하는 컨테이너를 실행하였다
  - 컨테이너의 `/root/` 경로에 볼륨이 마운트될 것이다
- 볼륨이 마운트 된 경로에 `hello`라는 파일을 생성하였다

```shell
# 동일한 볼륨을 사용하는 컨테이너 새로 생성
$ docker run -it -v test:/root/ ubuntu

# 파일이 존재
$ root@68e7d5f37440:/# cat /root/hello
test
```
- 동일한 볼륨을 사용한 다른 컨테이너를 실행하였다
  - 여러 컨테이너에서 동일한 볼륨 사용이 가능하기 때문에 정상 실행되었다
- 이전 컨테이너에 생성한 `/root/hello` 데이터가 존재함을 확인할 수 있었다

컨테이너에 마운트된 볼륨은 호스트 파일 시스템에 존재하는 디렉터리였다. 호스트의 디렉터리를 공유할 뿐, 특별할 건 없다.

```shell
$ sudo ls /var/lib/docker/volumes/test/_data
bye  hello

$ root@68e7d5f37440:/# ls /root/
bye  hello
```
- 호스트에서 볼륨이 생성된 경로에 접근하여 파일을 추가해보았다
- 컨테이너에서는 동일한 디렉터리를 사용하기 때문에, 호스트가 추가한 파일을 확인할 수 있다

**사용하지 않는 볼륨 제거하기**
```shell
$ docker volume prune
WARNING! This will remove anonymous local volumes not used by at least one container.
```
- `prune` 명령어를 사용하여 컨테이너에서 사용되지 않는 볼륨을 간편하게 제거 가능하다

### 바인드 마운트

**바인드 마운트**는 명시된 경로의 디렉터리 혹은 파일을 컨테이너에 마운트한다.
- 호스트 디렉터리의 경로는 절대 경로여야 한다
- 볼륨과 다르게 컨테이너에서 사용할 호스트 디렉터리의 경로를 직접 기입해주어야 한다

바인드 마운트는 호스트 디렉터리를 직접 사용해야 하기 때문에 호스트 파일 시스템에 종속적이게 된다.
따라서 도커 문서에서는 바인드 마운트보다는 볼륨 사용을 권장하고 있다.
- 바인드 마운트는 도커 CLI 명령어로 관리가 불가능하다

**바인드 마운트 사용하기**
```shell
$ docker run -it --mount type=bind,source=/home/ubuntu/lower1,target=/app ubuntu
```
- 마운트 타입을 `bind`로 설정하여 호스트의 `/home/ubuntu/lower1` 디렉터리를 컨테이너 내부로 마운트하였다

```shell
$ ls /home/ubuntu/lower1
a b

$ root@1ddd08a33c52:/# ls /app
a  b
```
- 컨테이너 내부에서 호스트 디렉터리와 동일한 데이터를 확인할 수 있었다

```shell
$ root@1ddd08a33c52:/# echo hi >> /app/hello

$ ls lower1
a  b  hello
```
- 볼륨과 마찬가지로 바인드 마운트도 컨테이너와 호스트가 동일한 디렉터리를 바라보게 된다
- 따라서 컨테이너에서 생성한 데이터를 호스트에서도 확인 가능하다

### 컨테이너에 데이터가 존재할 때
만약 컨테이너의 마운트 포인트에 이미 파일이 존재하면 어떻게 될까?

**볼륨**

볼륨을 사용할 경우, 컨테이너에 이미 데이터가 존재하면 도커가 해당 데이터들을 볼륨으로 복사해준다.

```shell
$ docker run -d -v nginx-volume:/usr/share/nginx/html nginx
```
- `nginx` 컨테이너에 볼륨을 마운트하여 실행하였다
- `nginx` 컨테이너의 `/usr/share/nginx/html` 경로에는 여러 html 파일들이 존재한다

```shell
$ docker run -it  -v nginx-volume:/root/ ubuntu

root@34b5eab7ed60:/# ls /root/
50x.html  index.html
```
- 동일한 볼륨을 마운트하여 다른 컨테이너를 실행하였다
- `nginx` 컨테이너의 파일들이 복사되었음을 확인할 수 있다.

**바인드 마운트**

바인드 마운트를 사용할 경우, 컨테이너에 이미 존재하던 데이터들은 호스트 디렉터리에 의해 덮어씌어진다.

```shell
$ docker run -d -it --mount type=bind,source=/tmp,target=/usr nginx

$ docker logs unruffled_wilber
exec /docker-entrypoint.sh: no such file or directory
```
- `nginx` 컨테이너의 `/usr` 디렉터리에 바인드 마운트를 수행하였다
- 바인드 마운트로 인해 실행에 필요한 파일이 사라져서 컨테이너가 종료되었다

## 참고자료
- docker, https://docs.docker.com/storage/
- 용찬호, 『시작하세요! 도커 / 쿠버네티스』, 위키북스(2020)

