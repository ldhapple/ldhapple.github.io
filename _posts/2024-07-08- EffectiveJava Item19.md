---
title: Item19 (상속을 고려해 설계하고 문서화하라. 그렇지 않으면 상속을 금지하라.)
author: leedohyun
date: 2024-07-08 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 상속을 고려해 설계하고 문서화하라. 그렇지 않으면 상속을 금지하라.

이전 아이템에서 상속을 염두에 두지 않고 설계한 상태로 어떤 클래스를 상속할 때 발생하는 문제 또한 문서화해놓지 않은 외부 클래스를 상속할 때의 위험성이 주된 이유였다. 

어떤 동작을 모르고 하위 클래스에서 재정의 혹은 새로운 구현을 했다가 상위 클래스의 변경 때문에 하위 클래스에서 오류가 발생한 것이다.

이러한 변경은 예를 들어 기획자에 의해 바뀔 수 있는 등 프로그래머의 통제권 밖에 있어 언제 어떻게 변경될 지 모르기 때문에 발생하는 문제이다.

그렇다면 상속을 고려한 설계와 문서화는 무엇을 뜻하길래 상속을 가능하게 해줄까?

### 상속을 고려한 설계와 문서화

#### API 문서

- 메서드를 재정의하면 어떤 일이 일어나는지 정확히 정리해 문서로 남겨야 한다.
- 상속용 클래스는 재정의할 수 있는 메서드들을 내부적으로 어떻게 이용하는 지, 어떤 상황에서 호출될(할) 수 있는지 문서로 남겨야 한다.
- final이 아닌 모든 메서드들에 대해 메서드가 호출되는 순서, 연쇄되어 호출되는 메서드 등 모든 상황을 문서로 남겨야 한다.

> API 문서의 메서드 주석의 예시

- implementation Requirements
	- 메서드 내부 동작 방식을 설명하는 곳을 의미한다.
	- 메서드 주석에 @implSpec 태그를 붙여주면 자바독 도구가 생성해준다.

```java
public abstract class AbstractCollection<E> implements Collection<E> {

/**  
  * {@inheritDoc}  
  *  
  * @implSpec  
  * This implementation iterates over the collection looking for the  
  * specified element.  If it finds the element, it removes the element * from the collection using the iterator's remove method. * * <p>Note that this   implementation throws an  
  * {@code UnsupportedOperationException} if the iterator returned by this  
  * collection's iterator method does not implement the {@code remove}  
  * method and this collection contains the specified object.  
  * * @throws UnsupportedOperationException {@inheritDoc}  
  * @throws ClassCastException {@inheritDoc}  
  * @throws NullPointerException {@inheritDoc}  
*/  
public boolean remove(Object o) { //...
}
``` 

- 이 메서드는 컬렉션을 순회하며 주어진 원소를 찾도록 구현되었다.
- 주어진 원소를 찾으면 반복자의 remove 메서드를 사용해 컬렉션에서 제거한다.
- 이 컬렉션이 주어진 객체를 갖고 있으나 이 컬렉션의 iterator 메서드가 반환한 반복자가 remove 메서드를 구현하지 않았다면 UnsupportedOperationException을 던진다.

AbstractCollection의 remove() 메서드 위에 적혀있는 주석이다.

설명에 따르면 iterator 메서드를 재정의 할 경우 remove 메서드의 동작에 영향을 끼칠 수 있음을 확실하게 알 수 있다.

> 좋은 API

좋은 API 문서는 '어떻게'가 아닌 '무엇'을 하는지를 설명해야 한다.

그런데 위의 AbstractCollection의 remove() 메서드의 주석이나, 상속을 고려한 문서화에 대한 설명을 보면 무엇을 하는지보다는 '어떻게'에 대한 설명이 주를 이룬다.

하지만 상속은 캡슐화를 해치기 때문에 어쩔 수 없다. 클래스를 안전하게 상속할 수 있도록 하려면 내부 구현 방식을 설명해주어야 한다.

#### 상속 설계 시 훅(hook)을 선별할 수 있어야 한다.

훅(hook)이란 클래스 내부 동작 과정 중간에 끼어들 수 있는 메서드를 의미한다.

상속을 설계할 때 이러한 훅을 잘 선별해 protected 메서드 형태로 공개해야 할 수 있다.

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {

	protected void removeRange(int fromIndex, int toIndex) {  
		ListIterator<E> it = listIterator(fromIndex);  
		for (int i=0, n=toIndex-fromIndex; i<n; i++) {  
			it.next();  
			it.remove();  
		}  
	}
}
```

- fromIndex부터 toIndex까지의 모든 원소를 리스트에서 제거하는 메서드이다.
	- toIndex 이후의 원소들은 앞으로 당겨진다.
- 훅은 이 리스트의 부분리스트에 정의된 clear() 연산이 이 메서드를 호출하는 것이다. 
	- 간단하게 말하면 clear() 연산에서 해당 메서드를 호출한다.
- 해당 메서드를 공개해 하위 클래스에서 clear() 메서드를 고성능으로 만들기 쉽게 한다.
	- 해당 공개 메서드를 재정의하거나 사용하여 하위클래스의 clear() 메서드를 효율적으로 구현할 수 있다.

만약 해당 메서드를 공개하지 않았다면 하위 클래스에서 clear() 메서드 호출 시 성능이 느려지거나 부분리스트의 메커니즘을 새로 구현해야 했을 것이다.

참고로 어떤 메서드를 protected로 공개해야 할 지는 하위 클래스를 구현해봐야 알 수 있는 일이다. 테스트를 위해서는 하위 클래스를 구현해야 한다.

#### 상속용으로 설계한 클래스는 배포 전에 반드시 하위 클래스를 만들어 검증해야 한다.

널리 사용될 클래스를 상속용으로 설계한다면 문서화한 내부 사용 패턴과 protected 메서드와 필드를 구현하면서 선택한 결정에 영원히 책임져야 함을 인식해야 한다.

따라서 상속용으로 설계한 클래스는 배포 전에 반드시 하위 클래스를 만들어 검증해보고 문제가 없음을 확인해야 한다.

#### 상속용 클래스의 생성자에서 재정의 가능한 메서드를 호출 X

상위 클래스의 생성자는 하위 클래스의 생성자보다 먼저 실행된다. 

따라서 상위 클래스의 생성자에서 하위 클래스에서 재정의한 메서드가 호출된다면 하위 클래스의 재정의 메서드가 하위 클래스의 생성자보다 먼저 호출되는 상황이 벌어질 수 있다.

```java
class Class1 {
    public Class1() {
        test();
    }

    public void test() {
        System.out.println("test");
    }
}

class Class2 extends Class1 {
    private String string;

    public Class2() {
        string = "override!";
    }

    @Override
    public void test() {
        System.out.println(string);
    }
}
```

```java
Class2 class2 = new Class2();  
class2.test();
```

- 위 결과가 어떻게 나타날까?
	- Class2를 생성할 때 Class1의 생성자가 실행되면서 test() 메서드를 수행하니 override가 출력될 것 같다.
	- 그리고 test()를 수행하면 또 한번 "override"가 출력될 것 같다.

그러나 결과는 null을 출력하고 override를 출력한다. Class2의 생성자가 호출되기 전에 test() 메서드가 호출되어 String 값이 초기화되지 않았기 때문이다.

따라서 상속용 상위 클래스에서는 생성자에서 재정의 가능한 메서드를 호출해서는 안된다.

#### Cloneable과 Serializable 인터페이스를 조심해야 한다.

위 두 인터페이스를 구현한 클래스를 상속 가능하게 설계하는 것은 좋지 않다.

Cloneable의 clone(), Serializable의 readObject()는 새 객체를 생성하는 생성자와 비슷한 효과를 낸다.

따라서 그 제약도 같다. 클래스의 상태가 초기화되기 전에 메서드부터 호출될 수 있다는 뜻이다. **재정의 가능한 메서드의 호출을 해서는 안된다.**

> Serializable을 구현한 상속용 클래스

Serializable을 구현한 상속용 클래스가 readResolve(), writeReplace() 메서드를 가질 때는 protected로 선언해야 한다.

private으로 선언 시 하위 클래스에서 무시되기 때문이다.

이 또한 상속을 허용하기 위해 내부 구현을 클래스 API로 공개하는 예시 중 하나이다.

### 상속.. 과연 선택할 수 있을까?

클래스를 상속용으로 설계하기 위해 엄청난 노력이 들고, 해당 클래스에 제약도 생기는 것을 확인할 수 있었다.

이를 거치지 않으면 클래스에 변화가 생길 때마다 하위 클래스를 오동작하게 만들 수 있다.

이 문제를 해결하는 가장 좋은 방법은 **상속용으로 설계하지 않은 클래스는 애초에 상속을 금지하는 것**이다.

그런데 인터페이스를 구현한 경우 상속을 금지해도 개발하는데 불편함이 없지만 그렇지 않은 경우 상속을 금지하면 불편할 수 있다.

이를 해결하기 위해서는 **클래스 내부에서 재정의 가능 메서드를 사용하지 않게 만들고 이를 문서화**하면 된다.

메서드를 재정의하더라도 다른 메서드의 동작에 아무런 영향을 미치지 않게 개발하면 된다.

> 상속 금지 방법(복습)

- 클래스를 final로 선언
- 모든 생성자를 private으로 막고 정적 팩토리 메서드 사용

> 클래스 내부에서 재정의 가능 메서드를 사용하지 않게 만드는 방법

클래스 동작을 유지하면서 재정의 가능한 메서드를 사용하는 코드를 제거해야 된다면 이 방법을 사용할 수 있다.

재정의 가능 메서드를 private 형식의 도우미 메서드로 옮기는 방법이다.

```java
class Class1 {
    public Class1() {
        helper();
    }

    public void test() {
        helper();
    }

    private void helper() {
        System.out.println("test");
    }
}

class Class2 extends Class1 {
    private String string;

    public Class2() {
        string = "override!";
    }

    @Override
    public void test() {
        System.out.println(string);
    }
}
```

## 정리

상속용 클래스를 설계하기 위해서는 문서화, 클래스 제약 등을 극복해야 한다.

그렇지 않으면 상위 클래스의 변경이 하위 클래스에 영향을 미치기 쉬워 예기치 못한 오류를 발생시키게 된다.

따라서 상속용 클래스로 설계하지 않았다면 상속을 막는 것이 좋다. 

만약 상속을 막아서 개발에 불편함이 생긴다면 인터페이스를 활용하자. 인터페이스를 구현한 클래스는 상속이 없어도 불편함이 없다. (컴포지션을 활용하는 방법도 있다.)