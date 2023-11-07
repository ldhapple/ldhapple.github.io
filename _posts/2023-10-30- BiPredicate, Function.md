---
title: BiPredicate, Function<>
author: leedohyun
date: 2023-10-30 20:13:00 -0500
categories: [JAVA, 정리]
tags: [JAVA, 정리]
---
Java가 제공하는 함수형 인터페이스에는 java.util.function에서 제공하는 것들이 존재한다.

크게 5가지로 

- Consumer
- Supplier
- Function
- Operator
- Predicate

이 중 이 포스트에서는 Predicate 계열의 BiPredicate, Function계열의 Function에 대해 소개하려고 한다.

Bi가 붙으면 2개의 매개변수를 받아 반환값을 전달한다고 보면 된다.

ex) BiFunction - 두 개의 매개변수를 받는 함수에 대해 동작.

## BiPredicate

```java
@FunctionalInterface
public interface BiPredicate<T, U> {

  boolean test(T t, U u);

  default BiPredicate<T, U> and(BiPredicate<? super T, ? super U> other) {
    Objects.requireNonNull(other);
    return (T t, U u) -> test(t, u) && other.test(t, u);
  }

  default BiPredicate<T, U> negate() {
    return (T t, U u) -> !test(t, u);
  }

  default BiPredicate<T, U> or(BiPredicate<? super T, ? super U> other) {
    Objects.requireNonNull(other);
    return (T t, U u) -> test(t, u) || other.test(t, u);
  }
}
```

BiPredicate Interface는 Java에서 함수형 프로그래밍을 구현하기 위해 Java 버전 1.8부터 도입된 함수형 인터페이스이다.

제네릭 타입인 두 개의 매개변수를 전달받아 특정 작업을 수행 후 Boolean 타입의 값을 반환하는 작업을 수행할 때 사용된다.

- T: 첫 번째 매개변수 타입
- U: 두 번째 매개변수 타입

BiPredicate 인터페이스 내부에는 한 개의 추상 메서드와 세 개의 디폴트 메서드가 존재한다. 

람다 표현식을 사용하면 추상 메서드인 test() 메서드를 구현하기 위한 클래스를 정의할 필요 없으며, BiPredicate 타입의 객체에 할당된 람다 표현식은 test() 메서드를 구현하기 위해 사용된다.

```java
BiPredicate<Integer, Boolean> check = (count, isMatch) -> count == 5 && !isMatch;
//(count, isMatch) -> count == 5 && !isMatch 가 test() 메서드를 구현하기 위해 사용된다.
```

### 추상 메서드 - test()

test() 메서드는 제네릭 타입인 두 개의 매개변수를 전달받아 특정 작업을 수행 후 Boolean 값을 반환한다.

```java
boolean test(T t, U u);
```

test() 메서드를 구현하기 위해 두 개의 매개변수를 가지며, Boolean 타입의 값을 반환하는 람다 표현식을 BiPredicate 타입의 객체에 할당한다. 그러면 람다 표현식은 test() 메서드를 구현하는데 사용된다.

```java
BiPredicate<String, String> predicate= (str1, str2) -> str1.equals(str2);

System.out.println(predicate.test("abc", "abd");

//false
```

### 디폴트 메서드

#### and()

디폴트 메서드인 and() 메서드는 BiPredicate 타입의 객체를 매개변수로 가진다.

```java
default BiPredicate<T, U> and(BiPredicate<? super T, ? super U> other) {
    Objects.requireNonNull(other);
    return (T t, U u) -> test(t, u) && other.test(t, u);
}
```

test() 메서드 반환 결과와 and() 메서드의 매개변수로 전달된 BiPredicate 객체의 test() 메서드 반환 결과에 대해 and 연산을 수행한다.

and() 메서드의 매개변수로 BiPredicate 객체 대신 두 개의 매개변수를 가지며, Boolean 타입의 값을 반환하는 람다 표현식을 전달할 수도 있다.

```java
BiPredicate<Integer, Integer> predicate1 = (num1, num2) -> num1 > num2;
BiPredicate<Integer, Integer> predicate2 = (num1, num2) -> num1 == num2;

System.out.println(predicate1.and(predicate2).test(1, 1));
System.out.println(predicate1.and((num1, num2) -> num1 < num2).test(1, 1);

//true
//false
```

#### negate()

디폴트 메서드 negate() 메서드는 매개변수를 가지지 않으며, test() 메서드의 반환 결과를 부정한다.

```java
default BiPredicate<T, U> negate() {
    return (T t, U u) -> !test(t, u);
}
```

```java
BiPredicate<Integer, Integer> predicate = (num1, num2) -> (num1 + num2) > 100;

System.out.println(predicate.negate().test(50, 30));

//true
//100보다 작아 false를 반환하는게 맞지만 negate()가 있기 때문에 부정 결과인 true가 반환된다.
```

#### or()

디폴트 메서드 or() 메서드는 and() 메서드와 매개변수가 동일하다. 따라서 BiPredicate 타입의 객체를 전달하거나 람다식을 전달할 수 있다.

```java
default BiPredicate<T, U> or(BiPredicate<? super T, ? super U> other) {
  Objects.requireNonNull(other);
  return (T t, U u) -> test(t, u) || other.test(t, u);
}
```

test() 메서드 반환 결과와 or() 메서드의 매개변수로 전달된 BiPredicate 객체의 test() 메서드 반환 결과에 대해 or 연산을 수행한다.

```java
BiPredicate<Integer, Integer> predicate1 = (num1, num2) -> num1 > num2;
BiPredicate<Integer, Integer> predicate2 = (num1, num2) -> num1 == num2;

System.out.println(predicate1.or(predicate2).test(10, 10);

//true
```

## Function<>

```java
@FunctionalInterface  
public interface Function<T, R> {  
  
	R apply(T t);  
	   
	default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {  
		Objects.requireNonNull(before);  
		return (V v) -> apply(before.apply(v));  
	}  
	  
	default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {  
		Objects.requireNonNull(after);  
		return (T t) -> after.apply(apply(t));  
	}  
	  
	static <T> Function<T, T> identity() {  
		return t -> t;  
	}  
}
```

함수형 인터페이스이다.

- T: 함수에 대한 입력 유형
- R: 함수 결과의 유형

T타입 인자를 받아 R타입을 리턴한다.

```java
Function<String, Object> function = Person::create;

위의 예시로 들면 Person.create(name) 메서드는 매개변수로 String name이 필요한 메서드이다.
따라서 String - T의 매개변수를 받고 Person을 생성해 반환(Object - R를 반환)하는 것이다.
```

Function 인터페이스 내부에는 한 개의 추상 메서드와 두 개의 디폴트 메서드, 그리고 한 개의 정적 메서드가 존재한다.

BiPredicate와 마찬가지로 람다 표현식의 사용이 가능하다.

### 추상 메서드 - apply()

```java
R apply(T t);
```

제네릭 타입인 한 개의 매개변수를 전달받아 특정 작업을 수행 후 값을 반환한다.

```java
Function<Integer, String> functionAdd = (num) -> Integer.toString(num + 100);

System.out.println(functionAdd.apply(10));

//110
```

### 디폴트 메서드

#### compose()

```java
default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
  Objects.requireNonNull(before);
  return (V v) -> apply(before.apply(v));
}
```

compose() 메서드는 매개변수로 전달받은 Function 객체의 apply() 메서드를 호출 후 반환 결과를 apply() 메서드에 전달한다.

```java
Function<Integer, Integer> functionAdd = (num) -> num + 100;
Function<Integer, Integer> functionMultiple = (num) -> num * 10;

System.out.println(functionAdd.compose(functionMultiple).apply(10));

//(10 * 10) + 100
//200
```

#### andThen()

```java
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
  Objects.requireNonNull(after);
  return (T t) -> after.apply(apply(t));
}
```

andThen() 메서드는 apply() 메서드 호출 후 반환 결과를 매개변수로 전달받은 Function 객체의 apply() 메서드에 전달한다.

```java
Function<Integer, Integer> functionAdd = (num) -> num + 100;
Function<Integer, Integer> functionMultiple = (num) -> num * 10;

System.out.println(functionAdd.andThen(functionMultiple).apply(10));
// 1100
```

- compose(): compose()에 전달한 Fucntion의 apply 결과를 전달 후 먼저의 함수 apply.
- andThen(): andThen()에 전달한 Function에게 먼저의 apply 값을 전달 후 andThen 메서드 apply.