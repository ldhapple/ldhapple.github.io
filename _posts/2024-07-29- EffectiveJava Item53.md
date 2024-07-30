---
title: Item53 (가변 인수는 신중히 사용하라)
author: leedohyun
date: 2024-07-29 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 가변 인수는 신중히 사용하라

가변 인수 메서드를 호출하게 되면 가장 먼저 인수의 개수와 길이가 같은 배열을 만든다. 그리고 인수들을 이 배열에 저장해 가변인수 메서드에 건네주게 된다.

동일한 타입의 여러 매개변수를 받을 수 있도록 설계된 것이다.

개수에 제한도 없고 사용도 자유롭다.

단, 이렇게 편리한 가변인수는 주의해서 사용해야 한다.

가장 심플하게 주의해야 할 점은 하나의 메서드에는 한 개의 가변 인수만 사용이 가능하다는 점이다. 이 외의 주의 사항들을 보자.

### 인수가 1개 이상이어야 하는 가변 인수 메서드

예를 들어 최솟값을 찾는 메서드인데 인수를 0개만 받을 수 있도록 설계하는 것은 좋지 않다.

인수 개수는 런타임에 생성된 배열의 길이로 알 수 있다.

```java
static int min(int... args) {
	if (args.length == 0) {
		throw new IllegalArgumentException("인수가 1개 이상 필요합니다.");
	}

	int min = args[0];

	for (int i = 1; i < args.length; i++) {
		if (args[i] < min) { 
			min = args[i];
		}
	}
	return min;
}
```

- 이 방식은 어떤 문제가 있을까?
- 가장 심각한 문제는 인수를 0개만 넣어 호출했을 때 컴파일 타임이 아닌 런타임에 실패한다.
	- args 유효성 검사를 할 때 예외를 던지게 되기 때문에 호출 시점에 바로 알 수 없다.
	- 코드도 지저분하다.
- min 초깃값을 Integer.MAX_VALUE로 설정하지 않고는 보다 명료하게 작성할 수 있는 for-each문도 사용할 수 없다.

인수가 1개 이상이어야 할 때 가변인수를 올바르게 사용하는 방법으로는 매개변수를 2개 받도록 하면 된다.

```java
static int min(int firstArg, int... remainingArgs) {
	int min = firstArg;
	for (int arg : remainingArgs) {
		if (arg < min) min = arg;
	}
	return min;
}
```

- 가변 인수 1개를 필수로 받아야 하는 경우 매개변수를 2개 받도록 하여 해결할 수 있다.

### 성능이 민감한 경우

성능에 민감한 상황에서도 가변 인수는 주의해서 사용해야 한다.

가변 인수 메서드는 호출될 때 마다 배열을 하나 할당하고 초기화한다고 했다. 이 비용이 크게 다가오는 경우 다중 정의를 통해 해결할 수 있다.

```java
public void foo() { }
public void foo(int a1) { }
public void foo(int a1, int a2) { }
//...
public void foo(int a1, int a2, int a3, int... rest) { }
```

- 이렇게 작성하면 인수가 4개 이상인 호출만 가변 인수가 사용되어 비용을 줄일 수 있다.
- 일반적으로는 크게 이득이 되지 않지만 메서드가 많이 사용되는 경우 성능 개선을 가져올 수 있을 것이다.

이렇게 사용한 예시로 EnumSet의 정적팩토리를 들 수 있다.

```java
public static <E extends Enum<E>> EnumSet<E> of(E e) {  
   EnumSet<E> result = noneOf(e.getDeclaringClass());  
   result.add(e);  
   return result;  
}

public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2) {  
   EnumSet<E> result = noneOf(e1.getDeclaringClass());  
   result.add(e1);  
   result.add(e2);  
   return result;  
}

//...

@SafeVarargs  
public static <E extends Enum<E>> EnumSet<E> of(E first, E... rest) {  
    EnumSet<E> result = noneOf(first.getDeclaringClass());  
    result.add(first);  
    for (E e : rest)  
      result.add(e);  
    return result;  
}
```

- 성능이 민감한 경우와 인수를 1개 이상 받아야 할 때 해결책들을 모두 적용한 것을 볼 수 있다.

## 정리

가변 인수를 사용할 때는 주의해서 사용해야 한다. 인수 개수가 일정하지 않은 메서드를 정의하려면 반드시 사용되기 때문에 사용하지 않을 수는 없다. 

인수가 1개 이상 필요할 때, 그리고 성능을 고려할 때 생각할 부분들이 있다.



[제네릭과 가변인수 사용 시 힙 오염 발생](https://ldhapple.github.io/posts/EffectiveJava-Item32/)도 주의해야 한다.