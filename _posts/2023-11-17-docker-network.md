---
layout: post
title: 도커 네트워크
author: rimrim990
categories: [infra]
tags: [docker]
toc: true
---
도커 네트워크에 대해 알아보자!
<!--break-->

## 도커 네트워크 이해하기
### 네트워크 설정 살펴보기
도커 컨테이너 내부에서 인터넷에 접속하고 싶다면 어떻게 해야할까?
혹은 컨테이너 간에 네트워크로 연결하고 싶다면 무엇을 설정해줘야 할까?

**컨테이너 네트워크 구성 정보 조회하기**

컨테이너 내부에서 `ipconfig`를 실행해 네트워크 구성 정보를 확인해보자.
```shell
root@c9fcde26af18:/# ifconfig

# 출력 정보
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
```
- 조회 결과 루프백 인터페이스 외에도 `eth0`라는 네트워크 인터페이스가 존재했다
- `eth0` 인터페이스에 할당된 IP 주소는 `172.17.0.3/16` 이었다

이번에는 `ip route`를 실행해 컨테이너에서 보낸 패킷이 어떤 경로로 라우팅되는지 조회해보자.

```shell
root@c9fcde26af18:/# ip route

# 출력 정보
default via 172.17.0.1 dev eth0
```
- 컨테이너에서 전송한 패킷은 `default` 인터페이스인 `eth0`로 보내진다
- `eth0`로 보내진 패킷은 `172.17.0.1`로 라우팅된다

그렇다면 `172.17.0.1`은 어디일까?

**호스트 네트워크 구성 정보 조회하기**
```shell
$ ifconfig

# 출력 정보
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
```
- `172.17.0.1`는 호스트의 `docker0` 인터페이스에 할당된 IP 주소였다

이번에는 호스트의 라우팅 정보를 조회해보자.
```shell
$ route

Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         ip-xx-xx-xx-x. 0.0.0.0         UG    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
...
```
- 호스트에서 `172.17.0.0/16` 서브넷으로 보내는 모든 패킷은 `docker0` 인터페이스로 보내진다
- 외부 네트워크로 전송하는 모든 패킷들은 `eth0` 인터페이스를 거쳐 `ip-xx-xx-xx-x` 게이트웨이로 라우팅 된다

**정리하기**

지금까지 조회한 내용을 정리하면 아래와 같다.
```
인터넷 <- (eth0) 호스트 (docker0) <- (eth0) 컨테이너
```
- 컨테이너에서 보낸 패킷은 호스트의 `docker0`로 라우팅 된다
- `docker0`로 라우팅된 컨테이너 패킷은 호스트의 네트워크 인터페이스를 통해 외부 인터넷으로 전달된다

```
호스트 (docker0) -> (eth0) 컨테이너
```
- 호스트에서 컨테이너로 보낸 패킷은 `docker0` 인터페이스를 통해 컨테이너로 전달된다

### 네트워크 실험해보기
`ping`으로 패킷을 보내 앞서 세운 가설들이 맞는지 검증해보자!

**컨테이너에서 호스트로 패킷 전송하기**
```shell
root@c9fcde26af18:/# ping 172.17.0.1

# 출력 결과
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.066 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.061 ms
```
- 컨테이너에서 호스트로 패킷 전송에 성공하였다

**컨테이너에서 외부로 패킷 전송하기**
```shell
root@c9fcde26af18:/# ping 8.8.8.8

# 출력 결과
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=0.892 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=0.881 ms
...
```
- 컨테이너에서 외부 네트워크로 패킷 전송에 성공하였다

**호스트에서 컨테이너로 패킷 전송하기**
```shell
$ ping 172.17.0.3

# 출력 결과
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.067 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.066 ms
```
- 호스트에서 컨테이너로 패킷 전송에 성공하였다

### NAT 설정 살펴보기

```shell
# 컨테이너는외부 네트워크로부터 응답을 받았다
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=0.892 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=0.881 ms
```

그런데 실험을 진행하면서 한 가지 의문이 드는 점이 있다.
도커 컨테이너는 `172.17.0.3`의 **사설 IP 주소**를 사용하고 있음에도 외부 네트워크에 `ping`을 보냈을 때 응답을 받을 수 있었다.
어떻게 가능했던 것일까?

이것이 가능했던 이유는 **호스트에서 `NAT` 설정을 통해 사설 IP 주소로 변환**을 수행했기 때문이다.

```shell
$ iptables -nL -t nat

# 출력 결과
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
```
- 호스트에서 `NAT` 테이블을 조회해보니 `POSTROUTING` 체인에 위와 같은 규칙이 설정되어 있었다
- `172.17.0.0/16` 서브넷에서 전송하는 모든 패킷의 소스 IP 주소를 목적지에 상관없이 변경한다

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/b25c5875-4924-4124-b418-237a5cfabe41">
<figcaption style="text-align: center">[이미지 출처] alice_k106 블로그</figcaption>
</figure>

- `POSTROUTING` 체인의 `MASQUERADE` 규칙은 패킷의 줄발지 주소를 변경한다
- `POSTROUING`에 정의된 룰에 의해 컨테이너에서 보낸 패킷의 사설 IP 주소가 공인 IP로 변경된다
- 외부 네트워크에서는 컨테이너의 사설 IP 주소가 아닌 공인 IP 주소로 응답을 전송한다

## 도커 네트워크 구조

지금까지 도커 컨테이너의 네트워크 설정 정보를 간략하게 살펴보았다.
이번에는 컨테이너 네트워크가 어떤 구조로 구성되어 있는지 알아보도록 하자.

### 컨테이너 네트워크 구조

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/45d2ed61-f129-4365-923c-89da72f44736">
</figure>

도커 네트워크는 크게 `docker0` 브릿지와 컨테이너의 `eth0`로 구성되어 있다.

**Docker0 인터페이스**
```shell
$ ifconfig

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255

```
- `docker0` 인터페이스는 도커를 설치할 때 호스트에 자동적으로 설정된다
- 도커는 컨테이너에 `172.17.0.0/16`의 IP를 순차적으로 할당하는데, `docker0`에 `172.17.0.1`을 부여한다

```shell
$ brctl show docker0
 brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242e2661c51       no              veth14cf085
                                                        vethb9f94ed
                                                        vethdff8406

```
- `docker0`는 일반적인 가상 네트워크 인터페이스가 아닌 가상 브릿지이다
- L2 계층의 브릿지와 같은 역할을 한다. 즉, 컨테이너 네트워크 인터페이스들을 서로 연결해준다

**veth와 eth0**
```shell
# 호스트
$ ifconfig
vethdff8406: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::980d:fbff:fee7:3bab  prefixlen 64  scopeid 0x20<link>

# 컨테이너
root@c9fcde26af18:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.4  netmask 255.255.0.0  broadcast 172.17.255.255
```
- 컨테이너를 생성하면 도커에서 자동으로 `eth0` 네트워크 인터페이스를 생성한다
- `eth0`와 연결된 `veth` 네트워크 인터페이스를 호스트 네트워크에도 생성한다
- `veth` 인터페이스는 `docker0` 브릿지에 연결되어 있다

**정리하기**

정리하자면, 도커는 컨테이너 생성 시 호스트와 컨테이너에 네트워크 인터페이스를 생성한다.
두 개의 인터페이스는 서로 쌍을 이루며 연결되어 있다.

연결된 네트워크 인터페이스를 통해 호스트의 `docker0` 브릿지로 패킷이 전달된다. 컨테이너는 이를 통해 외부 네트워크 혹은 다른 컨테이너에 패킷을 전송할 수 있다.

지금까지 서술한 내용들은 기본 브릿지 네트워크 사용을 전제로 한다.
별다른 설정을 하지 않았다면 컨테이너는 `docker0` 브릿지를 사용해 네트워크를 구성한다.

### 포트 포워딩

지금까지 여러 내용들을 다루었는데, 한 가지 지나친 것이 있다.
외부 네트워크에서는 어떻게 컨테이너에 접근할 수 있을까?

컨테이너에서 나가는 패킷은 `NAT` 주소 변환을 통해 호스트의 공인 IP로 변환되었다.
따라서 외부에서 응답을 보낼 수 있었다.

```shell
$ iptables -nL -t nat

# 출력 결과
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
```
- 앞서 `POSTROUTING` 체인 규칙에 의해 컨테이너에서 나가는 패킷은 `NAT`이 적용됨을 알아보았다

그러나 외부 네트워크에서 먼저 요청을 보내고 싶다면 어떻게 해야할까?
이럴 때는 포트포워딩을 설정해주어야 한다.

```shell
# 포트포워딩 설정
$ docker run -p 80:1234 ubuntu
```
- 컨테이너를 실행 할 때 `p` 옵션으로 포트포워딩을 설정할 수 있다
- 호스트의 80번 포트를 컨테이너의 1234번 포트로 포워딩해주었다

포트 포워딩을 적용하여 호스트의 특정 포트를 컨테이너 포트와 연결해줄 수 있다.
포트 포워딩을 설정하면 아래와 같이 호스트 네트워크에 `NAT` 규칙이 추가된다.

```shell
$ iptables -nL -t nat

# 출력 내용
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

# PREROUTING에 연결되어 있음
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.4:1234
```
- `PREROUTING` 체인의 `DANT` 규칙에 의해 외부에서 들어온 패킷의 목적지가 변환된다
- 패킷의 목적지가 호스트의 `80`번 포트라면, 이를 컨테이너의 `1234`번 포트로 변환한다

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/d002a1db-f943-4faf-b1d6-22388845f832">
<figcaption style="text-align: center">[이미지 출처] alice_k106 블로그</figcaption>
</figure>

- 외부 네트워크에서 호스트의 특정 포트로 요청을 전송한다
- 호스트 포트로 패킷이 들어오면, 패킷의 목적지는 `DNAT` 규칙에 의해 컨테이너의 사설 IP 주소로 변환된다

## 참고자료
- 용찬호, 『시작하세요! 도커 / 쿠버네티스』, 위키북스(2020)
- docker, https://docs.docker.com/network/
- alice_k106, https://blog.naver.com/alice_k106/221305928714


