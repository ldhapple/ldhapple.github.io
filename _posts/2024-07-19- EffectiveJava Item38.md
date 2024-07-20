---
title: Item38 (확장할 수 있는 열거 타입이 필요하면 인터페이스를 사용하라)
author: leedohyun
date: 2024-07-19 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 확장할 수 있는 열거 타입이 필요하면 인터페이스를 사용하라

이전 아이템들부터 다루고 있는 열거 타입(Enum)은 타입 안전 열거 패턴보다 우수하다.

타입 안전 열거 패턴이 무엇인가?

```java
public class Color {
	private final String name;

	private Color(String name) {
		this.name = name;
	}

	public static final Color RED = new Color("RED");
	public static final Color BLUE = new Color("BLUE");
	public static final Color YELLOW = new Color("YELLOW");

	@Override
	public String toString() {
		return name;
	}

	public static Color[] values() {
		return new Color[] { RED, BLUE, YELLOW };
	}
	
	//등등...
}
```

- 자바5 이전 열거형을 구현하기 위해 사용되던 패턴이다.
- 클래스의 인스턴스들이 각 열거형의 값으로 정의되어 있다. 따라서 타입 안전성도 보장된다.
	- 컴파일 시점에서 타입 체크로 오류를 잡을 수 있음.
	- 이 패턴에서도 각 상수를 인스턴스로 만든다. 마찬가지로 열거형도 각 상수를 인스턴스로 만든다.
- 메서드를 구현해 추가적인 동작을 나타낼 수도 있다.

타입 안전 열거패턴도 나쁘지 않아보인다. 다만, 열거형을 사용하면 훨씬 유연하고, 필요한 메서드를 하나하나 전부 구현할 필요가 없다.  사용하기도 쉽고 더 간결하게 나타낼 수 있다.

그런데 열거형에는 문제가 하나 있다. **타입 안전 열거패턴은 클래스이기 때문에 상속을 통한 확장이 가능하다. 반면 열거 타입은 확장이 불가능하다.**

- 열거형을 확장하지 못하는 것은 설계 실수가 아니다.
	- 대부분의 상황에서 열거 타입을 확장하는 것은 좋지 않은 생각이다.
	- 확장한 타입의 원소는 기반 타입의 원소로 취급하지만, 그 반대가 성립하지 않는다면 오히려 이상하다.

```java
UserRole { ADMIN, USER } 이라는 열거형이 있다고 치자. 더해
ExtendsUserRole { SUPER_ADMIN, GUEST } 로 확장했다고 가정해보자.

GUEST는 UserRole.USER로 취급된다. 그러나 실제 권한이 다를 것이다.
그런데 USER는 ExtendsUserRole.GUEST로 취급되지 않는다.

?? 역할 간의 상호 참조와 일관성에서 문제가 있다. 이상하다.
```

- 위에서 보다시피 열거형의 확장은 복잡하고, 고려해야 할 부분이 많다.
	- 기반 타입과 확장된 타입들의 원소 모두를 순회할 방법도 마땅치 않다.
	- 고려할 요소가 늘어 설계와 구현이 복잡해진다.

그러나 확장 가능한 열거 타입이 어울리는 경우가 몇 있다. 열거형을 확장해보자.

### 인터페이스로 열거형 확장하기

열거 타입은 인터페이스를 구현할 수 있다. 우리는 열거형이 Comparable과 Serializable을 구현하고 있다는 사실을 이미 알고 있다.

우선 확장 가능한 열거 타입이 어울리는 경우는 연산 코드가 있다.

인터페이스를 구현할 수 있다는 사실을 이용해 확장을 해보자.

```java
public interface Operation {
	double apply(double x, double y);
}
```

```java
public enum BasicOperation implements Operation {
	PLUS("+") {
		@Override
		public double apply(double x, double y) {
			return x + y;
		}
	},
	MINUS("-") {
		@Override
		public double apply(double x, double y) {
			return x - y;
		}
	},
	TIMES("*") {
		@Override
		public double apply(double x, double y) {
			return x * y;
		}
	},
	DIVIDE("/") {
		@Override
		public double apply(double x, double y) {
			return x / y;
		}
	}
	
	private final String symbol;

	//...
}
```

- 여기서 만약 ^의 지수 계산과, %의 나머지 계산을 추가하려면 어떻게 해야할까?
	- 물론 BasicOperation에 하나하나 추가해도 된다. 
	- 그러나 우리는 열거형의 확장을 공부하고 있기 때문에 확장하는 방법을 보자.

```java
public enum ExtendedOperation implements Operation {
	EXP("^") {
		@Override
		public double apply(double x, double y) {
			return Math.pow(x, y);
		}
	},
	REMAINDER("%") {
		@Override
		public double apply(double x, double y) {
			return x % y;
		}
	};
	
	private final String symbol;
	
	//...
}
```

```java
private static void test(Collection<? extends Operation> opSet, double x, double y) }
	for (Operation op : opSet) {
		System.out.printf("%f %s %f = %f%n",
						x, op, y, op.apply(x, y));
	}
} //한정적 와일드카드 타입을 통해 Operation의 하위 타입만 받아 순회한다.

public static void main(String[] args) {
	double x = Double.parseDouble(args[0]);
	double y = Double.parseDouble(args[1]);
	test(Arrays.asList(ExtenedOperation.values()), x, y);
}

// 4.000000 ^ 2.000000 = 16.000000
// 4.000000 % 2.000000 = 0.000000
```

- 우선 확장을 하는 이유에 대해 생각해보자.
	- SRP: 단일 책임 원칙을 지킴으로써 유지보수 및 관리에 도움이 된다. 기존 BasicOperation에 영향을 주지 않으면서 기능을 확장한다.
	- 모듈화: 각 부분이 독립적으로 동작할 수 있도록 한다.
	- 기존 코드의 변경없이 다양한 구현체를 쉽게 추가할 수 있다.
- apply()가 인터페이스에 선언되어 있어 열거 타입에 따로 추상 메서드로 선언하지 않아도 된다.
- 새로 작성한 연산이라도 기존에 Operation 인터페이스를 사용하도록 구현한 부분이라면 문제없이 동작한다.
	- 위의 예시에서도 확장한 연산들만을 순회한다. 매개변수에 기존 연산(사칙연산)을 그대로 넣어 사용해도 기존 연산을 순회하며 결과를 냈을 것이다.
	- 유연하게 동작한다. 인터페이스를 구현하는 여러 열거 타입을 사용할 수 있다.

그러나 인터페이스를 이용해 확장 가능한 열거 타입을 흉내내는 이 방식도 문제가 하나 있다. 상속이 아니기 때문에 열거 타입끼리 구현을 공유할 수 없다.

### 열거 타입끼리의 공통 부분 처리

기존 열거타입과 확장한 열거 타입에 공통 부분이 있다면 공유하여 중복 코드를 관리하고 싶을 것이다. 그런데 인터페이스를 통해 확장을 흉내낸것만으로 상속처럼 공유할 수 없다.

두 가지 방법이 있다.

- 공통 부분이 많지 않다면 인터페이스에 디폴트 메서드를 사용한다.
- 공통 부분이 많다면 별도의 도우미 클래스나 정적 도우미 메서드로 분리한다.

```java
public interface Operation { 
	double apply(double x, double y); 

	default void print(double x, double y) { 
		System.out.printf("%f %s %f = %f%n", x, this, y, apply(x, y)); 
	}
}
```

- 공통으로 사용될 수 있는 부분을 디폴트 메서드로 만든다. 해당 인터페이스를 구현한 열거형이라면 공유해서 사용할 수 있다.

```java
public interface Operation { 
	double apply(double x, double y); 
} 

public class OperationUtils { 
	public static void print(Operation op, double x, double y) { 
		System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y)); 
	} 
	//추가 공통 메서드들...
}

for (BasicOperation op : BasicOperation.values()) { 
	OperationUtils.print(op, x, y); 
} // 사용
```

- 공통되는 부분이 많다면 따로 클래스를 두어 정적 메서드를 통해 공통되는 부분을 관리할 수 있다.

## 정리

열거 타입은 확장할 수 없다. 열거 타입의 확장은 웬만하면 불필요하다. 하지만 가끔 확장이 필요하다고 느낄 수 있다.

그럴때 인터페이스를 구현하는 기본 열거 타입을 함께 사용해 비슷한 효과를 낼 수 있다. 해당 인터페이스를 사용하는 코드에 그대로 새로운 열거 타입을 사용할 수 있을 것이다.

인터페이스를 구현해 확장을 흉내내는 것으로는 공통된 부분의 중복 코드를 없앨 수 없다. 이런 경우 인터페이스의 디폴트 메서드나 정적 도우미 메서드를 활용해보자.