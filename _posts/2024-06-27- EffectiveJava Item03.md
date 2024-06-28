---
title: Item3 (private 생성자나 열거 타입으로 싱글턴임을 보증하자)
author: leedohyun
date: 2024-06-27 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## private 생성자나 열거 타입으로 싱글턴임을 보증하라

싱글톤 클래스를 만드는 방법은 다양하다.

[디자인패턴 포스트](https://ldhapple.github.io/posts/CS%EB%A9%B4%EC%A0%91-%EB%94%94%EC%9E%90%EC%9D%B8%ED%8C%A8%ED%84%B4%28%EC%8B%B1%EA%B8%80%ED%86%A4,-%ED%8C%A9%ED%86%A0%EB%A6%AC,-%EC%9D%B4%ED%84%B0%EB%A0%88%EC%9D%B4%ED%84%B0%29/)에 싱글톤을 구현하는 방법 7가지를 정리해놓았다.

클래스를 싱글톤으로 만들었을 때의 단점은 테스트가 어려워 진다는 점이다.

타입을 인터페이스로 정의하고 그 인터페이스를 구현해 만든 싱글톤이 아니라면 싱글톤 인스턴스를 mock 구현으로 대체할 수 없기 때문이다.

> 스프링에서의 대처

일반적으로 스프링의 빈들은 인터페이스를 구현한 클래스들이다.

- 생성자 주입을 통한 테스트
	- 테스트에서 생성자 주입을 통해 의존성을 명시적으로 주입할 수 있다.
	- 이 방법을 사용해 생성자를 통한 mock 객체를 주입할 수도 있다.
-  @MockBean 사용
	- 빈을 Mock으로 대체할 수 있다. 테스트 클래스에서만 적용된다.



### 방법 1. public static final 필드 방식

```java
public class Singlton{
	public static final Singleton INSTANCE = new Singleton();

	private Singleton(){
	}
}
```

- private 생성자는 static final 필드인 인스턴스를 초기화할 때 딱 한 번 호출된다.
	- 인스턴스가 하나뿐임을 보장한다.

> 장점

- 해당 클래스가 싱글톤임이 API에 명백하게 드러난다.
	- public static 필드가 final이기 때문에 절대 다른 객체를 참조할 수 없다.
- 간결하다.

> 단점

- 사용하지 않더라도 인스턴스가 생성된다. 

### 방법 2. 정적 팩토리 방식의 싱글톤

```java
public class Singleton {
	private static final Singleton INSTANCE = new Singlton();
	
	private Singlton() { }
	
	public static Singleton getInstance() { return INSTANCE; }
}
```

> 장점

- API를 바꾸지 않고도 싱글톤이 아니게 변경할 수 있다.
	- 유일한 인스턴스를 반환하던 팩토리 메서드가 호출하는 쓰레드별로 다른 인스턴스를 넘겨주도록 할 수 있다.
- 정적 팩토리 메서드를 제네릭 싱글톤 팩토리로 만들 수 있다.
	- getInstance() 메서드에 제네릭 클래스를 받아 그 제네릭 클래스에 해당하는 싱글톤 객체를 반환하도록 만들 수 있다.
- 정적 팩토리 메서드의 참조를 Supplier 로 사용할 수 있다.

이러한 장점들이 필요하지 않은 경우 방법 1을 추천한다.

> 단점

- 사용하지 않더라도 인스턴스가 생성된다.

### 방법1과 방법2의 직렬화

***두 방식으로 만든 싱글톤 클래스를 직렬화하기 위해서는 단순하게 Serializable을 구현한다고 선언하는 것만으로는 부족하다.***

모든 인스턴스 필드를 일시적(transient)이라고 선언한 후 readResolve 메서드를 제공해야 한다.

- 이 방식을 사용하지 않을 경우 직렬화(객체 -> 바이트 스트림으로 변환)된 인스턴스를 역직렬화할 때 마다 새로운 인스턴스가 만들어진다.

```java
private Object readResolve() {
	//직렬화된 객체가 역직렬화될 때 자동으로 호출된다.
	return INSTANCE;
}
```

- 직렬화된 객체가 역직렬화될 때 JVM은 readResolve() 메서드를 호출해 역직렬화 과정에서 반환되는 객체를 제어할 수 있다.
	- 이를 통해 복원된 객체를 원하는 객체로 교체할 수 있다.
	- 항상 같은 인스턴스를 반환하도록 구현해 싱글톤의 특성을 유지할 수 있다. 
- 역직렬화 과정에서 복원해야 하는 객체가 readResolve() 메서드를 가지고 있는지 체크해 존재하면 자동으로 호출한다.

### 열거 타입 방식의 싱글톤

```java
public enum Singleton {
	INSTANCE;

	public void method() {...} //객체 메서드
}
```

- public 필드 방식과 비슷하나 더 간결하다.
- 추가적인 노력 없이 직렬화/역직렬화 가능하다.
	- 제 2의 인스턴스가 생기는 부분을 완벽하게 막아준다.
- 리플렉션 공격에서도 안전하다. 

방법 1, 2는 권한이 있는 클라이언트의 경우 리플렉션 API를 통해 private 생성자를 호출 가능하기 때문에 두 번째 객체의 생성이 이루어질 때 예외를 던지도록 하는 방어를 구현해야 한다. 그러나 이 방법은 리플렉션 공격에서도 안전하다.

이렇게 원소가 하나 뿐인 열거 타입이 싱글톤을 만드는 가장 좋은 방법이다.

하지만 만들려는 싱글톤 클래스가 Enum 외의 클래스를 상속해야 하는 경우 이 방법을 사용할 수 없다.