---
title: Item5 (자원을 직접 명시하지 말고 의존 객체 주입을 사용하라)
author: leedohyun
date: 2024-06-30 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

많은 클래스들은 하나 이상의 자원에 의존한다.

하나 이상의 자원에 의존하는 클래스를 만약 정적 유틸리티 클래스로 구현했을 때 어떤 문제가 있는지 보자.

### 하나 이상의 자원에 의존하는 정적 유틸리티 클래스 

```java
public class RandomNumberGenerator {
	private RandomNumberGenerator() {...}
	
	public static int generate(int min, int max) {
		return Random.pickNumberInRange(min, max);
	}
}
```

위와 같은 지정한 범위 내의 랜덤한 숫자를 생성해주는 정적 유틸리티 클래스가 있다고 가정하자.

```java
public class Car {
	private String name;
	private int moveDistance;

	public Car(String name, int moveDistance) {
		this.name = name;
		this.moveDistance = moveDistance;
	}

	public void move() {
		if (canMove()) 	{
			this.moveDistance += MOVE_DISTANCE;
		}
	}

	private boolean canMove() {
		return RandomNumberGenerator.generate(1, 6) >= 3;
	}
	//...
}
```

그러한 유틸리티 클래스를 이용해 랜덤한 수가 일정 숫자를 넘기면 자동차를 움직일 수 있는 일종의 주사위 게임 클래스가 있다고 보자.

현재 상태에서는 아무런 문제가 없어보인다. 그런데 만약 이 게임이 이제 랜덤한 수가 아닌, 고정된 숫자를 넣어 그 숫자값에 기반해 진행되는 게임으로 바뀐다면?

현재 이 게임은 자동 생성을 위한 RandomNumberGenerator에 의존하고 있는데 위와 같이 설계가 바뀌어버린다면 게임이 구현되어있는 Car 클래스를 직접 수정해주어야 한다.

비즈니스 로직의 핵심 도메인을 수정해야만 한다는 의미이다. 심지어 테스트도 어렵다.

***RandomNumberGenerator를 일종의 자원이라고 볼 수 있고 이렇게 자원에 따라 동작이 달라지는 클래스에는 정적 유틸리티 클래스나 싱글톤 방식은 어울리지 않는다.***

### 의존 객체를 주입하자.

Car 클래스가 여러 자원 인스턴스를 지원해야 하고 클라이언트가 원하는 자원을 사용할 수 있어야 한다.

인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방식으로 이를 해결할 수 있다.

```java
@FunctionalInterface
public interface NumberGenerator {
	int generate();
}
```

NumberGenerator라는 인터페이스를 만들어 함수형 인터페이스로 활용해보자.

```java
public class RandomNumberGenerator implements NumberGenerator {
	private static final int MIN_RANGE = 1;
	private static final int MAX_RANGE = 6;
	
	@Override
	public int generate() {
		return Random.pickNumberInRange(MIN_RANGE, MAX_RANGE);
	}
	//...
}
```

인터페이스를 구현하면서 달라진 차이점은 이제 정적 메서드를 활용하지 않고, Car 클래스에서는 생성 시점에 전략 (위 구현 클래스)을 주입받게 된다.

```java
public class Car {
	private String name;
	private int moveDistance;
	private NumberGenerator numberGenerator;

	public Car(String name, int moveDistance, NumberGenerator numberGenerator) {
		this.name = name;
		this.moveDistance = moveDistance;
		this.numberGenerator = numberGenerator;
	}

	public void move() {
		if (canMove()) 	{
			this.moveDistance += MOVE_DISTANCE;
		}
	}

	private boolean canMove() {
		return numberGenerator.generate() >= 3;
	}
	//...
}
```

이렇게 구현하게 되면 만약 번호를 기준으로 움직인다는 전제의 설계가 변하지 않는 이상 번호를 생성하는 방식 어떤 것이든 구현해놓고 주입을 시켜주면 비즈니스 로직의 메인 도메인은 변경하지 않아도 된다.

객체지향설계원칙의 DIP, OCP를 지키는 방법이자 스프링의 의존성 주입의 장점을 간단한 코드로 확인할 수 있다.

### 의존 객체 주입의 장점

객체에 유연성을 부여해주고, 테스트를 용이하게 해준다.

위의 예시로 확인해보자.

만약 기존 구현 코드라면 테스트 시 랜덤으로 생성되는 번호를 바탕으로 테스트를 진행해야 하는데, 그렇다면 매번 결과가 달라져 테스트가 확실하지 않게 된다.

```java
@Test
void testMove() {
	Car car = new Car("car1", 0);

	car.move();

	assertThat(car).extracting("moveDistance").isEqualTo(0);
}
```

위와 같은 테스트는 사실상 작성할 수 없다. 매번 랜덤하게 수가 바뀌어 움직임이 실패했다는 것을 보장할 수 없기 때문이다.

그렇다고 RandomNumberGenerator를 테스트 해 랜덤값이 해당 범위 내에 있는가로 테스트를 마친다? 그것도 테스트의 목적성을 모두 지킬 수 없을 것이다.

```java
@Test
void testMove() {
	Car car = new Car("car1", 0, fixNumberGenerator);

	car.move();

	assertThat(car).extracting("moveDistance").isEqualTo(0);
}
```

이렇게 객체 생성 시점에 번호 생성 전략을 주입해 외부에서 번호 생성을 관리하도록 하면 테스트를 유연하게 할 수 있다.


## 정리

일종의 구현체에 의존하지 말고 인터페이스에 의존하라는 원칙을 다른 말로 확인해보았다. 

스프링을 학습하면서 의존성 주입에 관한 장점을 알고 있었지만 다시 한 번 코드를 작성해보며 확인해볼 수 있었다.

객체간의 결합도를 낮추어 확장이 용이하며, 테스트가 쉬워진다.