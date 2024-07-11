---
title: Item23 (태그 달린 클래스보다는 클래스 계층 구조를 활용하라)
author: leedohyun
date: 2024-07-10 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 태그 달린 클래스보다는 클래스 계층 구조를 활용하라

우선 태그 달린 클래스가 무엇인지 알아보자.

```java
class Figure {
	enum Shape { RECTANGLE, CIRCLE };
	
	final Shape shape; // 태그 필드

	double length; // 도형이 사각형일때만 사용하는 필드들
	double width;

	double radius; // 도형이 원 일때만 사용하는 필드

	Figure(double length, double width) { //... }
	Figure(double radius) { //... }

	double area() {
		switch (shape) {
			case RECTANGLE:
				return length * width;
			case CIRCLE:
				return Math.PI * (radius * radius);
			default:
				throw new AssertionError(shpae);
		}
	}
}	
```

- shape 이라는 태그를 활용해 각 도형마다 다르게 활용할 수 있도록 했다.

태그 달린 클래스란 말그대로 태그가 달린 클래스인데, 태그란 위 예시에서 Shape과 같은 것을 의미한다.

### 태그 달린 클래스의 단점

- 열거타입을 선언해서 사용한다
- 태그 필드 존재
- switch문까지 사용한다.
- 여러 구현이 하나의 클래스에 몰려있어 가독성이 나쁘다.
- 또 다른 요소를 추가하려면 쓸모없는 코드가 많아진다.
	- switch문을 전부 찾아가며 해당 요소에 대한 부분을 추가해주어야 한다.
- 필드들을 final로 선언해 안정성을 늘리고자 한다면 생성자에서 사용하지 않는 필드까지 초기화해줘야 한다.

가장 큰 단점은 인스턴스의 타입만으로 현재 나타내는 의미를 나타낼 수 없다는 점이다.

**태그 달린 클래스는 단점이 너무 많다.**

### 클래스 계층구조로 바꾸자.

자바같은 객체지향 언어는 타입 하나로 다양한 의미의 객체를 표현할 수 있도록 해준다.

태그달린 클래스를 클래스 계층구조를 통해 개선해보자.

- 계층 구조의 루트가 될 추상클래스를 선언한다.
	- class Figure
- 태그 값에 따라 달라지는 동작을 추상 메서드로 제공한다.
	- double area()
- 태그 값과 상관없이 동일하게 동작하는 메서드는 디폴트 메서드로 제공한다.
- 공통으로 사용하는 데이터 필드는 모두 추상 클래스에 선언한다.
- 태그 별로 추상 클래스를 확장한 구현 클래스를 만든다.

```java
abstract class Figure {
	final Color color;
	
	public Figure(Color color) {
		this.color = color;
	}

	final Color getColor() {
		return color;
	}
	
	abstract double area();
}
```
```java
final class Rectangle extends Figure {

	final double length;
	final double width;

	Rectangle(Color color, double length, double width) {
		super(color);
		this.length = length;
		this.width = width;
	}

	@Override
	double area() {
		return length * width;
	}
}
```

클래스 계층 구조로 바꾸면 태그 달린 클래스의 단점을 모두 없애준다.

- 쓸모없는 코드가 사라졌다.
	- switch, enum 등등
- 남은 필드들을 모두 final로 선언할 수 있다. 불필요한 값이 없다.
- 새 태그를 추가하더라도 번거롭게 Switch문 신경쓸 필요 없이 새 Figure 구현 클래스를 만들어주기만 하면 된다.
- 필드를 초기화하고, 추상 메서드를 모두 구현했는지 컴파일러가 체크해준다는 점도 큰 장점이다.
- 인스턴스가 어떤 타입인지도 명확해진다.

> 정사각형 추가하기

```java
class Square extends Rectangle {
	Square(Color color, double side) {
		super(color, side, side);
	}
}
```

정사각형을 직사각형의 구현 클래스로 선언해 더 쉽게 정의할 수 있었다.

계층 구조를 사용하는 것으로 이런 유연성도 얻을 수 있다.

## 정리

태그 달린 클래스를 실제로 본 적도 없고 사용해보려는 시도도 안해봤지만 정말 단점이 많은 것을 알았다.

만약 이런 클래스가 보인다면 계층 구조로 리팩토링하는 것을 고려해봐야겠다.

불필요한 코드가 생긴다는 점이나, 가독성은 차치하고서라도 인스턴스 타입으로 파악할 수 있다는 점과 확장성에서 큰 차이가 나는 것 같다.