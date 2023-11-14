---
layout: post
title: 자바 리액티브 프로그래밍
author: rimrim990
categories: [java]
tags: [java]
toc: true
---
자바 리액티브 프로그래밍에 대해 알아보자!

(2023.11.14에 수정됨)
<!--break-->

## 리액티브 프로그래밍
자바 9에 추가된 `java.util.concurrent.Flow` 클래스는 **리액티브 프로그래밍**을 지원한다.
리액티브 프로그래밍은 **리액티브 스트림 프로젝트 표준**에 따라 다양한 인터페이스를 제공한다.

### 리액티브 스트림
리액티브 프로그래밍이 사용하는 **리액티브 스트림**이란 무엇일까?

**리액티브 스트림의 목적**

> The purpose of Reactive Streams is to provide a standard for asynchronous stream processing with non-blocking backpressure.

리액티브 스트림은 **백프레셔** 방식을 사용하여, **스트림 데이터를 비동기적으로 처리**할 수 있는 표준 기술이다.

왜 리액티브 스트림은 백프레셔 방식을 도입하였을까?
옵저버 패턴을 통해 백프레셔가 무엇이며, 왜 필요한지 알아보자.

**용어 살펴보기**

리액티브 프로그래밍을 알아보기 앞서, **비동기**와 **논블로킹**이 무엇인지 정의하고 갈 필요가 있다.
리액티브 매니페스토 (reactive manifesto) 용어집에서는 이를 어떻게 정의하고 있을까?

`비동기 (Asynchronous)`
- A가 B로 요청을 전송했을 때, 요청의 처리가 전송 이후 임의의 시간에 처리됨을 의미한다
- A는 B가 언제 요청을 처리해줄지 모르기 때문에, B 내에서의 처리 과정을 직접 관찰할 수 없다
- B가 모든 요청을 처리해야 A가 다음 작업을 수행할 수 있는 **동기 작업 (Synchronous)** 과 반대되는 개념이다

`논블로킹 (Non-Blocking)`
- A가 B로 요청을 전송했을 때, B가 **요청 처리 여부와 관계없이 즉시 리소스를 반환**해줌을 의미한다
- 반환된 리소스에는 요청 처리의 결과, 혹은 아직 처리가 완료되지 않아 이용할 수 없음을 나타내는 정보가 담겨 있다
- A는 B의 요청 처리가 완료될 때까지 기다리기 보다는 **다른 작업을 진행**할 수 있다
- A는 B의 요청 처리가 끝났을 때 이를 **통지받을 수 있는 수단**을 등록할 수 있다. ex) 콜백 메서드

### 옵저버 패턴 알아보기

리액티브 스트림은 **옵저버 패턴**의 문제점을 보완하기 위해 백프레셔 방식을 도입하였다.
옵저버 패턴은 무엇일까?

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/5b913ab5-fb80-4df6-a6b7-9911cc2de9d5">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

옵저버 패턴에는 **메시지를 발행하는 발행자**와 이를 **구독하는 구독자**가 존재한다.
발행자는 특정 이벤트가 발생했을 때, 구독자들에게 이벤트 발생을 직접 통지해준다.
구독자는 메시지를 받기 위해 구독을 신청할 때, 발행자가 호출할 **콜백 메서드**를 함께 전달한다.

정리하면 구독자가 발행자에 구독을 신청했을 때, 발행자가 콜백을 호출해 비동기적으로 메시지를 전달해주는 구조이다.

### 옵저버 패턴의 문제점

옵저버 패턴에는 어떤 문제점이 있을까?

**첫 번째 문제점**

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/72e53286-f24e-4193-a283-000fb0a076ae">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

첫 번째로는, 발행자가 구독자가 감당할 수 없을 만큼의 메시지를 전송할 수 있다는 점이다.
- 예를 들어 발행자는 1초에 100개의 메시지를 전달하는데, 구독자는 1초에 10개 밖에 처리할 수 없다고 가정해보자
- 구독자는 처리하지 못한 메시지를 보관하기 위해 큐 같은 자료구조를 사용해야 할 것이다

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/a6dbd85f-191c-4e63-a2cf-1fab46c0594a">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

그런데 큐를 사용하더라도 발행자가 계속해서 메시지를 전송해 큐의 크기가 초과될 수도 있다.
- 큐에 쌓이지 못한 메시지는 유실될 것이다

이렇게 메시지를 발행하는 쪽에서 필요한 곳으로 전송해주는 방식을 `푸시 (push) 방식`이라고 한다.
푸시 방식에서는 구독자가 메시지 전송 속도를 감당할 수 없다는 문제점이 존재한다.

따라서 발행자가 메시지의 전송을 결정하는 게 아니라, 구독자로부터 **요청이 왔을 때만 메시지를 전송**하도록 제한할 필요가 있다.
이러한 방식을 `풀 (pull) 방식`이라고 한다.
백 프레셔는 구독자가 요청할 때만 메시지를 받을 수 있도록 풀 방식을 지원한다.

**두 번째 문제점**

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/72e53286-f24e-4193-a283-000fb0a076ae">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

발행자는 특정 이벤트가 발생했을 때 메시지를 전송한다.
그런데 구독자는 메시지의 전송이 전부 완료되었는지 어떻게 알 수 있을까?

혹은 발행자 측에서 메시지를 전송하던 중 예외가 발생해 콜백이 호출되지 못했다고 가정해보자.
그렇다면 구독자는 메시지를 정말로 보내지 않은 것인지, 예외로 인해 보내지 못한 것인지 어떻게 알 수 있을까?

옵저버 패턴에는 **완료나 예외 상황에 대한 처리 방식**이 정의되어 있지 않다.
반면에 리액티브 스트림은 메시지 전송 완료나 예외 상황에서 호출 할 수 있는 콜백 함수를 명시적으로 정의하고 있다.

### 리액티브 프로그래밍의 탄생

정리하면 옵저버 패턴에는 다음과 같은 문제점이 존재한다
- 발행자는 구독자가 처리 없을 만큼의 많은 메시지를 전달할 수도 있다
- 발행자의 메시지 전송 완료나 예외 상황에 대한 정의가 없다

리액티브 스트림은 이러한 문제점들을 보완하면서, 비동기적으로 데이터를 처리할 수 있는 표준을 정의하고 있다.
리액티브 스트림 표준을 따르는 `java.util.concurrent.Flow` 클래스를 살펴보면서, 리액티브 스트림이 어떻게 정의되어 있는지 알아보자.

## 자바 플로우 API

### 자바 플로우 API 구성 요소
`java.util.concurrent.Flow` 클래스에는 4개의 중첩 인터페이스가 정의되어 있다.
- `Subscriber`
- `Publisher`
- `Subscription`
- `Processor`

`Publisher`가 메시지를 발행하면, `Subscriber`는 메시지를 받아 소비한다.
- `Subscriber`는 `Publisher`에 이벤트가 발생했을 때 호출할 수 있도록 4가지 콜백 메서드를 정의하고 있다
- `Subscriber`는 `Publisher`에 메시지 발행을 요청할 수 있다

하나씩 차례대로 살펴보자.

**Publisher**
```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```
- `Subscriber`는 `Publisher`의 `subscribe` 메서드를 호출하여 구독을 신청할 수 있다

**Subscriber**

`Subscriber`는 `Publisher`에서 호출 가능한 4개의 콜백 메서드를 정의하고 있다.

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```
-
- `Subscriber`는 `Publisher`에 구독을 신청하면 `onSubscribe` 콜백을 호출받는다.
- `Subscriber`는 `onNext` 콜백 호출로 데이터를 전달받는다.
- `Subscriber`는 `onError` 콜백 호출로 데이터 처리 중 예외가 발생했음을 통지받는다.
- `Subscriber`는 `onComplete` 콜백 호출로 데이터 전달이 완료되었음을 통지받는다.

4개의 콜백 메서드는 리액티브 스트림에서 정의한 순서대로 호출되어야 한다

```
onSubscribe onNext* (onError | onComplete)?
```
- `onSubscribe`가 항상 제일 처음에 한 번 호출되어야 한다
- 이후 0번 이상의 `onNext`를 호출하여 메시지를 전송할 수 있다
- 모든 메시지 전송이 완료되면 `onComplete`를 호출해 종료를 알려야 한다
- 혹은 메시지 전송 중 예외가 발생하면, `onError`를 호출할 수 있다
- `onComplete`와 `onError` 중에서 하나의 메서드만 호출해야 하며, 이들은 아예 실행되지 않을 수도 있다

**Subscription**

`Subscription`을 통해 `Subscriber`는 `Publisher`에 메시지를 요청하거나, 더이상 메시지를 받지 않음을 통지할 수 있다.

```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```
- `Subscriber`는 `request` 메서드를 호출하여 `Publisher`에 n 개의 메시지 처리가 가능함을 전달할 수 있다
- `Subscriber`는 `cancel` 메서드를 호출하여 `Publisher`에 더이상 메시지를 받지 않음을 전달할 수 있다

### 자바 플로우 API의 제약사항
플로우 API에서는 `Flow` 클래스의 인터페이스를 어떻게 사용에 대해 다음과 같은 규칙을 정의하고 있다.

- `Publisher`는 반드시 `request` 메서드에 정의된 개수 이하의 요소만 `Subscriber`에 제공해야 한다
- `Subscriber`는 `request` 호출로 `Publisher`에 메시지를 받아 처리할 수 있음을 알려야 한다 (백프레셔)
- `Publisher`와 `Subscriber`는 `Subscription`을 공유해야 한다
- `onSubscribe`와 `onNext` 메서드에서는 `request` 메서드를 호출할 수 있어야 한다
- `cancel` 메서드는 여러 번 호출해도 한 번 호출한 것과 같은 효과를 가져야 한다

플로우 API를 구현한 애플리케이션의 생명 주기는 아래와 같다.

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/46119ca8-10fa-474a-be74-81a5537d2529">
</figure>

## 참고자료
- 토비의 봄 TV, https://www.youtube.com/watch?v=8fenTR3KOJo&list=PLOLeoJ50I1kkqC4FuEztT__3xKSfR2fpw&index=1
- 라울-게이브리얼 우르마, 『모던 자바 인 액션』, 한빛미디어(2019)
- reactive-streams, https://github.com/reactive-streams/reactive-streams-jvm
- Line Engineering, https://engineering.linecorp.com/ko/blog/reactive-streams-with-armeria-1
- Oracle, https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html
