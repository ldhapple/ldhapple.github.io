---
title: Item54 (null이 아닌, 빈 컬렉션이나 배열을 반환하라)
author: leedohyun
date: 2024-07-29 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## null이 아닌, 빈 컬렉션이나 배열을 반환하라

```java
private final List<Integer> numbers;

public List<Integer> getNumbers() {
	if (numbers.isEmpty()) {
		return null;
	}
	return new ArrayList<>(numbers);
}
```

- 위와 같이 컬렉션에 요소가 없는 경우 null을 반환하는 경우가 있다.
- 하지만 null을 반환하게 된다면 해당 메서드를 사용하는 클라이언트는 null을 처리하는 코드를 추가로 작성해야 한다.
	- 즉, 반드시 null값에 대한 처리를 해주어야 한다.

```java
List<Integer> numbers = generator.getNumbers();

if (numbers != null) {
	//...
}
```

- 위와 같이 null 처리를 하는 번거로움이 있다. (방어 코드)
- 더해 만약 메서드를 사용하는 측에서 null에 대해 인지하지 못하거나 실수로 null에 대한 처리를 하지 않았다면 null에 대한 방어 코드가 없어 오류가 발생할 수 있다.
	- 실제로 객체가 0개일 가능성이 거의 없는 상황에서 수년 뒤에 오류가 발생해 곤란한 경우도 있다고 한다.

### 그런데 빈 컬렉션보다 null이 낫지 않나?

빈 컬렉션을 반환하는 것보다 null을 반환하는 쪽이 낫다는 주장이 있다. 빈 컨테이너를 할당하는 데도 비용이 들기 때문이다.

하지만 이 주장은 두 가지 측면에서 틀린 주장이다.

- 성능 분석 결과 빈 컬렉션을 할당하는 것이 성능 저하의 주범이 아니라면 그 비용은 고려할 수준이 아니라는 뜻이다.
- 빈 컬렉션과 배열은 굳이 새로 할당하지 않고도 반환할 수 있다.

그렇다면 빈 컬렉션이나 배열을 새로 할당하지 않고 반환하는 방법을 보자.

```java
private final List<Integer> numbers;

public List<Integer> getNumbers() {
	return new ArrayList<>(numbers);
}
```

- 일반적으로는 위와 같이 기존 컬렉션을 복사하는 빈 컬렉션을 반환하도록 구현한다.
- 그런데 일부 주장에서는 이 비용이 성능에 영향을 미쳐 null이 낫다고 주장하는 것이다.

```java
public List<Integer> getNumbers() {
	if (numbers.isEmpty()) {
		return Collections.emptyList();
	}
	return new ArrayList<>(numbers);
}

// Collections.emptySet, Collections.emptyMap 등등..
```

- 가능성은 작지만 사용 패턴에 따라 빈 컬렉션 할당이 성능을 눈에 띄게 떨어뜨릴 수 있다.
- 그럴 때 **매번 똑같은 빈 '불변' 컬렉션을 반환**하면 된다.
	- [불변 객체는 자유롭게 공유해도 안전하다.](https://ldhapple.github.io/posts/EffectiveJava-Item17/) 쓰레드 안전하며, 방어적 복사도 필요 없다. 값이 변하지 않는다.
- **주의할 점은 이 또한 최적화에 해당한다는 것이다.**
	- 꼭 필요할 때만 사용해야 한다.
	- 최적화가 필요하다고 판단되면 수정 전과 후의 성능을 측정해 실제로 성능 개선이 되는지 확인해야 한다.

### 배열의 경우에도 null보다는 길이가 0인 배열을 반환하자

배열의 경우도 마찬가지이다. null을 반환하기보다 길이가 0인 배열을 반환해야 한다.

```java
public int[] getNumbers() {
	return numbers.toArray(new int[0]);
}
```

- 보통은 단순히 정확한 길이의 배열을 반환하면 된다. 그 길이가 0일 수도 있을 뿐이다.

마찬가지로 만약 위와 같은 방식이 성능을 떨어뜨린다면, 길이가 0인 배열을 미리 선언해두고 매번 선언한 배열을 반환하면 된다.

```java
private static final int[] EMPTY_NUM_ARR = new int[0];

public int[] getNumbers() {
	return numbers.toArray(EMPTY_NUM_ARR);
}
```

- 길이가 0인 배열은 모두 불변이다.
- 길이가 0인 배열을 미리 선언해두고 선언해둔 배열을 반환하도록 했다.
- 이렇게 구현하면 무조건 길이가 0인 배열을 반환할 것 같지만 numbers가 비어있을 때만 0인 배열을 반환한다.
- List.toArray(T[] a) 메서드는 주어진 배열 a가 충분히 크면 안에 원소를 담아 반환하지만, 그렇지 않다면 새로운 T[] 타입 배열을 만들어 그 안에 원소를 담아 반환한다.
	- 즉, numbers가 비어있을 경우 배열 a가 충분히 크기 때문에 그대로 반환하는 것이고, numbers가 비어있지 않은 경우는 배열 a가 작기 때문에 새로운 배열을 만들어 복사하여 반환하게 되는 것이다.

하지만 이 또한 성능 개선의 목적이고, 단순히 성능을 개선할 목적이라면 toArray()에 넘기는 배열을 미리 할당하는 것은 추천되지 않는다.

오히려 성능이 떨어진다는 연구 결과도 존재하기 때문이다.

## 정리

빈 컨테이너를 반환해야 하는 경우 null보다는 빈 컬렉션이나 길이가 0인 배열을 반환하도록 하자.

null을 반환하게 되면 그 메서드를 사용하는 클라이언트가 null에 대해 인지하고, null에 대한 경우를 처리하는 코드를 반드시 작성해주어야 후에 탈이 없다.

이러한 번거로움도 있는데, 성능에도 딱히 이점이 없기 때문에 null보다는 빈 컬렉션을 반환하자.