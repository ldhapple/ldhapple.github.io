---
title: Item18 (상속보다는 컴포지션을 사용하라)
author: leedohyun
date: 2024-07-08 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 상속보다는 컴포지션을 사용하라

상속은 코드를 재사용함에 있어 강력한 수단이 된다. 그러나 항상 최선이지 않다. 잘못 사용하면 오류를 내기 쉬운 소프트웨어를 만들 뿐이다.

### 상속의 문제점

상속은 캡슐화를 깨뜨리고 상위 클래스에 의존적이게 되어 결합도가 높아진다.

**상위 클래스의 구현이 하위 클래스의 동작에 영향**을 끼칠 수 있고, 상위 클래스의 구현을 수정했을 뿐인데 하위 클래스에서 오동작할 수 있다는 것이다.

```java
abstract class Vehicle {
	protected int energy;

	public Vehicle(int energy) {
		this.energy = energy;
	}

	public abstract void drive();

	public int getRemainingEnergy() {
		return energy;
	}
}
```

```java
class ElectricCar extends Vehicle {
	public ElectricCar(int energy) {
		super(energy);
	}
	
	@Override
	public void drive() {
		if (energy >= 10) {
			energy -= 10;
			System.out.println("ElectricCar. Remaining energy: ", + energy);
		} else {
			System.out.println("ElectricCar 운전 불가");
		}
	}
} 	
```

- 위 클래스는 당장 문제가 없어보인다.
- 그러나 만약 Vehicle 클래스의 요구사항이 변경되어 drive 메서드를 정의한다면?
	- 상위 클래스의 drive()를 정의하면 ElectricCar의 drive() 메서드는 무시된다.
	- 즉, 예상한 동작을 하지 않게 되는 것이고, 상위 클래스를 수정함에 따라 하위 클래스도 수정해야 한다는 문제가 생겼다.
	- 위와 같은 단순한 구조에서도 문제가 생겼는데, 구조가 조금만 더 복잡해져도 손을 대기 힘들정도의 더 큰 수정이 필요할 것이다.

또한 **상위 클래스와 하위 클래스의 관계가 컴파일 시점에 결정되는 것도 문제다. 구현에 의존하기 때문에 실행 시점에 객체의 종류를 변경하는 것이 불가능해진다.** 그렇다면 다형성과 같은 객체지향의 이점이 사라진다.

새로운 메서드를 추가하는 방식으로 해결한다해도 시그니처나 반환타입이 겹칠 경우 재정의한 것과 마찬가지가 되어 같은 문제가 발생한다.

이런 문제를 어떻게 해결할 수 있을까?

### 상속보다는 컴포지션

컴포지션이란 기존 클래스가 새로운 클래스의 구성 요소로 사용되는 것을 뜻한다.

새로운 클래스를 만들고 private 필드로 기존 클래스의 인스턴스를 참조하도록 하면 된다.

```java
interface Engine { 
	void run(Vehicle vehicle); 
} 

class ElectricEngine implements Engine { 

	@Override 
	public void run(Vehicle vehicle) { 
		if (vehicle.getEnergy() >= 10) { 
		vehicle.decreaseEnergy(10); 
			System.out.println("ElectricCar. Remaining energy: " + vehicle.getEnergy()); 
		} else { 
			System.out.println("ElectricCar 운전 불가"); 
		} 
	} 
}
```

```java
class Vehicle { 
	private int energy;  
	
	public Vehicle(int energy) { 
		this.energy = energy; 
	} 

	public void drive() { engine.run(this); } 
	public int getEnergy() { return energy; } 
	public void decreaseEnergy(int amount) { this.energy -= amount; } 
}
```
```java
class ElectricCar { 
	private Vehicle vehicle; 
	private Engine engine; 

	public ElectricCar(int energy) { 
		this.vehicle = new Vehicle(energy); 
		this.engine = new ElectricEngine(); 
	} 

	public void drive() { engine.run(vehicle); } 
	public int getRemainingEnergy() { return vehicle.getEnergy(); } 
}
```

-  기존 Vehicle 클래스를 사용해 기존 기능을 그대로 재사용할 수 있다.
	- **Vehicle 클래스에 메서드가 추가되어도 문제없다.**
	- 기존 동작에는 변동이 없다.
- 그러면서도 생성자에서 Engine을 갈아 끼우는 방식으로 의도한 ElectricCar를 구현할 수 있다.
	- Engine에서 기존 Vehicle 인스턴스의 기능을 사용했다.
	- **클래스 간의 관계가 컴파일 시점이 아닌 런타임 시점에 정해지도록 할 수 있다.**
		- ElectricCar 클래스는 Vehicle 객체를 필드로 가지고, 생성자를 통해 ElectricEngine을 런타임 시점에 할당받게 된다.
		
컴포지션의 사용은 위와 같이 래퍼 클래스로 구현할 적당한 인터페이스가 존재하는 경우 더더욱 견고하고 강력한 효과를 발휘한다.

### 종합하자면?

컴포지션을 사용하면 기존 클래스의 구현 변경에 영향을 받지 않아 유연하다. 상속은 상위 클래스에 의존적이고 캡슐화가 깨져 변화에 유연하지 못하다.

그러나 컴포지션이 상속보다 반드시 좋은 것은 아니다.

상속이 적절하게 이용되면 컴포지션보다 강력하고, 개발하기에도 편하다.

상속을 사용하는 경우는 아래와 같다.

- 확장을 고려하고 설계한 확실한 is - a 관계 일 때
	- 그런데 is - a 관계여도 상위 클래스가 확장을 고려해 설계되지 않았다면 문제가 발생할 수 있기 때문에 추천하지 않는다.
- API에 아무런 결함이 없는 경우
	- 결함이 있다면 하위 클래스까지 전파되어도 문제가 없는 경우

> is - a 관계

has - a 관계와 반대되는 개념으로, has - a 관계는 한 오브젝트가 다른 오브젝트에 속한 경우를 뜻한다.

반면 is - a 관계는 한 클래스가 다른 클래스의 파생 클래스임을 나타낸다. 즉, A is a B 관계라면 A는 B의 명세를 암시한다.

예를 들어 Car이 상위클래스이고 ElectricCar가 하위클래스라면 전기차가 자동차의 동작 방식을 따른 다는 것에는 변동될 가능성이 거의 없다.

Car - ElectricCar 같은 관계가 확실한 is - a 관계라고 볼 수 있다.

## 정리

상속을 이용한 확장 방식은 상위 클래스에서 메서드를 추가하거나 재정의하면서 문제가 발생할 수 있다. 상위 클래스의 변경에 따라 예기치 않은 하위 클래스의 오류가 발생할 수도 있다.

그래서 상속보다는 **기존 클래스를 구성 요소로 가지는 방식으로 기존 클래스의 동작을 이용하는 컴포지션**의 사용을 추천한다. 그러나 상속이 더 이득일 경우도 있다. 저러한 문제가 없을 경우이다. (is - a 관계 일 때, API에 아무런 결함이 없을 때)

***상속은 코드 재사용의 관점에서 사용하면 안되고, 확장의 관점에서 사용해야 한다.***