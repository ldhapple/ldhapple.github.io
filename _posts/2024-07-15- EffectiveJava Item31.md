---
title: Item31 (한정적 와일드카드를 사용해 API 유연성을 높이라)
author: leedohyun
date: 2024-07-15 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 한정적 와일드카드를 사용해 API 유연성을 높이라

[아이템 28](https://ldhapple.github.io/posts/EffectiveJava-Item28/)에서 매개변수화 타입(제네릭)은 불공변이라고 했다. 공변은 런타임에 잘못된 원소를 넣는 오류를 잡아내는 것에 반해 불공변 특성으로 컴파일 타임에 잡아냈었다.

```java
List<String>은 List<Object>의 하위 타입이 아니고, 그 반대도 상위 타입이 아니다.
```

- Object 리스트에는 어떤 객체든 넣을 수 있지만, String 리스트에는 String만 넣을 수 있다.
	- 즉 String 리스트는 Object 리스트가 하는 일을 제대로 수행하지 못한다.
	- 리스코프 치환 원칙에 어긋난다.

그러나 이러한 불공변 특성은 타입 안전성을 지켜주지만, 비교적 유연하지 않다. 유연하게 설계해야 하는 경우가 있을 것이다.

```java
public class Stack<E> {
	private E[] elements;
	
	//...
	
	public void push(E e) {
		ensureCapacity();
		elements[size++] = e;
		
	public void pushAll(Iterable<E> src) {
		for (E e : src) {
			push(e);
		}
	}
}
```

- Stack 클래스에 pushAll() 이라는 메서드를 추가했다.
	- 문제 없이 컴파일 된다.

하지만 만약 Iterable src의 원소 타입이 스택의 원소 타입과 일치하지 않는 경우라면?

```java
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ...;
numberStack.pushAll(integers);
```

문제없이 동작할 것 같지만 아래와 같은 오류가 발생한다.

![](https://blog.kakaocdn.net/dn/bl4cj8/btsIAVvzC1m/UgqnpRKYgqPK8bVfDDAGx0/img.png)

- 제네릭 타입이 불공변이기 때문이다.
- Number가 Integer의 상위 타입이라도 제네릭이 불공변이기 때문에 서로 관계가 없는 타입이 된다.

이런 경우 유연함을 가지고 싶을 수 있다. 자바는 이런 상황에 대처할 수 있는 한정적 와일드카드 타입이라는 특별한 제네릭 타입을 지원한다.

### 한정적 와일드카드 타입 사용

```java
public void pushAll(Iterable<? extends E> src) {
	for (E e : src) {
		push(e);
	}
}
```

- 이제 Number 스택에 Integer Iterator를 pushAll()해도 오류가 발생하지 않는다.
- ? extends E의 뜻은 E를 포함한 E의 하위 타입의 Iterable 이어야 한다는 뜻이다.
- 한정적 와일드카드 타입을 사용해 유연하게 pushAll()을 사용할 수 있게 됐다.

그렇다면 popAll()은 어떨까?

```java
public void popAll(Collection<E> dst) {
	while (!isEmpty()) {
		dst.add(pop());
	}
}
```

- Stack 내의 모든 원소를 주어진 컬렉션으로 옮겨 담는 역할을 하는 메서드이다.
- 단, 이렇게 구현하면 마찬가지로 꺼낸 원소의 타입이 컬렉션의 원소 타입과 동일하다면 문제가 없지만, 그렇지 않다면 똑같이 컴파일 에러가 발생한다.

```java
public void popAll(Collection<? super E> dst) { ... }
```

- 마찬가지로 한정적 와일드카드 타입을 사용해 E 상위 타입의 컬렉션이라고 변경해주면 된다.
- extends와 super의 차이가 발생하는 이유는 pushAll()은 원소들을 스택에 담아야 하기 때문에 스택의 하위타입 원소를 원하는 것이고, popAll()은 스택의 원소 타입 (E)를 컬렉션에 담아야 하기 때문에 컬렉션은 E의 상위 타입이어야 하기 때문이다.

정리하자면 ***유연성을 극대화하려면 원소의 생산자나 소비자용 입력 매개변수에 와일드카드 타입을 사용하라는 것이다.***

> 주의사항

입력 매개변수가 생산자와 소비자 역할을 동시에 한다면 와일드카드 타입을 써도 좋을게 없다.

```java
public static void eating(List<? extends Animal> animals) {
    for (Animal animal : animals) {
        animal.eat(); //Animal 소비
    }
	
	animals.add(new Dog()); //Animal 생산(추가) -> 불가능. 컴파일 에러 발생
}
```

- 매개변수가 소비와 생산을 둘 다 하고 있는 간단한 메서드이다.
- 그러나 이 때 생산, 즉 add()는 불가능하다. 와일드카드 타입은 쓰기 작업이 제한되어 있다.
	- animals가 실제로 어떤 타입의 리스트를 참조하고 있는지 알 수 없다.
	- 어떤 타입인지 확실하게 알아야 한다.

물론 이렇게 입력 매개변수를 통해 생산과 소비를 둘 다 적용하는 경우는 매우 드물 것이다. 

하지만 만약 이런 경우가 있다면 와일드카드 타입을 쓰지 말고 아래와 같이 쓰는게 낫다.

```java
public static void eating(List<Animal> animals) {
     for (Animal animal : animals) {
         animal.eat();
     }

     animals.add(new Dog());
}
```

### PECS 공식 (producer-extends, consumer-super)

이 공식은 와일드카드 타입을 사용할 때 헷갈리지 않고 수월하게 사용할 수 있도록 만든 것이다.

제네릭 타입 T가 생산자라면 (? extends T)를 사용하고, 소비자라면 (? super T)를 사용하라는 것이다.

여기서 생산자와 소비자의 개념이 조금 헷갈릴 수 있다. 

- 생산자: 제네릭 타입 T 타입의 객체를 생성하거나 반환하는 작업을 수행
- 소비자: 제네릭 타입 T 타입의 객체를 인자로 받아 사용하거나 저장하는 작업을 수행

```java
//생성자
public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2) { ... }
```

- 두 Set을 합쳐 E 타입의 객체를 생성해 반환한다.
	- 이를 위해서는 E 하위 타입의 Set들을 받아 E 타입의 Set으로 묶을 수 있어야 한다.
- 참고로 반환타입은 E 이다.
	- **반환 타입에는 한정적 와일드카드 타입을 사용하면 안된다.** 
	- 반환 타입에 와일드카드 타입을 사용하면 클라이언트 코드에서도 와일드카드 타입을 써야한다.

```java
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
Set<Number> numbers = union(integers, doubles);
```

- 이렇게 제대로 사용한다면 사용자는 와일드카드 타입이 쓰였다는 사실조차 의식하지 못한다.
	- 받을 수 있는 매개변수, 받을 수 없는 매개변수를 알아서 필터링 해준다.
- 만약 클래스 사용자가 와일드카드 타입을 신경써야 된다면 그 API에는 문제가 있을 확률이 높다.

이제 소비자로 사용되는 경우를 보자. 해당 제네릭 타입 객체를 사용하거나 저장하는 작업이다. 

```java
//소비자

public static <E> void addToSet(Set<? super E> set, E element) { 
	set.add(element);
}
```

- set에 E 타입의 객체인 element를 추가하는 역할을 한다. set 객체를 인자로 받아 저장하는 역할을 수행한다. 따라서 소비자이다.

```java
Set<Number> set1 = new HashSet<>();
Set<Object> set2 = new HashSet<>();

addToSet(set1, 1234);
addToSet(set2, "String");
```

- 제네릭은 불공변임에도 와일드카드 타입을 이용해 유연하게 하위 타입의 원소를 추가할 수 있게 만들었다.

조금 더 복잡한 경우를 보자.

```java
public static <E extends Comparable<? super E>> E max(List<? extends E> list)
{ ... }
```

- list에서 최댓값의 크기를 반환하는 메서드이다.
	- 최댓값의 크기를 반환하려면 비교할 수 있어야 하기 때문에 Comparable을 구현한 타입이어야 한다.
- Comparable은 언제나 소비자이다.
	- 타입 E를 인자로 받아 사용한다. (비교) 
	- 따라서 super.
- list는 왜 생산자일까?
	- (가장 큰 값) E 인스턴스를 생성하고 반환한다. 
	- 따라서 extends.

**결론은 생산자는 extends, 소비자는 super 이다.**

### 메서드 선언에 타입 매개변수가 한 번만 나오면 와일드카드로 대체하라.

제네릭과 와일드카드에는 공통되는 부분이 많아 메서드를 정의할 때 둘 중 어느 것을 사용해도 괜찮을 때가 있다.

```java
public static <E> void swap(List<E> list, int i, int j); //타입 매개변수 (제네릭)
public static void swap(List<?> list, int i, int j); //와일드카드
```

겉으로 보기엔 둘 다 문제없이 잘 동작할 것 같다. 둘 중 어떤 방법이 나을까?

기본 규칙은 ***메서드 선언에 타입 매개변수가 한 번만 나오면 와일드카드로 대체하는 것***이다.

```java
public static void swap(List<?> list, int i, int j) {
	list.set(i, list.set(j, list.get(i)));
}
```

- 그러나 와일드카드 방식에서 문제가 있다.
- list.get(i)를 set하는 과정에 컴파일 에러가 발생한다.
	- 방금 꺼낸 원소를 리스트에 넣을 수 없다.
	- 와일드카드 타입의 List에는 null 외에는 어떠한 값도 넣을 수 없다. 일반적으로 읽기 작업만 허용된다.

이럴땐 도우미 메서드를 사용하면 된다. 와일드카드 타입의 실제 타입을 알려주는 private 메서드이다.

```java
public static void swap(List<?> list, int i, int j) {
	swapHelper(list, i, j);
}

private static <E> void swapHelper(List<E> list, int i, int j) {
	list.set(i, list.set(j, liset.get(i)));
}
```

- swapHelper 메서드는 리스트가 E 타입이라는 것을 알고 있다.
- 따라서 E 타입의 값이라면 list에 값을 넣어도 안전함이 보장된다.

> 그런데 이러면서까지 와일드 카드 타입을 써야되는 이유가 뭐지..?

- public API라면 와일드카드 방식이 낫다.
	- 간단하다. 메서드 시그니처를 단순화할 수 있고, 가독성이 높아진다.
	- 클라이언트는 내부 구조를 알 필요가 없기 때문에 API를 더 쉽게 읽을 수 있도록 하는 것이 중요하다.
	- 별도의 오버로드나 복잡한 유형의 계층 구조없이 어떤 List도 허용한다.

결론적으로 API에서 클라이언트를 생각했을 때 더 쉽게 읽을 수 있는 와일드카드 타입이 더 낫다는 것이다.

더해 그 가독성에 대한 이익이 도우미 메서드를 추가적으로 구현하는 것에 드는 비용보다 더 가치가 있다는 뜻이다.

## 정리

조금 복잡하더라도 와일드카드 타입을 사용하면 API가 유연해지고 간단해진다.

와일드카드 타입을 사용하는 공식인 PECS를 기억하자. 생산자는 Extends, 소비자는 Super.

더해 메서드 선언에 타입 매개변수가 한 번만 나오면 와일드카드로 대체하는 것이 좋은데, 이유는 사용하는 클라이언트 입장에서 메서드 시그니처가 더 간단하기 때문이다.