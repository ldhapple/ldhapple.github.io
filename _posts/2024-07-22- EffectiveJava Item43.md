---
title: Item43 (람다보다는 메서드 참조를 사용하라)
author: leedohyun
date: 2024-07-22 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 람다보다는 메서드 참조를 사용하라

람다는 익명 클래스와 비교했을 때 간결함이 가장 큰 특징이다. 그런데 이보다 더 간결하게 함수 객체를 만드는 메서드 참조라는 방법이 존재한다.

### 메서드 참조

메서드 참조는 메서드를 참조해 매개변수 리턴 타입을 알아내어 불필요한 매개 변수를 제거하는 방법이다.

```java
map.merge(key, 1, (existingValue, incr) -> existingValue + incr);
```
```java
default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction)
```

- 키가 Map 안에 없다면 키와 숫자 1을 매핑하고, 이미 존재한다면 기존 매핑 값을 증가시키는 코드이다.
	- 즉, 키가 없으면 key와 value(1)을 매핑하고, 있다면 value값에 (existingValue + incr)인 BiFunction의 apply() 메서드 결과를 저장한다.

이 코드를 메서드 참조를 사용해 변경한다면?

```java
map.merge(key, 1, Integer::sum);
```

- 매우 간단하다.
- 기존의 BiFunction의 apply는 단순하게 값을 합해 반환하는 역할을 한다. 따라서 이와 같은 기능을 하는 Integer의 sum 메서드의 참조를 전달해 같은 기능을 구현 가능하다.

이렇게 메서드 참조를 사용하면 매개변수의 수가 많을 수록 제거할 수 있는 코드 양도 늘어난다.

보통 IDE가 메서드 참조를 권고해준다.

> Integer::sum이 어떻게 같은 동작을 할 수 있는 것일까?

Integer::sum의 식이 BinaryOperator로 변환되고 이 BinaryOperator는 BiFunction을 상속받은 것이다.

BinaryOperator는 BiFunction중 입력과 출력의 타입이 동일한 경우를 특화한 것이다.

컴파일러는 메서드 참조가 함수형 인터페이스로 변환할 수 있는지 확인해 호환되면 변환해준다.

### 메서드 참조를 항상 써야하는가?

메서드 참조를 사용한 코드가 더 간결한 것은 사실이다. 그러나 매개변수가 사라지기 때문에 이 부분을 고려해야 한다.

만약 람다의 매개변수 이름 자체가 코드를 이해하는데 도움이 된다면 오히려 유지보수에 더 유용하다.

심지어 때로는 람다가 오히려 메서드 참조보다 간결한 경우도 있다.

```java
//메서드 참조
service.execute(GoshThisClassNameIsHumongous::action);

//람다
service.execute(() -> action());
```

- 메서드 참조가 더 짧거나 명확하지 않다.

람다로 할 수 없는 일은 메서드 참조로도 불가능하다. 그러니 경우에 맞게 람다와 메서드 참조를 선택해서 사용하자. 웬만해서는 메서드 참조가 더 간결하다.


### 메서드 참조의 유형

메서드 참조의 유형은 다섯 가지가 있다.

- 정적 메서드

```java
//메서드 참조
Integer::parseInt

//람다
str -> Integer.parseInt(str)
```

- 한정적 인스턴스 메서드
	- 정적 메서드 참조와 비슷하지만, 수신 객체를 특정한다.

```java
//메서드 참조
Instant.now()::isAfter

//람다
Instant now = Instant.now();
t -> now.isAfter(t);
```

- 비한정적 인스턴스 메서드
	- 수신 객체를 특정하지 않고, 적용하는 시점에 수신 객체를 알려준다.
	- 이미 존재하는 외부 객체를 호출하는 상황에 사용한다.

```java
//메서드 참조
List<String upperCaseWords = words.stream()
									.map(String::toUpperCase)
									.collect(Collectors.toList());

//람다
List<String upperCaseWords = words.stream()
									.map(word -> word.toUpperCase)
									.collect(Collectors.toList());
```

- 클래스 생성자
	- 클래스 생성자를 가리키는 메서드 참조이다.

```java
//메서드 참조
TreeMap<K, V>::new

//람다
() -> new TreeMap<K, V>()
```

- 배열 생성자
	- 배열 생성자를 가리키는 메서드 참조이다.

```java
//메서드 참조
int[]::new

//람다
len -> new int[len]
```

## 정리

메서드 참조는 람다를 더욱 간결하게 하기 위한 람다의 대안이 될 수 있다.

하지만 경우에 따라 람다가 더 유용하거나 간결한 경우가 있으므로 상황에 따라 판단해 메서드 참조와 람다를 사용하자.
