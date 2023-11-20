---
layout: post
title: 스프링 UnexpectedRollbackException 해결하기
author: rimrim990
tags: [issue, spring]
toc: true
---
2023.07월 우아한 테크캠프 6기 미션을 수행하면서 마주한 `UnexpectedRollbackException` 해결 과정을 정리한 글입니다.
<!--break-->

## UnexpectedRolblackException

### 읭? 이게 뭐지!

**예외 로그**

우아한 테크캠프 4주차 쇼핑몰 미션을 진행하던 중, 다음과 같은 예외를 마주하였다.

```
org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:752)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:711)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:654)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:407)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
```

새내기 스프링 개발자에게는 낯설기만 한 `UnexpectedRollbackException`이라는 예외였다.
예외 메시지에 **트랜잭션** 관련 언급이 있는 것으로 보아, 트랜잭션의 잘못된 사용으로 발생한 예외인 것 같다.

**프로젝트 코드**

당시 예외가 던져진 코드는 다음과 같았다.

```java

@Service
@Transactional
public class CartProductService {

  private CartProduct findAndUpdateCartProduct(Member member, Product product) {
    try {
      CartProduct cartProduct = cartProductRepository.findOneByMemberIdAndProductId(member.getId(),
        product.getId());
      return cartProduct.increaseQuantity();
    } catch (CartException e) {
      return new CartProduct(member, product, DEFAULT_PRODUCT_QUANTITY);
    }
  }
}
```

- `CartProductService` 의 `findAndUpateCartProduct` 메서드에서 예외가 던져졌다
- `CartProductRepository`에서 장바구니 상품을 조회한 후, 상품이 존재하면 수량을 증가하여 반환한다
- 만약 장바구니 상품이 존재하지 않아 `CartException`이 던져지면 새로운 상품을 생성해 반환한다

```java
public class CartProductRepository {

  @Transactional(readOnly = true)
  public CartProduct findOneByMemberIdAndProductId(Long memberId, Long productId) {
    try {
      return entityManager.createQuery(
          "select c from CartProduct c where c.member.id = :memberId and c.product.id = :productId",
          CartProduct.class)
        .setParameter("productId", productId)
        .setParameter("memberId", memberId)
        // 조회 결과가 없으면 NoResultException 발생!
        .getSingleResult();
    } catch (NoResultException e) {
      throw new CartException(e.getMessage());
    }
  }
}
```

- `CartProductRepository`의 `findOneByMemberIdAndProductId` 메서드는 위와 같이 구현되었다
- 쿼리 조회 결과가 존재하지 않으면 `getSingleResult` 메서드에서 `NoResultException`를 던지는데, 이를 `CarException`으로 포장하여 던진다

### 어떻게 해결하지?

**일단 수정해보기**

예외 로그에서 트랜잭션을 언급했기 때문에, 트랜잭션 애너테이션을 적절히 조정하면 해결할 수 있을 것 같았다.
당시 `CartProductService`와 `CartProductRepository` 클래스 모두 `@Transaction` 애너테이션이 붙어있었다.

예측대로 `Repository`에 부착한 트랜잭션을 제거해보니 예외가 발생하지 않았다.

**이걸로 끝?**

생각보다 간단하게 문제를 해결할 수 있었다.
그런데 왜 이런 예외가 발생했는지 의문이 들었고, 왜 트랜잭션을 제거하면 해결이 되었는지도 궁금했다.

당시의 나는 캠프 코치님의 조언에 감명받아 **디버깅과 삽질**에 푹 빠져있었기 때문에, **트랜잭션 코드를 디버깅**하면서 직접 **내부 동작 원리**를 파보기로 결심하였다.
그리하여 이틀 간의 스프링 트랜잭션 공식 문서, 디버깅과의 전쟁이 시작되었다!

## 스프링 트랜잭션 디버깅

### 예외 발생 지점 추적하기

예외가 발생했을 당시 에러 로그는 아래와 같았다. 에러 로그를 참고하여 메서드 호출 스택을 따라가보자.

```java
org.springframework.transaction.UnexpectedRollbackException:Transaction silently rolled back because it has been marked as rollback-only
  at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:752)
  at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:711)
  at org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:654)
  at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:407)
  at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
```

**processCommit**
```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager,
  Serializable {

  private void processCommit(DefaultTransactionStatus status) throws TransactionException {

    // Throw UnexpectedRollbackException if we have a global rollback-only
    // marker but still didn't get a corresponding exception from commit.
    if (unexpectedRollback) {
      throw new UnexpectedRollbackException(
        "Transaction silently rolled back because it has been marked as rollback-only");
    }
  }
}
```
- `UnexpectedRollbackException`은 `processCommit` 메서드에서 던져졌다
- `unexpectedRollback`의 값이 참이면 예외를 던진다

`unexpectedRollback` 값은 어떤 상황에서 참이 되는 것일까?

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {

  private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    if (status.hasSavepoint()) {
      unexpectedRollback = status.isGlobalRollbackOnly();
    } else if (status.isNewTransaction()) {
      unexpectedRollback = status.isGlobalRollbackOnly();
    } else if (isFailEarlyOnGlobalRollbackOnly()) {
      unexpectedRollback = status.isGlobalRollbackOnly();
    }
  }
}
```
- `processCommit` 메서드를 거슬러 올라가보면 `unexpectedRollback` 값은 `status.isGloblRollbackOnly()`에 의해 결정된다

**isGlobalRollbackOnly**

```java
public class DefaultTransactionStatus extends AbstractTransactionStatus {
  @Override
  public boolean isGlobalRollbackOnly() {
    return (this.transaction instanceof SmartTransactionObject smartTransactionObject &&
      smartTransactionObject.isRollbackOnly());
  }
}
```
- `isGlobalRollbackOnly`는 트랜잭션 상태가 `rollbackOnly`로 마킹된 경우에 참을 반환한다

어딘가에서 내가 실행한 트랜잭션이 `rollbackOnly`로 마킹되었나 보다.
그럼 어느 시점에 `rollbackOnly`를 마킹했을까? 이를 추적하면 문제 해결에 꽤 큰 실마리가 될 것 같다.
관련 코드를 찾기 위해 무한 디버깅을 시도해보았다.

**CartProductRepository**

`rollbackOnly` 마킹은 전혀 예상치 못한 부분에서 일어났다.
결론부터 말하자면 `CartProductRepository`에서 던진 예외로 인해 발생한 것이었다.

```java
public class CartProductRepository {

  @Transactional(readOnly = true)
  public CartProduct findOneByMemberIdAndProductId(Long memberId, Long productId) {
    try {
        // 예외 발생
    } catch (NoResultException e) {
      throw new CartException(e.getMessage());
    }
  }
}
```
- `CartProductRepository`에서 쿼리를 수행했으나 결과가 존재하지 않아 `NoResultException`이 던져진다
- `NoResultException` 예외를 잡아 `CartException`으로 던진다

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
  private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    boolean unexpectedRollback = unexpected;

    if (status.hasTransaction()) {
        if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
            if (status.isDebug()) {
                logger.debug(
                    "Participating transaction failed - marking existing transaction as rollback-only");
            }
            doSetRollbackOnly(status);
        }
    }
  }
}
```
- 트랜잭션에서 예외가 발생했기 때문에 해당 트랜잭션은 롤백된다
- 롤백을 처리하는 `processRollback` 메서드에서 `doSetRollbackOnly`를 호출해 `rollbackOnly`를 참으로 마킹한다

**CartProductService**
```java
@Service
@Transactional
public class CartProductService {

  private CartProduct findAndUpdateCartProduct(Member member, Product product) {
    try {
      ...
    } catch (CartException e) {
      return new CartProduct(member, product, DEFAULT_PRODUCT_QUANTITY);
    }
  }
}
```
- `CartProductRepository`에서 예외를 던졌지만 `catch` 블록으로 잡아 정상 동작으로 처리하였다
- 따라서 `CartProductService`의 트랜잭션은 롤백되지 않고 정상적으로 커밋된다

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
  private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    unexpectedRollback = status.isGlobalRollbackOnly();
    doCommit(status);

    if (unexpectedRollback) {
      throw new UnexpectedRollbackException(
        "Transaction silently rolled back because it has been marked as rollback-only");
    }
  }
}
```
- `CartProductService`에서 트랜잭션을 커밋하기 위해 `processCommit` 메서드를 호출한다
- 트랜잭션을 커밋하던 중 `CartProductRepository`에서 설정한 `rollbackOnly` 마킹으로 인해 `unexpectedRollback`의 값이 참이 된다
- `UnexpectedRollbackException` 예외가 던져진다

### 정리하기
지금까지 디버깅한 내용을 정리하면, `UnexpectedRollbackException`은 `CartProductRepository`에서 던진 예외로 인해 발생한 것이다.
- `CartProductRepository`에서 던진 예외로 인해 트랜잭션이 롤백한다
- `CartProductReopsitory`의 트랜잭션을 롤백하는 과정에서 `rollbackOnly` 값을 참으로 마킹한다
- `CartProductService`를 커밋하는 과정에서 참으로 마킹된 `rollbackOnly`에 의해 `UnexpectedRollbackException`이 던져진다

코드를 디버깅하면서 어떤 흐름으로 예외가 던져졌는지 알게 되었다. 그런데 왜 `CartProductRepository`가 롤백했는데 `CartProductService`도 롤백시킨 것일까?
도무지 알 수가 없다. 이럴 때가 공식 문서가 필요한 시점이다! 스프링 문서를 살펴보면서 자세한 배경을 알아보자.

## 트랜잭션 전파속성
스프링에서 트랜잭션 관련 설명을 읽던 중, 트랜잭션 전파 속성이라는 개념을 발견하였다.

### 트랜잭션 전파속성이 뭐지?
트랜잭션 전파속성은 중첩된 트랜잭션이 생성되었을 때 이를 어떻게 관리할지 정의하는 방식이다.
`@Trasaction` 애너테이션에 전파 속성을 설정할 수 있다.
- 별도로 설정하지 않으면 `PROPAGATION_REQUIRED`가 기본 값으로 사용된다

스프링에서는 여러 가지 트랜잭션 전파속성을 제공하지만, 대표적으로 두 가지 속성만 살펴보도록 하겠다.

**PROPAGATION_REQUIRED**

스프링에서는 `@Transaction` 애너테이션으로 생성된 트랜잭션들을 **물리적 트랜잭션**과 **논리적 트랜잭션**으로 구분한다.
- 물리적 트랜잭션은 실제로 데이터베이스 트랜잭션을 시작한 진짜 트랜잭션이다
- 논리적 트랜잭션은 개념적인 트랜잭션으로, 기존에 생성된 물리적 트랜잭션에 참여한다
- 논리적 트랜잭션은 물리적 트랜잭션에 참여한 개념적인 트랜잭션이기 때문에, 물리적 트랜잭션의 설정 값들을 그대로 상속 받는다

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://camo.githubusercontent.com/0a1b949dbcdc28d07b9ebe56f3a9d5180ce040ac33b86ac811d871e565d9d8ad/68747470733a2f2f646f63732e737072696e672e696f2f737072696e672d6672616d65776f726b2f7265666572656e63652f5f696d616765732f74785f70726f705f72657175697265642e706e67">
<figcaption style="text-align: center">[이미지 출처] Spring Docs </figcaption>
</figure>

- `PROPAGATION_REQUIRED`에서는 기존에 트랜잭션이 존재하지 않으면 물리적 트랜잭션을 생성한다
- 이후에 중첩적으로 생성되는 모든 트랜잭션은 기존 트랜잭션에 참여하는 논리적 트랜잭션들이다
- 중첩된 논리적 트랜잭션들은 물리적 트랜잭션의 타임아웃이나 격리수준 등의 설정 값을 상속 받는다

**PROPAGATION_REQUIRES_NEW**

<figure>
<img style="display: block; margin:auto; width: 60%" src="https://camo.githubusercontent.com/6b362779192ecb2016972fca8240574fc95602632afc5b76464ed1af166a28a3/68747470733a2f2f646f63732e737072696e672e696f2f737072696e672d6672616d65776f726b2f7265666572656e63652f5f696d616765732f74785f70726f705f72657175697265735f6e65772e706e67">
<figcaption style="text-align: center">[이미지 출처] Spring Docs </figcaption>
</figure>

- `PROPAGATION_REQUIRES_NEW` 에서는 항상 새로운 물리적 트랜잭션을 생성한다
- 기존에 실행되던 트랜잭션은 잠시 중단되고, 새로운 트랜잭션이 종료되면 다시 재개된다
- 생성된 트랜잭션들은 모두 독립적인 설정 값을들 갖는다

### PROPAGATION_REQUIRED

**원인 찾기**

그렇다면 트랜잭션 전파 속성이랑 `UnexpectedRollbackException`이랑 무슨 연관이 있는 걸까? 스프링 문서에는 다음과 같은 설명이 있다.

> However, in the case where an inner transaction scope sets the rollback-only marker,
> the outer transaction has not decided on the rollback itself, so the rollback (silently triggered by the inner transaction scope) is unexpected.
> A corresponding UnexpectedRollbackException is thrown at that point.

- `PROPAGATION_REQUIRED` 전파 속성을 사용할 경우, 외부 트랜잭션은 내부 트랜잭션의 롤백으로 인해 의도치 않게 롤백될 수 있다
- 외부 트랜잭션은 롤백 마커에 의해 의도치 않은 롤백이 수행되었기 때문에 `UnexpectedRollbackException`을 던진다

`PROPAGATION_REQUIRED` 전파 속성에서는 중첩해서 생성된 모든 논리적 트랜잭션을 하나의 물리적 트랜잭션으로 매핑한다.
그렇기에 내부 트랜잭션이 롤백되면 하나의 물리적 트랜잭션을 공유하는 외부 트랜잭션도 롤백 시키는 전략을 취한다.

**정리하기**

`CartProductRepository`의 `@Transaction` 애너테이션에 별도의 설정을 하지 않았기 때문에 `PROPAGATION_REQUIRED` 전파 속성에 의해 트랜잭션이 생성된다.
따라서 `CartProductService`와 `CartProductRepository`는 동일한 물리적 트랜잭션을 공유한다.

`CartProductRepository`에서 런타임 예외가 발생해 트랜잭션이 롤백했기 때문에, 동일한 트랜잭션을 공유하는 `CartProductService`도 롤백된다.

## 문제 해결하기
이제 문제의 원인을 자세히 알게 되었으니, 정확한 해결 방법을 찾아보자!

### 이해하기
우선 왜 앞서 `@Transaction` 애너테이션을 제거했을 때 예외가 발생하지 않았는지 알게 되었다.

> 예측대로 `Repository`에 부착한 트랜잭션을 제거해보니 예외가 발생하지 않았다.

- `Service` 계층에만 `@Transaction` 애너테이션이 남게 된다
- `Service`에서는 `Repository` 계층에서 발생한 예외를 핸들링 했기 때문에, 정상적으로 커밋된다

### 해결 방법
**불필요한 애너테이션 제거하기**

가장 적절한 해결 방법은 `Repository` 계층에 부착한 트랜잭션을 제거하는 것이다.
트랜잭션의 동작 방식을 적절히 이해하지 못하고 불필요하게 모든 계층에 `@Transaction`을 부착해버렸다.
현재 프로젝트에서는 `Repository` 계층에서 중첩된 트랜잭션을 생성할 이유가 없다.

**noRollbackFor 설정하기**

최적의 방법은 아니라고 생각하지만, `noRollbackFor` 속성을 설정하여 문제를 해결할 수도 있다.
스프링 트랜잭션은 기본적으로 모든 런타임 예외에 대해서 롤백을 수행한다.
따라서 `noRollbackFor` 속성을 설정하여 특정 런타임 예외가 발생해도 롤백하지 않도록 수정할 수 있다.

```java
@Transactional(readOnly = true, noRollbackFor = {CartException.class})
```
- `findOneByMemberIdAndProductId` 메서드에서 `CartException`이 던져져도 트랜잭션이 롤백하지 않도록 수정한다

**메서드 로직 수정하기**

또 다른 해결책으로는 메서드 로직을 수정하는 방식이 있다.
`findOneByMemberIdAndProductId`에서 데이터베이스 조회 결과가 존재하지 않을 때 꼭 예외를 던질 필요가 있었을까?

예외는 진짜 예외 상황에서만 사용해야 한다고 생각하는데, 지금 봤을 때 데이터베이스 조회 결과가 존재하지 않는 건 예외 상황이 아닌 것 같다.
이를 어떻게 바라보는지 관점에 따라 달라질 수 있겠지만, `findOneByMemberIdAndProductId` 메서드에서 예외를 던지지 않도록 수정하는 것도 한 가지 방법이 될 수 있을 것 같다.

## 참고 자료
- Spring Docs, https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html
