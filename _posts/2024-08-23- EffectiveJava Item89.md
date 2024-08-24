---
title: Item89 (인스턴스 수를 통제해야 한다면 readResolve 보다는 열거 타입을 사용하라)
author: leedohyun
date: 2024-08-23 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 인스턴스 수를 통제해야 한다면 readResolve 보다는 열거 타입을 사용하라

implements Serializable을 추가하면 싱글턴이 될 수 없다.

기본 직렬화를 쓰지 않더라도, 명시적인 readObject를 제공하더라도 마찬가지다. 어떤 readObejct를 사용해도 이 클래스가 초기화될 때 만들어진 인스턴스와는 별개인 인스턴스를 반환하게 된다.

### readResolve

readResolve 기능을 이용하면 readObject가 만들어낸 인스턴스를 다른 것으로 대체할 수 있다.

역직렬화한 객체의 클래스가 readResolve 메서드를 적절히 정의해두었다면 역직렬화 후 새로 생성된 객체를 인수로 이 메서드가 호출되고, 이 메서드가 반환한 객체 참조가 새로 생성된 객체를 대신해 반환된다.

대부분의 경우 이 때 새로 생성된 객체의 참조는 유지하지 않아 가비지 컬렉션 대상이 된다.

```java
public class Elvis implements Serializable {
	public static final Elvis INSTANCE = new Elvis();
	private Elvis() { ... }

	private Object readResolve() {
		return INSTANCE;
	}
	
	public void leaveTheBuilding() { ... }
}
```

- 진짜 Elvis를 반환하고, 가짜 Elvis는 가비지 컬렉터에 맡긴다.
- readResolve() 메서드를 추가함으로써 역직렬화한 객체는 무시하고 클래스 초기화 때 만들어진 Elvis 인스턴스를 반환한다.
- Elvis 인스턴스의 직렬화 형태는 아무런 실 데이터를 가질 이유가 없어 모든 인스턴스 필드를 transient로 선언해야 한다.
	- transient 키워드로 지정된 필드는 직렬화되지 않는다.
	- 역직렬화될 때 해당 필드는 기본값으로 초기화된다.

사실 위의 Elvis 예시뿐 아니라 ***readResolve()를 인스턴스 통제 목적으로 사용한다면 객체 참조 타입 인스턴스 필드는 모두 transient로 선언해야 한다.*** 

그렇지 않으면 readResolve 메서드가 수행되기 전 역직렬화된 객체의 참조를 공격할 여지가 남게된다.

싱글턴이 transient가 아닌 참조 필드를 갖고 있다면, 그 필드의 내용은 readResolve()가 실행되기 전에 역직렬화된다. 잘 조작된 스트림을 써서 해당 참조 필드의 내용이 역직렬화되는 시점에 그 역직렬화된 인스턴스의 참조를 훔쳐올 수 있는 것이다.

readResolve 메서드를 사용해 순간적으로 만들어진 역직렬화된 인스턴스에 접근하지 못하게 하는 방법은 깨지기 쉽고 신경을 많이 써야 하는 작업이다.

### 열거 타입을 사용하자

직렬화 가능한 인스턴스 통제 클래스를 열거 타입을 이용해 구현하면 선언한 상수 외의 다른 객체는 존재하지 않음을 자바가 보장해준다.

```java
public enum Elvis {
	INSTANCE;

	private String[] favoriteSongs = 
		{ "Hound Dog", "Heartbreak Hotel" };
	
	public void printFavorites() {
		System.out.println(Arrays.toString(favoriteSongs));
	}
}
```

물론 공격자가 AccessibleObject.setAccessible 같은 특권 메서드를 악용하면 열거형을 통한 방어도 무력화된다.

## 정리

불변식을 지키기 위해 인스턴스를 통제해야 한다면 가능한 한 열거 타입을 사용하는 것을 추천한다.

여의치 않은 상황에서 직렬화와 인스턴스 통제가 모두 필요한 경우 readResolve 메서드를 작성해 넣어야 하고 그 클래스에서 모든 참조 타입 인스턴스 필드를 transient로 선언해야 안전하다.