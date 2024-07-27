---
title: Item49 (매개변수가 유효한지 검사하라)
author: leedohyun
date: 2024-07-25 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 매개변수가 유효한지 검사하라

메서드와 생성자의 대부분은 입력 매개변수의 값이 특정 조건을 만족하기를 바란다.

- 인덱스 값은 음수이면 안된다.
- 객체 참조는 null이 아니어야 한다.

이러한 제약은 반드시 문서화해야하고, 메서드 몸체가 시작되기 전에 검사해야 한다.

### 매개변수의 제약의 검사를 미룬다면?

- 메서드가 수행되는 중간에 모호한 예를 던지며 실패할 수 있다.
- 메서드는 잘 수행되는데 잘못된 결과를 반환할 수 있다.
- 메서드는 문제없이 수행됐는데, 그 메서드가 어떤 객체를 이상하게 만들어놔서 그 객체를 사용할 때 해당 메서드와 관련없는 오류를 발생시킬 수 있다.

```java
class Account {
	private String name;
	private double balance;

	//...

	public void deposit(double amount) {
		this.balance += amount;
	}

	public void withdraw(double amount) {
		if (amount <= 0) {
			throw new IllegalArgumentException();
		}

		if (amount > balance) {
			throw new IllegalArgumentException();
		}
		this.balance -= amount;
	}
}
```

- 위와 같이 deposit()에서 amount 매개변수 값을 검사하지 않는다고 가정해보자.
- 문제없이 사용하다가 deposit()에 음수를 전달하고 그 값이 balance값을 음수로 만들어버린 상황이 발생했다.
- 그러면 withdraw()를 사용할 때 예외를 맞이할 것이다.

분명 deposit()에서 잘못했는데 withdraw()를 사용할 때 예외를 맞게 되어 혼란이 올 수 있다. 물론 위 예시는 매우 간단하기 때문에 어디서 문제가 발생했는지 쉽게 찾을 수 있다.

하지만 조금만 더 복잡한 구조가 찾아온다면 문제를 찾기 쉽지 않을 수 있다.

***메서드가 직접 사용하지는 않지만 나중에 쓰기 위해 저장되는 매개변수는 신경써서 검사해야 한다.***

### 바로 검사하자. 그리고 문서화하자.

- 즉각적이고 깔끔한 방식으로 예외를 던질 수 있다.

또한 public과 protected 메서드는 매개변수 값이 잘못됐을 때 던지는 예외를 문서화하는 것이 좋다. @throws 자바독 태그를 사용하며 된다.

매개변수의 제약을 문서화한다면 그 제약을 어겼을 때 발생하는 예외도 함께 기술해야 한다.

```java
/**  
 * @param amount 예금할 금액이다. (양수여야 한다.)
 * @throws IllegalArgumentException amount가 0보다 작거나 같으면 발생한다.
 */
public void deposit(double amount) {  
	 if (amount <= 0) {  
		 throw new IllegalArgumentException("Deposit amount must be positive.");  
	 }  
	 this.balance += amount;  
}
```

> 유효성 검사의 예외

- 매개변수 유효성 검사 비용이 지나치게 높거나, 실용적이지 않거나, 계산 과정에서 암묵적으로 검사가 수행되는 경우도 있다.

예를 들어 Collections.sort(list)에서 list의 객체들은 상호 비교될 수 있는 타입이어야 한다. 하지만 이에 대해 검사를 하지는 않는다.

잘못된 객체가 들어가있다면 해당 객체를 비교 시 ClassCastException을 던질 것이기 때문에 미리 검사하는 것이 별 의미가 없어지는 것이다.

### null 검사하는 방법

- javautil.Obejcts.requireNonNull() 메서드 사용하기

```java
public Account(String name, double balance) {
	this.name = Objects.requireNonNull(name, "Required: name");
	this.balance = balance;
}
```

- @Nullable 애노테이션 사용하기

```java
class Account {
	private String name;
	private double balance;
	
	@Nullable
	private String nickName;

	//...
}
```

### 그 외 매개변수 유효성 검사

- public이 아닌 메서드라면?
	- assert(단언문)을 사용해 조건을 검사할 수 있다.
	- 단언문들은 자신이 단언한 조건이 무조건 참이라고 선언한다.
	- 실패하면 AssertionError를 던진다. 런타임에 아무런 효과도 성능 저하도 없다. 

```java
private static void sort(long[] a, int offset, int length) {
	assert a != null;
	assert offset >= 0 && offset <= a.length;
	assert length >= 0 && length <= a.length - offset;
	//.. 계산 수행
}
```

참고로 위 방법은 현업에서 자주 사용되지 않는다고 한다. 보통 명시적인 예외를 던져 매개변수를 검증한다.


## 정리

이번 아이템은 매개변수에 제약을 두는 것이 좋다는 뜻이 아니다. 오히려 메서드는 범용적으로 설계해야 한다. 매개변수의 제약은 적은 것이 좋다.

하지만 매개변수에 제약이 존재해야 한다면 해당 제약을 미리 검사하고 문서화해야 한다. 이러한 노력으로 실제 오류를 걸러낼 때 충분히 노력에 대한 보상을 받을 수 있다.