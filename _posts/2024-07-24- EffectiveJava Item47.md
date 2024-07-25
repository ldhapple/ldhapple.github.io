---
title: Item47 (반환 타입으로는 스트림보다 컬렉션이 낫다)
author: leedohyun
date: 2024-07-24 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 반환 타입으로는 스트림보다 컬렉션이 낫다

스트림 등장 이전 일련의 원소를 반환하는 메서드는 다양했다.

- Collection, Set, List, Array, Iterable 등등
	- 기본은 Collection이고, 경우에 따라 Iterable 인터페이스를 사용했다.

```java
Iterable<String> iter = new ArrayList<>(List.of("one", "two", "three"));
```

그런데 Java 8 이후 스트림이 등장하면서 이러한 선택이 복잡하게 되었다.

### 스트림은 반복을 지원하지 않는다.

**스트림은 반복을 직접적으로 지원하지 않는다.** 스트림은 반복을 내부적으로 처리하기 때문에 반복의 세부 사항을 개발자가 직접 제어할 수 없다.

이런 개념에 더해서 정리하면 [아이템 45](https://ldhapple.github.io/posts/EffectiveJava-Item45/)에서 스트림과 반복을 알맞게 조합해야 좋은 코드가 나온다고 했던 부분과 일맥 상통하다.

### Stream과 Iterable

**스트림 자체를 반복하는 것 또한 지원하지 않는다.** Stream 인터페이스는 Iterable를 확장한 것이 아닌, Iterable이 가지고 있는 추상 메서드를 전부 포함한다.

```java
default void forEach(Consumer<? super T> action) {
default Spliterator<T> spliterator()
```

- 스트림은 위 메서드들을 가지고 있다.

```java
public interface Collection<E> extends Iterable<E> {
```

```java
public interface Stream<T> extends BaseStream<T, Stream<T>> { }

public interface BaseStream<T, S extends BaseStream<T, S>>  
  extends AutoCloseable {}
```

- 하지만 Collection 처럼 Iterable을 확장하고 있지는 않다.

따라서 for-each로 Stream에 접근할 수 없는 것이다.

그렇다면 Stream과 Iterable을 같이 쓸 수 있는 방법은 없을까? (Stream을 반환하는 API를 통해 Stream을 얻고 for-each로 반복하는 방법)

#### Stream + Iterable 방법 1

```java
// ProcessHandle.allProcess()의 return -> Stream<ProcessHandle>
for(ProcessHandle processHandle : (Iterable<ProcessHandle>)ProcessHandle.allProcesses()::iterator){
    //...
}
```

- 동작은 하지만 실제로 쓰기에 너무 난잡하다. 직관성이 떨어진다.
	- 형변환을 해주지 않으면 컴파일 에러가 발생한다.

#### Stream + Iterable 방법 2

```java
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}

for(ProcessHandle processHandle : iterableOf(ProcessHandle.allProcesses())) {
    //...
}
```

- Stream을 Iterable로 중개해주는 메서드를 생성하면 가독성이 좋아진다.
- 자바의 타입 추론을 통해 어댑터 메서드 내부에서 따로 형변환을 하지 않아도 된다.

### Iterable과 Stream

위에서 Stream만을 반환하는 API가 반환한 값(스트림)을 for-each로 반복하길 원하는 경우의 방법을 보았다.

그렇다면 반대로 Iterable만을 반환하는 API가 반환한 값(Iterable)을 스트림 파이프라인에서 처리하고 싶은 경우는 어떨까?

```java
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

- 이 때도 Iterable을 Stream으로 중개해주는 메서드를 생성해 대처할 수 있다.

### 그래서 컬렉션을 반환하는게 왜 나은건데?

이번 아이템의 주제를 다시 생각해보자. 반환 타입으로는 스트림보다 컬렉션이 낫다는 것이다.

객체 시퀀스를 반환하는 메서드를 작성하는데 해당 메서드가 오직 스트림 파이프 라인에서만 쓰일 것을 안다면 스트림을 반환하게 해도 된다. 반대로 반복문에서만 쓰일 것을 안다면 Iterable을 반환하면 된다.

그러나 공개 API를 작성할 때는 한 쪽만 사용할 것이라는 보장이 없는 경우일 때 스트림 파이프라인을 사용하는 사람과 반복문에서 사용하려는 사람 모두를 배려해야 한다. 

- Collection 인터페이스는 Iterable의 하위타입이며 Stream 메서드도 제공한다.
	- 반복과 스트림을 동시에 지원한다.

따라서 ***원소 시퀀스를 반환하는 공개 API의 반환 타입에는 Collection이나 그 하위 타입을 쓰는 것이 일반적으로 최선이다.***

어댑터를 사용하는 것은 성능에도 좋지 않다.


> 덩치 큰 시퀀스의 경우

- 컬렉션을 반환하기 위해 덩치 큰 시퀀스를 메모리에 올리면 안된다.

예를 들어 주어진 집합의 멱집합 (한 집합의 모든 부분 집합을 원소로 하는 집합)을 반환하는 상황이라고 해보자.

원소의 개수가 N개이면 멱집합 원소의 개수는 2^N개가 된다.

표준 컬렉션 구현체에 저장하려고 한다면 메모리 사용량이 급격히 증가해 메모리 부족 문제를 초래할 수 있다. 각 부분집합을 생성하고 컬렉션에 추가하는 과정이 반복된다면 성능이 크게 저하된다.

이런 경우는 전용 컬렉션을 구현하는 것을 고려해야 한다. 

## 정리

반환 타입으로 Stream보다 Collection을 선택하는 것이 좋다.

Stream을 반환했을 경우 Stream이 Iterable을 확장하지 않았기 때문에 사용자의 목적에 따라 불편함을 느낄 수 있다. 그 반대의 경우도 마찬가지이다.

반면 Collection은 Stream과 Iterable을 둘 다 지원하기 때문에 공개 API를 만들 때 최선의 선택이 될 것이다.