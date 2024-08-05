---
title: Item58 (전통적인 for문보다는 for-each문을 사용하라)
author: leedohyun
date: 2024-08-04 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 전통적인 for문보다는 for-each문을 사용하라

```java
List<Integer> list = new ArrayList<>();

for (Iterator<Integer> iterator = list.iterator(); i.hasNext();) {
	Integer number = i.next();
	//...
}

for (int i = 0; i < list.size(); i++) {
	Integer number = list.get(i);
	//...
}
```

- 이러한 전통적인 for문으로 컬렉션을 순회할 수 있다.
- while문보다 낫지만 단점들을 가진다.
	- 반복자(Iterator)나 인덱스 탐색을 위한 변수들은 코드를 지저분하게 만든다. 실제 필요한 원소를 위한 부수적인 코드일 뿐이다.
	- 반복자같이 사용되는 요소가 늘어난다면 잘못된 사용으로 예상치 못한 오류를 만날 확률이 커진다.
	- 또 위의 경우, 단순 List지만 만약 배열의 경우라면 같은 for문을 사용하더라도 코드 형태가 달라져야 된다.

이러한 전통적인 for문의 단점을 해결해줄 수 있는 것이 for-each이다.

### for-each

for-each문의 정식 이름은 향상된 for문(enhanced for statement)이다.

반복자와 인덱스 변수를 사용하지 않아 코드가 깔끔해지고 오류가 발생할 일도 없다.

하나의 관용구로 컬렉션과 배열을 모두 처리할 수도 있다.

```java
for (Integer number : list) {
	// number..
}
```

- for-each문은 사람이 손으로 최적화한 경우와 사실상 같기 때문에 어떤 컨테이너를 써도 성능 차이는 나지 않는다.

이러한 for-each문의 장점은 반복문을 **중첩해서 순회**할 때 더 커진다.

```java
enum Suit {HEART, DIAMOND, CLUB, SPADE}
enum Rank {ACE, ONE, TWO, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT, NINE,TEN, JACK, QUEEN,KING}

static Collection<Suit> suits = Arrays.asList(Suit.values());
static Collection<Rank> ranks = Arrays.asList(Rank.values());

List<Card> deck = new ArrayList<>();
      
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); ) {
        deck.add(new Card(i.next(), j.next()));
    }
}
```

- 위 코드에는 문제가 있다.
	- 바깥 컬렉션의 반복자에서 next 메서드가 너무 많이 불린다는 점이다.
	- 마지막 줄의 i.next()가 Suit 하나당 한번씩 불려야 하는데, Rank 하나당 한번씩 불리고 있다.
	- 그래서 실제로 코드를 동작시켜 보면 Suit이 바닥나 NoSuchElementException이 발생하게 된다.

![](https://blog.kakaocdn.net/dn/dzQtru/btsITkoL3pe/0ylDLKTaPUrko4Zicdkhzk/img.png)

덱에 넣을 때 각 suit과 rank를 출력해보면 바로 알 수 있다.

만약 바깥 컬렉션(suit)의 크기가 안쪽 컬렉션(rank) 크기의 배수라면?

```java
enum Suit {HEART, DIAMOND, CLUB, SPADE}  
enum Rank {ACE, ONE}
```
![](https://blog.kakaocdn.net/dn/AAhXZ/btsIUJt9TWh/paRGkSyMovi9Rhrz5Z1wk0/img.png)

- 반복문이 이전과 같이 예외를 던지지 않고 종료된다.
- 의도한 동작이 아닌데도 정상적으로 수행했다.
- 물론 이렇게 동작을 출력해서 직접 보면 금방 잘못된 부분을 찾을 수 있지만 이렇게 하기 어려운 경우가 있을 것이다.

물론 실수하지 않고 suit을 관리하는 변수를 바깥 for문에 두면 쉽게 해결된다. 하지만 for-each문을 사용하면 더욱 더 간단히 해결된다.

```java
for (Suit suit : suits) {
	for (Rank rank : ranks) {
		deck.add(new Card(suit, rank));
	}
}
```

- 가독성도 좋아지고, 오류없이 원하는 동작을 해내도록 아주 간단하게 해결했다.

### for-each를 사용할 수 없는 경우

for-each의 강점을 알았지만, 안타깝게도 for-each를 사용할 수 없는 경우가 세 가지 존재한다.

- 파괴적인 필터링(destructive filtering)
- 변형(transforming)
- 병렬 반복(parallel iteration)

세 가지 경우를 각각 보자.

#### 파괴적인 필터링

컬렉션을 순회하면서 선택된 원소를 제거해야 한다면 반복자의 remove 메서드를 호출해야 한다.

Java 8부터는 Collection의 removeIf 메서드를 이용하여 컬렉션을 명시적으로 순회하는 일을 피할 수 있다.

```java
List<Card> cards = new ArrayList<>();  
cards.add(new Card(1));  
cards.add(new Card(2));  
cards.add(new Card(3));  
cards.add(new Card(4));  
  
for (Card card : cards) {  
  if (card.num == 1) {  
	  cards.remove(card);  
  }  
}
```

- 이 코드는 ConcurrentModificationException이 발생한다.
- for-each 루프가 내부적으로 Iterator를 사용하여 컬렉션을 순회한다.
	- Iterator가 컬렉션의 요소를 제거하는 방식과, Collection의 remove 메서드의 방식이 다르기 때문에 예외가 발생한다.
	- ArrayList의 remove()는 modCount를 업데이트 하는데 Iterator는 이 변경을 감지해 예외를 발생시킨다. 이는 여러 쓰레드가 동시에 컬렉션을 수정하려 할 때 발생할 수 있는 문제를 예방하기 위함이다.

따라서 이런 경우는 for-each는 불가능하고, Iterator를 직접 사용하면 된다. Java 8부터는 removeIf 메서드를 통해 해결이 가능하다.

```java
cards.removeIf(card -> card.num == 1);
```

#### 변형

리스트나 배열을 순회하면서 그 원소의 값 일부 혹은 전체를 교체해야 한다면 리스트의 반복자나 배열의 인덱스를 사용해야 한다.

```java
List<String> names = new ArrayList<>();
names.add("홍길동");
names.add("이길동");
names.add("삼길동");
names.add("사길동");
names.add("오길동");

for (String name : names) {
	// ..?
}
```
```java
for (int i = 0; i < names.size(); i++) {
	names.set(i, "홍길동");
```

- 원소의 값을 변경하기 위해서는 반복자나 인덱스를 사용해야 한다.

#### 병렬 반복

여러 컬렉션을 병렬로 순회해야 한다면 각각의 반복자와 인덱스 변수를 사용해 엄격하면서도 명시적으로 제어해야 한다.

```java
List<String> names = new ArrayList<>();
names.add("홍길동");
//...

List<Integer> ages = new ArrayList<>();
ages.add(13);
//...

for (String name : names) {
	//...
}

for (int age : age) {
	//...
}
```

- 두 리스트를 for-each문을 사용해 병렬로 순회하는 것은 불가능하다.
	- 반복자나 인덱스를 사용해야 한다.

## 정리

전통적인 for문과 비교하여 for-each문은 장점뿐이다. 성능 저하도 없고 명료하며 예상치 못한 동작을 방지해준다.

따라서 for문을 사용해야 하는 경우를 제외하고는 for-each문을 쓰도록 하자.

for-each를 쓰기 위해 Iterable을 새로 구현하는 것은 까다롭다. 하지만 원소들의 묶음을 표현하는 타입을 작성할 경우 Iterable을 구현하는 방향으로 고민해볼 가치가 있을 정도로 for-each가 더 좋다고 말해도 과언이 아니다.