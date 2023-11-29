---
layout: post
title: 인덱스로 조인 성능 개선하기
author: rimrim990
tags: [issue, mysql]
toc: true
---
조인 테이블에 인덱스를 추가해 쿼리 성능을 개선해보자!
<!--break-->

## 문제상황
### 문제의 쿼리
쇼핑몰 팀 프로젝트를 진행하면서 **키워드로 상품을 검색한 후 주문 많은 순으로 정렬하는 쿼리**를 작성하였다.
당시 작성한 쿼리는 아래와 같다.

```sql
SELECT p.* FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE p.name LIKE '%keyword%'
GROUP BY p.id
ORDER BY sum(oi.quantity) DESC
LIMIT 20;
```
- 상품명과 구매 수량을 얻기 위해 `products` 테이블과 `order_items` 테이블을 조인한다
- `like` 연산을 사용해 이름에 `keyword`가 들어간 상품을 검색한다

**DB 스키마**

쿼리 이해를 돕기 위해 DB 스키마의 일부를 가져왔다.
상품 상세 정보는 `products`에 저장되며, 구매 수량을 비롯한 구매한 상품에 대한 값은 `order_items`에 저장된다.

연관된 상품 간에 참조관계가 존재하긴 하나, 외래키 제약을 적용하진 않았다.

<figure style="margin-bottom: 10px">
<img src="https://github.com/rimrim990/TIL/assets/62409503/0eab5cd5-2ee3-472c-9176-ecf851f98032" width="500"/>
<figcaption style="text-align: center">데이터베이스 스키마 일부</figcaption>
</figure>

- 상품을 주문하면 주문 정보인 `orders`와 주문 상품인 `order_items` 레코드가 생성된다
- `order_itmes`는 `products`와 N:1 관계를 갖는다

**데이터 구성**

당시 DB에 존재하던 데이터 수량은 다음과 같았다.
- 1만여 건의 `products` 레코드
- 100만여 건의 `orders` 레코드
- 250만여 건의 `order_items` 레코드

각 상품마다 평균적으로 250개의 주문 상품 정보가 존재한다. 즉 다음과 같은 쿼리를 수행하면 250이 평균값으로 도출된다.
```sql
# 250.3262
SELECT AVG(a.cnt) AS avg
FROM
    (SELECT COUNT(oi.product_id) AS cnt
    FROM order_items oi
    GROUP BY oi.product_id) AS a;
```

상품별 주문 상품 개수의 최소값과 최대값은 다음과 같았다. **주문 상품의 수는 평균이 250인 정규 분표**를 갖는다.
```sql
# 192,312
SELECT MIN(a.cnt) AS min, MAX(a.cnt) AS max
FROM
    (SELECT COUNT(oi.product_id) AS cnt
    FROM order_items oi
    GROUP BY oi.product_id) AS a;
```

### 쿼리 성능 측정
이제 쿼리를 실행해보자. 사용자 A가 **상품명에 '니트'를 포함하는 상품**을 검색하는 상황을 가정해보자.

<figure style="margin-bottom: 10px">
<img width="806" src="https://github.com/rimrim990/TIL/assets/62409503/27ffec08-b22c-42b6-b772-2434d7c209ec">
<figcaption style="text-align: center">'니트'가 포함된 상품을 검색하는 쿼리</figcaption>
</figure>

데이터를 가져오는데 무려 11초나 소요되었다. 심지어 한 글자 키워드로 검색하면 더 오래 걸린다.

<figure style="margin-bottom: 10px">
<img width="806" src="https://github.com/rimrim990/TIL/assets/62409503/4ec0a959-6a57-41fa-93dd-d905ed32f7fa">
<figcaption style="text-align: center">'니'가 포함된 상품을 검색하는 쿼리</figcaption>
</figure>

상품명에 '니'가 포함된 데이터를 가져오는데는 1분 30초가 소요되었다. 상품을 검색하는데 1분 30초나 걸린다면 사용자가 모두 떠나갈 것이다.

## 쿼리 실행 계획 분석
**쿼리 실행계획**이이란 DBMS의 옵티마이저가 선택한 **최적의 쿼리 실행 방법**을 의미한다.
DBMS는 쿼리를 파싱하여 가능한 모든 실행 방법을 생성해 비교한 후, 최소 비용의 방법을 실행 계획으로 선택한다.

MySQL에서는 `EXPLAIN` 키워드를 추가해 실행 계획을 살펴볼 수 있다.

### 쿼리 실행 계획 확인
쿼리에 `EXPLAIN` 커맨드를 추가해 실행 계획을 출력해보았다.

```sql
EXPLAIN SELECT p.* FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE p.name LIKE '%니%'
GROUP BY p.id
ORDER BY sum(oi.quantity) DESC
LIMIT 20;
```

문제 쿼리의 실행 계획은 다음과 같았다.

| id  | select_type | table | type | key  | key_len | ref  | rows     | Extra                                           |
|-----|-------------|-------|------|------|---------|------|----------|-------------------------------------------------|
| 1   | SIMPLE      | p     | ALL  | null | null    | null | 9143     | Using where; Using temporary; Using filesort    |
| 1   | SIMPLE      | oi    | ALL  | null | null    | null | 20947799 | Using where; Using join buffer (flat, BNL join) |


### EXPLAIN 출력 형식
쿼리 실행 계획을 분석하기 위해 `EXPLAIN` 출력 형식을 필요한 것만 간략하게 살펴보자.

**출력 순서**

> EXPLAIN returns a row of information for each table used in the SELECT statement.
> It lists the tables in the output in the order that MySQL would read them while processing the statement.

`EXPLAIN`은 `SELECT` 쿼리 실행을 위해 **접급한 테이블**마다 **순차적으로** 행을 생성한다.

예를 들어, 앞선 쿼리 실행 계획에서는 `p` 테이블 다음에 `oi` 테이블 행이 존재했다.
이는 쿼리 수행을 위해 `products` 테이블에 먼저 접근한 다음, `order_items` 테이블에 접근했음을 의미한다.

**id 칼럼**

id 칼럼은 쿼리 내에서 실행된 `SELECT` 쿼리에 순차적으로 부여된 식별자 번호이다.
하나의 `SELECT`에서 여러 테이블을 조인하면 하나의 id 번호를 부여받는다.

**select_type 컬럼**

select_type 칼럼은 `SELECT`의 타입을 나타낸다.
select_type에는 다음과 같은 값들이 올 수 있다.

| 타입       | 설명                                              |
|----------|-------------------------------------------------|
| SIMPLE   | UNION이나 서브쿼리를 사용하지 않는 단순 SELECT                 |
| PRIMARY  | UNION이나 서브쿼리를 갖는 SELECT 쿼리의 가장 바깥쪽 SELECT       |
| UNION    | UNION 으로 결합된 SELECT에서 첫 번째를 제외한 두 번째 이후의 SELECT |
| SUBQUERY | FROM절 이외에서 사용되는 SUBQUERY의 SELECT                |

**type 칼럼**

tpye 칼럼은 조인 타입을 나타내는데, **테이블 접근 방식**이라고 볼 수 있다.
type에는 다음과 같은 값들이 올 수 있으며 **접근 성능이 빠른 순서대로 정렬**되어 있다.
- `ALL`을 제외한 모든 type은 인덱스를 사용해 접근한다

| 타입     | 설명                               |
|--------|----------------------------------|
| const  | 최대 1건의 데이터를 반환하는 접근 방식           |
| eq_ref | 조인시 프라이머리 혹은 유니크 키를 사용해 동등 조건 검색 |
| ref    | 조인시 인덱스를 사용해 동등 조건 검색            |
| range  | 인덱스 레인지 스캔을 사용하는 접근 방식           |
| index  | 인덱스 풀 스캔 접근 방식                   |
| ALL    | 풀 테이블 스캔 접근 방식                   |

**key 칼럼**

keys 칼럼은 옵티마이저가 쿼리 실행 계획에서 사용한 **인덱스**를 의미한다.

**rows 칼럼**

rows 칼럼은 쿼리 실행 계획을 수행하기 위해 얼마나 많은 레코드를 읽고 비교해야 하는지 옵티마이저가 예측한 비용이다.
옵티마이저가 통계 정보를 바탕으로 산출한 값이기 때문에 100% 정확하지는 않다.

**Extra**

Extra 칼럼은 MySQL이 어떻게 쿼리를 처리했는지 부가적인 정보를 제공한다.
Extra 칼럼에 아래와 같은 값들이 올 수 있다.

| 코멘트               | 설명                          |
|-------------------|-----------------------------|
| Using index       | 커버링 인덱스 사용                  |
| Using join buffer | 조인을 위한 적절한 인덱스가 없어 조인 버퍼 생성 |
| Using where       | 스토리지에서 읽어온 레코드를 필터링         |

### 쿼리 실행 계획 분석
쿼리 실행 계획에 어떤 값이 올 수 있는지 간단하게 살펴보았다.
이제 문제 쿼리의 실행 계획을 분석해보자.

| id  | select_type | table | type | key  | key_len | ref  | rows     | Extra                                           |
|-----|-------------|-------|------|------|---------|------|----------|-------------------------------------------------|
| 1   | SIMPLE      | p     | ALL  | null | null    | null | 9143     | Using where; Using temporary; Using filesort    |
| 1   | SIMPLE      | oi    | ALL  | null | null    | null | 20947799 | Using where; Using join buffer (flat, BNL join) |

- type 칼럼이 ALL 이므로 모든 테이블을 **풀 테이블 스캔** 했음을 알 수 있다
- key 칼럼이 null 이므로 모든 테이블에서 **인덱스를 사용하지 않았음**을 알 수 있다
- Extra 칼럼이 Using join buffer 키워드가 존재하므로 **조인을 위한 인덱스가 없어** 조인 버퍼를 생성했음을 알 수 있다.

## 쿼리 성능 개선
이제 쿼리 성능을 개선해보자.
Extra 칼럼의 **Using join buffer** 가 이번 성능 개선의 키 포인트다.
이를 이해하려면 **조인 쿼리가 내부적으로 어떻게 처리**되는지 알아야 한다.

### 조인 방식
MySQL은 테이블 조인을 위해 기본적으로 **Nested Loop Join** (NL) 알고리즘을 사용한다.
NL 알고리즘은 **이너 테이블에 인덱스**가 있을 때 효율적으로 실행되기 때문에, 마땅한 인덱스가 없을 경우 최적화를 위해 다른 조인 알고리즘을 사용한다.

**Nested Loop Join**

NL 조인 알고리즘은 **중첩 루프**를 돌며 조인을 처리한다.
첫 번째 테이블을 순회하면서 각 레코드에 대해 두 번째 조인 테이블 레코드를 순회한다.

```
for each row in t1 {
  for each row in t2 {
    if row satisfies join conditions, send to clint
  }
}
```
- 조인의 아우터 테이블인 t1에 N 개의 레코드가 존재한다고 가정하자
- t1의 모든 레코드는 한 번씩만 접근되는 반면에, t2의 모든 레코드는 N 번씩 접근된다

NL 조인에서 이너 조인 테이블은 아우터 테이블에 비해 더 많이 접근된다.
그렇기 때문에 빠른 쿼리 실행을 위해서는 **이너 테이블의 조인 컬럼**에 효율적으로 접근할 수 있어야 한다.

**인덱스의 필요성**

이너 테이블의 조인 컬럼에 효율적으로 접근하도록 **인덱스를 추가**할 수 있다.
인덱스를 추가하면 기존의 NL 조인 수행 과정이 다음과 같이 개선될 수 있다.

```
for each row in t1 {
  for each row in t2 matching reference key {
  }
}
```
- t1의 모든 레코드에 대해, 조인 조건을 만족하는 t2 레코드만 접근한다

인덱스가 없을 때는 어떤 레코드가 조인 조건을 만족하는지 알 수 없었기에 매번 테이블을 풀 스캔해야 했다.
그러나 인덱스가 존재하면 **조인 조건을 만족하는 레코드만** 더 **효율적으로** 찾아낼 수 있다.

### 인덱스 추가
이제 이너 테이블인 `order_items`에 인덱스를 추가해보자.

**단일 칼럼 인덱스 추가하기**

```sql
EXPLAIN SELECT p.* FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE p.name LIKE '%니%'
GROUP BY p.id
ORDER BY sum(oi.quantity) DESC
LIMIT 20;
```

문제의 쿼리를 다시 살펴보면, `order_items` 테이블은 `product_id` 칼럼을 조인 조건으로 사용하고 있다.
따라서 `product_id` 칼럼에 인덱스를 추가해야 한다.

`product_id` 칼럼에 인덱스를 추가하였다.
```sql
create index idx_products on order_items(product_id);
```

이제 쿼리 수행 시간을 다시 측정해보자.

<figure style="margin-bottom: 10px">
<img width="806" src="https://github.com/rimrim990/TIL/assets/62409503/b3cca52a-24de-4433-80cd-ad6a7d6f8887">
<figcaption style="text-align: center">'니'가 포함된 상품을 검색하는 쿼리</figcaption>
</figure>

와우! 무려 1분 30초나 소요되던 쿼리 수행 시간이 9초 300밀리초로 줄어들었다.
놀라운 변화지만 여전히 너무 느리다. 검색하는데 10초나 걸리는 쇼핑몰은 아무도 이용하지 않을 것이다.

어떻게 더 개선할 수 있을까?

**커버링 인덱스로 변경하기**

기존의 인덱스를 **커버링 인덱스**로 변경하여 쿼리 수행 시간을 더 단축할 수 있다.
문제의 쿼리를 살펴보면, `order_items` 레코드에서 사용되는 칼럼은 `product_id`와 `quantity` 밖에 없음을 알 수 있다.

```sql
EXPLAIN SELECT p.* FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE p.name LIKE '%니%'
GROUP BY p.id
ORDER BY sum(oi.quantity) DESC
LIMIT 20;
```
- `order_items` 테이블에서 `product_id`와 `quantity` 칼럼 값만 사용되고 있다

**커버링 인덱스**란 데이터 파일을 읽지 않고 **인덱스만으로 쿼리를 처리**할 수 있도록 구성된 인덱스이다.
인덱스 검색을 통해 일치하는 키 값들을 찾더라도, 데이터 파일에 접근해 실제 레코드 값을 가져와야 한다.
만약 커버링 인덱스를 사용한다면 **데이터 파일에 접근하는 시간을 절약**할 수 있다.

따라서 앞서 생성한 인덱스를 `product_id`와 `quantity`를 키로 하는 다중 인덱스가 되도록 수정해야 한다.
기존의 인덱스를 제거한 후 새로운 인덱스를 다시 생성해보자.
- 프로젝트에서 `product_id`와 `quantity`는 불변 값이었기에 인덱스를 생성하기 적절하다고 판단했다

```sql
drop index idx_products on order_items;
create index idx_products on order_items(product_id, quantity);
```
- `product_id`와 `quantity`를 키 값으로 사용하도록 수정하였다

커버링 인덱스를 사용한 쿼리의 수행 시간은 어떻게 개선되었을까? 측정해보자.

<figure style="margin-bottom: 10px">
<img width="806" src="https://github.com/rimrim990/TIL/assets/62409503/d58fdd51-efcb-4330-b1a4-eaba685c059d">
<figcaption style="text-align: center">'니'가 포함된 상품을 검색하는 쿼리</figcaption>
</figure>

와! 수행시간이 100밀리초로 대폭 감소하였다.
**조인 쿼리에서 인덱스의 중요성**을 정말 크게 체감할 수 있었다.

### 개선 결과 측정
인덱스를 추가하여 1분 30초가 걸리던 쿼리 수행 시간을 10초대로 개선하였다. 또한 기존 인덱스를 커버링 인덱스로 변경해 최종적으로 100밀리초로 단축할 수 있었다.

인덱스가 추가되고 문제 쿼리의 실행 계획은 다음과 같이 변했다.

| id  | select_type | table | type  | key          | key_len | ref           | rows | Extra        |
|-----|-------------|-------|-------|--------------|---------|---------------|------|--------------|
| 1   | SIMPLE      | p     | index | PRIMARY      | 8       | null          | 9143 | ...          |
| 1   | SIMPLE      | oi    | ref   | idx_products | 8       | shopping.p.id | 193  | Using index; |

- type 칼럼이 ref 이므로 `order_items` 테이블에 접근하기 위해 인덱스를 사용했음을 알 수 있다
- Extra 칼럼에 Using index 코멘트가 있으므로 커버링 인덱스를 사용했음을 알 수 있다

## 참고자료
- MySQL, https://dev.mysql.com/doc/refman/8.0/en/execution-plan-information.html
- 백은빈, 이성욱,  『Real My SQL 8.0 (1권)』, 위키북스(2021)
