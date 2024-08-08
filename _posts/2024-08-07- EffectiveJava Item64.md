---
title: Item64 (객체는 인터페이스를 사용해 참조하라)
author: leedohyun
date: 2024-08-07 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 객체는 인터페이스를 사용해 참조하라

[아이템 51](https://ldhapple.github.io/posts/EffectiveJava-Item51/)에서 매개변수 타입으로 클래스가 아닌 인터페이스를 사용하라고 했다. 이것을 확장하면 객체는 인터페이스를 통해 참조하라 라고 할 수 있다.

***적합한 인터페이스만 있다면 매개변수뿐 아니라 반환값, 변수, 필드를 전부 인터페이스 타입으로 선언하는 것이 좋다.***

***인터페이스를 타입으로 사용하는 습관을 길러둔다면 프로그램이 훨씬 유연해질 것이다.***

### 인터페이스 사용의 장점

- 초기 요구사항으로 인증 코드를 이메일로 전송하는 기능을 만들어 달라고 요청이 왔다.

```java
public class EmailService {
	public void sendEmail(User user, Code code) {
		//...
	}
}
```
```java
public class AuthService {
	private final EmailService emailService;

	//...
	
	public void sendAuthCode(User user, Code code) {
		emailService.sendEmail(user, code);
	}
}
```

그런데 다 개발하고 보니, 인증 코드를 SMS로도 보낼 수 있도록 해달라는 요청이 왔다.

```java
public class SMSService {
	public void sendSMS(String phoneNumber, Code code) {
		//...
	}
}
```
```java
public class AuthService() {
	private final EmailService emailService;
	private final SMSService smsService;

	//...

	public void sendAuthCode(Type type, User user, Code code) {
		if (type == Type.EMAIL) {
			emailService.sendEmail(user.getEmail(), code);
		} else if (type == Type.SMS) {
			smsService.sendSMS(user.getPhoneNumber(), code);
		} else {
			throw new IllegalArgumentException();
		}
	}
}
```

- 매개변수도 달라지고, 코드 가독성도 안좋아지게끔 코드가 변경되고 있다.
- 기존에 작성했던 EmailService도 바뀌게 되었다.

사실 여기까지는 괜찮을 수 있지만 만약 인증에 관한 요구사항이 점점 추가된다면 복잡해질 수 있다.

***인터페이스를 사용하도록 수정한다면?***

```java
public interface AuthCodeSender {
	void sendCode(User user, Code code);
}
```
```java
public class EmailService implements AuthCodeSender {
	@Override
	public void sendCode(User user, Code code) {
		//...
	}
}

public class SMSService implements AuthCodeSender {
	@Override
	public void sendCode(User user, Code code) {
		//...
	}
}

public class PassService implements AuthCodeSender {
	@Override
	public void sendCode(User user, Code code) {
		//...
	}
}
```
```java
public class AuthService {
	private final AuthCodeSender authCodeSender;

	//...
		
	public void sendAuthCode(User user, Code code) {
		authCodeSender.sendCode(user, code);
	}
}
```

- 이제 필요한 경우에 맞추어 AuthCodeSender를 구현하는 구현 클래스들을 만들고, 사용할 때 그 구현 클래스를 참조하도록 하게 만들면 된다.
- 기존 코드의 구조를 수정할 필요가 없어진 것이다.
- 코드가 더 유연하고, 확장이 쉽도록 바뀌었다. 유지보수가 쉽다. 요구사항이 더 많다면 장점은 크게 와닿을 것이다.
- 수정이 필요하더라도 하위 클래스만 변경하면 되고, 비즈니스 코드는 변하지 않는다.
- 테스트 또한 용이해진다.

### 주의 사항

인터페이스가 장점을 가진다고 해서 모든 부분을 추상화하려고 하면 안된다. 추상화 또한 비용이기 때문에 추상화가 필요한 상황에서 사용하면 된다.

따라서 기존에는 추상화를 하지 않다가 추가적인 요구사항이 들어온다거나 할 때 인터페이스를 도입하는 리팩토링을 하는 것을 권장한다.

또, 만약 적합한 인터페이스가 없다면 클래스(구현체)로 참조해야 하는데, 클래스의 계층 구조 중 필요한 기능을 만족하는 가장 덜 구체적인 상위의 클래스를 타입으로 사용하는 것을 권장한다.

ex) PriorityQueue, String 같은 값 클래스

## 정리

적합한 인터페이스가 있다면 구현체보다는 인터페이스를 사용하는 것이 좋다. 객체 지향 설계 원칙을 준수할 수 있고, Spring은 DI를 통해 이를 더 쉽게 가능하도록 지원해준다.

여러 장점이 많으니 습관화하는 것이 좋다. 단 추상화도 비용이기 때문에 인터페이스를 도입하는 것은 필요할 때 하면 된다.