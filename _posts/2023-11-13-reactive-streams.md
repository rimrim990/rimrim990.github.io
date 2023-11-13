---
layout: post
title: 리액티브 스트림
author: rimrim990
categories: [java]
tags: [java]
toc: true
---
자바 비동기, 논블로킹 프로그래밍을 위해 리액티브 스트림에 대해 알아보자!
<!--break-->

## 리액티브 스트림
리액티브 스트림은 Netflix, Pivotal, Lightbend 등의 기업들이 정의한 **비동기 스트림 처리**의 표준 스펙이다.
자바 버전 9에서부터 리액티브 스트림 원칙을 반영한 `Flow API`를 도입하면서, 리액티브 스트림은 자바의 공식 기능이 되었다.

### 리액티브 스트림이란?
> The purpose of Reactive Streams is to provide a standard for asynchronous stream processing with non-blocking backpressure.

공식 문서에 따르면, 리액티브 스트림은 **논블로킹 백프레셔**를 사용한 **비동기 스트림** 처리의 표준을 제공한다고 한다.

논블로킹 백프레셔? 비동기 스트림? 어려워 보이는 단어들이 잔뜩 나열되어 있다. 스트림과 백프레셔가 무엇인지 잠시 짚고 넘어가보자.

### 스트림

스트림은 이미 자바 8에서부터 존재했던 개념이다. 자바 스트림은 일반적인 데이터와 어떤 점이 다를까?

```java
// 자바 스트림 예시
Stream.iterate(0, i -> i + 1)
    .filter(i -> i % 2 == 1)
    .limit(20)
    .forEach(System.out::println);
```
- 스트림은 여러 값들의 시퀀스이다. 스트림을 구성하는 값들에는 순서가 존재한다.
- 스트림 파이프라인은 최종 연산이 정의되어야만 수행된다.
- 스트림의 원소들은 하나씩 순차적으로 처리된다.
예를 들어, 첫 번째 원소가 `filter`와 `limit` 파이프라인을 완료해야 두 번째 원소가 `filter`와 `limit`에서 처리될 수 있다.
- 스트림은 실제로 연산에 필요한 데이터만 처리한다.
예시에서는 `Stream.iterate`로 무한 스트림을 생성했지만, `limit`로 인해 20개의 원소만 메모리에 올라간다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/742b8f24-612a-4d62-a92a-4355df60c0c9">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

일반적인 데이터와 스트림을 비교한 그림이다.
만약 배열과 같은 자료구조를 사용했다면 모든 데이터가 우선 메모리에 올라가야 데이터를 처리할 수 있다.
수많은 데이터를 전부 메모리에 올린다면 `out of memory` 에러가 발생할 수도 있다.
혹은 `out of memory`가 발생하지 않더라도, 대량의 객체 생성으로 인해 `GC`가 수행되어 서버 서능에 영향을 줄 수 있다.

반면에 스트림은 실제로 데이터가 사용될 때 메모리에 올라가기 때문에, 많은 양의 데이터를 더욱 효율적으로 처리할 수 있다.

### 백프레셔

백 프레셔를 이해하기 위해 옵저버 패턴이 사용되는 상황을 가정해보자.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/5b913ab5-fb80-4df6-a6b7-9911cc2de9d5">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

옵저버 패턴에서는 발행자(`publisher`)가 구독자(`subscriber`)에게 데이터를 발행해준다. 구독자가 다음과 같은 과정을 통해 데이터를 전달받는다.
- 구독자가 발행자 객체를 콜백과 함께 구독한다
- 발행자는 데이터가 생성되면 모든 구독자들을 순회하며 등록된 콜백을 수행해준다

그런데 만약 발행자가 1초에 100개의 데이터를 전달하는데, 구독자는 1초에 10개 밖에 처리하지 못하면 어떻게 해아할까?
이런 상황에서는 큐와 같은 자료구조를 사용해 데이터를 저장해놔야 할 것이다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/a6dbd85f-191c-4e63-a2cf-1fab46c0594a">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

하지만 큐를 사용하더라도, 발행자가 큐 길이를 초과할 만큼 너무 많은 데이터를 전달하면 데이터가 드롭될 수 밖에 없다.

> This back-pressure is an important feedback mechanism that allows systems to gracefully respond to load rather than collapse under it.

발행자가 구독자의 **처리 능력을 넘어갈 만큼의 데이터를 전달**하는 상황을 방지하기 위해, **백프레셔**가 도입되었다.
발행자가 제한없이 구독자에게 데이터를 전달했던 것과 달리, 백프레셔 방식에서는 구독자가 발행자에게 **처리 가능한 데이터의 양**을 통지한다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/45e1e16c-e0e5-48ba-aea4-1635830520a2">
<figcaption style="text-align: center">[이미지 출처] Line Engineering</figcaption>
</figure>

- 구독자는 발행자에게 처리 가능한 만큼의 데이터를 요청한다.
- 발행자는 요청된 크기 내의 데이터를 전송해준다.

백프레셔처럼 데이터를 사용하는 측에서 필요한 데이터를 가져오는 방식을 **풀(pull) 방식**이라고 한다.
반면에 이전처럼 제공하는 측에서 데이터를 제한 없이 전달하는 방식을 **푸시(push) 방식**이라고 한다.

### 정리하기
이제 스트림과 백프레셔가 무엇인지 알게 되었다.
앞서 리액티브 스트림은 논블로킹 백프레셔를 사용해 비동기 스트림을 지원한다고 했었다.

**비동기**는 현재 스레드가 요청을 보낸 후, **응답을 기다리지 않고 다른 작업을 처리**할 수 있음을 의미한다.
이를 스트림에 적용해보면 아래와 같을 것이다.

```java
// 스트림 파이프라인 처리가 완료될 때까지 기다려야 한다
Stream.iterate(0, i -> i + 1)
    .filter(i -> i % 2 == 1)
    .limit(20)
    .forEach(System.out::println);
```
- 스트림 파이프라인 작업이 끝날 때까지 다른 작업을 처리할 수 없다
- 만약 비동기 스트림이었다면, 파이프라인이 처리되는 동안 다른 작업을 수행할 수 있을 것이다.
파이프라인 처리가 완료되면 콜백 메서드 등으로 작업이 완료되었음을 통지 받는다.

논블로킹은 요청된 **작업의 수행이 완료되었는지 여부와 관계없이 원래의 실행 흐름으로 돌아갈 수 있음**을 의미한다.
이를 구독자, 발행자 모델에 적용하면 아래와 같을 것이다.

<figure>
<img style="display: block; margin:auto; width: 50%" src="https://github.com/rimrim990/rimrim990.github.io/assets/62409503/55e0dea3-73a8-4732-af97-0ab207adc3eb">
</figure>

구독자는 발행자에게 수용 가능한 데이터를 통지한 후 다시 원래의 실행 흐름으로 돌아올 수 있다.
- 구독자는 발행자가 요청한 만큼의 데이터를 생성하는 동안 기다리지 않아도 된다

## 리액티브 스트림 API

### 리액티브 스트림 API의 구성 요소
리액티브 스트림 API에는 4개의 컴포넌트가 존재한다
- `Subscriber`
- `Publisher`
- `Subscription`
- `Processor`

리액티브 스트림의 근간은 옵저버 패턴이기 때문에, 발행자인 `Publisher`와 `Subscriber`가 구성 요소로 존재한다.

### Publisher
`Publisher`는 데이터 시퀀스를 보유하고 있는 컴포넌트이다. `Subscriber`의 요청을 받으면 데이터를 전달해준다.

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```
- 구독자는 `Publisher` 인터페이스의 `subscribe` 메서드를 수행하여 구독을 신청할 수 있다

리액티브 스트림에서는 `Publisher`의 `subscriber` 메세드는 반드시 아래와 같은 실행 흐름을 가져야 한다고 제한하고 있다.
```java
onSubscribe onNext* (onError | onComplete)?
```
- 한 번의 `s.onSubscribe` 메서드가 호출되어야 한다
- 0 번 이상의 `s.onNext` 메서드가 호출되어야 한다
- `s.onError` 혹은 `onComplete` 가 최대 한 번 수행되어야 한다

### Subscriber
`Subscriber`는 `Publisher`를 구독하고, 필요한 만큼의 데이터를 요청한다.

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```
- `Subscriber`가 특정 구독자 객체에 구독을 신청하면, `onSubscribe` 콜백을 호출받는다.
- 구독자는 `onNext` 콜백 호출로 데이터를 전달받는다.
- 구독자는 `onError` 콜백 호출로 데이터 처리 중 예외가 발생했음을 통지받는다.
- 구독자는 `onComplete` 콜백 호출로 데이터 전달이 완료되었음을 통지받는다.

리액티브 스트림은 옵저버 패턴과 다르게 폴 방식을 사용한다고 했었다.
그것 외에도 중요한 차이점이 더 존재하는데, 바로 `Subscriber` 인터페이스의 `onError`, `onComplete` 메서드이다.
- 옵저버 패턴에서는 데이터 전달이 완료되거나, 데이터 처리 중 예외가 발생했을 때 어떻게 대응할 것인지 언급하지 않는다.
- 반면에 리액티브 스트림은 `onError`와 `onComplete` 메서드를 인터페이스로 정의하고 있다

### Subscription
`Subscription`은 `Publisher`와 `Subscriber` 사이의 데이터 교환을 위해 정의된 인터페이스이다.
- `Publisher`와 `Subscriber`가 공유한다

```java
// Subscriber
public void onSubscribe(Subscription s);
```
- `Publisher`의 `subscribe`에서는 반드시 `onSubscribe`를 한 번 호출해야 한다
- `onSubscribe`는 인자로 `Subscription`을 받기 때문에, `Publisher`로부터 `Subscription`을 공유받게 된다.

`Subscription` 인터페이스에 정의된 메서드는 다음과 같다.
```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```
- `request` 메서드를 호출하여 발행자에 n 개의 데이터를 요청할 수 있다
- `cancel` 메서드를 호출하여 구독을 취소할 수 있다

`Subscription` 인터페이스 메서드는 구독자가 발행자의 데이터 생성을 조정하기 위해 사용되어야 한다.
- 따라서 구독자 객체 내부에서만 메서드가 호출되어야 한다.
- 또한 구독자 객체의 `onSubscribe` 메서드를 통해서만 전달되어야 한다.

## 참고자료
- reactive-streams, https://github.com/reactive-streams/reactive-streams-jvm
- Line Engineering, https://engineering.linecorp.com/ko/blog/reactive-streams-with-armeria-1
- Oracle, https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html
