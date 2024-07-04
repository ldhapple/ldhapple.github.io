---
title: Item11 (equals를 재정의하려거든 hashCode도 재정의하라)
author: leedohyun
date: 2024-07-03 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## equals를 재정의하려거든 hashCode도 재정의하라

equals를 재정의한 클래스에서 hashCode를 재정의하지 않으면 hashCode 일반 규약을 어기게 되어 해당 클래스의 인스턴스를 HashMap이나 HashSet같은 컬렉션의 원소로 사용할 때 문제를 일으키게 된다.

- equals가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 똑같은 값을 반환해야 한다.
	- 논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.

하지만 Object의 기본 hashCode는 둘을 다르다고 판단해 서로 다른 값을 반환하여 문제가 생기는 것이다.

> HashCode

객체의 주소값을 변환하여 생성한 객체의 고유한 정수값을 의미한다.

### hashCode에 문제가 있는 경우?

이전 아이템에서 정의한 Point 클래스로 예시를 들어보자.

```java
Point p1 = new Point(0, 1);
Point p2 = new Point(0, 1);

p1.equals(p2); // true
```

equals는 이전 아이템에서 재정의해주었기 때문에 문제가 없다. 문제는 HashMap 같은 컬렉션에서의 사용에서이다. Hash를 이용하는 컬렉션의 원소로 사용할 때 문제가 발생한다.

```java
Map<Point, String> map = new HashMap<>();
map.put(new Point(0, 1), "반지름1");
```

- 만약 위와 같은 상황에서 map.get(new Point(0, 1))을 실행한다면 "반지름1" 이라는 값이 나와야 할 것 같다.
	- 하지만 null을 반환한다.

이 상황에서 Point 인스턴스는 2개가 사용되었다. 하나는 Map에 "반지름1" 이라는 값을 넣을 때 사용되었고, 다른 하나(논리적 동치)는 맵에서 꺼내려할 때 사용되었다.

Point 클래스는 hashCode를 재정의하지 않아 논리적 동치인 두 객체에 서로 다른 해시코드를 반환하여 규약을 지키지 못했다.

## HashCode 재정의 방법

HashCode를 재정의해야 하는 이유는 알았다. 그렇다면 어떻게 재정의 해주어야 할까?

> 최악의 구현법

```java
@Override
public int hashCode() {
	return 77;
}
```

동치인 모든 객체에 똑같은 해시코드를 반환하긴 한다. 따라서 HashMap에서 동치인 인스턴스를 키로 값을 불러올 때 정상적으로 불러올 수 있다.

그렇지만 이런 구현은 모든 객체에게 똑같은 값만 내어주어 모든 객체가 해시 테이블의 버킷 하나에 담겨 마치 연결리스트처럼 동작하게 된다.

수행 시간이 O(1)에서 O(N)으로 느려진다.

### int 변수 result를 선언한 후 값 c로 초기화

c는 해당 객체의 첫 번째 핵심 필드를 아래 방식으로 계산한 해시코드이다.

- c의 계산 방식
	- 기본 타입 필드(int, boolean 등)라면 Type.hashCode(필드)를 수행. (Type은 기본 타입의 박싱 클래스)
	- 참조 타입 필드(class, String 등)면서 equals 메서드가 필드의 equals를 재귀적으로 호출해 비교한다면 그 필드의 hashCode를 재귀적으로 호출해 구한다.
		- Person (String name, Address address) 라면 Person의 equals()는 Address.equals도 호출할 것
		- 따라서 이런 경우 hashCode도 재귀적으로 호출해 구한다는 뜻.
		- 위 계산이 복잡해질 것 같은 경우 해당 필드의 표준형을 만들어 그 표준형의 hashCode로 값을 구한다. 
	- 필드가 배열일 경우 핵심 원소 각각을 별도 필드처럼 다룬다.
		- 배열에 핵심 원소가 없다면 단순히 상수(0 추천)를 사용한다.
		- 모든 원소가 핵심 원소라면 Arrays.hashCode()를 사용한다.

이렇게 계산한 c로 result를 갱신한다.

```java
result = 31 * result + c;
```

- 31을 쓰는 이유
	- 곱셈 연산에서 시프트 연산을 사용해 곱셈의 속도를 향상시킬 수 있다. 컴파일러에서 최적화가 가능한 연산이다.
	- 31은 소수인데 해시 함수에서 사용되는 소수는 해시 충돌 확률을 줄이는데 도움을 준다.

이렇게 재정의한 hashCode도 equals와 마찬가지로 단위테스트를 통해 검증해보는 것이 좋다.

**주의할 점은 equals 비교에 사용되지 않은 필드는 반드시 제외되어야 한다는 점이다.**

```java
@Override 
public int hashCode() { 
	int result = Integer.hashCode(y); 
	result = 31 * result + Integer.hashCode(x); 
	return result; 
}
```

Point 클래스의 hashCode를 위 방법대로 구현하면 위와 같은 식이다. 이 방법도 충분히 좋긴하지만 해시 충돌을 더욱 줄이고 싶다면 com.google.common.hash.Hashing을 참고하면 좋다.

### Objects 클래스의 해시 메서드 사용

Objects 클래스는 임의의 개수만큼 객체를 받아 해시코드를 계산해주는 정적 메서드를 가지고 있다.

```java
@Override
public int hashCode() {
	return Objects.hash(y, x);
}
```

물론 위 방법보다는 속도가 느리다. 입력 인수를 담기 위한 배열이 만들어지고 입력 중 기본 타입이 존재한다면 박싱과 언박싱도 거치기 때문이다.

따라서 성능이 중요하다면 이 방법은 사용하지 말자.

### 해시 코드의 캐싱

클래스가 불변이고 해시코드를 계산하는 비용이 크다면, 매번 새로 계산하기 보다는 값을 캐싱하는 방식을 고려해야 한다.

```java
private int hashCode;

@Override
public int hashCode() {
	int result = hashCode;
	if (result == 0) {
		//해시코드 계산
		hashCode = result;
	}
	return result;
}
```

### 주의사항

- 캐싱 방법에서 hashCode 필드의 초기값은 흔히 생성되는 객체의 해시코드와는 달라야 한다.
- 성능을 위해 해시코드를 계산할 때 핵심 필드를 생략해서는 안된다.
	- 속도가 당장은 빨라질 수 있지만 해시 품질이 나빠져 해시테이블의 성능을 떨어뜨리게 된다.
- hashCode 값의 생성 규칙을 API 사용자에게 자세히 공표해서는 안된다.
	- 클라이언트가 해당 값에 의지하지 않게 되어 추후 계산 방식을 바꿔도 문제를 일으키지 않는다.

## 정리

equals를 재정의할 때는 hashCode도 반드시 재정의해야 한다.

재정의한 hashCode 또한 규약을 따라야한다. (같은 인스턴스라면 같은 해시값, 다른 인스턴스라면 다른 해시값) 

재정의할 때에는 위 방법으로 직접 재정의해도 되지만 일부 IDE나 프레임워크에서 제공하는 부분을 이용해도 좋을 것이다.