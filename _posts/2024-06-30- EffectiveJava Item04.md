---
title: Item4 (인스턴스화를 막으려거든 private 생성자를 사용하라)
author: leedohyun
date: 2024-06-30 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 인스턴스화를 막으려거든 private 생성자를 사용하라

정적 메서드와 정적 필드만을 담은 클래스는 남용될 수 있어 안좋게 볼 수 있으나 때때로 유용하게 쓰일 수 있다.

### 정적 메서드와 정적 필드만을 담은 클래스

- 기본 타입 값이나 배열 관련 메서드를 모은 클래스

java.util.Arrays를 예시로 들 수 있다.

```java
public class Arrays {
	public static boolean isArray(Object o)
	public static void sort(long[] a)
	public static boolean equals(int[] a, int[] a2)
	public static List<Object> asList(Object array)
	public static <T> boolean isNullOrEmpty(T[] array)
	
	//...
	private Arrays() { }
```

Arrays 클래스를 살펴보면 위와 같은 정적 메서드들이 매우 많이 정의되어 있다.

그런데 우리가 사용할 때

```java
Arrays array = new Arrays();
```

이렇게 생성하지는 않는다.

```java
Arrays.sort(arr);
Arrays.asList(arr);
```

정적 메서드이기 때문에 이러한 방식으로 사용한다.

- 특정 인터페이스를 구현하는 객체를 생성해주는 정적 메서드 / 팩토리를 모은 클래스

java.util.Collections를 보자.

```java
public class Collections {
	public static <T extends Comparable<? super T>> void sort(List<T> list)
	public static <K,V> SortedMap<K,V> unmodifiableSortedMap(SortedMap<K, ? extends V> m)

	//...
	private Collections() { }
}
```

```java
Set<Object> objects = Collections.emptySet();
```

- final 클래스와 관련된 메서드들을 모아놓는 경우

final 클래스는 상속이 금지되어 있다.

상속을 통해 기존 클래스의 동작을 변경할 경우 보안이나 동작에 있어 예기치 못한 에러가 발생할 수 있어 final 클래스를 사용한다.

우리가 많이 사용하는 Math 클래스가 그 예시이다.

```java
public final class Math {

	@IntrinsicCandidate  
	public static double sqrt(double a) {...}

	public static long round(double a) {...}

	@IntrinsicCandidate 
	public static long abs(long a) {...}

	private Math() {}
}
```

Math 클래스의 역할은 수학적 계산을 수행하는 정적 메서드들을 제공하는 역할이다. 따라서 단순히 입력을 받아 계산을 수행하여 반환하는 것이 목적이므로 상속을 통해 기능의 확장이나 변경을 할 필요가 없다.

거기에 만약 상속이 가능해 오버라이딩을 열어두면, 명확한 계산과 답이 정해져있음에도 안정성과 일관성을 해칠 수 있다.

따라서 final로 선언해 이러한 문제들을 예방한다.

## 그래서?

이 3가지 경우 모두 공통적으로 private 생성자로 인스턴스화를 막아두었고, 메서드들이 static으로 선언되어 바로 호출할 수 있다.

이렇게 정적 멤버만 담은 유틸리티 클래스는 인스턴스로 만들어 사용하려고 설계한 것이 아니다.

따라서 인스턴스화를 막아두어야 한다.

### 만약 생성자를 따로 두지 않는다면?

컴파일러가 자동으로 public 기본 생성자를 만들어준다.

이 부분 때문에 의도치 않게 인스턴스화를 열어둔 클래스가 보인다.

### 추상 클래스로 만들어서 인스턴스화를 막는다면?

결론부터 말하면 추상 클래스로 만드는 것으로 인스턴스화를 막을 수 없다.

물론 abstract 클래스는 인스턴스화 하는 것이 불가능하기 때문에 이러한 생각이 들 수 있다.

- 그러나 하위 클래스를 생성하면 인스턴스화가 가능해진다.
- 더군다나 추상클래스는 보통 공통 필드나 공통 메서드를 정의해두고 상속해서 쓰라는 뜻으로 오해하기 쉽다.

## 정리

정적 멤버만을 담은 유틸리티 클래스는 private 생성자를 통해 인스턴스화를 막아주어야 한다.

다른 방법으로는 인스턴스화를 막을 수 없다.

```java
private UtilityClass() {
	throw new AssertionsError();
}
```

이런식으로 에러를 던져주면 클래스 내부에서 실수로라도 생성자를 호출하는 것을 방지해줄 수 있다.

이렇게 인스턴스화를 막았을 때의 장점을 마지막으로 정리한다.

- 불필요한 객체 생성을 막음
	- 메모리 낭비 방지
- 안정성 보장
	- 인스턴스화를 하면 예상치 못한 동작을 유발할 수 있다.
- 상속 막음
	- 유틸리티 클래스는 보통 기능의 확장이나 변경이 필요하지 않다. 상속에 의한 변경을 막는다.  