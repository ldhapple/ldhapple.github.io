---
title: Item30 (이왕이면 제네릭 메서드로 만들라)
author: leedohyun
date: 2024-07-14 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 이왕이면 제네릭 메서드로 만들라

클래스처럼 메서드도 제네릭으로 만들 수 있다.

매개변수화 타입을 받는 정적 유틸리티 메서드는 보통 제네릭으로 만들어진다.

```java
public static <E extends Enum<E>> EnumSet<E> of(E e) {  
	  EnumSet<E> result = noneOf(e.getDeclaringClass());  
	  result.add(e);  
	  return result;  
}
```

그런데 메서드(형 변환을 해야하는 메서드)를 왜 제네릭 메서드로 만들어야 할까? 이유를 알아보자.

### 제네릭 메서드로 만들어야 하는 이유?

```java
public static <E extends Enum<E>> EnumSet<E> of(E e) {  
	  EnumSet<E> result = noneOf(e.getDeclaringClass());  
	  result.add(e);  
	  return result;  
}
```

우선 위의 EnumSet 정적 유틸리티 메서드는 왜 제네릭일지 생각해보자.

- EnumSet에는 어떤 Enum 타입이 들어와도 된다.
- 그러나 하나의 EnumSet에는 같은 Enum 타입임이 보장되어야 한다.
- 따라서 타입을 제네릭으로 만들어 사용한 것이다.

만약 제네릭을 쓰지 않고 로 타입을 쓰게 된다면?

```java
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1); // 경고!
    result.addAll(s2); // 경고!
    return result;
}
```

- 로(raw) 타입을 사용하고 있다.
- 매개변수의 타입이나, 반환 타입으로 로 타입을 사용하면, 컴파일은 가능하지만 경고가 발생한다.
	- 위의 두 경고를 없애려면 메서드를 타입 안전하게 만들어야 한다.
	- 제네릭 메서드로 수정하면 된다.

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet(s1); // 경고!
    result.addAll(s2); // 경고!
    return result;
}
```

- 이 제네릭 메서드는 경고 없이 컴파일이 가능하고, 쓰기도 쉽다.
- 직접 형변환하지 않아도 어떤 오류나 경고 없이 컴파일 된다.

종합적으로 기존에 했던 얘기들과 같은 맥락이다. **타입 안전성을 위한 부분과 타입 캐스팅을 번거롭게 하지 않아도 되기 때문이다.**

### 제네릭 메서드 만드는 법

메서드에 제네릭을 사용할 때도 클래스에 제네릭을 사용하는 것 처럼 타입이 오는 자리에 제네릭을 넣어주면 된다.

다만 메서드에 제네릭을 사용할 때는 메서드의 제한자와 반환 타입 사이에 타입 매개변수 목록을 넣어주어야 한다.

```java
public static <E extends Enum<E>> EnumSet<E> of(E e) { 
```

- public static과 EnumSet 사이에 타입 매개변수를 괄호 사이에 넣어 명시해주어야 한다.

```java
public class Stack<E> {

    private E[] elements;

    //...

    public E pop() { // 지정안해도 됨!
        if (size == 0) {
            throw new EmptyStackException();
        }
        E result = elements[--size];
        elements[size] = null;
        return result;
    }

    //...
}
```

- 단, 클래스가 제네릭을 사용하는 클래스라면 클래스 내 메서드에서 같은 제네릭을 사용할 때는 타입 매개변수를 지정해주지 않아도 된다.

### 제네릭 싱글톤 팩토리

제네릭은 런타임 시점에 타입이 소거된다고 했다. 이로 인해 하나의 객체를 어떤 타입으로든 매개변수화 할 수 있다.

하지만 이를 구현하기 위해서는 요청한 타입 매개변수에 맞게 매번 그 객체의 타입을 바꿔주는 정적 팩토리를 만들어야 한다.

말로는 조금 이해가 어려울 수 있다. 코드를 보자.

```java
public class GenericSingletonFactory {
	private static final Function<Object, Object> IDENTITY_FN = (t) -> t;

	@SuppressWarnings("unchecked")
	public static <T> Function<T, T> identityFunction() {
		return (Function<T, T>) IDENTITY_FN;
	}
}
```

- IDENTITY_FN은 입력 값을 그대로 반환하는 함수이다.
- 이러한 항등함수 객체는 상태가 없다. 따라서 요청할 때 마다 객체를 새로 생성하는 것은 낭비이다.
	- 제네릭이 만약 실체화되었다면 항등함수를 타입별로 하나씩 만들어야 했다.
- IDENTITY_FN을 형변환하면 경고가 발생한다.
	- 하지만 입력값을 단순 반환하는 항등함수임으로 타입이 안전하기 때문에 경고를 제거해줄 수 있다.
- 하나의 Function 객체를 어떤 타입으로든 매개변수화 한 것이다. 
	- 요청한 타입에 맞게 해당 객체의 타입을 바꾸어주는 역할을 한 것.

```java
Function<String, String> fun = GenericSingletonFactory.identityFunction();

for (String s : strings) {  
  System.out.println(fun.apply(s));  
}

Function<Integer, Integer> fun = GenericSingletonFactory.identityFunction();

for (Integer i : ints) {  
  System.out.println(fun.apply(i));  
}
``` 

- 형변환을 하지 않아도 컴파일 오류나 경고가 발생하지 않는다.

### 재귀적 타입 한정

자기 자신이 들어간 표현식을 사용해 타입 매개변수의 허용 범위를 한정할 수 있다는 개념이다.

주로 타입의 자연적 순서를 정하는 Comparable 인터페이스와 함께 쓰인다.

```java
public static <E extends Comparable<E>> E max(Collection<E> c) {
    ...
}
```

- Collection의 요소들 중 가장 큰 요소를 반환하는 max() 메서드이다.
	- 가장 큰 요소를 반환하려면 매개변수로 받아오는 컬렉션의 요소들은 비교, 정렬이 가능해야 한다.
- 타입 한정을 보면 모든 타입 E는 자신과 비교할 수 있는 E 이다. 라는 의미를 내포한다. 
	- 자기 자신 E를 포함한 표현식으로 타입 매개변수의 허용 범위를 Comparable이 구현된 타입으로 한정한 것이다.

## 정리

메서드에서도 제네릭을 사용하지 않고 로 타입이나 Object 타입을 사용하게 되면 사용할 때마다 형변환을 해주어야 하며, 타입 안전성에서도 문제가 생길 확률이 있다.

**따라서 형변환을 해줘야 하는 메서드들은 제네릭을 사용하도록 수정하거나, 처음부터 그렇게 만들자.**

기존 클라이언트의 코드도 건드리지 않으면서도 새로운 사용자의 편의성 그리고 타입 안전성을 모두 챙길 수 있는 방법이다.