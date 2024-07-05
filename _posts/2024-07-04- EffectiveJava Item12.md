---
title: Item12 (toString을 항상 재정의하라)
author: leedohyun
date: 2024-07-04 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## toString을 항상 재정의하라

Object의 기본 toString 메서드는 우리가 작성한 클래스에 적합한 문자열을 반환하는 경우가 거의 없다.

많이 겪어봤겠지만 해당 메서드는 보통 아래와 같은 결과를 나타내게 된다.

```
Test$Point@4d
```

클래스이름@16진수로 표시한 해시코드를 반환한다.

- toString의 규약
	- 간결하면서 사람이 읽기 쉬운 형태의 유익한 정보를 반환해야 한다.
	- 위와 같은 방식도 사람이 읽기 쉽고 간결하지만, (0, 1) 과 같이 Point에 대한 정보를 제공하는 것이 훨씬 유익할 것이다.
- toString의 쓰임
	- println, printf
	- 문자열 연결 연산자 (+)
	- assert 구문에 넘길 때
	- 디버거가 객체를 출력할 때
		- 컬렉션의 원소 객체를 출력할 때에도 사용된다.
	- 직접 호출하지 않더라도 위와 같이 자동으로 불일 일이 많아 재정의 해주면 좋다.

***toString을 재정의하는 것은 equals와 hashCode만큼 매우 중요하진 않지만 사용하기에 훨씬 편하고 디버깅하기 쉽다.***

### toString 재정의하는 방법

#### 간결하면서 사람이 읽기 쉬운 형태의 + 유익한 정보를 담아야 한다.

```
Point@4d -> (0, 1)
```

주의할 점은 객체가 거대하거나 객체의 상태가 문자열로 표현하기 적합하지 않다면 이런 방식은 어울리지 않는다.

```
Thread[main,5,main]
전화번호부(총 1231589개)
```

이런식으로 객체의 상태를 문자열로 표현하기 적합하지 않는다면 요약 정보를 담는 것이 더 이상적이다.

#### 객체가 가진 주요 정보는 모두 반환하자.

```java
class Person {
	private final String name;
	private final Address address;
	private final int age;
	private final String phoneNumber;

	//...
	@Override
	public String toString() {
		return "Name = " + name + "\n"
			+ "age = " + age;
	}
}
```

위와 같은 방식으로 일부 정보만 반환하는 것은 좋은 방법이 아니다.

#### toString을 구현할 때면 반환값의 포맷을 문서화할 지 정해야 한다.

전화번호부나 행렬과 같은 값 클래스라면 문서화가 권장된다. 

```java
@Override
public String toString() {
	return String.format("%03d-%03d-%04d",
			areaCode, prefix, lineNum);
}
```
```
/*
이 전화번호의 문자열 표현을 반환한다.
이 문자열은 "XXX-YYY-ZZZZ" 형태의 12글자로 구성된다.
XXX는 지역코드, YYY는 프리픽스, ZZZZ는 가입자 번호이다.
각각의 대문자는 10진수 숫자 하나를 나타낸다.

전화번호 각 부분의 값이 너무 작아 자릿수를 채울 수 없다면 앞에서부터 0으로 채운다.
*/
```

위와 같이 포맷을 명시할 수 있을 것이다.

- 장점
	- 포맷을 명시하면 그 객체는 표준적이고, 명확하며 사람이 읽을 수 있게 된다.
- 단점
	- 포맷을 한 번 명시했고, 그 클래스가 많이 쓰인다면 평생 그 포맷에 얽매이게 된다.
		- 수정이 어렵다.
		- 따라서 오히려 포맷을 명시하지 않는다면 포맷을 개선할 수 있는 유연성을 얻을 수 있다.

> 포맷 명시 여부와 상관없이 toString이 반환한 값에 포함된 정보를 얻어올 수 있는 API를 제공하자. 

위의 phoneNumber의 경우 지역 코드, 프리픽스, 가입자 번호용 접근자를 제공해야 한다. 그렇지 않으면 이 정보가 필요한 프로그래머는 toString의 반환값을 파싱할 수 밖에 없다.

성능이 나빠지고 필요하지도 않은 작업이다. 더해 포맷이 바뀐다면 시스템이 망가질 것이다.

***따라서 toString이 반환한 값에 포함된 정보는 그 정보를 이용할 수 있도록 API를 제공해야 한다.***


### toString을 재정의하지 않아도 되는 경우

- 정적 유틸리티 클래스인 경우
- 이미 완벽한 toString이 제공되고 있는 열거형 타입
- 상위 클래스에서 이미 알맞게 재정의한 경우

## 정리

toString을 알맞게 재정의한 클래스는 사용도 편하고 디버깅하기 쉽게 해준다.

따라서 toString은 명확한 정보를 담도록 재정의하는 것이 좋다.

경우에 따라 toString을 재정의할 때 인텔리제이의 기능을 써도 좋다. Point 클래스를 예시로 든다면 아래와 같은 toString 메서드를 만들어준다.

```java
@Override  
public String toString() {  
  return "Point{" +  
            "y=" + y +  
            ", x=" + x +  
            '}';  
}
```