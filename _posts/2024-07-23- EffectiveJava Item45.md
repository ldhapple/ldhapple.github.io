---
title: Item45 (스트림은 주의해서 사용하라)
author: leedohyun
date: 2024-07-23 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

[스트림과 Optional 정리](https://ldhapple.github.io/posts/Stream,-Optional/)

## 스트림은 주의해서 사용하라

스트림 API는 다량의 데이터 처리 작업을 돕고자 Java 8부터 도입되었다.

이 스트림 API는 두 가지 추상 개념을 제공한다.

- 스트림
	- 데이터 원소의 유한 혹은 무한 시퀀스 개념
- 스트림 파이프라인
	- 원소들로 수행하는 연산 단계를 표현하는 개념

> 스트림

스트림의 원소들은 어디로부터든 올 수 있다. 컬렉션, 배열, 파일, matcher, 다른 스트림 등등 그리고 이 원소들은 객체 참조나 기본 타입 값(int, long, double 지원)이다.

> 스트림 파이프라인

스트림 파이프라인은 소스 스트림에서 시작해 종단 연산으로 끝난다. 그 사이에 하나 이상의 중간 연산이 있을 수 있다. 

- 중간 연산은 스트림을 변환하는 역할을 한다. 
	- 원소에 함수를 적용하거나 특정 조건을 만족하지 못하는 원소를 거르는 등의 것을 뜻한다.
- 종단 연산은 마지막 중간 연산이 내놓은 스트림에 최종적인 연산을 가한다.
	- 원소를 정렬해 컬렉션에 담거나, 특정 원소를 하나 선택하거나, 모든 원소를 출력하는 등의 것을 뜻한다.

그리고 스트림 파이프라인은 지연 평가된다.

평가는 종단 연산이 호출될 때 이루어지며, 종단 연산에 쓰이지 않는 데이터 원소는 계산에 쓰이지 않는다.

이러한 지연 평가가 무한 스트림을 다룰 수 있게 해준다. 종단 연산이 없는 스트림 파이프라인은 아무 일도 하지 않으므로 주의하자.

### 스트림 API

- 메서드 연쇄를 지원하는 플루언트 API (Fluent API) 다.
	- 파이프라인 하나를 구성하는 모든 호출을 연결해 단 하나의 표현식으로 완성할 수 있다.
	- 파이프라인 여러 개를 연결해 표현식 하나로 만드는 것도 가능하다.
- 기본적으로 스트림 파이프라인은 순차적으로 수행된다.
	- 파이프라인을 병렬로 실행하려면 파이프라인을 구성하는 스트림 중 하나에서 parallel 메서드를 호출해주기만 하면 된다.
	- 단, 효과를 볼 수 있는 상황은 많지 않다. 
- 스트림 API는 다재다능하다. 어떠한 계산도 할 수 있다.
	- 스트림을 제대로 사용하면 프로그램이 깔끔해진다.
	- 그러나 반드시 스트림을 써야한다는 뜻은 아니다. 단지 할 수 있다는 뜻이다. 
		- 스트림을 잘못 사용하면 오히려 읽기 어렵고 유지보수도 힘들다.
- 스트림을 언제 써야하는 지에 대해 규정하는 규칙은 없다.
	- 참고할만한 노하우 정도가 존재한다.

### 스트림은 주의해서 사용해야 한다.

사전 파일에서 단어를 읽어 사용자가 지정한 값보다 원소 수가 많은 아나그램(구성하는 알파벳은 같고 순서만 다른 단어들. ex - apple, elppa) 그룹을 출력하는 프로그램을 예시로 보자.

#### 스트림 없이 구현한 코드

```java
public class Anagrams {
	public static void main(String[] args) throws IOException {
		File dictionary = new File(args[0]);
		int minGroupSize = Integer.parseInt(args[1]);
	
		Map<String, Set<String>> groups = new HashMap<>();
		try (Scanner s = new Scanner(dictionary)) {
			while (s.hasNext()) {
				String word = s.next();
				groups.computeIfAbsent(alphabetize(word), (unused) -> new TreeSet<>()).add(word);
			}
		}

		for (Set<String> group : groups.values()) {
			if (group.size() >= minGroupSize) {
				System.out.println(group.size() + ": " + group);
			}
		}
	}

	//키 값 추출 - 단어를 구성하는 철자를 알파벳 순으로 정렬한 값
	private static String alphabetize(String s) {
		char[] a = s.toCharArray();
		Arrays.sort(a);
		return new String(a);
	}
}
```

- 맵 안에 키가 있는지 찾고, 있으면 해당 키에 매핑된 값을 반환한다. (Set 반환)
	- 만약 키가 없다면, 건네진 함수 객체를 키에 적용해 값을 계산 후 그 키와 값을 매핑해놓고 계산된 값을 반환한다.
	- 즉, 키가 있다면 그 키에 매핑된 Set에 word를 추가하고, 키가 기존에 없다면 새로운 Set을 만들고 거기에 word를 추가한다.
 
#### 스트림을 과하게 활용한 코드

```java
public class Anagrams {
	public static void main(String[] args) throws IOException {
		Path dictionary = Paths.get(args[0]);
		int minGroupSize = Integer.parseInt(args[1]);

		try (Stream<String> words = Files.lines(dictionary)) { //복습: try with resources
			// 사전을 여는 부분을 제외하고 프로그램 전체가 단 하나의 표현식으로 처리
			words.collect(
					groupingBy(word -> word.chars().sorted()
							.collect(StringBuilder::new,
									(sb, c) -> sb.append((char) c),
									StringBuilder::append).toString()))
					.values().stream()
					.filter(group -> group.size() >= minGroupSize) //
					.map(group -> group.size() + " : " + group)
					.forEach(System.out::println);
		}
	}
}
```

- 코드 자체는 짧지만 추출한 word를 어떻게 조작하고 그룹화하는지 잘 읽히지 않는다.
	- 심지어 우리는 어떤 프로그램을 만드는 지 아는데도 읽기 힘들다.
	- 스트림에 익숙하지 않은 사람에게는 더더욱 어려울 것이다.

이렇게 스트림을 과하게 사용하면 오히려 가독성이 떨어지고 유지보수하기도 힘들다.

#### 스트림을 적절히 활용한 코드

```java
public class Anagrams {
	public static void main(String[] args) throws IOException {
		Path dictionary = Paths.get(args[0]);
		int minGroupSize = Integer.parseInt(args[1]);

		try (Stream<String> words = Files.lines(dictionary)) { 
			words.collect(groupingBy(Anagrams::alphabetize)) // alphabetize 메서드로 단어들을 그룹화함
					.values().stream()
					.filter(anagramGroup -> anagramGroup.size() >= minGroupSize) // 문턱값보다 작은 것을 걸러냄
					.forEach(g -> System.out.println(g.size() + " : " + g)); // 필터링이 끝난 리스트 출력
		}
	}

	private static String alphabetize(String s) {
		char[] a = s.toCharArray();
		Arrays.sort(a);
		return new String(a);
	}
```
- 도우미 메서드를 잘 활용해야 한다.
	- alphabetize() 메서드를 이용하여 단어들을 아나그램 조건에 맞게 그룹화했다.
	- 스트림 파이프라인에서는 타입 정보가 명시되지 않거나 임시 변수를 자주 사용하기 때문이다.
- 그 값들에서 또 스트림 파이프라인을 더 만들어서 그 그룹의 size가 문턱값보다 작다면 거르고, 출력했다.
- 확실히 스트림을 적절히 사용한 경우는 읽기도 편하고 코드도 깔끔하다.
- 람다의 매개변수 이름을 잘 지어야 파이프라인의 가독성이 유지된다.
	- ex) anagramGroup

스트림을 모르는 사람도 어느정도 이해할 수 있는 코드가 되었다. 한 번 스트림 코드를 명확히 설명해보자.

- 첫 스트림의 파이프라인에서는 중간 연산이 없고 종단 연산으로 groupingBy를 통해 각 단어를 Map으로 수집했다.
- 그리고 이 Map을 통해 새로운 스트림을 열고 그 스트림에서 중간 연산으로 필터링을 하고 종단 연산을 통해 출력하는 것이다.

스트림을 적절히 활용하면 코드가 짧아짐은 물론 명확해진다. 가독성도 좋다.

> 주의: 자바는 기본 타입인 char용 스트림을 지원하지 않는다.

위 예시에서 alphabetize() 메서드를 스트림을 사용해 다르게 구현할 수도 있다.

그러나 그렇게 하면 명확성이 떨어지고 잘못 구현할 가능성이 있다. 심지어 느려질 수도 있다. 자바가 char용 스트림을 지원하지 않기 때문이다.

```java
"Hello World!".chars().forEach(System.out::print);
```

- 위 코드의 동작은 Hello World!를 출력하지 않는다.
	- 72101108108111328711111410810033을 출력한다.
- 스트림의 원소가 char가 아닌 int값이기 때문이다. 이름은 chars() 인데 int 스트림을 반환하니 헷갈린다.
	- 올바르게 출력하려면 형변환을 명시적으로 해주어야 한다.

**따라서 char 값들을 처리할 때는 스트림을 사용하지 않는 것이 낫다.** 

### 기존 코드의 리팩토링

스트림을 보면 기존 코드의 모든 반복문을 스트림으로 바꾸고 싶은 유혹이 생길 수 있다.

하지만 위 예시에서 보다시피 스트림을 사용한다고 무조건 더 나은 코드가 되는 것은 아니다. 스트림과 반복문을 적절히 조합해야 한다.

가져가야 하는 기조로는 ***기존 코드는 스트림을 사용하도록 리팩토링을 하되, 새 코드가 더 나아 보일 때만 반영하도록 해야 한다.*** 가 될 것이다.

> 코드 블록 vs 람다 블록

되풀이되는 계산을 할 때 스트림은 함수 객체로 표현하고, 반복문은 코드블록을 사용해 표현한다.

```java
for (int i = 1; i <= 10; i++) { 
	if (i % 2 == 0) { 
		int square = i * i; 
		System.out.println(square); 
	} 
}
```
```java
IntStream.rangeClosed(1, 10) 
		.filter(i -> i % 2 == 0) 
		.map(i -> i * i) 
		.forEach(System.out::println);
```

여기서 코드 블록만 할 수 있는 일이 있다.

- 범위 안의 지역변수를 읽고, 수정이 가능하다.
	- 람다에서는 final이거나 사실상 final인 변수만 읽을 수 있다.
	- 지역 변수를 수정하는 것은 불가능하다.
- return을 사용해 메서드에서 빠져나갈 수 있다.
- break나 continue문으로 블록 바깥의 반복문을 종료하거나 반복을 한 번 건너뛸 수 있다.
- 메서드 선언에 명시된 검사 예외를 던질 수 있다.

람다는 위에 명시된 내용을 아무 것도 할 수 없다.

따라서 이런 부분을 고려해서 둘 중 어느 것을 사용하는게 더 깔끔한 코드가 될 지 판단해서 더 나은 것을 선택해야 한다.

### 스트림에 어울리는 일과 어울리지 않는 일

- 스트림으로 처리하기에 좋은 일
	- 원소들의 시퀀스를 일관되게 변환하기 (ex- 문자열 리스트의 원소들을 대문자로 변경하기)
	- 원소들의 시퀀스를 필터링하기
	- 원소들의 시퀀스를 하나의 연산을 사용해 결합하기 (ex - 원소 전체를 더한 값 구하기)
	- 원소들의 시퀀스를 컬렉션에 모으기
	- 원소들의 시퀀스에서 특정 조건을 만족하는 원소 찾기 ( findFirst() )

그렇다면 스트림으로 처리하기 어려운 일들에는 무엇이 있을까?

- 스트림으로 처리하기 어려운 일
	- 파이프라인의 여러 단계에서의 값들에 동시에 접근해야 하는 경우
		- 스트림 파이프라인은 한 값을 다른 값에 매핑하고 나면 원래의 값을 잃는 구조이다.

```java
static Stream<BigInteger> primes() {
	return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

- 무한 스트림을 반환하는 메서드이다.
- iterate() 정적 팩토리는 매개변수 2개를 받는다.
	- 첫 번째 매개변수는 스트림의 첫 번째 원소이다.
	- 두 번째 매개변수는 스트림에서 다음 원소를 생성해주는 메서드이다.

```java
primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
		.filter(mersenne -> mersenne.isProbablePrime(50))
		.limit(20)
		.forEach(System.out::println);
```

- 첫 20개의 메르센 소수를 출력하는 프로그램이다.
	- 메르센 수는 2^p - 1 형태의 수이다.
	- 이 때 p가 소수이면 메르센 수도 소수일 수 있는데 이 조건을 만족하는 수를 메르센 소수라고 한다.

이 예시에서 만약 각 메르센 소수의 앞에 지수 p를 출력하길 원한다고 가정해보자. 위 예시에서는 단지 메르센 소수만 출력이 가능하다. 종단 연산에서 접근할 수 없다. 스트림에서 어떻게 p에 접근해 출력할 수 있을까?

- 중간 연산에서 수행한 매핑을 거꾸로 수행하여 지수를 계산하는 방법이 있다.

이렇게 파이프라인의 여러 단계에서의 값들에 모두 접근해야 하는 경우에는 확실히 스트림을 활용하기 어렵다. 반복문을 사용하면 훨씬 쉽게 구현할 수 있을 것이다.

## 정리

스트림을 사용한다고 해서 무조건 가독성이 좋아지고 코드가 깔끔해지지 않는다. 오히려 과도하게 사용하면 읽고 이해하기 어렵다.

스트림과 반복문은 각각에게 알맞은 일이 있으므로 더 나은 방법을 찾아 서로 조합하면 가장 깔끔한 코드가 될 것이다.

만약 스트림과 반복문 중 어느것이 더 나은지 판단할 수 없다면 직접 해보고 더 나은 쪽을 택하면 된다.