---
title: Item50 (적시에 방어적 복사본을 만들라)
author: leedohyun
date: 2024-07-27 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 적시에 방어적 복사본을 만들라

자바는 안전한 언어이다. 네이티브 메서드를 사용하지 않아 C, C++과 같이 안전하지 않은 언어에서 흔히 보는 버퍼 오버런, 배열 오버런, 와일드 포인터 같은 메모리 충돌 오류에서 안전하다.

- 배열 오버런이란 자바에서 배열의 경계를 벗어나는 접근을 하게되면 ArrayIndexOutOfBoundsException을 만나는데, 다른 언어에서는 그렇지 않아 문제가 발생할 수 있는 것을 의미한다.
- 또, 자바에서는 직접 포인터 연산을 하지 않고 참조를 통해 메모리를 간접적으로 다루기 때문에 임의의 메모리 영역을 수정해 발생하는 문제가 생기지 않는다는 뜻이다.

자바로 작성한 클래스는 시스템의 다른 부분에서 무슨 짓을 해도 그 불변식이 지켜진다. 메모리 전체를 하나의 거대한 배열로 다루는 언어에서는 누릴 수 없는 강점이다.

### 그러나 방어적으로 프로그래밍 하자.

자바가 비교적 안전하다해도 다른 클래스로부터의 침범을 아무런 노력 없이 막을 수 있는 것은 아니다.

- 악의적인 의도로 시스템의 보안을 뚫으려는 시도가 늘고 있다.
- 평범한 프로그래머라도 실수로 클래스를 오작동하게 할 수 있다.

따라서 ***우리는 클라이언트가 불변식을 깨뜨리려고 노력한다고 가정해 방어적으로 프로그래밍을 해야 한다.***

불변식을 깨뜨리는 경우를 알아야 우리가 방어적으로 프로그래밍해야 한다는 것을 깨달을 것이다. 어떤 경우 불변식을 지키지 못하게 되는 지 알아보자.

### 객체가 자기도 모르게 내부를 수정하도록 허락하는 경우

흔히 하는 실수이다.

```java
public final class Period {
	private final Date start;
	private final Date end;

	public Period(Date start, Date end) {
		if (start.compareTo(end) > 0) {
			throw new IllegalArgumentException();
		}
		this.start = start;
		this.end = end;
	}
	
	public Date start() {
		return start;
	}

	public Date end() {
		return end;
	}

	//...
}
```

- 시작 시각이 종료 시각보다 늦을 수 없다는 불변식이 지켜질 것 같은 클래스이다. 
	- 생성자에서 매개변수를 검사해 예외를 던지도록 했으니 말이다.
- 거기에 class에도 final을 붙이고, 필드에도 final을 붙였다.
	- Period 클래스를 상속하는 하위 클래스를 만들 수 없다. 불변 클래스를 만들 목적이다.
	- 또한 필드가 초기화된 이후 값을 변경할 수 없도록 final을 붙였다. 생성자에서만 초기화 될 목적이다.

불변을 보장하고 있는 것 같다. 하지만 간과하고 있는 사실이 하나 있다. **필드에 사용한 Date는 가변 객체이다.**

```java
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end);
end.setYear(78);
```

- Period의 필드인 end를 수정해 Period의 내부를 손쉽게 변경할 수 있었다. 
- end를 수정하는 것은 막혀있지 않다.

Java 8 이후 이 부분은 쉽게 해결할 수 있다. Date 대신 불변 인스턴스인 Instant를 사용하면 된다. 혹은 LocalDateTime이나 ZonedDateTime을 사용하면 된다. 해당 객체들은 final이다.

### 불변 객체만 쓰면 해결되는 걸까? (방어적 복사를 하자)

문제가 하나 있다. 앞으로 위와 같이 가변 객체를 쓰지 않는다고 해서 이전 코드들이 알아서 수정되는 것은 아니다.

여전히 많은 API와 내부 구현에 그 잔재가 남아있을 수 있다.

외부 공격으로부터 Period 인스턴스 내부를 보호하려면 어떻게 해야 할까?

```java
public Period(Date start, Date end) {
	this.start = new Date(start.getTime());
	this.end = new Date(end.getTime());
	
	if (this.start.compareTo(this.end) > 0) {
		throw new IllegalArgumentException();
	}
}
```

- ***생성자에서 받은 가변 매개변수 각각을 방어적으로 복사했다.***
	- 외부에서 값을 수정하더라도 Period에서는 복사한 값을 사용하고 있기 때문에 이전 예시에서의 문제는 일어나지 않는다.
- **매개변수의 유효성을 검사하기 전에 방어적 복사본을 만들고 이 복사본으로 유효성을 검사한 부분을 명심해야 한다.** 
	- 멀티 쓰레드 환경에서 원본 객체의 유효성 검사 후 복사본을 만드는 찰나에 다른 쓰레드가 원본 객체를 수정할 수도 있다.
	- 검사시점/사용시점(time-of-check/time-of-use) 공격 혹은 TOCTOU 공격이라고 한다.
- **또 매개변수가 제 3자에 의해 확장될 수 있는 타입이라면 방어적 복사본을 만들 때 clone을 사용하면 안된다.**
	- 예를 들면 Date는 final이 아니기 때문에 상속해 확장할 수 있다.
	- 그렇다면 clone 메서드가 Date가 정의한 것이 아닌 Date를 확장한 하위 클래스가 재정의한 것일 수도 있다.
	- clone()이 악의를 가진 하위 클래스의 인스턴스를 반환한다면 문제가 발생할 수 있다.

```java
Date start = new Date();
ExtendDate end = new ExtendDate();
Period period = new Period(start, end);
```

- 생성자에서 clone을 사용하여 복사할 경우 ExtendDate가 재정의한 clone()이 호출될 것이다.

결론적으로는 이렇게 방어적 복사를 이용하면 문제는 끝일까? 여전히 문제는 남아있다.

### 접근자 메서드가 여전히 내부의 가변 정보를 드러내고 있다. (접근자 메서드에서도 방어적 복사본을 반환하자.)

```java
public Date start() {
	return start;
}

public Date end() {
	return end;
}
```
```java
Date start = new Date();
Date end = new Date();
Period period = new Period(start, end);
period.start().setYear(77);
```

- 기껏 방어적 복사를 했지만, 접근자 메서드를 통해 내부의 가변 필드를 수정할 수 있다.
- final은 재할당을 막아줄 뿐 내부 필드의 참조를 반환한 Date 객체를 통해 그 객체의 상태를 변경하는 것은 막아주지 않는다.

이 공격을 막기 위해서는 물론 접근자를 없애는게 가장 쉬운 방법이지만, 접근자가 필요한 경우라면 **가변 필드의 방어적 복사본을 반환하도록 하면 된다**.

```java
public Date start() {
	return new Date(start.getTime);
}

public Date end() {
	return new Date(end.getTime);
}
```

- 새로운 객체를 생성해 반환했기 때문에 그 객체는 수정해도 인스턴스 내부의 필드에는 접근하지 못한다. 단순히 값만 같은 객체일 뿐이다.
- 생성자에서 방어적 복사를 하고, 접근자를 방어적 복사본을 반환하도록 만들면 완벽한 불변이 완성된다.
	- 시작 시각이 종료 시각보다 나중일 수 없다는 불변식을 위배할 방법이 없다. (네이티브 메서드나 리플렉션 같은 수단을 동원하지 않는 이상은..)
	- 완벽하게 캡슐화되었다. 자신 말고는 가변 필드에 접근할 방법이 없다.
- 참고로 생성자와 달리 접근자 메서드에서는 방어적 복사에 clone()을 사용해도 된다.
	- Period가 가지고 있는 Date 객체는 java.util.Date임이 확실하기 때문이다. (이미 생성자에서 새로운 Date객체를 만들어 방어적 복사를 했다.)
	- 그러나 인스턴스 복사에는 일반적으로 생성자나 정적 팩토리를 쓰는 것이 좋다. ([아이템 13](https://ldhapple.github.io/posts/EffectiveJava-Item13/))

## 정리

가변 객체라면 그 객체가 클래스에 넘겨진 뒤 임의로 변경되어도 그 클래스가 문제없이 동작할 지를 따져보아야 한다. (Map의 키로 사용한다고 생각하면 Map의 불변식이 깨질 수 있다.)

확신할 수 없다면 클래스가 클라이언트로부터 받는(ex - 매개변수), 혹은 클라이언트로 반환(ex- 접근자 메서드)하는 구성요소가 가변일 경우 그 요소는 반드시 방어적으로 복사해야 한다.

그러나 복사 비용이 너무 크거나, 클라이언트가 그 요소를 잘못 수정할 일이 없음을 신뢰한다면 방어적 복사를 수행하는 대신 그 구성 요소를 수정했을 때의 책임이 클라이언트에 있음을 명시하면 된다.