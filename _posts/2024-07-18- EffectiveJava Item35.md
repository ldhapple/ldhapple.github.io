---
title: Item35 (ordinal 메서드 대신 인스턴스 필드를 사용하라)
author: leedohyun
date: 2024-07-18 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## ordinal 메서드 대신 인스턴스 필드를 사용하라

대부분의 열거 타입 상수는 자연스럽게 하나의 정숫값에 대응된다. 그리고 모든 열거 타입은 해당 상수가 그 열거 타입에서 몇 번째 위치인지를 반환하는 ordinal() 이라는 메서드를 제공한다.

```java
public enum Ensemble {
	SOLO, DUET, TRIO, QUARTET, ... , DECTET;

	public int numberOfMusicians() { return ordinal() + 1; }
}
```

- 합주단의 각 상수가 몇 명의 연주자인지 반환하는 메서드이다.
	- 당연히 정상적으로 동작한다.

위 코드는 정상적으로 동작하지만, 유지보수가 매우 어렵다.

- 상수 선언 순서를 바꾸는 순간 해당 메서드는 오동작한다.
- 이미 8명이 연주하는 octet 상수가 존재하기 때문에 똑같이 8명이 연주하는 복4중주 (double quartet) 상수는 추가가 불가능하다.
- 값을 중간에 비워둘 수도 없다.
	- 12중주가 있다고 하면 11중주가 그 전에 선언이 되어있어야 한다. 그런데 11중주를 일컫는 말은 없다.
	- 더미 상수를 같이 추가해야 하는 상황인 것이다.

코드가 깔끔하지 못하고, 실용성도 떨어진다.

### 대처법

열거 타입 상수에 연결된 값은 ordinal 메서드로 얻지 말고 인스턴스 필드에 저장하면 된다.

```java
public enum Ensemble {
	SOLO(1), DUET(2), TRIO(3), QUARTET(4), ... , DECTET(10);
	
	private final int numberOfMusicians;
	//...
	public int numberOfMusicians() { return numberOfMusicians; }
}
```

## 정리

Enum의 API 문서를 보면 ordinal에 대해 "대부분 프로그래머는 이 메서드를 쓸 일이 없다" 라고 되어 있다.

ordinal() 메서드는 EnumSet과 EnumMap 같이 열거 타입 기반의 범용 자료구조에 쓰일 목적으로 설계된 것이다. 따라서 이러한 용도가 아닌 경우 ordinal() 메서드는 없다고 생각하는 것이 낫다.

> EnumMap 사용 이유?

- Array를 사용하기 때문에 성능적으로 우수하다.
- 해싱 과정이 필요없어 HashMap보다 빠르다.
- 키를 제한할 수 있다.
	- Enum을 Key값으로 제한하면서 상수의 순서를 활용할 수 있다.
- Thread-Safe 하지 않다.
	- Collections.synchronizedMap()으로 감싸 대응할 수 있다.