---
title: Item34 (int 상수 대신 열거 타입을 사용하라)
author: leedohyun
date: 2024-07-18 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

이번 아이템부터는 클래스의 일종인 Enum 열거 타입과, 인터페이스의 일종인 애노테이션에 대해 다룬다.

내용을 훑어 보았을 때 단순히 Enum을 어떻게 사용하는 지, 주의해야할 점은 무엇인지에 더해 Enum 타입의 다양한 활용성에 대해 고민해볼 수 있는 장인 것 같다.

## int 상수 대신 열거 타입을 사용하라

열거 타입은 일정 개수의 상수 값을 정의한 후, 그 외의 값은 허용하지 않는 타입이다.

색상, 요일 등등으로 많이 연습해봤을 것이다.

```java
enum Day {
	MONDAY, TUESDAY, WEDNESDAY, THURSDAY, 
	FRIDAY, SATURDAY, SUNDAY;
}
```

이러한 Enum이 나오기 전에는 정수 상수를 한 묶음으로 선언해 사용했다.

```java
public static final int MONDAY = 0;
public static final int TUESDAY = 1;
//...
```

이러한 정수 열거 패턴에는 단점이 있다. 단점을 알아보자.

### 정수 열거 패턴의 단점

- 타입 안전을 보장할 방법이 없고, 표현력 또한 좋지 않다.
	- 예를 들어 int today = Day.MONDAY; 이렇게 사용하던 코드에 int today = 7;을 할당해도 컴파일러는 오류를 발생시키지 못한다.
	- 또한 모든 상수가 같은 네임스페이스에 정의되어 다른 열거형과 상수 이름이 충돌할 수 있다.
- 정수 열거 패턴을 사용한 프로그램은 깨지기 쉽다.
	-  새로운 상수가 추가되거나 기존 상수의 값이 바뀌면 문제가 생긴다.
	- 단순 상수를 나열한 것이기 때문에 컴파일 시점에 그 값이 클라이언트 파일에 그대로 새겨진다.
	- 따라서 값이 바뀌면 클라이언트도 반드시 다시 컴파일해야 하는 것이다. 그렇지 않으면 엉뚱한 동작을 할 수 있다.
- 문자열로 출력하기 까다롭다.
	- 그 값을 출력하거나 디버거로 살펴보아도 단지 숫자로만 보여서 도움이 되지 않는다.
- 같은 정수 열거 그룹에 속한 모든 상수를 순회하는 방법도 마땅치 않다.
	- 상수가 몇 개인지도 알 수 없다.  

정수 대신 문자열 상수를 사용하는 방법으로 의미 있는 값을 출력하게 할 수도 있지만, 이런 방법은 더 나쁘다.

```java
public class Day {
	public static final String MONDAY = "MONDAY"; 
	public static final String TUESDAY = "TUESDAY"; 
	public static final String WEDNESDAY = "WEDNESDAY";
	//...
}
```
```java
String today = Day.MONDAY;
String today = "MONDAY";
```

- 경험이 부족한 프로그래머가 문자열 상수의 이름 대신 문자열 값 그대로를 하드코딩하게 할 수 있다.
	- 오타가 발생해도 컴파일러는 체크할 수 없다.
- 문자열 비교를 하게 되어 성능 저하가 발생한다.

그러나 이러한 열거 패턴의 단점들을 상쇄해주는 것이 존재한다. Enum 열거 타입이다.

### Enum 열거 타입

```java
public enum Genre {
	ADVENTURE, THRILLER, COMIC;
}
```

겉보기에는 C, C++과 같은 다른 언어의 열거 타입과 비슷하지만, 자바의 열거 타입은 완전한 형태의 클래스이기 때문에 단순한 정숫값일 뿐인 타 언어의 열거타입보다 강력하다는 것을 알아야 한다.

- Java의 열거 타입
	- 열거 타입 자체는 클래스이다.
	- 상수 하나당 자신의 인스턴스를 하나씩 만들어 public static final 필드로 공개한다.
	- 이러한 인스턴스는 밖에서 접근할 수 있는 생성자를 제공하지 않으므로 사실상 final이다.
	- 따라서 클라이언트가 인스턴스를 직접 생성하거나 확장할 수 없어 열거 타입 선언으로 만들어진 인스턴스들은 딱 하나씩만 존재한다.
		- 싱글톤 패턴에 열거 타입을 이용했던 것을 기억!
	
Enum 열거 타입을 이용하면, 정수 열거 패턴의 단점을 모두 상쇄할 수 있을지 확인해보자.

> Enum 열거 타입의 장점

- 컴파일 타임 타입 안전성을 제공한다.
	- 예를 들어 열거 타입을 매개변수로 받는 메서드를 선언했다면, 건네받은 참조는 해당 열거 타입의 값 중 하나임이 보장된다.
	- 다른 타입을 넘기면 컴파일 에러가 발생한다.
- 열거 타입에는 각자의 네임 스페이스가 있다.
	- 이름이 같은 상수여도 공존이 가능하다.
	- 열거 타입에 새로운 상수를 추가하거나 순서를 바꾸더라도 다시 컴파일 하지 않아도 된다는 의미이다.
	- 공개되는 부분이 오직 필드의 이름뿐이다. 따라서 상수 값이 클라이언트로 컴파일되어 각인되지 않는다.
-  열거 타입의 toString 메서드를 통해 출력하기에 적합한 내용을 출력할 수 있다.

정수 열거 패턴의 단점들을 모두 제거하면서도, 유연하게 사용할 수 있다.

이 외에도 Enum 열거 타입의 강점은 더 있다.

### 열거 타입의 활용

열거 타입에는 임의의 메서드나 필드를 추가할 수 있고, 임의의 인터페이스를 구현하게 할 수도 있다.

Object 메서드들, Comparable, Serializable을 구현했고, 그 직렬화 형태 또한 웬만큼 변형을 가해도 문제없이 동작하게끔 구현이 되어있다.

그렇다면 이렇게 열거 타입에 메서드나 필드를 추가해서 어떤 부분에 활용할 수 있을지 알아보자.

단순하게 Apple과 Orange를 예로 들면 과일의 색을 알려주거나, 과일 이미지를 반환하는 메서드를 추가하고 싶을 수 있다.

실제로 열거 타입은 클래스이기 때문에 고차원의 추상 개념 하나를 완벽하게 표현해낼 수 있다.

태양계의 행성을 열거 타입으로 선언한 것을 예시로 장점을 확인해보자.

```java
public enum Planet {
	MERCURY(3.302e+23, 2.439e6),
	VENUS(4.869e+24, 6.052e6),
	EARTH(5.975e+24, 6.378e6),
	MARS(6.419e+23, 3.393e6),
	JUPITER(1.899e+27, 7.149e7),
	SATURN(5.685e+26, 6.027e7),
	URANUS(8.683e+25, 2.556e7),
	NEPTUNE(1.024e+26, 2.477e7);

	private final double mass; //질량
	private final double radius;

	private final double surfaceGravity; //표면 중력

	private static final double G = 6.67300E-11;
	
	Planet(double mass, double radius) {
		this.mass = mass;
		this.radius = radius;
		surfaceGravity = G * mass / (radius * radius);
	}

	//getter..

	public double surfaceWeight(double mass) {
		return mass * surfaceGravity;
	}
}
```


코드와 함께 열거 타입 사용법을 보자.

- 열거 타입 상수 각각을 특정 데이터와 연결지으려면 생성자에서 데이터를 받아 인스턴스 필드에 저장하면 된다.
- 근본적으로 열거타입은 불변이기 때문에 필드는 final이어야 한다. 
	- 그리고 필드를 public으로 선언해도 되지만, private으로 두고 별도의 public 접근자 메서드를 두는 것이 낫다. (불필요한 방어적 복사 방지)
- 열거 타입도 마찬가지로 공개하지 않아도 될 부분은 public으로 선언하지 않아야 한다.
- 보통 열거 타입은 톱레벨 클래스로 만들지만, 특정 톱레벨 클래스에서만 사용되는 것이라면 해당 클래스의 멤버 클래스로 만든다.
- 이렇게 열거 타입은 일반적으로 클래스를 다루는 방식과 동일하게 다뤄주면 된다.

지금 이 상태만으로도 충분히 강력하지만, 조금 더 활용해보자.

```java
double mass = earthWeight / Planet.EARTH.surfaceGravity();

for (Planet p : Planet.values()) {
	System.out.printf("%s 에서의 무게는 %f이다. \n", p, p.surfaceWeight(mass);
}

//MERCURY에서의 무게는 69.912739이다.
//VENUS에서의 무게는 167.434436이다.
//...
```

- 이렇게 열거 타입의 모든 원소를 순회하며 원소들을 이용할 수도 있다.
- 단순 출력뿐 아니라, 열거 타입 원소에 boolean 변수를 넣어두고 어떤 조건에 맞는지를 체크할 수도, Function같은 함수형 인터페이스를 넣어 필요한 구현을 유연하게 할 수 있다.
	- 각 상수 값의 필드 값을 비교할 수도 있다.
- 이러한 메서드를 꼭 밖에서 작성하지 않고, 열거 타입 내부에서 클래스 메서드로 만들어도 무방하다.

거기에 더해 열거 타입에서 상수를 하나 제거하더라도 클라이언트에는 영향이 가지 않는다. 삭제한 타입을 참조하는 경우 컴파일 에러가 발생할 것이다.

### 열거 타입의 활용 2 (상수별 메서드 구현)

```java
enum Operation {
	PLUS("+") {
		public double apply(double x, double y) { return x + y; }
	},
	MINUS("-" {
		public double apply(double x, double y) { return x - y; }
	}, //...

	private final String symbol;
	
	Operation(String symbol) { this.symbol = symbol; }

	public abstract double apply(double x, double y);

	@Override
	public String toString() { return symbol; }
}
```

- 추상 메서드를 선언하고, 각 상수별 클래스 몸체에서 재정의하는 방식을 사용했다.
	- 상수별 메서드를 구현할 수 있다.
	- 추상 메서드로 선언해놓았기 때문에 새로운 상수를 추가해도 컴파일 에러가 발생해 놓치지 않을 수 있다.
-  toString()을 재정의하여 의미있는 출력을 유도할 수도 있다.

```java
public enum Operation { 
	PLUS, MINUS, TIMES, DIVIDE; 

	public double apply(double x, double y) { 
		switch (this) { 
			case PLUS: return x + y; 
			case MINUS: return x - y; 
			case TIMES: return x * y; 
			case DIVIDE: 
				if (y == 0) { 
					throw new ArithmeticException("Cannot divide by zero"); 
				} 
				return x / y; 
			default: 
				throw new AssertionError("Unknown operation: " + this); 
		} 
	} 
}
```

- 이렇게 switch문을 이용해 값에 따라 분기하며 만들어볼 수도 있다.
- 물론 새로운 값을 열거 타입에 추가하려면 그 값을 처리하는 case 문을 잊지말고 넣어줘야 한다는 단점이 있다.
	- 실수하기 쉽다.

### 열거 타입의 활용 3 (전략 열거 타입 패턴)

요일에 따라 일당을 계산해주는 열거 타입이 있다고 해보자. 그런데 평일과 주말은 잔업 수당의 계산 방식이 다른 체계이다.

이는 각 요일에 따라 계산하는 메서드를 하나하나 넣어줘도 된다. 하지만 일당을 계산하는 부분은 공통적인 부분일 것이고, 하나하나 넣어주는 상수별 메서드 구현은 코드를 공유하기 어렵다.

코드를 공유할 수 없어 중복되는 코드를 하나하나 작성해주어야 하고, 가독성이 떨어진다.

switch문으로 분기해서 코드를 공유하더라도, 새로운 상수가 추가되었을 때 관리가 어렵다는 단점이 존재한다.

그럼 이런 경우 어떻게 대처할 수 있을까? 전략 열거 타입 패턴을 쓰면 된다.

잔업 수당 계산을 private 중첩 열거 타입으로 옮기고(전략) 기존 열거 타입의 생성자에서 적당한 전략을 선택해 넣어주면 된다.

```java
enum PayrollDay {
	MONDAY(WEEKDAY), TUESDAY(WEEKDAY), WEDNESDAY(WEEKDAY),
	THURSDAY(WEEKDAY), FRIDAY(WEEKDAY),
	SATURDAY(WEEKEND), SUNDAY(WEEKEND);

	private final PayType payType; // private 중첩 열거 타입

	PayrollDay(PayType payType) { this.payType = payType; }
	
	int pay(int minutesWorked, int payRate) {
		return payType.pay(minutesWorked, payRate);
	}

	enum PayType {
		WEEKDAY {
			int overtimePay(int minsWorked, int payRate) {
				return minsWorked <= MINS_PER_SHIFT ? 
					0 : (minsWorked - MINS_PER_SHIFT) * payRate / 2;
			}
		},
		WEEKEND {
			int overtimePay(int minsWorked, int payRate) {
				return minsWorked * payRate / 2;
			}
		};

		abstract int overtimePay(int mins, int payRate);
		private static final int MINS_PER_SHIFT = 8 * 60;

		int pay(int minsWorked, int payRate) {
			int basePay = minsWorked * payRate;
			return basePay + overtimePay(minsWorked, payRate);
		}
	}
}
``` 

- 일종의 전략 패턴을 구현했는데, 그 전략을 내부 열거 타입으로 선언했다.
	- 내부 열거 타입에서 그 타입에 해당하는 방식의 메서드를 선언하여 톱 레벨 클래스의 열거 타입에서는 그 메서드를 공유해서 사용할 수 있게 되었다.
- 상수별 메서드 구현이 필요 없어 코드의 중복이 줄고, 관리도 쉬워졌다.
- Switch문보다 복잡하지만 관리에 용이하고 유연하다. 새로운 상수가 추가되어도 전략을 선택해주기만 하면 된다.

## 정리

***필요한 원소를 컴파일타임에 다 알 수 있는 상수 집합이라면 항상 열거 타입을 사용하자.***

다른 방식에 비해 훨씬 강력하고, 안전하며 다양한 활용을 할 수 있다. 열거 타입에 정의된 상수 개수가 영원히 고정 불변일 필요도 없다.

대다수의 열거 타입이 생성자나 메서드 없이 사용되지만 생성자나 메서드, 상수 등을 추가적으로 활용해 다양한 구현을 할 수 있다.

활용도가 무궁무진하다.