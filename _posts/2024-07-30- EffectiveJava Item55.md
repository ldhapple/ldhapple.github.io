---
title: Item55 (옵셔널 반환은 신중히 하라)
author: leedohyun
date: 2024-07-30 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 옵셔널 반환은 신중히 하라

Java 8 이전에는 메서드가 특정 조건에서 값을 반환할 수 없을 때 취할 수 있는 선택지가 두 가지 있었다.

- 예외를 던진다.
- null을 던진다. (반환 타입이 객체 타입일 경우)

그러나 두 방법에는 모두 허점이 있다.

- 예외는 진짜 예외적인 상황에서만 사용해야 한다.
	- 예외를 생성할 때 스택 추적 전체를 캡처하는데 드는 비용 문제도 생각해야 한다.
- null을 반환하는 것은 이전 아이템에서 다루었듯 null을 처리하는 방어 코드를 따로 구현해주어야 한다.

### 그러나 Optional의 등장

Java 8 이후 같은 경우의 선택지가 하나 더 늘었다. 바로 Optional이다.

- Optional
	- Optional은 T 타입 참조를 하나 담고 있거나, 아무것도 담지 않을 수 있다.
	- 원소를 최대 1개 가질 수 있는 불변 컬렉션이다.
		- 실제 Collection을 구현한 것은 아니지만, 원칙적으로 그렇다.
	- 보통 T를 반환하는데 특정 조건에서 아무것도 반환하지 않아야 할 경우 T 대신 Optional을 반환하도록 하면 된다.
		- 유효한 반환값이 없을 때 빈 결과를 반환하는 메서드가 만들어진다.
		- 옵셔널을 반환하는 메서드는 예외를 던지는 메서드보다 유연하고 사용하기 쉽다.
		- null을 반환하는 메서드보다 오류 가능성 또한 적다.

```java
//null을 반환하는 경우
private Map<String, User> users = new HashMap<>();

public User findUserByUsername(String username) {
	return users.get(username);
}
```
```java
//Optional을 반환하는 경우
public Optional<User> findUserByUsername(String username) {
	User user = users.get(username);
	if (user == null) {
		return Optional.empty();
	}
	return Optional.of(user);
}
```

- Optional로 반환하는 구현 코드는 어렵지 않다.
	- 적절한 정적 팩토리를 사용해 옵셔널을 생성해주면 된다.
	- 그리고 결과가 없을 경우 빈 결과를 반환하는 빈(empty) 옵셔널을 반환한다.
- Optional.empty()
	- 빈 옵셔널을 생성한다.
- Optional.of(user)
	- user 값이 들은 옵셔널을 생성한다.
	- 단, of() 메서드에 null을 넣으면 안된다. NullPointerException이 발생한다.
	- null 값을 허용하는 옵셔널을 만들고 싶다면 Optional.ofNullable(value)를 사용하면 된다.

> **그러나 옵셔널을 반환하는 메서드에서는 null을 반환하지 말자.**

Optional.ofNullable(null); 을 통해 null을 담은 옵셔널을 생성할 수는 있다.

그런데 Optional이 등장한 취지를 다시 생각해보자. 예외를 던지거나, null을 반환하는 방식에는 문제가 있기 때문이었다.

null을 포장하는 객체를 사용해 다시 null을 담을 이유가 있을까?

### Optional을 사용하는 경우

스트림의 종단 연산 중 상당수가 옵셔널을 반환하도록 구현되어 있다. 그 종단 연산들에서는 옵셔널을 선택하는 것이 합리적이라고 판단한 것이다.

그렇다면 우리가 메서드를 작성할 때 예외를 던지거나 null을 반환하지 않고 옵셔널을 선택해야 하는 경우는 언제일까?

우선 Optional의 기능들을 보자.

#### Optional의 기능들

기본적으로 Optional은 검사 예외와 취지가 비슷하다. 반환 값이 없을 수도 있음을 API 사용자에게 명확하게 알려준다.

만약 Optional이 비검사 예외를 던지는 것, null을 반환하는 것과 비슷한 개념이라면 API 사용자가 오류가 날 수 있는 상황에 대한 인지를 하지 못해 예상치 못한 오류를 만나게 될 것이다. 

반면 검사 예외는 클라이언트는 **반드시 이에 대해 처리하는 코드를 작성해야 하기 때문에 그런 취지에서 Optional은 검사 예외와 비슷**하다고 한 것이다.

따라서 메서드가 Optional을 반환한다면 클라이언트는 **값을 받지 못했을 때 취할 행동을 선택해야 한다.**

Optional에 대해 어떤 행동을 취할 수 있는 지 보자.

- orElse()
	- 기본값을 설정할 수 있다.

```java
User basicUser = new User(//...);

User findUser = findUserByUsername("홍길동").orElse(basicUser);
```

- orElseThrow()
	- 원하는 예외를 던질 수 있다.

```java
User findUser = findUserByUsername("홍길동").orElseThrow(UserNotFoundException::new);
```

- get()
	- Optional에 값이 존재할 때 그 값을 반환한다.
- 항상 값이 채워져 있다고 가정할 수 있다.
	- 곧바로 값을 꺼내 사용하는 것이며 항상 값이 채워져 있다고 잘못 판단한 경우라면 NoSuchElementException이 발생한다.

```java
User findUser = findUserByUsername("홍길동").get();
```

- isPresent()
	- Optional에 값이 존재하는 지 여부를 확인한다. boolean을 반환한다.

```java
boolean find = findUserByUsername("홍길동").isPresent();
```

- ifPresent()
	- Optional에 값이 존재할 때, 주어진 Consumer와 함께 그 값을 처리한다.

```java
findUserByUsername("홍길동").ifPresent(findUser -> System.out.println(findUser);
```

- map
	- Optional에 값이 존재하면 그 값을 주어진 Function을 적용한 결과로 변환하고 그렇지 않다면 빈 Optional을 반환한다.
- filter
	- Optional에 값이 존재하고, 그 값이 주어진 Predicate를 만족하면 그 값을 포함하는 Optional을 반환하고 그렇지 않으면 빈 Optional을 반환한다.

> orElseGet()

orElse()에 설정하는 기본 값은 Optional의 값이 있어도 생성된다. 이 말은 Optional의 값이 null이 아니더라도 쓸데없는 값이 생긴다는 뜻이다.

따라서 만약 비용이 큰 값을 기본값으로 설정하게 된다면 orElseGet()을 사용해보자.

- Optional에 값이 존재하면 그 값을 반환하고, 없다면 주어진 Supplier에 의해 생성된 값을 반환한다.
	- 값이 처음 필요할 때 Supplier로 생성하는 방식이므로 초기 설정 비용 문제를 해결할 수 있다.

```java
User findUser = findUserByUsername("홍길동").orElseGet(() -> getBasicUser());
```

> Optional과 Stream

Optional은 Stream을 지원한다.

스트림을 사용한다면 Optional들을 Stream에 담아 그 중 채워진 옵셔널들에서만 값을 뽑아 처리하는 경우가 드물지 않다.

```java
Stream<Optional<User>> streamOfOptionals = Stream.of(
	findUserByUsername("홍길동"),
	findUserByUsername("김철수"),
	findUserByUsername("이짱구")
);

List<User> nonEmptyUsers = streamOfOptionals
	.filter(Optional::isPresent)
	.map(Optional::get)
	.toList();
```

더해 Java 9 이후로 Optional에 stream() 메서드가 추가되었다. Optional을 Stream으로 변환해주는 어댑터이다.

- stream()
	- Optional에 값이 있으면 그 값을 원소로 담은 스트림으로, 값이 없다면 빈 스트림으로 변환한다.
	- 이를 Stream의 flatMap 메서드와 조합하면 위 코드를 명료하게 바꿀 수 있다.

```java
List<User> nonEmptyUsers = streamOfOptionals
	.flatMap(Optional::stream)
	.toList();
```

#### 결과가 없을 수 있으며, 클라이언트가 이 상황을 특별하게 처리해야 하는 경우

기능들을 확인했으니, 이제 어떤 경우에 옵셔널을 쓰고 어떤 경우에 옵셔널을 쓰면 안되는 지 알아보자.

옵셔널을 사용하는 기본 규칙은 결과가 없을 수 있고, 클라이언트가 이 상황을 특별하게 처리해야 하는 경우에 Optional을 반환하면 된다.

단 주의해야 할 점이 있다.

- Optional도 새로 할당하고 초기화해야 하는 객체이다.
	- 즉 비용이 든다는 이야기이다.
- 따라서 성능이 중요한 상황에서는 옵셔널이 맞지 않을 수 있다.
	- 성능을 고려해서 사용해야 한다.

#### 컬렉션, 스트림, 배열, 옵셔널 같은 컨테이너 타입은 옵셔널로 감싸면 안된다.

우선 컨테이너 타입은 옵셔널로 감싸면 안된다.

사실 당연한 얘기이다. 당장 [이전 아이템](https://ldhapple.github.io/posts/EffectiveJava-Item54/)에서 null 대신 빈 컬렉션이나 배열을 반환했다.

단순히 빈 컨테이너를 그대로 반환하면 방어 코드를 작성해야 하는 문제 등이 없다. 굳이 옵셔널로 한 번 더 감쌀 필요가 없는 것이다.

#### 박싱 타입을 Optional로 감쌀 필요 없다.

박싱된 기본 타입을 담는 옵셔널은 기본 타입보다 무거울 수 밖에 없다. 기본 타입을 두 번 감싼 형태이기 때문이다.

그래서 int, long, double 전용 Optional 클래스가 존재한다.

```java
OptionalInt, OptonalLong, OptionalDouble
```

이 옵셔널들도 기본 옵셔널이 제공하는 메서드를 거의 다 제공한다. 단, 위에 언급된 기본 타입이 아닌 기본 타입들 Boolean, Byte, Character, Float 등은 예외일 수 있다.

#### 그 외의 경우

Optional을 사용해야 하는 기본 원칙을 제외하고는 사용해야 하는 경우를 언급하지 않고 있다.

그 이유는 다른 쓰임으로는 적절하지 않기 때문이다.

- 옵셔널을 맵의 값으로 사용한다?
	- 맵 안에 키가 없다는 사실을 나타내는 방법이 2가지가 된다.
		- 키 자체가 없는 경우 / 그 키가 빈 옵셔널일 경우
	- 복잡성만 높아져 혼란과 오류 가능성을 키울 뿐이다.

**옵셔널을 컬렉션의 키, 값, 원소나 배열의 원소로 사용하는 것이 적절한 상황은 거의 없다.** 

> 조금 예외적인 경우 (기본 타입 선택적 필드)

옵셔널을 인스턴스 필드에 저장해두는 것이 필요한 경우가 있을까?

거의 없지만, 가끔 유용하게 쓸 수 있는 곳이 있다.

Class에 필수 필드와 선택적 필드가 있는 경우이다. 필수 필드가 아닌데 그 타입이 기본 타입이라면 값이 없음을 나타낼 방법이 마땅치 않다.

따라서 이런 경우는 필드 자체를 옵셔널로 선언하고 게터 메서드들이 옵셔널을 반환하도록 해주면 유용하다.

## 정리

결과가 없을 수 있고, 클라이언트가 이 상황을 특별하게 처리해야 하는 경우의 메서드라면 Optional을 반환하는 것을 고려해야 한다.

그러나 Optional에는 성능 저하가 있을 수 밖에 없어 성능에 민감한 메서드라면 null을 반환하거나 예외를 던지는 것도 생각해야 한다.

그리고 Optional을 반환값 이외에 쓰는 용도는 거의 없다. 만약 Optional을 사용하게 된다면 쓸데없이 컨테이너나 박싱 타입을 Optional로 감싸는 경우를 주의해야 한다.