---
title: Item22 (인터페이스는 타입을 정의하는 용도로만 사용하라)
author: leedohyun
date: 2024-07-09 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 인터페이스는 타입을 정의하는 용도로만 사용하라

인터페이스는 자신을 구현한 클래스의 인스턴스를 참조할 수 있는 타입 역할을 한다.

이 말 뜻은 **자신의 인스턴스로 무엇을 할 수 있는지를 클라이언트에 알려주는 것**이다.

인터페이스는 오로지 이 용도로만 사용되어야 한다.

### 인터페이스의 잘못된 사용

인터페이스를 어떻게 다른 용도로 잘못 사용하는지 보자.

#### 상수 인터페이스 안티패턴

메서드 없이 상수를 뜻하는 static final 필드로만 구성된 인터페이스를 뜻한다.

이 상수들을 사용하려는 클래스에서는 정규화된 이름을 사용하는 것을 피하고자하는 것이 목적이다.

```java
interface Constants {
	static final int MAX_SIZE = 1000;
	static final String DEFAULT_NAME = "Hong";
	static final double PI = 3.141592;
	//...
}
```

이렇게 사용하는 것이 뭐가 문제가 될까?

- 사용하지 않을 수도 있는 상수를 포함하여 모두 가져온다.
	- 상수들을 더 쓰지 않게 되어도 바이너리 호환성을 위해 여전히 상수 인터페이스를 구현하고 있어야 한다. (해당 인터페이스 상수를 다른 프로그램에서 사용될 수 있어 유지해야 함.)
- 클라이언트가 상수 인터페이스 값에 직접 접근할 수 있다, 캡슐화가 깨진다.
	- 불필요한 정보를 노출해 클라이언트에 혼란을 야기할 수 있다.
- 캡슐화가 깨진다.
	- 인터페이스에 private이나 protected를 사용할 수 없는데, 클래스가 내부적을 사용하는 상수들은 구현 세부사항임에도 모든 클래스에서 접근이 가능해 캡슐화가 깨진다.
- 클래스가 어떤 상수 인터페이스를 사용하든 사용자에게는 아무런 의미가 없다. 

상수를 공개하는 목적이라면 다른 선택지가 있다.

> 바이너리 호환성

#### 상수 인터페이스 대안

- 특정 클래스나 인터페이스와 강하게 연관된 상수라면 그 클래스나 인터페이스 자체에 추가한다.

```java
public final class Integer extends Number  
        implements Comparable<Integer>, Constable, ConstantDesc {  
	@Native public static final int MIN_VALUE = 0x80000000;  
	  
	@Native public static final int MAX_VALUE = 0x7fffffff;
	//...
}
```

- 인스턴스화할 수 없는 유틸리티 클래스에 담아 공개한다.

```java
class Constants {
	private Constants() {}

	public static final int MAX_SIZE = 1000;
	public static final String DEFAULT_NAME = "Hong";
	public static final double PI = 3.141592;
	//...
}	
```

- 열거 타입으로 나타내기 적합한 상수라면 열거 타입으로 만들어 공개한다.

```java
public enum Day { 
	SUNDAY, MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY 
}
```

## 정리

인터페이스는 타입을 정의하는 용도로 사용하는 것이지, 상수 등을 관리하는 용도가 아니다.

타입을 정의한다는 것은 인터페이스에 해당하는 인스턴스로 클라이언트에게 자신의 동작이 어떤 것인지 알려주는 용도이다.

다른 용도로 사용했을 경우 부작용이 있을 수 있으므로 기존 인터페이스의 목적에 맞추어 사용하자.