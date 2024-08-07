---
title: Item62 (다른 타입이 적절하다면 문자열 사용을 피하라)
author: leedohyun
date: 2024-08-06 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 다른 타입이 적절하다면 문자열 사용을 피하라

문자열은 Text를 표현하도록 설계되었다. 실제로도 잘 처리한다.

그런데 너무 흔하고, 너무 잘 처리해서 의도하지 않은 용도로 쓰이는 경우가 있다.

### 문자열은 다른 값 타입을 대신하기에 적합하지 않다.

많은 경우 파일, 네트워크, 키보드 입력으로부터 데이터를 받을 때 문자열을 사용한다. 

그러나 입력받을 데이터가 문자열일 때만 그렇게 하는 것이 좋다.

- 받은 데이터가 수치 라면?
	- int, float, BigInteger 등 적합한 수치 타입으로 변환해주어야 한다.
- 예, 아니오 같은 데이터라면?
	- 적절한 열거 타입이나 boolean으로 변환해야 한다.  

***기본 타입이든 참조 타입이든 적절한 값 타입이 존재한다면, 그것을 사용하고 없을 경우 새로 작성하자.***

### 문자열은 열거 타입을 대신하기에 적합하지 않다.

상수를 열거할 때는 문자열보다 열거 타입이 월등히 낫다. 오타 발생 시 컴파일러 체크 불가 등 문제가 많다.

[아이템 34](https://ldhapple.github.io/posts/EffectiveJava-Item34/)에서 이미 사례를 보았다.

### 문자열은 혼합 타입을 대신하기에 적합하지 않다.

여러 요소가 혼합된 데이터를 하나의 문자열로 표현하는 것은 좋지 않은 생각이다.

```java
String compoundKey = className + "#" + i.next();
```

- 실제 시스템에 존재하는 코드라고 한다.
- 두 요소를 구분해주는 #이 className이나 i.next()에 쓰였다면 혼란스러울 것이다.
- 각 요소를 개별적으로 접근하려면 문자열을 파싱해야 해서 느리고, 귀찮으며 오류 가능성 또한 커진다.
- 적절한 equals, toString, compareTo 메서드를 제공할 수 없다. String이 제공하는 기능에만 의존해야 한다.

이런 경우 전용 클래스를 새로 만드는 편이 낫다. 이러한 클래스는 보통 private 정적 멤버 클래스로 선언한다.

```java
public class example {

	private static class CompoundKey {
		private final String className;
		private final String identifier;
		//...
	}

	public static void main(String[] args) {
		//...
	}
}
```

### 문자열은 권한을 표현하기에 적합하지 않다.

예를 들어 쓰레드 지역변수 기능을 설계한다고 가정하자. 쓰레드가 자신만의 변수를 갖게 해주는 기능이다.

자바 2부터 이 기능을 지원하지만, 그 전에는 프로그래머가 직접 구현했는데, 그 당시 클라이언트가 제공한 문자열 키로 쓰레드별 지역 변수를 식별하도록 설계했다.

문제가 뭘까?

```java
public class ThreadLocal {
	private ThreadLocal() { } // 객체 생성 방지

	// 현 쓰레드의 값을 키로 구분해 저장한다.
	public static void set(String key, Object value);

	// 키가 가리키는 현 쓰레드의 값을 반환한다.
	public static Obejct get(String key);
}
```

- 쓰레드 구분용 문자열 키가 전역 이름 공간에서 공유되는 것이 문제이다.
- 각 클라이언트가 고유한 키를 전달해야 하는데, 만약 같은 키를 사용하게 된다면?
	- 의도치 않게 같은 변수를 공유한다.
	- 두 클라이언트 모두 제대로 기능하지 못한다.
- 보안에 취약하다.
	- 악의적인 클라이언트라면 의도적으로 같은 키를 사용해 다른 클라이언트의 값을 가져올 수 있다. 

위 API는 키를 문자열을 사용하는 대신 위조할 수 없는 키를 사용하면 문제는 해결된다.

```java
public class ThreadLocal {
    private ThreadLocal() {}

    public static class Key {
        key() {}
    }

    // 위조 불가능한 고유 키를 생성
    public static Key getKey() {
        return new Key();
    }

    public static void set(Key key, Object value);
    public static Object get(Key key);
}
```

- 이 방법은 문자열 기반의 Key 사용의 문제를 해결해주긴하지만 개선할 여지가 있다.
- set, get 메서드는 더 이상 정적 메서드일 필요가 없다.
	- Key 클래스의 인스턴스 메서드로 바꾸면 된다.
	- 이렇게 하면 Key는 더 이상 단순 쓰레드 지역변수를 구분하기 위한 역할이 아닌, 그 자체가 쓰레드 지역 변수가 되는 것이다.

이렇게 개선하면 현 톱레벨 클래스인 ThreadLocal은 이제 할 일이 없기 때문에, 중첩 클래스 Key의 이름을 ThreadLocal로 바꾸면 된다.

```java
public final class ThreadLocal {
    public ThreadLocal();
    public void set(Object value);
    public Object get();
}
```

위 코드도 개선할 부분이 있다. get()으로 얻은 Object를 실제 타입으로 형변환해야 하기 때문에 타입 안전하지 못하다.

```java
public final class ThreadLocal<T> {
    public ThreadLocal();
    public void set(T value);
    public T get();
}
```

- 최종적으로 문자열 기반의 키를 사용한 API보다 더 빠르고, 발생했던 문제들도 해결할 수 있었다.
- 결론은 권한에 대한 표현을 하고 싶을 때, 굳이 문자열을 사용할 필요가 없다는 점이다.

## 정리

마구잡이로 문자열을 사용하지 말고, 더 적합한 데이터 타입이 있다면 그 타입으로 변환해 사용하자.

그러한 타입이 없다면 새로 작성하는 편이 낫다. 문자열은 문자를 다루는 용도로만 사용하자.