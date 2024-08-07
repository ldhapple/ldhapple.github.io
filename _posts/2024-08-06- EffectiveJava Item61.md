---
title: Item61 (박싱된 기본 타입보다는 기본 타입을 사용하라)
author: leedohyun
date: 2024-08-06 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 박싱된 기본 타입보다는 기본 타입을 사용하라

Java의 데이터 타입에는 기본 타입과 참조 타입이 있다.

각각의 기본 타입에 대응하는 참조 타입이 박싱된 기본 타입이라고 불린다.

우리가 무심코 사용하게 되는 Integer, Long, Double 등이 그 예이다.

이러한 박싱된 기본 타입과 기본 타입간에는 오토박싱과 오토언박싱이 이루어져 사용에는 크게 차이가 없을 수 있다.

하지만 이 둘은 분명한 차이가 있다. 차이를 파악하고 경우에 맞게 사용하는 것이 중요하다.

### 기본 타입과 박싱된 기본 타입의 차이

- 기본 타입은 값만 갖지만, 박싱된 기본 타입은 식별성을 갖는다.
- 기본 타입의 값은 언제나 유효하지만 박싱된 기본 타입은 유효하지 않은 값을 가질 수 있다.
- 기본 타입이 박싱된 기본 타입보다 시간과 메모리 사용면에서 더 효율적이다.

#### 박싱된 기본 타입은 값 + 식별성을 갖는다.

```java
@Test  
void boxingTypeTest() {  
  Integer int1 = Integer.valueOf(127);  
  Integer int2 = Integer.valueOf(127);  
  
  Assertions.assertThat(int1 == int2).isTrue();  
}
```
```java
@Test  
void boxingTypeTest2() {  
  Integer int1 = Integer.valueOf(128);  
  Integer int2 = Integer.valueOf(128);  
  
  Assertions.assertThat(int1 == int2).isTrue();  
}
```

- 첫 번째 테스트는 통과하지만 두 번째 테스트는 통과하지 못한다. 이유가 뭘까?
- 박싱된 기본 타입은 식별성을 갖는다고 했다.
- 이 말의 뜻은 '==' 연산자로 비교할 때 값이 같은지만을 따지는 것이 아닌, 같은 객체인지도 따지는 것이다.
- Integer.valueOf() 메서드는 성능 최적화를 위해 valueOf 팩토리 메서드에서 작은 범위의 값을 캐싱한다.
	- -128부터 127까지의 값을 캐싱한다.
- 따라서 이 범위의 값인 127은 같은 객체를 반환하기 때문에 True를 반환하지만, 128은 범위를 벗어나 다른 객체가 반환되어 False를 반환하게 되는 것이다.

***종합적으로 박싱된 기본 타입은 식별성을 가져 기본 타입과는 다르게 같은 값이라도 서로 다르다고 식별될 수 있다.***

#### 박싱된 기본 타입은 null을 가질 수 있다.

기본 타입에는 null을 넣을 수 없다. 항상 초기화되어야 한다. 

하지만 박싱된 기본 타입은 참조 타입이기 때문에 null 값을 넣을 수 있다.

#### 기본 타입이 시간과 메모리 사용면에서 더 효율적이다.

시간 효율성 측면에서 비교해보자.

- 기본 타입
	- 기본 타입은 직접 값을 저장하고 접근해 연산 속도가 빠르다.
	- CPU가 기본 타입의 연산을 직접 수행하는 것이 가능하다.
	- 메모리의 스택 영역에 저장되어 메모리에 접근하는 시간이 빠르다.
- 박싱 타입
	- 메모리 힙 영역에 저장되어 메모리에 접근하는 시간이 더 느리다.
	- 박싱과 언박싱 과정이 필요해 추가적인 시간이 소요된다.

메모리 측면에서는 어떨까?

- 기본 타입
	- 기본 타입은 메모리 스택 영역에 직접 값을 저장한다.
	- 기본 타입은 고정된 크기를 가지고 메모리 오버헤드가 없다.
- 박싱 타입
	- 박싱 타입은 메모리 힙 영역에 객체로 저장된다.
	- 객체는 값을 저장하기 위한 추가적인 메모리 오버헤드가 발생한다.
		- 일반적으로 객체 헤더(12바이트) + 참조 포인터(4 / 8바이트)를 가진다.


### 박싱 타입이 문제를 일으키는 경우 1 (식별성 문제)

Integer 값을 오름차순으로 정렬하는 비교자를 보자.

```java
Comparator<Integer> naturalOrder = 
	(i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

- 코드를 보아도 별거 없다. 단순히 원소가 더 작을 경우 음수, 같으면 0 그리고 크다면 0을 반환하는 간단한 구조이다.
- Collections.sort에 원소 100만개짜리 리스트와 이 비교자를 넣어 테스트해도 문제가 없다.

그러나 문제를 일으키는 코드가 있다.

```java
@Test  
void test() {  
    Comparator<Integer> naturalOrder = (i ,j) -> (i < j) ? -1 : (i == j ? 0 : 1);  
    int compare = naturalOrder.compare(new Integer(42), new Integer(42));  
    Assertions.assertThat(compare).isEqualTo(0);  //fail (1)
}
```

- 책에서는 이 코드를 예시로 설명하고 있지만, Java 9 이후 new 키워드를 이용해 Integer를 생성하는 방식은 deprecated가 나타나긴 한다.
- 우선 이 코드가 왜 0을 반환하지 못하고 1을 반환하게 되는 지 보자.
	- 첫 비교인 (i < j)는 잘 동작한다.
	- 그런데 그 다음 동등성을 '=='으로 검사하고 있다.
	- 박싱된 기본 타입은 식별성을 가지고 있는데, new 키워드를 써서 서로 다른 객체이기 때문에 비교자는 잘못된 값을 반환하게 되는 것이다.

***이처럼 박싱된 기본 타입에 '==' 연산자를 사용하면 오류가 일어날 수 있다.***

실무에서 기본 타입을 다루는 비교자가 필요한 경우 아래와 같이 구현하자.

```java
Comparator<Integer> naturalOrder = (iBoxed, jBoxed) -> {
	int i = iBoxed;
	int j = jBoxed;

	return i < j ? -1 : (i == j ? 0 : 1);
}
```

- 지역변수 2개를 두어 각각 박싱된 매개변수의 값을 기본 타입으로 저장한 후 비교를 해당 기본 타입 변수로 수행한다면 식별성 검사를 하지 않게 되어 문제가 없다.

### 박싱 타입이 문제를 일으키는 경우 2 (null값을 가져 발생하는 문제)

```java
public class Unbelievable {
	static Integer i;

	public static void main(String[] args) {
		if (i == 42) {
			//...
		}
	}
}
```

- i는 초기화되지 않았기 때문에 42가 아니라 if문 내부의 로직을 실행하지 않을 것이다.
- 그러나 i는 박싱 기본 타입이므로 초깃값이 다른 참조 타입 필드와 마찬가지로 null이다.
	- if문 내부의 조건을 검사할 때 NullPointerException을 발생시킨다.
		- Cannot invoke "java.lang.Integer.intValue()" because "this.i" is null
		- 언박싱 실패
	
***기본 타입과 박싱된 기본 타입을 위와 같이 혼용한 연산에서는 박싱된 기본 타입의 박싱이 자동으로 언박싱된다.*** 따라서 예외를 던지게 된 것이다.

이 부분은 단순히 i를 int로 선언해주면 해결된다.

### 박싱 타입이 문제를 일으키는 경우 3 (오토 박싱/언박싱 문제)

아이템 6의 코드를 다시 보자.

```java
Long sum = 0L;
for (long i = 0; i <= Integer.MAX_VALUE; i++) {
	sum += i;
}
System.out.println(sum);
```

- 지역변수 sum이 박싱된 기본 타입으로 선언되어 있다. 그런데 for문 내부의 변수는 기본 타입이다.
- 박싱된 기본 타입과 기본 타입간의 연산은 박싱된 기본 타입이 언박싱 되어 연산된다.
- 즉, 박싱과 언박싱을 반복한다.

박싱과 언박싱을 반복하는 과정에서 심각한 성능 문제가 발생한다.

### 박싱 타입을 사용해야 하는 경우

- 컬렉션의 원소, 키 값, 매개변수화 타입이나 매개변수화 메서드의 타입 매개변수
	- 컬렉션은 기본 타입을 담을 수 없다.
	- 제네릭의 경우도 마찬가지
- 리플렉션을 통해 메서드를 호출하는 경우
- null 값 처리가 용이해 ORM 프레임워크와 연동할 경우 처리를 원활하게 할 수 있다.
	-  `We recommend that you declare consistently-named identifier attributes on persistent classes and that you use a nullable (i.e., non-primitive) type` [Hibernate ORM docs](https://docs.jboss.org/hibernate/orm/5.3/userguide/html_single/Hibernate_User_Guide.html#entity-pojo-identifier)

## 정리

기본 타입과 박싱된 기본 타입중에는 웬만하면 기본 타입을 선택하자.

기본 타입이 간단하며, 빠르고 오류 발생 가능성도 줄여준다.

다만 박싱된 기본 타입을 사용해야 하는 경우에는 주의해서 사용하자. 식별성, null 값을 가지는 것, 그리고 성능 문제를 신경써야 한다.

별개로 JPA같은 ORM 프레임워크와 사용할 때, Entity를 만드는 경우 등에는 박싱된 기본 타입을 쓰는 것이 더 나을 수 있다.

