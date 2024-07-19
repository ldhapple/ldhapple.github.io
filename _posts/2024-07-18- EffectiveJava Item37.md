---
title: Item37 (ordinal 인덱싱 대신 EnumMap을 사용하라)
author: leedohyun
date: 2024-07-18 21:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## ordinal 인덱싱 대신 EnumMap을 사용하라

배열이나 리스트에서 원소를 꺼낼 때 ordinal 메서드로 인덱스를 얻는 코드가 존재한다.

```java
public enum Color {
	RED, GREEN, BLUE, YELLOW, ORANGE
}

public enum Palette {
	PRIMARY, SECONDARY, NEUTRAL
}
```

```java
Set<Palette>[] paletteColors = new Set[Palette.values().length];

paletteColors[Palette.PRIMARY.ordinal()].add(Color.RED);
paletteColors[Palette.PRIMARY.ordinal()].add(Color.BLUE);
paletteColors[Palette.PRIMARY.ordinal()].add(Color.YELLOW);

paletteColors[Palette.SECONDARY.ordinal()].add(Color.YELLOW);
//...
```

- 색상을 각 팔레트별로 분류하는 예시이다.
	- 각 팔레트에 원하는 색상을 Set 컬렉션에 넣어주고 있다.

이렇게 구현했을 때 문제가 무엇일까?

- 배열은 제네릭과 호환되지 않는다. 따라서 비검사 형변환을 수행해야 하고 깔끔하게 컴파일 되지 않는다. 
- 정확한 정숫값을 사용한다는 것을 작성자가 직접 보증해야 한다.
	- 정수는 열거 타입과 달리 타입 안전하지 않다.
	- 잘못된 값을 사용하면 잘못된 동작을 수행하거나 운이 좋다면 ArrayIndexOutOfBoundsException을 받게 될 것이다.
- 출력 시 모두 레이블을 달면서 표현해주어야 한다.

따라서 이런 방법은 사용하면 안된다. 다른 대안도 있다. 바로 EnumMap이다.

### EnumMap 사용

위의 ordinal()을 사용한 배열에서 배열은 사실 열거 타입 상수를 값으로 매핑하는 일을 할 뿐이다. 따라서 Map도 사용 가능하다.

열거 타입을 키로 사용하도록 설계한 EnumMap을 써서 코드를 수정해보자.

```java
Map<Palette, Set<Color>> paletteColors = new EnumMap<>(Palette.class);

paletteColors.put(Palette.PRIMARY, EnumSet.of(Color.RED, Color.BLUE, Color.YELLOW)); 
paletteColors.put(Palette.SECONDARY, EnumSet.of(Color.GREEN, Color.ORANGE)); 
paletteColors.put(Palette.NEUTRAL, EnumSet.noneOf(Color.class));

for (Palette palette : Palette.values()) { 
	System.out.printf("%s colors: %s%n", palette, paletteColors.get(palette)); 
}
```

- 더 명료하면서도 안전하다. 성능도 비슷하다.
	- 안전하지 않은 형변환은 쓰이지 않는다.
- 맵의 키인 열거 타입이 그 자체로 출력용 문자열을 제공해 출력 결과에 직접 레이블을 달 일도 없다.
- 인덱스 접근을 직접 하지 않으므로 배열 인덱스를 계산하는 과정에서 오류나는 일 같은건 일어나지 않는다.

내부 구현 방식은 배열이기 때문에 배열의 성능을 가져오면서도, Map의 타입 안전성까지 챙겼다.

### EnumMap vs HashMap

그렇다면 왜 EnumMap이 존재하고, 왜 EnumMap을 사용해야 할까?

둘의 차이는 겉보기에 EnumMap의 키 값에 열거 타입만 올 수 있다는 것이다.

> 성능이 더 좋다.

HashMap은 key의 Hash값을 계산해 테이블의 인덱스로 사용하여 노드의 Key가 일치할 때까지 연결리스트 또는 트리를 탐색한다.

반면 EnumMap은 열거 타입의 ordinal()을 인덱스로 사용해 배열에 저장하는 방식으로 동작하기 때문에 성능 효율이 더 좋다.

> 순서를 보장한다.

HashMap은 Hash값을 이용해 저장하기 때문에 put을 이용하여 값을 삽입할 때 순서가 보장되지 않는다.

반면 EnumMap은 Enum 클래스가 선언 순서를 이용한 배열을 내부적으로 구현하고 있어 Enum의 ordinal() 값과 같은 순서를 보장한다.

## 정리

열거형 타입과 Map을 같이 쓸 상황이 온다면 EnumMap을 사용하는 것을 추천한다.

성능에서도 이점이 있고, 순서를 보장해준다.

다만 순서 면에서 단순한 순서가 아닌 특별한 조건의 순서가 보장된 것을 원한다면 TreeMap이나 LinkedHashMap을 사용할 수도 있다.

[EnumMap 정리](https://ldhapple.github.io/posts/Enum-Map/)