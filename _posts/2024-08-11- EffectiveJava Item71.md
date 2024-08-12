---
title: Item71 (필요 없는 검사 예외 사용은 피하라)
author: leedohyun
date: 2024-08-11 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 필요 없는 검사 예외 사용은 피하라

이전 아이템인 아이템 70에서도 살짝 언급한 부분이다.

물론 검사 예외를 잘 활용하면 발생한 예외를 프로그래머가 처리하여 안전성을 높이고, 프로그램의 질을 높일 수 있다.

하지만 검사 예외를 과하게 사용하면 사용하기 불편한 API가 된다.

검사 예외를 사용하게 되면, catch 블록을 두어 그 예외를 붙잡아 처리하거나 더 바깥으로 던져 문제를 전파해야 한다. 바깥으로 전파하는 과정에서 던져지는 예외에 의존하게 되는 사태까지 벌어질 수 있다.

또한 두 경우 모두 API 사용자에게 부담을 준다. 더해 검사 예외를 던지는 메서드는 스트림 안에서 직접 사용할 수 없어 Java 8 이후부터는 그 부담이 더욱 커졌다.

- API를 제대로 사용해도 발생할 수 있는 예외
- 프로그래머가 의미있는 조치를 취할 수 있는 경우

위 두가지 경우라면 검사 예외를 사용해도 좋을 것이다. 하지만 그렇지 않다면 비검사 예외를 사용하는 것이 좋다.

```java
} catch (TheCheckedException e) {
	throw new AssertionError();
}
```
```java
} catch (TheCheckedException e) {
	e.printStackTrace();
	System.exit(1);
}
```

- 이 두 가지 검사 예외를 처리하는 방법이 과연 맞는 방법일까?
- 두 처리 방법 모두 당장 발생한 검사 예외를 잡는 역할만 수행할 뿐 근본적인 조치를 취하지 못했다.
- 검사 예외는 프로그래머가 그 예외를 잘 다룰 수 있는 상황일 때 비로소 사용할 수 있다고 생각한다.
	- 더 나은 방법이 없다면 비검사 예외를 선택하면 된다.

### 단 하나의 검사 예외만 던질 때 부담은 더 커진다.

검사 예외는 프로그래머에게 그 예외를 처리해야 하는 부담감을 지운다.

이러한 부담은 메서드가 단 하나의 검사 예외만 던질 때 더 커진다.

이미 다른 검사 예외도 던지는 상황에서 또 다른 검사 예외를 추가하는 경우를 가정한다면, 기껏해야 catch문 하나 추가하는 정도밖에 되지 않는다.

하지만 검사 예외가 단 하나 뿐이라면, 그 예외 하나 때문에 API 사용자는 try 블록을 추가해야 하고 스트림에서 직접 사용하지 못한다.

따라서 이러한 경우 다른 방법을 생각해야 한다.

### 단 하나의 검사 예외만 던지는 경우의 대처법

단 하나의 검사 예외만 던지는 것은 부담이 크다는 것을 알았다.

이 상황을 회피하려면, 검사 예외를 던지지 않는 방법밖에 없다.

검사 예외를 던지지 않는 방법 2가지가 있다.

#### 적절한 결과 타입을 담은 옵셔널을 반환한다.

- 검사 예외를 밖으로 던지는 대신 단순하게 빈 옵셔널을 반환한다.
	- 이 방식의 단점은 예외가 발생한 이유를 알려주는 부가 정보를 담을 수 없다는 점이다.

```java
try {
	User user = repository.findUserById(userId);
} catch (CheckedException e) {
	//...
}
```

- findUserById() 메서드에서 체크 예외를 발생시켜 try-catch 블록으로 처리하거나 밖으로 던져주어야 하는 상황이라고 가정하자.

```java
public Optional<User> findUserById(String userId) {
	try {
		return Optional.ofNullable(repository.findUserById(userId));
	} catch (SQLException e) {
		log.error("Database Error", e);
		return Optional.empty();
	}
}
```

- 핵심은 예외를 밖으로 던지지 않아, 해당 API를 사용하는 사용자는 부담이 없어진다는 것이다.

#### 검사 예외를 던지는 메서드를 2개로 쪼개 비검사 예외로 바꾼다.

- 검사 예외를 던지는 메서드를 두 개로 쪼개어 비검사 예외로 바꾼다
- 문제가 발생하는 지점을 boolean으로 받아 true일 경우에만 로직을 실행시키는 방법이다.

```java
public boolean isExist(String userId) {
	try {
		repository.findUserById(userId);
		return true;
	} catch (SQLException e) {
		return false;
	}
}
```
```java
public User findUserById(String userId) {
	if (!isExist(userId)) {
		throw new RuntimeException("유저가 존재하지 않습니다.");
	}

	try {
		return repository.findUserById(userId);
	} catch (SQLException e) {
		throw new RuntimeException("DataBase Error", e);
	}
}
```

이러한 리팩토링을 모든 상황에 적용할 수는 없지만, 적용할 수 있다면 쓰기 편한 API를 제공할 수 있다.

의존적이지 않고 보다 유연하다.

> 주의 사항

isExist() 메서드는 상태 검사 메서드에 해당한다.

[아이템 69](https://ldhapple.github.io/posts/EffectiveJava-Item69/)에서 상태 검사 메서드 대신 옵셔널이나 null을 반환하는 선택지가 있다고 했다.

이 경우는 외부 동기화 없이 여러 쓰레드가 동시에 접근할 수 있을 때 외부 요인에 의해 상태가 변할 수 있다면 상태 검사 메서드가 오동작할 수 있다.

따라서 이런 경우에는 위와 같은 리팩토링이 유효하지 않다. 또한 isExist()에서 작업의 일부를 대신 수행해주고 있는데 이 또한 성능에서 손해이다.

따라서 검사 예외를 피하려는 노력도 경우에 맞게 리팩토링해 대처해야 한다.

## 정리

검사 예외는 프로그램의 안전성을 높여줄 수 있다.

처음 자바 설계 당시 검사 예외가 더 나은 선택지라고 생각되기도 했다.

하지만 복구할 수 없는 예외가 많아지면서 오히려 검사 예외는 예외를 밖으로 던지는 행위를 통해 API 사용자의 부담감만 지우는 상태까지 왔다.

따라서 예외 상황에서 복구할 방법이 없는 경우에는 비검사 예외를 던지도록 하고, 복구가 가능하고 호출자가 그 처리를 해주기를 원하는 경우 검사 예외를 고민할 수 있다.

하지만 이 또한 검사 예외보다도 옵셔널을 우선적으로 고민하자. 상황을 처리하기에 충분한 정보를 제공할 수 없을 때 검사 예외를 사용하자.