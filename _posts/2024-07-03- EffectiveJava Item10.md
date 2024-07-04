---
title: Item10 (equals는 일반 규약을 지켜서 재정의하라)
author: leedohyun
date: 2024-07-03 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## Object

Object는 객체를 만들 수 있는 구체 클래스이다. 그러나 기본적으로는 상속해서 사용하도록 설계되어 있다.

Object에서 final이 아닌 메서드는 아래와 같다.

- equals
- hashCode
- toString
- clone
- finalize

위 메서드들은 모두 재정의를 염두에 두고 설계된 것이기 때문에 재정의 시 지켜야 하는 일반 규약이 명확히 정의되어 있다.

해당 메서드들을 규약에 맞지 않게 정의하면 대상 클래스가 이 규약을 지킨다고 가정하는 HashMap과 HashSet같은 클래스들에서 오동작할 수 있다.

이번 아이템부터 아이템 14까지는 이러한 Object 메서드들을 언제 어떻게 재정의해야 하는지에 대해 다룬다.

## equals는 일반 규약을 지켜 재정의하라

equals 메서드는 재정의하기 쉬워보인다. 하지만 곳곳에 함정이 있기 때문에 자칫하면 끔찍한 결과를 가져오게 된다.

문제를 회피하는 가장 쉬운 방법은 아예 재정의하지 않는 것이다.

재정의하지 않는게 나은 경우와 재정의 해야하는 경우를 나누어 알아보자.

### 재정의하지 않는 것이 최선인 경우

#### 각 인스턴스가 본질적으로 고유한 경우

값을 표현하는게 아닌 동작하는 개체를 표현하는 클래스가 해당된다.

```java
public class Thread implements Runnable {
	//...
	@Override
	public boolean equals(Object obj) {
		if (obj == this) return true;

		if (obj instanceof WeakClassKey) {
			Class<?> referent = get();
			return (referent != null) &&
					(((WeakClassKey) obj).refersTo(referent));
		} else {
			return false;
		}
	}
}
```

Thread 같은 경우 각 인스턴스가 독립적인 개체이고, 값이나 상태를 통해 동일성을 정의할 수 없다.

Thread 클래스는 각 객체가 서로 다른 쓰레드를 나타내고, 설령 동일한 상태를 갖고 있더라도 독립적인 서로 다른 개체이기 때문이다.

위 Thread 클래스의 equals 메서드를 보자.

- Thread 클래스는 각 객체가 고유한 리소스를 나타내기 때문에, equals를 재정의하더라도 단순하게 참조 비교를 통해 동일성을 확인하는 것이 적합하다.
- obj == this
	- 두 객체가 동일한 메모리 주소를 가리키는지 확인.
- obj instanceof WeakClassKey
	- WeakClassKey는 WeakReference를 사용해 약한 참조를 유지하는 키를 나타낼 때 사용한다.
- get(), refersTo()
	- get()으로 현재 객체가 참조하는 클래스를 가져온다.
	- 현재 객체와 obj가 참조하는 클래스가 동일한 클래스인지를 판단한다.

#### 인스턴스의 논리적 동치성을 검사할 일이 없을 때

예를 들어 Pattern 같은 경우 각 인스턴스가 같은 정규표현식을 나타내는지 equals 메서드를 재정의해서 판별할 수 있지만 그럴 필요가 없다.

일반적으로 패턴 자체를 비교하는 것이 목적이 아닌 문자열을 정규 표현식과 일치하는 지 판단하기 위해 사용하는 객체이기 때문이다. 따라서 기본 동작인 Object의 equals를 사용해도 해결된다.


> 논리적 동치성

두 개체가 메모리의 동일한 인스턴스인지의 여부가 아닌 동일한 개념의 인스턴스인지를 판별.

ex) 문자 순서가 일치하는 String 객체는 논리적으로 동일하다.

> 참조 동치성

두 개의 참조가 메모리의 동일한 개체를 가리키는지의 여부. 기본적으로 Object의 equals 메서드는 '==' 연산자를 사용해 참조 동치성을 확인한다.

ex) Thread

#### 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는 경우

예를 들어 대부분의 Set 구현체는 AbstractSet이 구현한 equals를 상속 받아 쓴다.

List의 구현체들은 AbstractList로부터, Map 구현체들은 AbstractMap으로부터 상속받아 그대로 사용한다.

```java
public boolean equals(Object o) {
	if (o == this) return true;

	if (!(o instanceof Set)) return false;

	Collection<?> c = (Collection<?>) o;
	if (c.size() != size()) return false;

	try {
		return containsAll(c);
	} catch (ClassCastException | NullPointerException unused) {
		return false;
	}
}
```

위 코드는 AbstractSet의 equals 메서드이다.

- 객체가 같은 참조인지 체크하고, Set인지 체크한다.
- size를 비교한다.
- 그리고 내부 인스턴스를 비교한다.

위 과정이면 하위클래스도 마찬가지로 같은 것인지 체크하는데에 충분하다.

#### 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없을 때

더해 equals가 실수로라도 호출되는 것을 막고 싶다면 아래와 같이 구현해보자.

```java
@Override
public boolean equals(Object o) {
	throw new AssertionError();
}
```

### equals를 재정의해야 하는 경우

- 논리적 동치성을 확인해야 하는데 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의 되어있지 않았을 때이다.

주로 값 클래스들이 대부분 해당된다. (Integer, String 등등)

이런 클래스들은 객체가 같은지를 알고 싶은 것이 아니라 값이 같은 지 알고 싶을 것이다.

equals가 논리적 동치성을 확인하도록 재정의해두면 Map의 Key와 Set의 원소로도 사용할 수 있게 된다.

> 예외

- Enum

값 클래스라 하더라도 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스라면 equals를 재정의하지 않아도 된다.

이러한 클래스는 어차피 같은 인스턴스가 2개 이상 만들어지지 않으므로 논리적 동치성과 참조 동치성이 사실상 같은 의미가 되기 때문에 Object의 equals가 논리적 동치성도 확인해준다고 볼 수 있다.

## equals의 일반 규약

null이 아닌 모든 x,y,z에 대하여.

- 반사성
	- x.equals(x) == true 이다.
- 대칭성
	- x.equals(y) == true 라면 y.equals(x)도 true이다.
- 추이성
	- x.equals(y) == true이고 y.equals(z) 도 true라면 x.equals(z) 또한 true이다.
- 일관성
	- x.euqals(y)를 반복해서 호출해도 같은 값을 반환해야 한다.
- null - 아님
	- x.equals(null)은 항상 false이다.

어떻게 보면 수학적으로 당연한 부분이다. 그렇지만 이 규약을 실수로라도 어기게 되면 프로그램이 이상하게 동작하거나 종료될 수 있고, 그 이유를 찾기 매우 힘들 것이다.

> 세상에 홀로 존재하는 클래스는 없다 - 존 던(John Donne)

한 클래스의 인스턴스는 당연히 다른 곳으로 빈번하게 전달된다. 이러한 전달은 전달되는 객체가 equals 규약을 지킨다고 가정하고 동작한다.

따라서 규약을 반드시 지켜야 한다.

### 반사성

객체는 자기 자신과 같아야 한다는 것이다.

이 부분은 일부러 어기는 경우가 아니라면 만족시키지 못하는 경우가 더 적을 것이다.

### 대칭성

두 객체는 서로에 대한 동치 여부에 똑같이 답해야 한다는 것이다.

대소문자를 구별하지 않는 문자열을 구현한 클래스를 예시로 들어 어떻게 대칭성을 못지키게 되는 지 알아보자.

```java
public final class CaseInsensitiveString {
	private final String str;

	public CaseInsensitiveString(String str) {
		this.str = Objects.requireNonNull(str);
	}

	@Override
	public boolean equals(Object o) {
		if (o instanceof CaseInsensitiveString) {
			return str.equalsIgnoreCase(((CaseInsensitiveString) o).str);
		}

		if (o instanceof String) {
			return str.equalsIgnoreCase((String) o);
		}
		return false;
	}
}
```

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Abcd");
String str = "abcd";

System.out.println(cis.equals(str)); //true
System.out.println(str.equals(cis)); //false
```

- 대문자를 무시하는 cis는 equals를 호출해도 true를 반환했지만 반대로 str이 cis와의 동치를 확인할 때는 false가 반환되었다.
- x.equals(y) != y.equals(x)가 되어버린 것이다.

위 예시에서 대칭성을 지키고 싶다면 CaseInsensitiveString 클래스가 String과도 연동하겠다는 두 번째 조건문을 버리면 된다. 버려야만 한다.

### 추이성

첫 번째 객체와 두 번째 객체가 같고 두 번째 객체와 세 번째 객체가 같다면 첫 번째 객체와 세 번째 객체도 같아야 한다는 특성이다.

어떻게 추이성을 지키지 못하게 되는 지 보자.

```java
public class Point {
	private final int y;
	private final int x;

	public Point(int y, int x) {
		this.y = y;
		this.x = x;
	}

	@Override
	public boolean equals(Object o) {
		if (!(o instanceof Point)) {
			return false;
		}

		Point p = (Point) o;
		return this.y == p.y && this.x == p.x;
	}
}
```

기존 포인트에 Color 정보까지 포함하는 Point의 하위 클래스인 ColorPoint를 보자. equals를 재정의해 색상까지 같은 지 비교를 수행해야 할 것이다.

```java
public class ColorPoint extends Point {
	private final Color color;
	
	public ColorPoint(int y, int x, Color color) {
		super(y, x);
		this.color = color;
	}

	@Override
	public boolean equals(Object o) {
		if (!(o instanceof Point)) {
			return false;
		}
		
		// o가 Point일 경우 색상을 무시하고 위치만 비교한다.
		if (!(o instanceof ColorPoint)) {
			return o.equals(this);
		}

		// o가 ColorPoint일 경우 색상까지 비교한다.
		return super.equals(o) && ((ColorPoint) o).color == color;
	}
}		
```

```java
ColorPoint cp1 = new ColorPoint(1, 7, Color.YELLOW);
ColorPoint cp2 = new ColorPoint(1, 7, Color.BLUE);
Point p = new Point(1, 7);

cp1.equals(p); // true
p.equals(cp1); // true
cp2.equals(p); // true

cp1.equals(cp2); // false;
```

- 대칭성을 지켜주지만 추이성을 깬다.
	- 위치는 다 같지만 색상이 다르기 때문에 cp1과 p 그리고 p와 cp2가 같아도 cp1과 cp2는 다르다.
- 이러한 방식은 무한 재귀에 빠질 위험도 존재한다.

```java
public class SmellPoint extends Point {

    private final Smell smell;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) {
            return false;
        }

		// SmellPoint가 아닐 경우 위치만 비교
        if (!(o instanceof SmellPoint)) {
            return o.equals(this);
        }

        // o가 SmellPoint라면 냄새까지 비교
        return super.equals(o) && this.smell == ((SmellPoint) o).smell;
    }
}
```
 
```java
Point cp = new ColorPoint(1, 7, Color.BLUE);
Point sp = new SmellPoint(1, 7, Smell.SWEET);

cp.equals(sp); //무한 재귀
```

- ColorPoint가 아니기 때문에 SmellPoint의 equals를 호출하는데, SmellPoint의 equals에서도 비교군이 ColorPoint가 되어 ColorPoint의 equals를 호출하는 무한 재귀에 빠진다.

***구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 없다.***

#### 만약 getClass 검사로 바꾼다면? - 리스코프 치환 원칙 위반

```java
@Override
public boolean equals(Object o) {
	if (o == null | o.getClass() != getClass())
		return false;
	Point p = (Point) o;
	return p.x == x && p.y == y;
}
```

객체지향적 추상화의 이점을 포기하고 추이성을 지키기 위해 getClass()를 사용해 비교해보았다.
클래스가 다르다면 바로 false를 반환하기 때문에 추이성을 지킬 수 있다.

그러나 하위 클래스는 상위 클래스의 규약을 지켜야한다. 즉, Point의 하위클래스 또한 어디서든 Point로 활용될 수 있어야 된다.

```java
private static final Set<Point> unitCircle = Set.of(
		new Point(1, 0), new Point(0, 1),
		new Point(-1, 0), new Point(0, -1));

public static boolean onUnitCircle(Point p) {
	return unitCircle.contains(p);
}
```

- unitCircle은 반지름이 1인 단위 원이다.
- 해당 원 안에 Point p가 포함되어 있는 지 판별하는 메서드가 있다.

만약 해당 Point p값에 ColorPoint(equals getClass()로 구현)와 같은 값을 넣는다면?

- Set과 같은 대부분의 Collection은 equals를 기반으로 contains() 작업을 수행한다.
	- 위에서 onUnitCircle을 호출하면 Point의 equals를 호출하게 될 것이다.
	- 그런데 getClass()를 이용하면 ColorPoint는 어떠한 Point와도 같을 수 없다.
	- 따라서 y, x의 값(위치값)에 상관없이 무조건 false를 반환하게 되는 것이다.

리스코프 치환 원칙을 지켰다면 위와 같은 코드도 동작해야 한다. instanceof로 구현했다면 동작했을 것이다. (***물론 위에서 말했듯 새로운 값을 추가한 확장 클래스라면 추이성이 안지켜진다.***) 

#### 구체 클래스의 하위 클래스에서 euqals 규약을 지키면서 값을 추가할 방법

우회하는 방법이다. ***상속 대신 컴포지션을 사용하는 것*** (아이템 18)이다.

```java
public class ColorPoint {
	private Point point;
	private Color color;
	
	//...
	
	//ColorPoint의 Point 뷰를 반환한다.
	public Point asPoint() {
		return point;
	}

	@Override
	public boolean equals(Object o) {
		if (!(o instanceof ColorPoint)) return false;

		ColorPoint ocp = (ColorPoint) o;
		return ocp.point.equals(point) && ocp.color.equals(color);
	}
	//...
}
```

- Point를 하위 클래스의 private 필드로 두었다. (컴포지션)
- 그리고 일반 Point를 반환하는 뷰 메서드를 public으로 추가했다.
	- asPoint()를 두어 비록 상속은 아니지만 Point의 기능을 재사용할 수 있도록 했다.
- 그러면서도 equals 메서드는 규약을 지킬 수 있게 되었다.
	
#### 추상 클래스의 하위 클래스라면?

추상 클래스의 하위 클래스에서라면 equals의 규약을 지키면서도 새로운 값을 추가할 수 있다.

상위 클래스를 직접 인스턴스로 만드는 것이 불가능하기 때문에 위 문제들은 모두 일어나지 않기 때문이다.

### 일관성

두 객체가 같다면 앞으로도 영원히 같아야 한다는 뜻이다.

가변 객체는 비교 시점에 따라 서로 다를 수도, 같을 수도 있지만 불변 객체는 한 번 다르면 끝까지 달라야 한다.

불변 클래스로 만들기로 정했다면 equals가 한 번 같다고 한 객체와는 영원히 같다고 결과가 나와야 한다는 뜻이다.

그런데 ***클래스가 불변이든 가변이든 equals의 판단에 신뢰할 수 없는 자원이 끼어들게 해서는 안된다.*** 

```java
URL url1 = new URL("www.abcd.com");
URL url2 = new URL("www.abcd.com");

url1.equals(url2); //항상 동일하지 않다.
```

- java.net.URL 클래스는 URL과 매핑된 host의 IP주소를 이용해 비교한다.
	- 같은 도메인 주소라도 IP 주소가 달라질 수 있기 때문에 결과가 달라질 수 있다.

URL을 온전히 비교하는 것이 아닌 IP주소가 끼어들어 동등성 비교에 신뢰할 수 없는 자원을 사용하게 된 것이다.

***따라서 equals는 항상 메모리에 존재하는 객체만을 사용한 결정적 계산을 수행해야 한다.***

### null-아님

모든 객체가 null과 같지 않아야 한다.

보통 equals 메서드를 보면 null 인지 체크를 하고 null이라면 false를 반환하는 식으로 구현한다.
하지만 그럴 필요 없다.

```java
@Override  
public boolean equals(Object o) {  
	  if (this == o) {  
		  return true;  
	  }  
	  if (o == null || getClass() != o.getClass()) {  
		  return false;  
	  }  
	  Vertex vertex = (Vertex) o;  
	  return v == vertex.v && cost == vertex.cost;  
}
```

- 인텔리제이에서 제공하는 equals 재정의 코드이다. null 체크를 하는 것을 볼 수 있다.
	- 하지만 이는 필요 없다고 주장한다.

```java
if (!(o instanceof Vertex)) {
	return false;
}
```

- 이렇게 instanceof를 사용하게 되면 매개변수가 올바른 타입인지 파악하면 된다.
	- instanceof는 두 번째 피연산자와 무관하게 첫 번째 피연산자(o)가 null이라면 false를 반환한다.
	- 또한 동치성을 검사하기 위해 equals는 건네받은 객체를 적절히 형변환하여 필수 필드들의 값을 알아내야 한다.
		- 그러기 위해서는 형변환에 앞서 입력 매개변수가 올바른 타입인지 검사해야 한다.
		- 이러한 부분도 해결해줄 수 있다.

## 양질의 equals 메서드 구현 방법

- == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다.
- instanceof 연산자로 입력이 올바른 타입인지 확인한다.
	- 보통은 equals를 정의하는 클래스인지 확인하지만, 그 클래스가 구현한 특정 인터페이스를 확인할 수 있다.
	- 인터페이스를 구현한 다른 클래스끼리도 비교할 수 있도록 하기 위함이다.
- 입력을 올바른 타입으로 형변환한다.
	- 앞서 instanceof로 체크했기 때문에 형변환은 실패할 수 없다.
-  입력 객체와 자기 자신의 대응되는 '핵심' 필드들이 모두 일치하는지 하나하나 확인한다.
	- 모든 필드가 일치했을 때 true를 반환한다.

equals를 다 구현했다면 대칭적인가? 추이성이 있는가? 일관적인가?를 따져보자. 단위 테스트를 작성해 체크하자.

추가적으로 Object 외의 타입을 매개변수로 받는 equals 메서드는 선언하지 말자.

> 필드 비교 시 주의 사항

- float와 double를 제외한 기본 타입 필드는 ==을 사용해 비교한다.
	- float와 double는 compare() 메서드로 비교한다.
	- 특수한 부동 소수 값을 다뤄야 하기 때문이다.
- 필드가 참조 타입이라면 equals를 사용해 비교한다.
- null 값을 정상 값으로 취급할 것이라면 Object.equals()를 사용해 NullPointerException 발생을 예방한다.
- 최상의 성능을 바란다면 다를 가능성이 더 크거나 비교하는 비용이 더 싼 필드를 먼저 비교하면 된다.

## 정리

꼭 필요한 경우가 아니라면 equals 메서드를 재정의하지 말아야 한다. 많은 경우에 Object의 equals가 원하는 비교를 수행해줄 것이다.

재정의를 해야하는 경우는 ***논리적 동치성을 확인해야 하는데 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의 되어있지 않았을 때이다.***

재정의를 해야한다면 그 ***클래스의 핵심 필드를 모두 빠짐없이 규약을 지켜가며 비교***해야 한다.