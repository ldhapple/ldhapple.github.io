---
title: Item32 (제네릭과 가변인수를 함께 쓸 때는 신중하라)
author: leedohyun
date: 2024-07-17 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 제네릭과 가변인수를 함께 쓸 때는 신중하라

가변인수 메서드는 제네릭과 같이 자바 5때 함께 추가되었다. 그래서 서로 잘 어우러질 것이라 생각할 수 있지만 그렇지 않다.

```java
public static void printNumbers(int... numbers) { ... }
```

- 이러한 가변 인수 메서드가 호출되면 가변 인수를 담기 위해 배열이 자동으로 하나 만들어진다.
- 실체화 불가 타입(ex - 제네릭)은 런타임에 컴파일 타임보다 타입 정보가 적다.

위 두 정보를 기억하고, 왜 제네릭과 가변인수 메서드를 같이 쓰면 문제가 발생하는 지 알아보자.

### 힙 오염의 발생

```java
public static void method(List<String>... stringLists) {
	List<Integer> intList = List.of(42);
	Object[] objects = stringLists;
	objects[0] = intList; // 힙 오염 발생
	String s = stringLists[0].get(0); // ClassCastException - 컴파일러가 형변환
```

- 제네릭 타입의 변수에 다른 타입의 객체를 할당하게 되어 힙 오염이 발생한다.
	- String List -> Integer List 할당
	- String으로 선언했지만, Integer 객체를 참조하게 된다. (ClassCastException)

가변 인수와 제네릭을 같이 쓰면 왜 힙오염이 생기는걸까?

- 가변 인수 메서드에서 제네릭 타입을 사용할 때, 위에서 언급했듯 컴파일러는 가변 인수를 담기 위해 배열을 생성한다.
	- 제네릭 배열을 생성해야 한다.
- 그러나 자바에서는 제네릭 배열은 허용하지 않는다. (타입 안전성 - 컴파일 타임 / 런타임)
	- 따라서 Object 배열을 생성한다.
	- 그리고 이 Object 배열을 제네릭 타입 배열로 캐스팅한다.
	- 코드를 예시로 보면 Object배열의 Object는 사실상 String List인 것이다.
	- 따라서 컴파일 에러는 발생하지 않지만, 런타임에 문제가 생길 수 있는 것이다.

***이렇게 타입 안전성이 깨지기 때문에 제네릭 varargs 배열 매개변수에 위와 같이 값을 저장하는 것은 안전하지 않다.***

> 그런데 제네릭 배열의 생성은 막아뒀으면서 왜 제네릭 varargs 매개변수를 받는 메서드는 허용했을까?

제네릭이나 매개변수화 타입의 varargs 매개변수를 받는 메서드가 실무에서 매우 유용하기 때문이다.

자바 라이브러리에서도 이런 메서드를 여럿 제공한다.

```java
Arrays.asList(T... a)
Collections.addAll(Collection<? super T> c, T... elements) //소비자!
EnumSet.of(E first, E... rest)
```

### @SafeVarargs

제네릭 가변 인수 메서드를 작성하면 클라이언트에 경고를 준다.

그러나 메서드 작성자가 타입 안전을 확신한다면?

작성자는 @SafeVarargs 애노테이션을 사용해 경고를 지워 타입 안전함을 보장할 수 있다.

@SafeVarargs를 사용하는 것에 기본적인 방침이 있다.

***제네릭이나 매개변수화 타입의 varargs 매개변수를 받는 모든 메서드에 @SafeVarargs를 달면 된다.***

컴파일러 경고를 없애준다. 위 방침의 핵심은 타입 안전하지 않은 제네릭 varargs 메서드는 작성조차 하지 말라는 것이다.

> 메서드가 타입 안전한지 확신하는 방법

- 메서드가 가변인수를 담는 배열에 저장하는 로직이 없을 것. (아무것도 추가로 저장해서는 안된다.)
- 가변 인수를 담은 배열의 참조가 밖으로 노출되지 않을 것.

```java
static <T> T[] toArray(T... args) {
	return args; // X!
}
```

- 이렇게 반환하면 args 배열의 타입은 해당 메서드에 인수를 넘기는 컴파일 타임에 결정된다.
	- 그런데 그 시점에는 컴파일러에게 충분한 정보가 주어지지 않아 타입을 잘못 판단할 수 있다.

위 메서드만 보고는 잘 이해가 안될 수 있다. 저장하는 경우는 어디서 타입 안전성이 깨지는 지 확인했다. 그런데 이 가변 인수 배열의 참조가 밖으로 노출되면 어디서 문제가 발생할까?

해당 메서드를 사용해서 타입 안전성이 깨질 수 있는 부분을 보자.

```java
public static <T> T[] pickTwo(T a, T b, T c) {
	switch (ThreadLocalRandom.current().nextInt(3) {
		case 0: return toArray(a, b);
		case 1: return toArray(a, c);
		case 2: return toArray(b, c);
	}
	throw new AssertionError();
}

public static void main(String[] args) {
	String[] argsArray = pickTwo("굿", "abc", "String");
}
```

- 전부 String 타입의 매개변수를 넘겨 메서드를 호출했기 때문에 문제가 없어보인다. 가변인수 배열에 어떤 값을 저장하지도 않았다. 컴파일에서도 문제가 발견되지 않는다.
- 그런데 실행하면 ClassCastException이 발생한다.
	- pickTwo() 메서드는 Object[]를 반환한다.
		- 제네릭을 담기에 가장 구체적인 타입이 Object이기 때문이다.
	- Object[]로 반환된 pickTwo의 반환 결과값이 컴파일러에 의해 String[]으로 자동 형변환된다.
		- Object[]는 String[]의 하위 타입이 아니기 때문에 문제 발생! 
	
그러나 타입 안전을 확신할 수 있는 두 가지 경우에서 예외가 있다.

- @SafeVarargs가 제대로 사용된 또 다른 가변인수 메서드로 넘기는 경우
- 배열의 일부를 가변인수 메서드를 받지 않는 일반 메서드에 넘기는 경우

```java
@SafeVarargs
public static <T> List<T> flatten(List<? extends T>... lists) {
	List<T> result = new ArrayList<>();
	for (List<? extends T> list : lists) {
		result.addAll(list);
	}
	return result;
}
```

### @SafeVarargs의 대안?

@SafeVarargs가 유일한 해결책은 아니다.

실체는 배열인 varargs 매개변수를 List 매개변수로 바꿀 수도 있다.

```java
static <T> List<T> flatten(List<List<? extends T>> lists) { ... }
```

```java
public static <T> List<T> pickTwo(T a, T b, T c) {
	switch (ThreadLocalRandom.current().nextInt(3) {
		case 0: return List.of(a, b);
		case 1: return List.of(a, c);
		case 2: return List.of(b, c);
	}
	throw new AssertionError();
}

// String[] argsArray = pickTwo("굿", "abc", "String");
List<String> attributes = pickTwo("굿", "abc", "String");
```

- List로 대체했을 경우 메서드의 타입 안전성을 컴파일러가 검증해준다.
- 애노테이션을 달 필요가 없다.
- 작성자의 실수로 안전하지 않은데 안전하다고 판단할 경우가 없다.

결과적으로 배열이 사용되지 않아 컴파일러가 타입 안전을 보장해준다.

이 방법은 가변인수를 사용하는 것에 비해 당연히 단점도 존재한다.

- 클라이언트 코드가 지저분해진다.
- 속도가 조금 더 느리다.

## 정리

가변인수와 제네릭은 같이 사용했을 때 문제가 발생할 수 있다.

가변인수가 배열로 생성되어 제네릭 배열이 생기기 때문이다. 

제네릭 배열은 배열은 실체화되고 제네릭은 실체화 불가 타입이기 때문에 런타임 시 제네릭은 타입 정보가 사라져 아무 타입이나 들어가 컴파일러의 자동 형변환에 의해 예외가 발생하는, 즉 배열의 타입 안전성이 깨지게 된다.

이를 해결할 수 있는 방법은 타입 안전성을 보장할 수 있는 해당 제네릭 배열에 값을 저장하지 않거나, 그 배열을 밖으로 공개하지 않을 경우 @SafeVarargs 애노테이션으로 경고를 지우는 방법이 있다.

혹은 가변인수 (배열) 대신 리스트를 사용해 타입 안전성을 애초에 지키면서 사용할 수도 있을 것이다.

결론적으로는 가변인수와 제네릭을 함께 사용할 경우는 타입 안전에 신경을 써야한다.