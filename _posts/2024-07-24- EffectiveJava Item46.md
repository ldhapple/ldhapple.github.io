---
title: Item46 (스트림에서는 부작용 없는 함수를 사용하라)
author: leedohyun
date: 2024-07-24 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 스트림에서는 부작용 없는 함수를 사용하라

스트림은 단순하게 또 하나의 API가 아닌 함수형 프로그래밍에 기초한다.

스트림 패러다임의 핵심은 계산을 일련의 변환으로 재구성하는 것이다. 각 변환의 단계는 이전 단계의 결과를 받아 처리하는 순수 함수여야 한다.

- 순수 함수
	- 오직 입력만이 결과에 영향을 주는 함수
	- 다른 가변 상태를 참조하지 않고 함수 스스로도 다른 상태를 변경하지 않는다.

이 부분을 지키려면 스트림 연산에 건네는 함수 객체는 모두 부작용이 없어야 한다.

```java
Map<String, Long> sideEffectExample = new HashMap<>();

try (Stream<String> words = new Scanner(file).tokens()) {
	words.forEach(word -> {
		sideEffectExample.merge(word.toLowerCase(), 1L, Long::sum);
	});
}
```

- 이 코드는 스트림, 람다, 메서드 참조를 모두 사용했다.
- 텍스트 파일에서 단어별로 빈도 수를 세어 Map에 저장하는 코드로, 동작도 의도한대로 수행된다.
- **그러나 이 코드는 스트림 코드라고 할 수 없다.**
	- 위 스트림 파이프라인은 forEach 종단 연산에서 Map의 상태를 변경하고 있다.
	- forEach의 사용은 스트림을 사용하는 것이 아닌 단순 반복문 사용에 불과하다.
	- 스트림 API의 이점을 살리지 못한 것으로 같은 기능의 반복문 코드보다 오히려 길고 읽기 어렵다. 유지보수에도 좋지 않다.

위 코드는 종단 연산에서 외부의 상태를 수정하는 람다를 실행한다. 그러면서 문제가 생긴다.

사실 이 정리만 보면 단순히 반복문과 비교했을 때 크게 다르지 않아보인다 정도일 뿐 와닿지 않는다. 올바르게 스트림 API를 사용한 코드를 보며 이해해보자.

```java
Map<String, Long> goodStreamExample;

try (Stream<String> words = new Scanner(file).tokens()) {
	goodStreamExample = words
			.collect(groupingBy(String::toLowerCase, counting()));
}
```

- 앞선 코드와 같은 역할을 하지만 코드 길이부터 줄어들고 명확해졌다.
- 스트림을 활용한 코드를 보고 앞선 잘못 사용한 코드를 보면, 사실 잘못 사용한 코드는 스트림을 사용했지만, 오히려 구조는 반복문과 동일한 느낌이다.
- forEach 연산은 종단 연산 중 기능이 가장 적고 가장 스트림답지 않은 기능이다.
	- **forEach는 스트림 계산 결과를 보고할 때만 사용하고, 계산하는데에는 쓰지 않는게 좋다.**

### 부작용을 일으키는 함수를 사용하는 경우 예시

그렇다면 중간에 부작용을 일으키는 함수 객체를 사용하는 예시를 보자.

```java
List<String> list = List.of("hello", "fine", "abc");  
List<String> modifiableList = new ArrayList<>(list);  
  
Stream<String> stream = modifiableList.stream()  
  .peek(System.out::println) //스트림 요소 출력
  .map(String::toUpperCase); //대문자로 변환
  
modifiableList.add("new"); //스트림 생성 후 원본 리스트 변경  
```

- 스트림을 생성했지만 아직 최종 연산을 실행하지 않았다.
```java
//최종 연산 수행  
stream.filter(str -> str.contains("E"))  
  .forEach(System.out::println);
```

- 위 코드의 결과는 어떻게 될까?
	- 우선 peek()를 통해 소문자 리스트의 문자열들을 모두 출력하기를 기대할 수 있다.
		- 그리고나서 "E"를 포함하는 대문자 문자열들을 출력하기를 기대한다.
	- 그러나 peek()는 우선 forEach() 종단 연산 전까지 지연된 상태이다.

```
hello
HELLO
fine
FINE
abc
new
NEW
``` 

- 실제 결과는 이렇게 나타난다.
- 스트림이 지연 실행되기 때문에 스트림이 설정된 이후 원본 리스트가 변경되면 스트림 연산이 실행될 때 변경된 리스트가 사용된다.
	- 스트림이 설정될 때는 없었던 new라는 요소가 스트림에 추가되어 최종 연산에는 포함되는 것을 볼 수 있다.

만약 위 예시에서 부작용을 일으키는 함수 객체를 사용한다면?

```java
Stream<String> stream = modifiableList.stream()
		.map(str -> {
			if (str.equals("fine")) {
				modifiableList.add("new");
			}
			return str.toUpperCase();
		});

stream.filter(str -> str.contains("E"))
	.forEach(System.out::println);
```

- 이 코드는 ConcurrentModificationException이 발생한다.
	- 스트림이 생성된 후 원본 리스트를 변경해 발생한 예외이다.
- 우리는 중간에 NEW 를 추가해서 종단 연산을 통해 E가 포함된 문자열으로 HELLO, FINE, NEW를 출력하길 원했을 것이다.
	- 만약 중단 없이 수행되었다고 가정하면 실제로는 HELLO, FINE만을 출력한다.
	- NEW라는 요소가 스트림 연산 중간에 추가되었지만, 스트림 파이프라인이 이미 시작된 이후이므로 filter 연산에 포함되지 않는다.

우리는 결과적으로 순수 함수를 쓰지 않고, 어떤 상태를 변경하는 것을 적용했을 때 우리가 생각한 결과가 나오지 않음을 볼 수 있다.

> 스트림 중간에서 원본 객체를 수정하는 것과 스트림에서 벗어나 원본 객체를 수정하는 것의 차이

위 예시에서 스트림 중간에서 원본 객체를 수정하면 오히려 종단 연산에서 반영이 되지 않고, 스트림의 바깥에서 원본 객체를 수정하면 오히려 종단 연산에서 반영되는 것을 볼 수 있다.

무슨 차이일까?

- 스트림 바깥에서 원본 객체를 수정하는 경우
	- 스트림을 생성한 후 최종 연산을 실행하지 않아 스트림 연산은 지연된 상태이다.
	- 실제 스트림은 원본 객체에 원소가 추가된 후 실행된다.
- 스트림 중간에서 원본 객체를 수정하는 경우
	- 스트림이 생성될 때 스트림은 원본 객체의 스냅샷을 캡처한다.
	- 스트림은 생성된 이후 원본 컬렉션이 변경되지 않는다는 가정하에 동작한다. (따라서 예외 발생)
	- 예외가 발생하지 않는다고 가정하고 보자.
	- 우선 중간에 원본 객체를 수정하는데, 이 때 이 연산은 지연된 상태이다.
	- 스트림 연산이 시작될 때는 new를 추가할 때 이미 캡처된 스냅샷에는 영향을 끼치지 않는다.
		- 스냅샷은 최종 연산이 실행될 때 캡처되어 바깥에서 원본 객체를 수정하는 경우에는 반영이 되는 것이다.

이러한 동작의 차이로 다른 결과를 보인 것이다.

결과적으로 어느 경우던 모두 스트림의 일관성을 해쳐 예측을 벗어나게 된다. 따라서 스트림을 사용할 때는 원본 데이터를 수정하지 않도록 주의해야 한다. 스트림에서는 순수 함수를 사용하도록 하자.

원본 데이터를 불변으로 설계하는 것도 방법이다. 

### Collectors

결국 스트림을 올바르게 사용하기 위해서는 중간 연산에서는 순수 함수만을 사용하고, 종단 연산에서 forEach 같은 것으로 계산하지 말아야 한다, 계산은 중간 연산에서 해야한다.

종단에서는 Collector를 사용하여 컬렉션으로 수집(취합)하는 방식 등을 이용해야 한다.

많이 사용하는 Collector에 대해 알아보자.

java.util.stream.Collectors 클래스는 메서드를 39개나 가지고 있다. 스트림의 원소들을 객체 하나에 취합하는 용도이다.

- toList()
	- 스트림의 원소들을 리스트로 수집한다.

```java
List<String> result = stream.collect(Collectors.toList());
```

- toSet()
	- 스트림의 원소들을 Set으로 수집한다.
- toMap()
	- 스트림의 원소들을 키와 값으로 매핑하여 Map으로 수집한다. 

```java
Map<Integer, String> result = stream.collect(Collectors.toMap(
	String::length,
	e -> e));
```

- joining()
	- 스트림의 원소들을 문자열로 이어 붙인다.

```java
String result = stream.collect(Collectors.joining(", "));
```

- groupingBy()
	- 스트림의 원소들을 특정 기준으로 그룹화하여 맵으로 수집한다.

```java
Map<Integer, List<String>> result = stream.collect(
	Collectors.groupingBy(String::length));
//문자열의 길이를 키로 해당 길이인 문자열들을 value값으로.
```

- counting()
	- 스트림의 원소들을 세어 개수를 반환한다.

```java
long count = stream.collect(Collectors.counting());
```

- reducing()
	- 스트림의 원소들을 하나의 값으로 합친다.

```java
Optional<String> concat = stream.collect(
	Collectors.reducing((s1, s2) -> s1 + s2));
``` 

Collectors 클래스는 위와 같이 스트림의 원소들을 다양한 방법으로 수집하고 변환할 수 있는 메서드들을 제공한다.

 따라서 스트림의 중간 연산에서는 순수 함수를 사용하고, 최종 연산에서는 forEach 대신 collect를 사용하여 결과를 모으는 방식으로 코드를 작성해보자. 

코드의 가독성과 유지보수성을 높이고, 함수형 프로그래밍의 장점을 잘 활용할 수 있는 방법이다.

## 정리

스트림 파이프라인 프로그래밍의 핵심은 부작용 없는 함수 객체에 있다. 스트림 관련 객체에 건네지는 모든 함수 객체는 부작용이 없는 순수 함수여야 한다.

그렇지 않으면 스트림의 일관성이 깨져 예측할 수 없는 결과가 나올 수 있다.

그리고 종단 연산인 forEach는 계산에 이용하지 말아야 한다. 단순 반복문보다 나은 부분이 없다.

forEach대신 Collectors를 활용해 스트림의 원소들을 수집하는 방향으로 구현하는 것을 생각해보자.