---
title: Item13 (clone 재정의는 주의해서 진행하라)
author: leedohyun
date: 2024-07-04 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## clone 재정의는 주의해서 진행하라

```java
public interface Cloneable {  
}
```

Cloneable은 복제해도 되는 클래스임을 명시하는 용도의 믹스인 인터페이스(mixin interface) 이다.

그런데 Cloneable 인터페이스에는 메서드조차 하나 없다. clone() 메서드는 Obejct 클래스에 protected로 구현되어 있다.

```java
@IntrinsicCandidate  
protected native Object clone() throws CloneNotSupportedException;
```

- protected이기 때문에 Cloneable을 구현하는 것 만으로는 외부 객체에서 clone 메서드를 호출할 수 없다.
- 리플렉션을 사용하면 가능하지만, 100% 성공을 보장하지 않는다.
	- 해당 객체가 접근이 허용된 clone 메서드를 제공한다는 보장이 없기 때문이다.

이러한 문제점들에도 Cloneable 방식은 널리 쓰이고 있어 잘 알아두는 것이 좋다.

### Cloneable의 쓰임

아무 메서드도 없는 이 인터페이스는 놀랍게도 Object의 clone() 동작 방식을 결정한다.

- Cloneable을 구현한 클래스의 인스턴스에서 clone을 호출하면 그 객체의 필드들을 하나하나 복사한 객체를 반환한다.
- 구현하지 않은 클래스의 인스턴스에서 clone을 호출하면 CloneNotSupportedException을 던진다.

참고로 이렇게 인터페이스를 사용하는 것은 이례적이니 사용하지 않는 것이 좋다.

### Cloneable의 구현

```java
class Point implements Cloneable {
	private int y;
	private int x;

	@Override
	public Point clone() throws CloneNotSupportedException {
		return (Point) super.clone();
	}
}
```

- Object의 clone 메서드는 protected이기 때문에 public으로 재정의해주어야 한다.
 
***명세에서는 이야기 하지 않지만 실무에서 Cloneable을 구현한 클래스는 위와 같이 clone 메서드를 public으로 제공하며, 그 사용자는 당연히 복제가 제대로 이루어 질 것이라고 기대한다.***

그러나 이 기대를 만족시키려면 그 클래스와 모든 상위 클래스는 복잡하고, 강제할 수 없고, 허술하게 기술된 프로토콜을 지켜야만 한다.

그 결과로는 깨지기 쉽고, 위험하고, 모순적인 메커니즘이 탄생한다. 

**생성자를 호출하지 않고도 객체를 생성할 수 있게 된다.**

### Cloneable의 문제점

#### A클래스의 clone을 호출할 때 상위 클래스에서 정의한 clone이 호출된다면 A 클래스가 아닌 상위 클래스의 객체가 반환된다.

```java
class Parent implements Cloneable {
	private int value;

	//...

	@Override
	public Parent clone() throws CloneNotSupportedException {
		return (Parent) super.clone();
	}
}

class Child extends Parent {
	private String name;
	
	//...
	
	@Override
	public Child clone() throws CloneNotSupportedException {
		return (Child) super.clone(); //공변 반환 타이핑
	}
}
```
```java
Child child = new Child(10, "Lee");

Child clonedChild = child.clone();
```

- 이를 처리하기 위해 공변 반환 타이핑 (재정의한 메서드의 반환 타입은 부모 클래스의 메서드가 반환하는 타입의 하위 타입일 수 있다.) 을 이용할 수는 있다.

#### super.clone()을 호출하는 방식의 clone은 동일 참조의 필드를 전달해 오류가 생길 수 있다.	

```java
class Point implements Cloneable {
	private int y;
	private int x;

	//...

	@Override
	public Point clone() throws CloneNotSupportedException {
		return (Point) super.clone();
	}
}

class Line implements Cloneable {
	private Point start;
	private Point end;
	
	//...

	@Override
	public Line clone() throws CloneNotSupportedException {
		Line clonedLine = (Line) super.clone();
		cloned.start = this.start.clone();
		cloned.end = this.end.clone();
		return clonedLine;
	}
}
```

- 이를 막기 위해 위와 같이 재귀적으로 필드에 대한 clone()을 호출할 필요가 있다.

#### 그 외 문제

- CloneNotSupportedException을 체크 예외로 던져 예외 처리를 반드시 하도록 명시한다.
	- Cloneable을 구현해 예외가 발생할 가능성이 없는 코드에서도 마찬가지.
- 생성자와 동일한 역할을 해 생성자의 역할을 모호하게 한다.

### 가변 객체를 참조하는 객체의 올바른 clone 구현 방법

가변 객체를 참조하지 않는 경우에는 일반적인 clone() 구현 방법을 사용해도 별 문제가 없다.
그러나 가변 객체를 참조하는 경우는?

```java
class Stack {
	private Object[] elements;

	@Override
	public Stack clone() throws CloneNotSupportedException {
		return (Stack) super.clone();
	}
}
```

- 복제된 객체와 원본 객체가 동일한 주소의 elements 배열을 참조하는 문제가 발생한다.
	- 복사 대상인 객체가 불변인 경우 문제가 되지 않지만 가변인 경우 동일한 객체를 참조해 한쪽에서의 변경이 다른 쪽에도 영향을 미치게 되어 문제가 발생한다.

```java
class Stack {
	private Object[] elements;
	
	//...
	
	@Override
	public Stack clone() {
		Stack result = (Stack) super.clone();
		result.elements = elements.clone();
		return result;
	}
}
```

- 위와 같은 구현의 문제점이 하나 있다. 
	- elements가 final이 아니어야 한다.
	- 가변 객체를 참조하는 필드는 final로 선언하라는 일반 용법과 충돌한다.
	- 결론적으로 복제 가능한 클래스 구현을 위해서는 필드에서 부득이하게 final을 제거해야 될 수도 있다.

#### HashTable의 경우의 예외

```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        Object key, value;
        Entry next;

        Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
        }
    }

    @Override 
    public HashTable clone() {
        HashTable result = (HashTable) super.clone();
        result.buckets = new Entry[buckets.length];
        
        for (int i = 0; i < buckets.length ; i++) {
            if (buckets[i] != null) {
                result.buckets[i] = buckets[i].deepCopy();
            }
        }
        return result;
    }
}
```

- 필드인 배열이 가지고 있는 값이 연결 리스트의 첫 번째 엔트리이다.
	- 단순히 배열을 복제한다고 해서 원본 객체와 복제본 객체가 분리되지 않는다.
	- 연결리스트를 구성하는 엔트리 객체가 동일한 인스턴스가 되기 때문에 복제된 객체의 연결리스트가 수정되면 원본 객체의 연결 리스트도 영향을 받게 된다.

따라서 위와 같이 연결 리스트를 구성하는 모든 엔트리를 새롭게 생성해주는 방식으로 복사를 해야 한다.

### 주의 사항

- 배열의 deep copy
	- arr.clone()으로 배열을 복사하면 새로운 배열을 만들고 기존 배열의 원소를 채워넣어 반환해준다.
	- 그런데 만약 배열이 갖는 값이 참조 객체인 경우 해당 객체의 값을 수정하면 원본 배열의 객체가 같이 변하게 된다.
	- 따라서 깊은 복사를 원한다면 새롭게 배열을 만들고 내부 원소들을 순회하며 원소들을 clone()해 넣어주어야 한다.
- 쓰레드 안전 클래스
	- Object.clone()은 멀티 쓰레드 환경을 고려하지 않았다.
	- 따라서 멀티 쓰레드 환경에 안전한 클래스를 만들기 위해서는 clone() 메서드가 아무런 작업을 하지 않더라도 재정의하고 동기화를 해줘야 한다.
- clone() 에서는 재정의 가능한 메서드를 호출해서는 안된다.
	- clone()에서 재정의 가능한 메서드를 호출하게 되면 하위 클래스에서 super.clone()을 호출 했을 때 상위 클래스에서의 호출임에도 하위 클래스의 재정의 된 메서드를 호출하게 되고 예측할 수 없는 객체가 복사된다.

## Clone 보다 나은 방법 (복사 생성자, 복사 팩토리)

확장하려는 클래스가 Cloneable을 구현한 경우 어쩔 수 없이 clone()을 재정의 해주어야 한다.

하지만 그렇지 않은 상황이라면 ***복사 생성자, 복사 팩토리***라는 더 나은 방식이 있다.   

```java
public class Point {
	public Point(Point point) {
		//복사 로직
		return copy;
	}
}
```
```java
public class Point {
	public static Point copy(Point point) {
		//복사 로직
		return copy;
	}
}
```

- 이 방식은 인자를 받기 때문에 구현 클래스가 아닌 인터페이스도 받을 수 있다.
	- 따라서 해당 인터페이스를 구현하는 클래스끼리는 다른 구현 클래스로의 복사도 가능하다.
	- ex) HashSet 객체를 TreeSet 객체로 복제 가능

## 정리

Cloneable은 다양한 문제가 있어 새로운 인터페이스를 만들고자 한다면 Cloneable을 확장하지 않는 것이 좋다.

- 얕은 복사 문제
	- 가변 객체의 복사 따로 구현
- 상속 관계에서의 문제
- 예외 처리 문제
- 생성자와의 혼동 문제
- Thread Safe 문제
	- 가변 상태의 객체를 컨트롤해야 한다.

새로운 클래스도 이를 구현해서는 안된다.

final 클래스라면 상속이 막혀있기 때문에 비교적 위험이 크지 않지만 이 경우에도 성능 최적화 관점에서 검토 후 문제가 없을 때만 드물게 허용해야 한다.

***기본적으로 복제 기능을 구현하는 것은 생성자와 팩토리를 이용하는 것이 가장 좋은 방법이다.***
