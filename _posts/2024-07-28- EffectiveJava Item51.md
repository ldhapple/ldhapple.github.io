---
title: Item51 (메서드 시그니처를 신중히 설계하라)
author: leedohyun
date: 2024-07-28 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 메서드 시그니처를 신중히 설계하라

- 메서드 시그니처
	- 메서드 명
	- 매개변수의 타입, 순서

리턴 타입과  Exception은 포함되지 않는다.

API 설계 요령들을 한 번 살펴보자.

### 메서드 이름을 신중히 짓자

항상 표준 명명 규칙을 따라야 한다.

이해할 수 있고, 같은 패키지에 속한 다른 이름들과 일관되게 짓는 것이 최우선 목표이다. 그 후 개발자 커뮤니티에서 널리 받아들여지는 이름을 사용하면 된다.

- Java 메서드 명명 방법
	- 소문자 카멜 케이스
		- 메서드 이름은 소문자로 시작하고, 각 단어의 첫 글자를 대문자로 작성한다.
	- 의미있는 이름을 사용하자
		- 메서드 이름은 메서드가 수행하는 작업을 명확히 나타내야 한다.
		- ex) calculateTotalPrice()
	- 동사로 시작해야 한다.
		- 메서드 이름을 동사로 시작해 메서드의 동작을 명확하게 표현한다.
		- ex) getName, setPrice
	- 커뮤니티에서 널리 사용되는 표준 동사를 사용하자.
		- get, set은 접근자와 설정자 메서드에 사용한다. 
		- is, has, can은 boolean 값을 반환하는 메서드에 사용한다.
		- create는 새로운 객체 생성 후 리턴하는 메서드에 사용한다.
		- to는 다른 형태의 객체로 반환할 때 사용한다.
		- ex) add, remove, create, delete, update, find 등등..
	- 약어와 축약어를 지양하자.
		- 혼동을 일으킬 수 있고 가독성을 떨어뜨린다.
		- ex) calculateTotalPrice => calcTotPrice
	- 일관성을 유지하자.
		- 같은 패키지나 클래스 내에서 메서드 이름의 일관성을 유지하자.
		- addCustomer가 있다면 제거에는 removeCustomer로 사용하는 식.
	- 명사의 사용을 지양하자.
		- 메서드이름에 명사만을 사용하면 메서드의 동작을 명확히 표현하기 어렵다.
		- ex) totalPrice()

> 참고: 네이밍 시 고려할 수 있는 사항

- 왜 존재해야 하는가
- 무슨 작업을 하는가
- 어떻게 사용하는가

```java
// 왜 존재해야 하는가: 돈을 입금해야 한다. 
// 무슨 작업을 하는가: 지정된 금액을 계좌에 추가한다. 
// 어떻게 사용하는가: 매개변수로 입금할 금액을 전달한다. 
public void deposit(double amount) { 
	if (amount > 0) { 
		balance += amount; 
	} 
}
```

### 편의 메서드를 너무 많이 만들지 말자

모든 메서드는 각각의 역할을 다해야 한다. 

그러나 메서드가 너무 많은 클래스는 익히고, 사용하고, 문서화하고, 테스트하고, 유지보수하기 어렵다. 인터페이스 또한 마찬가지이다.

```java
public class User {  
  
	private String username;  
	private String email;  
	private int age;  
	  
	public User(String username, String email, int age) {  
		validateUsername(username);  
		validateEmail(email);  
		validateAge(age);  
		  
		this.username = username;  
		this.email = email;  
		this.age = age;  
	}  
	  
	private void validateUsername(String username) {  
		if (username == null || username.isEmpty()) {  
			throw new IllegalArgumentException();  
		}  
	}  
	  
	private void validateEmail(String email) {  
		if (email == null || !email.contains("@")) {  
			throw new IllegalArgumentException();  
		}  
	}  
	  
	private void validateAge(int age) {  
		if (age < 0 || age > 150) {  
			throw new IllegalArgumentException();  
		}  
	}  

	//...
}
```

- 위와 같이 검증하는 로직들도 편의 메서드로 분리할 수 있다.
- 하지만 이렇게 필요한 부분일지라도 과도해지면 단점이 많아진다. (사용, 테스트, 문서화 등)
- 물론 이런 검증 메서드는 애노테이션 등을 사용하거나, 유틸리티 클래스로 분리하는 등의 방식으로 대처할 수도 있을 것이다.

결과적으로는 편의 메서드를 너무 많이 만들지 말도록 하자.

### 매개변수 목록은 짧게 유지하자.

4개 이하가 좋다고 한다.

4개가 넘어가면 매개변수를 전부 기억하기도 어렵고, 만약 같은 타입의 매개변수가 여러 개 연달아 나온다면 매개변수의 순서를 기억해야 해 실수를 할 확률이 높아진다.

> 매개변수가 너무 많을 때 줄이는 방법

- 메서드 쪼개기
	- 메서드를 쪼개 여러 메서드로 만들고 매개변수 목록을 그 메서드들이 나누어 갖도록 하면 된다.
	- 물론 메서드의 숫자가 많아질 수 있지만 직교성을 높여 오히려 메서드 수를 줄여주는 효과가 있다. (분리된 각 메서드를 조합해 여러 기능을 더 쉽게 만들 수 있다.) 
		- 직교성: 공통점이 없는 기능들이 잘 분리되어 있다.

```java
public void createReport(String title, String content, String author, 
		String footer, int fontSize, String fontColor, 
		String backgroundColor, String orientation) { 
	//... 
}
```
```java
validateString();
setFont();
//...
```

- 도우미 클래스 만들기
	- 보통 정적 멤버 클래스로, 매개변수 몇 개를 독립된 하나의 개념으로 볼 수 있을 때 추천된다.
	- 하나의 개념으로 묶일 수 있는 매개변수들을 정적 멤버 클래스로 묶어 가독성 향상 및 구현을 깔끔하게 할 수 있다.

```java
public void giveCard(String suit, int number, int height, int width) {}
```

```java
class Blackjack {
    // 도우미 클래스 (정적 멤버 클래스)
    static class Card {
        private String suit;
        private int number;
        private int height;
        private int width;
    }
    public void giveCard(Card card);
    //...
}
```

- 빌더 패턴의 활용
	- 매개변수가 많고 그 중 일부는 생략해도 괜찮을 때 사용하면 도움이 된다.

```java
public class Car {  
	private String make;  
	private String model;  
	private int year;  
	private String color;  
	private String engineType;  
	  
	private Car(Builder builder) {  
		this.make = builder.make;  
		this.model = builder.model;  
		this.year = builder.year;  
		this.color = builder.color;  
		this.engineType = builder.engineType;  
	}  
	  
	public static class Builder {  
		private String make;  
		private String model;  
		private int year;  
		private String color;  
		private String engineType;  
		  
		public Builder(String make, String model) {  
			this.make = make;  
			this.model = model;  
		}  
		  
		public Builder year(int year) {  
			this.year = year;  
			return this;  
		}  
		  
		public Builder color(String color) {  
			this.color = color;  
			return this;  
		}  
		  
		public Builder engineType(String engineType) {  
			this.engineType = engineType;  
			return this;  
		}  
		  
		public Car build() {  
			return new Car(this);  
		}  
	}  
}
```

### 매개변수의 타입으로는 클래스보다 인터페이스가 낫다.

매개변수로 적합한 인터페이스가 있다면 이를 구현한 클래스가 아닌 그 인터페이스를 직접 사용하는 것이 좋다.

예를 들어 메서드에 HashMap을 넘길 일은 전혀 없다. 대신 Map을 사용하면 된다. 그러면 HashMap뿐 아니라 TreeMap, ConcurrentHashMap 등등 하위 구현체를 인수로 줄 수 있다.

더해 아직 존재하지 않는 Map도 가능하다.

만약 클래스를 사용한다면 클라이언트에게 특정 구현체만 사용하도록 제한하는 형태이고, 입력 데이터가 만약 다른 형태로 존재한다면 특정 구현체로 옮겨 담기 위한 비싼 복사 비용을 치뤄야 한다.

### boolean보다 원소 2개짜리 열거 타입이 낫다

메서드 이름상 boolean을 받아야 의미가 명확할 때를 제외하고, 열거 타입을 사용하면 코드를 읽고 쓰기 더 쉬워진다.

```java
public void switchLight(boolean isOn) { 
	if (isOn) { 
		System.out.println("불이 켜졌습니다."); 
	} else { 
		System.out.println("불이 꺼졌습니다."); 
	} 
}
```
```java
public enum State { ON, OFF }

public void switchLight(State state) { 
	if (state == State.ON) { 
		System.out.println("불이 켜졌습니다."); 
	} else if (state == State.OFF) { 
		System.out.println("불이 꺼졌습니다."); 
	} 
}
```

열거 타입을 사용하면 코드의 가독성과 명확성이 향상된다. 잘못된 값을 전달하는 실수도 줄일 수 있다.