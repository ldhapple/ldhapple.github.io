---
title: Item7 (다 쓴 객체 참조를 해제하라) feat. 약한 참조와 강한 참조
author: leedohyun
date: 2024-07-01 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 다 쓴 객체 참조를 해제하라

자바는 JVM을 통해 가비지 컬렉터를 갖추어 메모리 관리에 신경을 쓰지 않아도 될 것 같다.

하지만 메모리 관리에 신경을 써야한다. (메모리 누수에 신경써야 한다.)

## 메모리 누수?

가비지 컬렉터로 관리해주는데 어디서 메모리의 누수가 발생하고 왜 신경써야 하는 걸까?

아래 스택을 구현한 코드를 보자.

```java
public class Stack {
	private Object[] elements;
	private int size = 0;
	private static final int DEFAULT_INITIAL_CAPACITY = 16;

	public Stack() {
		elements = new Object[DEFAULT_INITIAL_CAPACITY];
	}

	public void push(Object e) {
		ensureCapacity();
		elements[size++] = e;
	}
	
	public Object pop() {
		if (size == 0) {
			throw new EmptyStackException();
		}
		return elements[--size];
	}
	
	private void ensureCapacity() {
		if (elements.length == size) {
			elements = Arrays.copyOf(elements, 2 * size + 1);
		}
	}
```

- 원소를 위한 공간을 마련하는 ensureCapacity() 메서드에서 배열 크기를 늘려야 할 때 2배씩 늘린다.
- 그런데 스택이 커졌다가 줄어들었을 때 스택에서 꺼내진 객체들을 가비지 컬렉터가 회수하지 않는다.
	- 프로그램에서 그 객체들을 더 이상 사용하지 않더라도 회수하지 않는다.
	- 스택 객체가 그 객체들의 ***다 쓴 참조***를 여전히 갖고 있기 때문이다.

이렇게 가비지 컬렉션 언어에서는 의도치 않게 객체를 살려둠으로써 발생하는 메모리 누수를 찾기 매우 까다롭다.

객체 참조를 하나 살려두면 가비지 컬렉터는 해당 객체 뿐만 아니라 그 객체가 참조하는 모든 객체를 회수하지 못한다. 또 그렇게 살아남은 객체가 참조하는 객체들도 마찬가지이다.

위 예시에서의 Stack은 정리되지 않는다. Stack의 Object 배열인 elements가 참조하는 Object들의 다 쓴 참조를 갖고 있기 때문이다.

> 다 쓴 참조

앞으로 다시 쓰지 않을 참조를 뜻한다.

위 예시에서는 elements 배열의 활성 영역 (인덱스가 size보다 작은 원소들로 구성) 밖의 참조들을 가리킨다.

## 그래서 어떻게 해결하나?

가장 쉬운 방법은 다 쓴 참조를 null 처리해주는 것이다.

### 다 쓴 참조를 null 처리하자

```java
public Object pop() {
	if (size == 0) {
		throw new EmptyStackException();
	}
	return elements[--size];
} 

//다 쓴 참조 처리
public Object pop() {
	if (size == 0) {
		throw new EmptyStackException();
	}

	Object result = elements[--size];
	elements[size] = null;
	
	return result;
}
```

스택 구현 클래스를 예시로 보면 위와 같다. 해당 참조를 다 썼을 때 null 처리 해주어 참조 해제를 한다.

이렇게 다 쓴 참조를 null 처리했을 때 또 다른 장점도 있다. 다른 참조를 사용하려할 때 NullPointerException을 발생시킬 것이다.

***그러나 객체 참조를 null 처리하는 것은 예외적인 경우여야 한다.***

다 쓴 객체 참조를 해제하는 가장 좋은 방법은 해당 참조를 담은 변수를 유효 범위 밖으로 밀어내는 것이다. 변수의 범위를 최소화되도록 정의했다면 자연스럽게 이루어진다.

Stack 구현 클래스 내부에서 null처리를 하지 않고 pop() 메서드를 통해 객체를 외부로 반환하고, 그 반환된 객체를 외부 변수에서 사용하다가 그 외부 변수가 유효 범위 밖으로 밀어내진다면 자연스럽게 객체 참조가 해제되어 GC 대상이 된다.

```java
{
	//...
	Object o = stack.pop();

	//...
}

// 위의 범위를 벗어나게 되면 객체 참조 해제
```

> 그렇다면 null 처리는 언제?

우선 위의 Stack 클래스가 메모리 누수에 취약한 이유를 알아야 한다.

Stack 클래스는 자기 메모리를 직접 관리하고 있다. Stack 구현 클래스는 elements 배열로 저장소 풀을 만들어 원소들을 관리하고 있다.

배열의 활성 영역에 속한 원소들이 사용되고 비활성 영역은 쓰이지 않는데 가비지 컬렉터가 이를 알 수가 없다.

가비지 컬렉터의 입장에서는 비활성 영역에서 참조하는 객체도 결국 참조하고 있기 때문에 GC 대상이라고 파악하지 못하는 것이다.

***일반적으로 위와 같이 자기 메모리를 직접 관리하는 클래스라면 위와 같은 부분을 프로그래머밖에 모르기 때문에 항상 메모리 누수에 주의해야 한다.***

원소를 다 사용한 즉시 원소가 참조한 객체를 null 처리 해주어야 한다.

## 캐시에서의 메모리 누수

캐시 또한 메모리 누수를 일으키는 주범이다.

객체 참조를 캐시에 넣은 후 그 객체를 다 쓴 뒤에도 캐시에 놔두면서 발생하게 된다.

```java
public class CacheExample {
	private static Map<String, Object> cache = new HashMap<>();

	private static void addCache(String key, Object value) {
		cache.put(key, value);
	}

	private static Object getCache(String key) {
		return cache.get(key);
	}
	//..
}
```

위와 같이 캐시를 구현했다고 가정해보자.

캐시에 객체를 추가하고, 캐시에서 객체를 꺼내 사용한다. 그런데 다 사용하고도 계속 해당 key값의 객체를 제거하지 않고 Map에 남겨두면 메모리 누수가 발생하게 되는 것이다.

이를 해결할 수 있는 방법은 여러가지가 있다.

### WeakHashMap 사용

캐시 외부에서 키를 참조하는 동안만 엔트리가 살아 있는 캐시가 필요한 상황이라면 WeakHashMap을 사용해보자.

다 쓴 엔트리는 자동으로 제거된다.

```java
public class CacheExample {
	private static Map<String, Object> cache = new WeakHashMap<>();

	private static void addCache(String key, Object value) {
		cache.put(key, value);
	}

	private static Object getCache(String key) {
		return cache.get(key);
	}
	//..
}
```

Key가 강한 참조가 아닌 약한 참조로 저장된다. 이로 인해 키가 가리키는 객체가 다른 곳에서 더 이상 강한 참조를 가지지 않는다면 GC 대상이 된다.

- 캐시를 만들 때 보통 캐시 엔트리의 유효 기간을 정확하게 정의하기 어렵다.
	- 시간이 지날수록 엔트리의 가치를 떨어뜨리는 방식 등을 채택해 사용하곤 한다.
	- 백그라운드 스레드나 캐시에 새 엔트리를 추가할 때 정리하는 방식도 활용한다.

> 엔트리?

WeakHashMap 내부에서 각 요소는 Map.Entry 라는 객체로 저장된다.

- Key (약한 참조)
- Value (강한 참조)
- Hash code (키의 해시 코드)

엔트리 객체는 위와 같은 구성으로 되어 있다. 

```java
Map<Key, String> weakCache = new WeakHashMap<>();

Key key1 = new Key("key1");
weakCache.put(key1, "value");

Key strongReference = key1; //key1 객체를 다른 변수에 할당해 강한 참조
key1 = null;

strongReference = null;

System.gc();
```

엔트리는 키에 대해 약한 참조를 가지고 있으므로 키가 다른 곳에서 강한 참조를 가지지 않게 되면 해당 엔트리는 GC 대상이 된다.

위의 예시에서는 Key에 대한 강한 참조를 가지고 있던 strongReference 변수가 null이 될 때 강한 참조가 사라지면서 GC 대상이 되고, 해당 엔트리 또한 GC 대상이 된다.

> 약한 참조 (Weak Reference)

강한 참조는 일반적으로 new 할당 후 새로운 객체를 만들어 해당 객체를 참조하는 방식이다. 할당된 객체를 해지하기 위해 null 값을 넣어도 새로운 객체가 참조를 하고 있어 GC 대상이 되지 않는다.

약한 참조는 메모리에 객체가 있을 수 있을 정도로만 유지하는 강하지 않은 참조이다.

WeakReference를 이용해 new 할당된 객체를 참조하는 방식이다. 할당된 객체를 해지하기 위해 null 값을 넣을 경우 WeakReference 객체가 참조하고 있어도 GC의 대상이 되고 WeakReference 객체는 null 값을 가지게 된다.

약한 참조로 저장된 객체는 다른 곳에서 강한 참조가 없을 경우 GC 대상이 된다는 뜻이다.

## 콜백이나 리스너의 메모리 누수

클라이언트가 콜백(특정 이벤트 처리 등)을 등록만 해놓고 명확히 해지하지 않는다면, 어떤 조치가 있지 않는 이상 콜백은 계속 쌓이게 된다.

이런 경우 콜백을 약한 참조로 저장하면 가비지 컬렉터가 즉시 수거해 간다.

## 정리

약한 참조와 강한 참조에 대한 개념을 이해하고, 메모리 누수가 발생하지 않도록 이 개념들을 이용해 미리 메모리 누수 문제를 예방하는 것이 중요하다.

메모리 누수는 겉으로 잘 드러나지 않기 때문이다.