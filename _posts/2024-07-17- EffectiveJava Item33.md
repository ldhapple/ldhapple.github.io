---
title: Item33 (타입 안전 이종 컨테이너를 고려하라)
author: leedohyun
date: 2024-07-17 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 타입 안전 이종 컨테이너를 고려하라

제네릭은 컬렉션과 단일 원소 컨테이너(AtomicReference.. ) 등등에 흔히 쓰인다. 이 때 매개변수화되는 대상은 원소가 아닌 컨테이너 자신이다.

따라서 하나의 컨테이너에서 매개변수화할 수 있는 타입의 수가 제한된다. 

예를 들면 Set에는 원소의 타입을 뜻하는 단 하나의 타입 매개변수만 있으면 되고 Map의 경우에는 Key, Value의 2가지 타입만 필요하다는 뜻이다.

그러나 더 유연한 수단이 필요할 때도 종종 있을 것이다. 그럴 때 타입 안전 이종 컨테이너 패턴을 사용하면 된다.

### 타입 안전 이종 컨테이너 패턴

어떤 경우에 유연하게 구현해야 할까?

타입별로 즐겨 찾는 인스턴스를 저장하고 검색할 수 있는 Favorite 클래스가 있다고 해보자.

단순하게 Map을 떠올릴 수 있다. 그러나 Map은 키 값으로 타입을 정의할 수는 있겠지만 즐겨 찾는 인스턴스의 타입을 다양하게 할 수 없다. 가능하다 해도 복잡하게 구현해야 할 것이다.

이럴 때 조금 더 유연하게 타입 안전 이종 컨테이너 패턴을 써볼 수 있다.

```java
public class Favorites {
	private Map<Class<?>, Object> favorites = new HashMap<>();
	
	public <T> void putFavorite(Class<T> type, T instance) {
		favorites.put(Objects.requireNonNull(type), instance);
	}
	
	public <T> T getFavorite(Class<T> type) {
		return type.cast(favorites.get(type));
	}
}
```

- 각 타입의 Class 객체를 매개변수화한 키 역할로 사용한다.
- cast()를 이용해 값의 타입이었던 Object대신 T 타입으로 형변환해 반환한다.

```java
Favorites f = new Favorites();

f.putFavorite(String.class, "Java");
f.putFavorite(Class.class, Favorites.class);

String favoriteString = f.getFavorite(String.class);
Class<?> favoriteClass = f.getFavorite(Class.class);
//...
```

- Favorites 인스턴스는 타입 안전하다. String을 요청했는데 Integer을 반환할 일이 없다.
- 모든 키의 타입이 제각각이기 때문에 일반적인 Map과 달리 여러 가지 타입의 원소를 담을 수 있다.

**종합하자면 타입 안전 이종 컨테이너 패턴은 컨테이너 대신 '키를 매개변수화'하고, 컨테이너에 값을 넣거나 뺄 때 매개변수화한 키를 제공하는 방식이다.**


> Key 비한정적 와일드카드 타입

위 구현에서 Map의 키에 비한정적 와일드카드 타입을 쓰고 있다. 비한정적 와일드카드 타입은 읽기만을 지원하기 때문에 Map안에 아무것도 넣을 수 없다고 생각될 수 있다.

그러나 맵이 아니라 키의 Class가 와일드카드 타입인 것을 파악해야 한다. 모든 키가 서로 다른 매개변수화 타입일 수 있다는 뜻이다. 각 키에 해당하는 값은 해당 클래스의 인스턴스여야 한다는 뜻으로 쓰인 것이다.

> 값 타입은 Object?

위 구현에서 키에 대응하는 값 타입으로 Object를 선택했다. 이 뜻은 키와 값 사이의 타입 관계를 보증하지 않는다는 것이다.

자바의 타입 시스템에서는 이 관계를 명시할 방법이 없다. 

하지만 작성자는 이 관계가 성립할 수 있음을 알고 있기 때문에 이점만을 누리면 된다.

### 타입 이종 컨테이너 패턴의 한계

- 악의적인 클라이언트가 Class 객체를 제네릭이 아닌 로 타입으로 넘기면 타입 안전성이 깨진다.
	- 이 문제는 HashMap같은 일반 컬렉션 구현에도 적용된다.
- 실체화 불가 타입에는 사용할 수 없다.
	- 키에 String List가 들어갈 수 있을까?
	- 불가능하다. String List는 List.class로 대체된다. Integer List 등등과 같은 Class 객체를 공유하기 때문에 사용할 수 없는 것이다.

### 한정적 타입 토큰

위 패턴의 예시 코드를 보면, 어떤 타입의 Class든 받을 수 있도록 되어 있다.

하지만 일부 타입을 제한시키고 싶을 수도 있다. 이런 경우 한정적 타입 토큰을 사용하면 된다.

한정적 타입 토큰이란 단순히 한정적 타입 매개변수나 한정적 와일드카드를 사용해 표현 가능한 타입을 제한하는 타입 토큰이다. 

```java
public <T extends Annotation> T getAnnotation(Class<T> annotationType) {...}
```

- Annotation의 하위 타입으로 제한했다.

## 정리

일반적인 제네릭 형태에서는 하나의 컨테이너가 다룰 수 있는 타입 매개변수의 수가 고정되어 있다.

그런데 때때로 더 유연한 방식이 필요한 경우가 있을 것이다.

이럴 때 타입 안전 이종 컨테이너를 만들어 대처할 수 있을 것이다. 핵심은 Class를 키로 사용하는 것이다.