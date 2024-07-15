---
title: Item29 (이왕이면 제네릭 타입으로 만들라)
author: leedohyun
date: 2024-07-14 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 이왕이면 제네릭 타입으로 만들라

제공되는 제네릭 타입과 메서드를 사용하는 일은 쉽다. 하지만 제네릭 타입을 새로 만드는 일은 쉽지만은 않다.

```java
public class Stack {
	private Object[] elements; //문제!
	private int size = 0;
	private static final int DEFAULT_INITIAL_CAPACITY = 10;

	public Stack() {
		elements = new Object[DEFAULT_INITIAL_CAPACITY];
	}

	public void push(Object e) {
		ensureCapacity();
		elements[size++] = e;
	}

	public Object pop() {
		if (size == 0) throw new EmptyStackException();
		Object result = elements[--size];
		elements[size] = null; // (복습) 다 쓴 참조 해제
		return result;
	}

	//...
}	
```

[아이템 7](https://ldhapple.github.io/posts/EffectiveJava-Item07/)에서 다뤘던 Stack 코드이다. 이제는 아이템 28을 학습했고 위의 코드의 문제를 알 수 있다.

```java
Stack stack = new Stack();
stack.push(1);
String str = (String) stack.pop(); // 문제!
```

- 매번 형변환을 해줘야 한다.
- ClassCastException이 발생한다.
	- 컴파일에는 문제를 찾지 못하고, 런타임 시 문제가 발생한다.

그렇다면 제네릭을 써야한다. 어떻게 만들지 알아보자.

### 배열 + 제네릭 타입으로..

```java
public class Stack<E> {
	private E[] elements;
	private int size = 0;
	private static final int DEFAULT_INITIAL_CAPACITY = 16;

	public Stack() {
		elements = new E[DEFAULT_INITIAL_CAPACITY]; // 문제!
	}

	public void push(E e) {
		ensureCapacity();
		elements[size++] = e;
	}

	public E pop() {
		if (size == 0) throw new EmptyStackException();
		E result = elemtns[--size];
		elements[size] = null;
		return result;
	}
	
	//...
}
```

- 위 코드에서의 문제가 보여야 한다. 보이지 않는다면 아이템 28을 제대로 학습하지 않은 것이다.
- 실체화 불가 타입으로는 배열을 만들 수 없다.
	- 배열은 실체화되어 런타임에도 자기 타입을 계속 확인한다.
	- 반면 리스트는 런타임에는 타입 정보가 소거됐었다.

그렇다면 배열을 사용하면서도 제네릭을 어떻게 사용할 수 있을까?

#### 우회방법 1. 배열을 Object로 생성하고 제네릭으로 형변환

```java
@SuppressWarnings("unchecked")
public Stack() {
	elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
}
```

- 비검사 형변환이 안전함을 확신한다면 범위를 좁혀 @SuppressWarnings로 경고를 숨긴다.
- 배열 elements는 private 필드에 저장되고, 클라이언트로 반환되거나 다른 메서드에 전달되는 일이 없다.
	- push 메서드를 통해 배열에 저장되는 원소의 타입은 항상 E다.
	- **따라서 이 사례의 비검사 형변환은 안전하다.**

#### 우회 방법 2. elements의 타입을 Object[]로 바꾸고 pop() 에서 형변환

```java
public E pop() {
	if (size == 0) throw new EmptyStackException();

	@SuppressWarnings("unchecked")
	E result = (E) elements[--size];
	
	elements[size] = null;
	return result;
}
```

- 배열이 반환한 원소를 E로 형변환한다.
	- 그러면 오류대신 경고가 뜬다.

#### 두 방법의 비교

두 방법은 모두 어느정도 지지를 받고 있다. 그렇다면 둘 중 어느 방법이 더 좋을까?

- 첫 번째 방법
	- 가독성이 더 좋다. 
		- 배열의 타입을 E[]로 선언해 오직 E 타입 인스턴스만 받음을 어필한다.
	- 그러나 런타임에는 E[]가 아닌 Object[]로 동작한다는 단점이 있다.
		- 제네릭은 타입 소거되어 Object로 바뀐다.
	- 힙 오염의 가능성도 존재한다.
		- 런타임에 Object[]로 동작하기 때문
- 두 번째 방법
	- 처음부터 Object[] 이기 때문에 힙 오염의 가능성이 없다.
		- pop() 메서드의 타입 안전성은 push 시 E 타입만 들어오기 때문에 Object[]에 저장되는 요소가 모두 E 타입임이 보장되어 괜찮다.
	- 배열에서 원소를 읽을 때 마다 형변환을 해줘야 한다.

현업에서는 주로 첫 번째 방식을 더 선호하고 자주 사용한다.

하지만 힙 오염이 걱정된다면 두 번째 방법을 사용하곤 한다.

> 힙 오염?

제네릭 타입을 사용할 때 발생할 수 있는 문제이다.

**컴파일 시점에서는 경고나 에러가 발생하지 않지만 런타임 시점에 ClassCastException 등의 예외를 발생시킬 수 있는 상황을 의미한다.**

아이템 28에서 많이 보았던 경우이다.

### 그런데 배열보다는 리스트를 쓰라며..?

위 Stack 예시는 제네릭을 사용하면서도 배열을 쓰는 예시를 보여준다. 리스트를 사용하라던 아이템 28과 대비되는 예시이다.

그런데 자바가 리스트를 기본 타입으로 제공하지 않기 때문에 ArrayList 같은 제네릭 타입도 결국 기본 타입인 배열을 사용해 구현한다. 또한 HashMap 같은 제네릭 타입은 성능을 높일 목적으로 배열을 사용하기도 한다.

따라서 배열 + 제네릭을 우회하는 방법을 설명하는 것이다.

### 제네릭 타입 사용 시 주의사항

Stack의 예시처럼 대다수의 제네릭 타입은 타입 매개변수에 아무런 제약을 두지 않는다.

```java
Stack<Object>
Stack<int[]>
Stack<List<String>>
```

등등 어떤 참조 타입으로도 Stack을 만들 수 있다.

그러나 주의 사항이 있다.

- 타입 매개변수에 제약이 없다. 단, 기본 타입은 안된다.
	- 기본 타입을 사용할 시 박싱 타입으로 우회한다.
- 타입 매개변수에 제약을 두는 제네릭 타입도 있다.
	- ex) Collections.binarySearch()
	- Comparable을 구현한 타입만 List의 타입으로 사용 가능하다.

```java
public static <T>  
int binarySearch(List<? extends Comparable<? super T>> list, T key) {  
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)  
        return Collections.indexedBinarySearch(list, key);  
	else return Collections.iteratorBinarySearch(list, key);  
}
```

## 정리

클라이언트에서 직접 형변환을 해야하는 타입보다는 제네릭 타입을 사용하는 것이 더 안전하고 편하다.

따라서 새로운 타입을 설계할 때 제네릭 타입을 사용하는 것이 좋다. 기존 코드 중 제네릭 타입이었어야 하는 코드들이 있다면 수정해보자.

제네릭과 배열을 같이 사용해야 하는 경우에도 우회 방법이 있으니, 상황에 따라 선택해서 구현해보자.

물론 리스트를 사용할 수 있는 경우에는 리스트를 사용해도 좋다.