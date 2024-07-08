---
title: Item16 (public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라)
author: leedohyun
date: 2024-07-07 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

인스턴스 필드들을 모아놓는 일 외에는 아무 목적도 없는 퇴보한 클래스를 작성하는 경우를 보자.

```java
class Point {
	public double y;
	public double x;
}
```

이런 클래스는 코딩테스트 풀 때나 허용된다. 이 클래스의 문제점을 보자.

### public 클래스에 public 필드를 사용했을 때의 문제점

캡슐화의 이점을 제공하지 못한다.

- API를 수정하지 않고는 내부 표현을 바꿀 수 없다.
- 불변 식을 보장할 수 없다.
- 외부에서 필드에 접근할 때 부수 작업을 수행할 수 없다.

### public 클래스 캡슐화를 살리는 방법

그렇다면 위와 같은 클래스를 어떻게 작성하는 것이 좋을까?

```java
public class Point {
	private double x;
	private double y;

	public Point(double y, double x) {
		this.y = y;
		this.x = x;
	}

	public double getY() { return y; }
	public double getX() { return x; }
	
	public void setY(double y) { this.y = y; }
	public void setX(double x) { this.x = x; }
}
```

- 필드를 모두 private으로 변경한다.
- public 접근자(getter), 변경자(setter)를 추가한다.

public 클래스라면 이러한 구현이 확실히 옳다. 필드를 private으로 바꾸고 getter, setter을 제공하면 무엇이 좋을까?

> 장점

- 유연성을 얻는다.
	- getter, setter을 이용해 클래스 내부 표현 방식을 언제든 바꿀 수 있다.
	- 만약 필드를 노출하는 방법을 사용한다면 클라이언트가 그 필드를 그대로 사용하고 있어 필드를 수정할 때 마음대로 바꿀 수 없다.

### private 중첩 클래스 혹은 package-private 클래스의 경우??

위에서 public 클래스의 경우 필드를 private으로 바꾸고 접근자를 제공해 유연성을 주는 방식을 선택하라고 했다. 

그렇다면 많이 사용하는 package-private 클래스나, private 중첩 클래스의 경우라면 어떨까?

```java
class DataCalc {
	public int[] data;
	
	//...
	
	int calculateData() {
		int sum = 0;
		for (int val : data) {
			sum += val;
		}
		return sum;
	}
}
```
```java
int[] test = {1, 2, 3, 4, 5};

DataCalc calc = new DataCalc(test);
int result = calc.calculateData();

calc.data[0] = 7;
result = calc.calculateData();
```

- 해당 클래스가 포함되는 패키지 내에서만 조작이 가능하다.
- private 중첩 클래스도 마찬가지이다.
	- 상위 클래스에서만 필드에 접근할 수 있다.

클래스와 필드를 선언하는 입장에서나, 클라이언트 입장에서나 훨씬 깔끔하다.

> 그런데 같은 패키지에 한정할지라도 클래스의 캡슐화가 깨지면 안좋은 것 아닌가?

캡슐화가 깨지는 것은 맞다.

하지만 결과적으로 클래스를 통해 표현하려는 추상 개념만 상세하고 올바르게 표현하면 된다.

- 클래스가 특정 데이터나 동작을 추상화하고, 그 추상 개념을 명확하게 표현한다면 public으로 열어두었을 때 더 직관적이고 간단하다.
	- 추상 개념만 명확하다면 구현이나 가독성이 더 깔끔한 public으로 열어두는 방법을 택하는 것도 나쁘지 않다는 것이다.

추상 개념이 명확하다는 것을 예시로 들면 데이터 구조를 다루는 클래스나 기능을 제공하는 클래스가 있을 것이다. (위의 DataCalc 클래스도 마찬가지이다.)

이러한 클래스는 일반적으로 어떤 데이터나 기능을 캡슐화하고 단순히 사용자에게 제공하는 추상화된 개념을 명확하게 표현할 수 있다.

> 정리 후 개인적인 생각

추상 개념이 명확하다는 것이 주관적이고 애매하다고 생각이 들어서, 웬만하면 캡슐화를 지켜 유지보수 및 확장성에 도움이 되는 것이 좋다고 생각한다.

현재 생각은 이렇지만 후에 경험이 쌓여 어떤 부분에서 단순하고 명확한 클래스를 작성하게 된다면 적용해보면서 생각이 바뀔 수도 있겠다.

### 자바 플랫폼 라이브러리에서 public 클래스의 필드를 노출한 사례

java.awt.package 패키지의 Point와 Dimension 클래스가 그 예시이다.

```java
public class Point extends Point2D implements java.io.Serializable {
	public int x;
	public int y;

	//...
}
```
```java
public class Dimension extends Dimension2D implements java.io.Serializable {
	public int width;  
	public int height;

	//...
}
```

여기서 Dimension의 getSize() 메서드를 봐보자.

```java
public Dimension getSize() {  
  return new Dimension(width, height);  
}
```

- public 타입을 가변으로 만들어 내부 데이터를 변경할 수 있게 되면 불필요하게 방어적 복사를 해야할 수 있다.
	- 방어적 복사를 하지 않았다면 원본 객체를 반환했을텐데, public으로 열어둔 필드가 가변이기 때문에 원본 객체가 손상됐을 것이다.
- 방어적 복사를 위해 getSize() 메서드는 매번 새로운 Dimension 객체를 생성해서 반환해주고 있다.

이렇게 객체를 계속 생성하게 되는 성능 문제를 고치려면, 필드를 private으로 바꿔주고 그것을 관리할 수 있는 접근자, 변경자를 제공해주면 될 문제다.

그래서 실제로 getWidth(), getHeight() 메서드가 추가되었다. 그런데 문제는 이미 getSize()를 사용하고 있는 기존 클라이언트 코드가 존재한다는 것이다.

원래 결정했던 API 설계때문에 성능 문제를 인지해도 쉽게 고칠 수 없게 되는 것이다.

### public 클래스에서 불변 필드를 노출한다면?

```java
public final class Time {
	private static final int HOURS_PER_DAY = 24;
	private static final int MINUTES_PER_HOUR = 60;

	public final int hour;
	public final int minute;

	//...
}
```

- Time 인스턴스 각각이 유효한 시간을 표현함이 보장된다.
	- 불변성이 보장된다. 
	- 방어적 복사를 안해도 된다.

- API를 수정하지 않고는 내부 표현을 바꿀 수 없다.
- 불변 식을 보장할 수 없다.
- 외부에서 필드에 접근할 때 부수 작업을 수행할 수 없다.

그러나 이게 다다. 불변 식을 보장할 수 있을 뿐 다른 문제는 해결되지 않는다. 

여전히 API를 수정하지 않고 내부 표현을 바꿀 수 없다.

## 정리

public 클래스는 가변 필드를 직접 노출해서는 안된다. 불변 필드일 경우 비교적 덜 위험하지만 여전히 문제가 있다. (주로 API 수정 문제가 큰 것 같다.)

private 중첩 클래스나 package-private 클래스에서는 경우에 따라 public으로 필드를 노출하는 것이 나은 경우도 있다. (나는 우선 최대한 캡슐화를 지키는 것이 좋다고 본다.)