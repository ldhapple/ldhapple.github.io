---
title: Item36 (비트 필드 대신 EnumSet을 사용하라)
author: leedohyun
date: 2024-07-18 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 비트 필드 대신 EnumSet을 사용하라

열거한 값들이 주로 집합으로 사용될 경우, 예전에는 각 상수에 서로 다른 2의 거듭제곱 값을 할당한 정수 열거 패턴을 사용해왔다.

```java
public class Text {
	public static final int STYLE_BOLD = 1 << 0;
	public static final int STYLE_ITALIC = 1 << 1;
	public static final int STYLE_UNDERLINE = 1 << 2;
	public static final int STYLE_STRIKETHROUGH = 1 << 3;

	public void applyStyles(int styles) { ... }
}
```

이렇게 구현하는 이유가 뭘까?

```java
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

- 위와 같이 비트 OR 연산자를 사용해 여러 상수를 하나의 집합으로 모을 수 있다.
	- 겉으로 보기에 매우 유연하고 좋아보인다.

이 방식은 장단점이 있다.

- 장점
	- 비트별 연산을 사용해 합집합/교집합 같은 집합 연산을 효율적으로 수행 가능하다.
- 단점
	- 정수 열거 패턴의 단점을 그대로 가지고 있다. (깨지기 쉽고, 타입 안전 보장 X 등)
	- 비트 필드 값이 그대로 출력될 경우 단순한 정수 열거 상수를 출력할 때보다 해석하기가 어렵다.
	- 비트 필드 하나에 녹아있는 모든 원소를 순회하기 까다롭다.
	- 최대 몇 비트가 필요한지 API 작성 시 미리 예측하여 적절한 타입을 선택해야 한다. 
		- API를 수정하지 않고는 비트 수를 늘릴 수 없다.

사실 내 입장에서 보면 여러 상수를 하나의 집합으로 쉽게 모을 수 있는 점이 인상깊지만, 우선 비트 연산도 익숙하지 않고 Enum 처럼 다양한 활용도를 가지지 못하는게 아쉬워보인다.

Enum이 존재하는 지금은 어떻게 처리할 수 있을까?

### 비트 필드를 대체하는 방법

결론부터 말하면 EnumSet을 사용해주면 된다.

```java
public class Text {
	public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }

	public void applyStyles(Set<Style> styles) { ... }
}
```

```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

- 우선 메서드 선언부에서 EnumSet이 아닌 Set으로 받은 이유가 있다.
	- 모든 클라이언트가 EnumSet을 건네리라 짐작되는 상황이라도 이왕이면 인터페이스로 받는게 일반적으로 좋은 습관이다.
	- 특이한 클라이언트가 다른 Set 구현체를 넘기더라도 처리할 수 있다.

그럼 EnumSet을 사용하는 것의 장점은 뭘까?

- 장점
	- 열거 타입 상수의 값으로 구성된 집합을 효과적으로 표현할 수 있다.
	- Set 인터페이스를 완벽히 구현해 타입 안전하다. 
		- 다른 Set 구현체와도 함께 사용 가능하다.
	- removeAll()과 retainAll(교집합) 같은 대량 작업은 비트를 효율적으로 처리할 수 있는 산술 연산으로 구현되어 성능도 비트 연산에 밀리지 않는다.
	- 난해한 작업들은 EnumSet에서 처리되어 비트를 직접 다룰때 겪을 수 있는 오류들로부터 자유롭다.

다만 조금의 단점이라면, 현재 자바 21까지에서도 불변 EnumSet을 만들 수 없다. 대안으로는 Collections.unmodifiableSet으로 EnumSet을 감싸 사용할 수 있다. (명확성과 성능이 조금 희생된다.)

## 정리

열거할 수 있는 타입을 모아 집합 형태로 사용하더라도 비트 필드를 사용할 이유가 없다.

EnumSet을 사용하면 열거 타입의 장점은 살리면서도, 비트 필드 수준의 명료함과 성능을 제공하기 때문이다.