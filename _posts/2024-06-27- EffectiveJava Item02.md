---
title: Item2 (생성자에 매개변수가 많다면 빌더를 고려하라)
author: leedohyun
date: 2024-06-27 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 생성자에 매개변수가 많다면 빌더를 고려하라

아이템 1에서의 정적 팩토리 메서드와 생성자에는 똑같은 제약이 있다.

### 생성자와 정적 팩토리 메서드는 매개변수가 많을 때 적절히 대응하기 힘들다.

```java
public class User {
	private int age;             // 필수
    private int phoneNumber;     // 필수
    private int weight;          // 선택
    private int tall;            // 선택
    private int birthDay;        // 선택
	//....
}
```

이러한 User 객체가 있다고 생각해보자. birthDay와 같은 변수의 경우 사용자가 선택적으로 입력할 수 있는 부분이라고 가정한다.

***과거에는 이런 경우 점층적 생성자 패턴을 즐겨 사용했다.***

> 점층적 생성자 패턴

```java
private User(int age, int phoneNumber) {
    this.age = age;
    this.phoneNumber = phoneNumber;
}

private User(int age, int phoneNumber, int weight) {
    this(age, phoneNumber);
    this.weight = weight;
}

private User(int age, int phoneNumber, int weight, int tall) {
    this(age, phoneNumber);
    this.weight = weight;
    this.tall = tall;
}

private User(int age, int phoneNumber, int weight, int tall, int birthDay ) {
    this(age, phoneNumber);
    this.weight = weight;
    this.tall = tall;
    this.birthDay = birthDay ;
}

//...
```
```java
User user = new User(25, 01012345678, 70, 180, 0305);
```

클래스의 인스턴스를 만들기 위해 원하는 매개변수를 모두 포함한 생성자 중 가장 짧은 것을 선택할 수 있도록 한 것이다.

그러나 이런 구현 방법은 단점이 많다.

지금의 예시는 사실 매개변수가 그렇게 많지는 않다. 그럼에도 코드를 작성하거나 읽는데 꽤나 피곤하다.

- 모든 경우의 수를 전부 구현하는 것이 아니라면, 어떤 매개변수를 위해 필요 없는 매개변수의 값까지 지정해주어야 하는 경우가 있다.
- 각 매개변수의 의미, 값의 순서 등등을 알기 힘들고 코드를 완성해도 읽기 어렵다.
- 가장 큰 문제는 매개변수의 순서가 바뀌었을 때 컴파일러단에서 에러를 잡을 수 없다.

***또 매개변수가 많을 때 활용하는 방법 중 자바 빈즈 패턴이라는 것이 있다.*** 

> 자바 빈즈 패턴

매개변수가 없는 생성자로 객체를 만든 후 setter 메서드를 호출해 원하는 매개변수의 값을 설정하는 방식이다.

```java
public class User {
    private int age;
    private int phoneNumber;
    private int weight;
    private int tall;
    private int birthDay;

	public User() {}

    public void setAge(final int age) { this.age = age; }
    public void setPhoneNumber(final int phoneNumber) { this.phoneNumber = phoneNumber; }
    public void setWeight(final int weight) { this.weight = weight; }
    public void setTall(final int tall) { this.tall = tall; }
    public void setBirthday(final int birthDay) { this.birthDay = birthDay; }
}
```

```java
User user = new User();
user.setAge(25);
user.setPhoneNumber(01011111234);
//...
```

점층적 생성자 패턴의 단점들이 보완되었다. 코드는 비교적 길어졌지만 인스턴스를 생성하기 보다 쉽고 무엇보다 읽기 쉬운 코드라는 점이 장점이다.

그러나 자바빈즈 패턴도 심각한 단점이 존재한다.

- 객체 하나를 만들기 위해 메서드를 여러 개 호출해야 한다.
- 객체가 완전히 생성되기 전까지 일관성이 무너진 상태가 된다.
	- 생성자와 다르게 매개변수의 유효성을 확인할 수 없다.
	- 이러한 객체가 만들어지면 버그가 발생해도 찾기 어렵다.
- 클래스를 불변으로 만들 수 없다. 스레드 안정성을 얻기 위해 추가 작업이 필요하다.

이러한 단점을 완화하고자 생성이 끝난 객체를 수동으로 얼리고, 얼리기 전에는 사용하지 못하도록 막는 일종의 락을 획득하는 방법으로 대처하고는 했다.

그러나 이런 방법도 컴파일러단에서 객체가 얼려져있는 것인지 확인할 수 없어 런타임 오류에 취약하다는 단점이 있었다.

### 빌더 패턴의 등장

점층적 생성자 패턴의 안정성, 그리고 자바 빈즈 패턴의 가독성을 모두 챙긴 빌더 패턴이 등장했다.

필요한 객체를 직접 만드는 대신, 필수 매개변수만으로 생성자나 정적 팩토리 메서드를 호출해 빌더 객체를 얻는다.

이러한 빌더 객체가 제공하는 일종의 setter 메서드들로 원하는 매개변수를 선택해 설정하고 build() 메서드를 호출하면 원하는 객체를 얻는 방식이다.

```java
public class User {
    private final int age;
    private final int phoneNumber;
    private final int weight;
    private final int tall;
    private final int birthday;

    private User(Builder builder) {
        this.age = builder.age;
        this.phoneNumber = builder.phoneNumber;
        this.weight = builder.weight;
        this.tall = builder.tall;
        this.birthday = builder.birthday;
    }

    public static class Builder {
	    //필수 매개변수
        private final int age;
        private final int phoneNumber;
        
        //선택 매개변수 - 기본값으로 초기화
        private int weight = 0;
        private int tall = 0;
        private int birthDay = 0;
        
        public Builder(int age, int phoneNumber) {
            this.age = age;
            this.phoneNumber = phoneNumber;
        }

        public Builder weight(int weight) {
						// validation 가능
            this.weight = weight;
            return this;
        }

        public Builder tall(int tall) {
            this.tall = tall;
            return this;
        }

        public Builder birthday(int birthday) {
            this.birthday = birthday;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
}
```
```java
User user = new User.Builder()
		.age(25)
		.birthDay(0305)
		.build();
```

간단한 빌더패턴이다. 우선 생성자는 private으로 막아두는 걸로 했다.

값을 세팅할 때 빌더객체를 반환해 연쇄적으로 호출하여 인스턴스를 만드는 과정을 추상화한다고 볼 수 있다.

위 User 클래스는 불변하게 되었고, 모든 매개변수의 기본 값 또한 한 곳에 모아둘 수 있게 되었다.

### 빌더 패턴은 계층적으로 설계된 클래스와 함께 사용하기 좋다.

```java
public abstract class Pizza {
	public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
	final Set<Topping> toppings;

	abstract static class Builder<T extends Builder<T>> {
		EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

		public T addTopping(Topping topping) {
			toppings.add(Objects.requireNonNull(topping));
			return self();
		}
		
		abstract Pizza build();

		protected abstract T self();
	}
	
	Pizza(Builder<?> builder) {
		toppings = builder.toppings.clone();
	}
}
```

- Pizza클래스는 추상 클래스이고, 각각의 피자는 여러 종류의 토핑을 갖는다.
- addTopping() 메서드를 통해 토핑을 추가하고, 메서드 체이닝을 지원하기 위해 자기 자신을 반환한다. (Builder)
- Pizza의 생성자는 Builder 객체를 받아 빌더를 통해 세팅된 토핑을 복사한다.

```java
public class NyPizza extends Pizza {
	public enum Size { SMALL, MEDIUM, LARGE }
	private final Size size;

	public static class Builder extends Pizza.Builder<Builder> {
		private final Size size;

		public Builder(Size size) {
			this.size = Objects.requireNonNull(size);
		}

		@Override
		public NyPizza build() {
			return new NyPizza(this);
		}

		@Override
		protected Builder self() {
			return this;
		}
	}

	private NyPizza(Builder builder) {
		super(builder);
		size = builder.size;
	}
}
```

- Size라는 Enum 열거형이 추가되었다. Builder를 통해 Size를 세팅할 수 있으며 NyPizza의 생성 시 토핑 그리고 size 필드를 초기화 한다.

```java
public class Calzone extends Pizza {
	private final boolean sauceInside;

	public static class Builder extends Pizza.Builder<Builder> {
		private boolean sauceInside = false;

		public Builder sauceInside() {
			sauceInside = true;
			return this;
		}

		@Override
		public Calzone build() {
			return new Calzone(this);
		}

		@Override
		protected Builder self() {
			return this;
		}
	}

	private Calzone(Builder builder) {
		super(builder);
		sauceInside = builder.sauceInside;
	}
}
```

- 위의 뉴욕 피자의 사이즈와 비슷하게 sauce를 넣을 지 말지 여부를 추가적으로 초기화 한다.

사용 부분을 보자.

```java
NyPizza pizza = new NyPizza.Builder(SMALL)
		.addTopping(SAUSAGE)
		.addTopping(ONION)
		.build();

Calzone calzone = new Calzone.Builder()
		.sauceInside()
		.addTopping(HAM)
		.build();
```

위와 같이 빌더 객체 하나로 여러 객체를 순회 (Pizza -> NyPizza)하며 만들 수 있고, 빌더에 넘기는 매개변수에 따라 다른 객체를 만들 수 있다.

NyPizza의 build() 메서드를 호출하면 NyPizza를 생성하는데, 그 NyPizza의 생성자 안에 Pizza 추상클래스의 생성자를 호출해 Builder에 세팅된 토핑 필드들을 추가한 Pizza가 생성될 수 있다.

종합해보면 계층적으로 잘 설계된 클래스들에 Builder 패턴을 사용하면 객체 생성의 구현에 있어서도, 생성하는 코드를 읽는데에서도 장점이 있다.

### 그러나 Builder 패턴에도 단점이 있다.

결국 코드들을 종합적으로 보면, 어찌됐건 어떠한 객체를 만들기 위해 Builder라는 객체를 만들어야 한다.

Builder 생성 비용이 크지는 않지만 성능에 민감한 상황이라면 문제가 될 수 있는 것이다.

또한 점층적 생성자 패턴보다 구현에 있어 코드가 복잡하기 때문에 매개변수가 4개 이상은 되어야 값어치를 한다. (이런 부분은 사실 @Builder 같은 롬복의 애노테이션으로 극복할 수 있는 문제인 것 같다.)

### 정리

성능에 아주 민감한 상황이 아니라면 처음부터 웬만하면 Builder 패턴을 사용하는 것이 나을 수 있다.