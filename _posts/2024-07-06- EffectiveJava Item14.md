---
title: Item14 (Comparable을 구현할 지 고려하라)
author: leedohyun
date: 2024-07-06 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## Comparable을 구현할 지 고려하라

Comparable 인터페이스의 compareTo는 단순 동치성 비교에 더해 순서까지 비교할 수 있고 제네릭하다.

Comparable을 구현했다는 것은 그 클래스의 인스턴스들에는 자연적인 순서가 존재함을 뜻한다.

```java
class Point implements Comparable<Point> {  
	 private final int y;
	 private final int x;
	 private final int dist;  
	  
	 public Point(int y, int x, int dist) {  
		 this.y = y;  
		 this.x = x;  
		 this.dist = dist;  
	 }  
	  
	  @Override  
	  public int compareTo(Point other) {  
		  if (this.dist == other.dist) {  
			  if (this.y == other.y) {  
				  return this.x - other.x;  
			  }  
			  return this.y - other.y;  
		  }  
	  return this.dist - other.dist;  
	  }  
}
```

### CompareTo의 규약

기본적으로 객체가 주어진 객체보다 작으면 음의 정수를, 같으면 0을, 크면 양의 정수를 바노한한다.

만약 객체와 비교할 수 없는 타입의 객체가 주어지면 ClassCastException을 던진다.

- sgn(x.compareTo(y)) == -sgn(y.compareTo(x))
	- 비교 메서드의 부호 결과와 반대 비교 메서드의 부호 결과는 반대가 되어야 한다.
	- x.compareTo(y)는 y.compareTo(x)가 예외를 던질 때에 한해 예외를 던져야 한다.
- x.compareTo(y) > 0 && y.compareTo(z) > 0 이면 x.compareTo(z) > 0이어야 한다.
	- 추이성을 지켜야 한다.
- x.compareTo(y) = 0 이면 sgn(x.compareTo(z)) == sgn(y.compareTo(z)) 이다.  
- x.compareTo(y) = 0 이면 x.equals(y) 여야 한다.
	- 이 조건은 필수는 아니지만, 지키는 것이 좋다.
	- Comparable을 구현하고 이 권고를 지키지 않는 모든 클래스는 그 사실을 명시해야 한다.
	- 정렬된 컬렉션들은 동치성을 비교할 때 equals 대신 compareTo를 사용하기 때문에 이 규약을 지키는 것이 좋다.

> equals와의 공통점

규약이 equals와 비슷한 것을 볼 수 있다. 똑같이 반사성, 대칭성, 추이성을 충족해야 한다.

그래서 문제점도 같다.

- 기존 클래스를 확장한 구체 클래스에서 새로운 값 컴포넌트를 추가한다면 규약을 지킬 방법이 없다.
	- 기존 클래스와 확장 클래스를 비교하게 되면 확장 클래스의 추가된 값 때문에 대칭성, 추이성 등의 규약을 지키기 어렵다.
	- 물론 마찬가지로 객체 지향적 추상화의 이점을 포기하면 문제는 없다.
-  위 문제에 대한 우회법이 같다.
	- 확장 대신 독립된 클래스에 기존 클래스의 인스턴스를 가리키는 필드를 두면 된다.
	- 이후 그 필드에 대한 반환값을 제공하는 뷰 메서드를 제공하자.

### CompareTo 구현

compareTo의 규약이 equals와 비슷하기 때문에 메서드 작성 요령 또한 비슷하다.

하지만 compareTo는 제네릭 인터페이스이기 때문에 타입을 확인하거나 형변환할 필요가 없다. 인수의 타입이 잘못되면 컴파일이 되지 않는다.

#### 객체 참조 필드를 비교하는 방법

```java
public final class CaseInsensitiveString 
		implements Comparable<CaseInsensitiveString> {
	private final String s;
	
	public int compareTo(CaseInsensitiveString cis) {
		return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
	}
	//...
}	
```

- compareTo를 재귀적으로 호출한다.
	- 만약 Comparable을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야 한다면 비교자를 대신 사용한다.
	- 비교자는 직접 만들거나 자바가 제공하는 것 중 골라쓰면 된다.
	- 위 코드는 자바가 제공하는 비교자를 사용하고 있다.
		- CaseInsensitiveString용 compareTo 메서드이다.

#### 정수 기본 타입 필드를 비교하는 방법

- 기존에는 정수 기본 타입 필드를 비교할 때 관계 연산자인 (< , >)를 사용했다.
	- 실수 타입 필드는 Double.compare, Float.compare을 사용하도록 했다.
- ***하지만 자바 7이후 부터는 박싱된 기본 타입 클래스들에 새로 추가된 정적 메서드 compare을 사용해주면 된다.***
	- 비교 연산자의 사용은 오류가 발생할 수 있다.

```java
class Point implements Comparable {
	private final int y;
	private final int x;

	//...
	
	@Override
	public int compareTo(Point p) {
		int result = Integer.compare(y, p.y);
		if (result == 0) {
			result = Integer.compare(x, p.x);
		}

		return result;
	}
}
``` 

위 코드에서 더 체크해볼 경우는 핵심 필드가 여러 개인 경우이다.

> 핵심 필드가 여러 개인 경우

어느 것을 먼저 비교하는 지가 중요해진다.

가장 핵심이 되는 필드가 같다면 같지 않은 필드를 찾을 때 까지 그 다음으로 중요한 필드를 비교해나가는 방식으로 구현한다.

#### 비교자 생성 메서드를 활용한 비교자

```java
private static final Comparator<Point> COMPARATOR = 
		comparingInt((Point p) -> p.y)
			.thenComparingInt(p -> p.x);
	
public int compareTo(Point p) {
	return COMPARATOR.compare(this, p);
}
```

- 자바 8부터 제공된 비교자 생성 메서드이다.
	- 메서드 연쇄 방식으로 가독성 좋게 비교자를 생성할 수 있다.
	- 정적 메서드를 활용할 수도 있다.
- 가독성이 좋지만 주의할 점은 성능 저하가 뒤따르게 된다.
- Comparator는 수많은 보조 생성 메서드들을 가지고 있어 자바의 숫자용 기본 타입을 모두 커버한다.
	- 객체 참조용 비교자 생성 메서드도 있다. 

## 정리

순서를 고려해야 하는 값 클래스를 작성한다면 꼭 Comparable을 구현해 그 인스턴스들을 쉽게 정렬하고, 검색하고, 비교할 수 있도록 해주어야 한다. 컬렉션과 어우러져 시너지를 낼 수 있다.

- compareTo를 구현할 때는 규약을 지켜야 하며, 비교를 할 때 연산자의 사용보다 박싱타입의 compare 메서드를 쓰자.
	- 비교 연산자의 사용은 거추장스럽고 오류를 일으킬 수 있기 때문이다.
- Comparator 인터페이스가 제공하는 비교자 생성 메서드의 사용도 추천된다. 