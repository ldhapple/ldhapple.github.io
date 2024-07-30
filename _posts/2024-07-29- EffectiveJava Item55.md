---
title: Item52 (다중 정의는 신중히 사용하라)
author: leedohyun
date: 2024-07-28 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 다중 정의는 신중히 사용하라

다중 정의는 우리가 아는 오버로딩을 뜻하는 말이다.

```java
public class CollectionClassifier {
	public static String classify(Set<?> s) {
		return "Set";
	}
	
	public static String classify(List<?> lst) {
		return "List";
	}
	
	public static String classify(Collection<?> c) {
		return "그 외";
	}
}
```
```java
Collection<?>[] collections = {
	new HashSet<String>(),
	new ArrayList<BigInteger>(),
	new HashMap<String, String>().values()
};

for (Collection<?> c : collections) {
	System.out.println(classify(c));
}
```

- 위 코드의 결과로 어떤 것들이 출력될까?
	- Set, List, 그 외 를 차례대로 출력할 것이라고 기대한다.
- 하지만 실제로 실행해보면 "그 외"만 3번 연달아 출력하게 된다.
	- **오버로딩된 세 개의 classify() 중 어느 메서드를 호출할 지 컴파일 타임에 정해지기 때문이다.**
	- 컴파일 타임에는 for문에서의 c 타입이 Collection으로 고정되어 있다.
	- 런타임에는 타입이 매번 달라지지만 호출할 메서드를 선택할 때에는 영향을 주지 못하는 것이다.

이렇게 직관과 어긋나는 이유는 ***재정의한 메서드는 동적으로 선택되고, 다중정의한 메서드는 정적으로 선택되기 때문이다.***

### 오버로딩과 오버라이딩의 차이

- 메서드 재정의
	- 해당 객체의 런타임 타입이 어떤 메서드를 호출할 지의 기준이된다.
	- 동적으로 선택된다.
- 오버로딩
	- 객체의 런타임 타입은 전혀 중요하지 않다.
	- 선택은 컴파일 타임에 오직 매개변수의 컴파일 타임 타입에 의해 이루어진다. 
	- 정적으로 선택된다.

메서드 재정의의 경우를 보자.

```java
class Wine {
	String name() { return "포도주"; }
}

class SparklingWine extends Wine {
	@Override
	String name() {	return "발포성 포도주"; }
}

class Champagne extends SparklingWine {
	@Override
	String name() { return "샴페인"; }
}
```
```java
List<Wine> wineList = List.of(
	new Wine(), new SparklingWine(), new Champagne());

for (Wine wine : wineList) {
	System.out.println(wine);
}
```

- Wine 클래스에 정의된 name()은 하위 클래스에서 재정의되었다.
- 오버로딩을 사용했을 때와 다르게 포도주, 발포성 포도주, 샴페인을 차례로 출력한다.
	- for문에서의 컴파일 타입은 오버로딩때와 마찬가지로 모두 Wine이다.
	- 하지만 **오버라이딩은 가장 하위에서 정의한 재정의 메서드가 실행되는 구조**이기 때문에 우리가 원하는 결과를 얻을 수 있는 것이다.

결과적으로 다중정의를 사용한 경우는 헷갈린다. 공개 API의 경우라면 API 사용자가 런타임에 이상하게 동작하는 상황에서 문제를 진단하느라 시간을 허비할 것이다.

다중 정의가 혼동을 일으키는 상황을 피해야 한다.

> 해결책

다중 정의를 하지 않는 것이다. 특히 가변 인수(varargs)를 사용하는 메서드라면 다중 정의를 아예 하지 말아야 한다.

```java
public void print(int a) {
	System.out.println("int: " + a);
}

public void print(int... numbers) {
	System.out.println("varargs: " + Arrays.toString(numbers));
}
```

- 어떤 메서드를 호출할 지 컴파일러가 모호할 수 있다. 혹은 의도치않게 충돌할 수 있는 상황이 벌어진다.

다중 정의는 이렇게 혼란을 줄 수 있다.

이러한 ***오버로딩의 대체재로 메서드의 이름을 다르게 지어주는 방법이 있다. 이를 통해 해결할 수 있다.***

- writeInt(int), wirteLong(long)
- readInt(), readLong()

### 생성자의 경우..?

생성자는 이름을 다르게 지을 수 없다. 그리고 생성자는 재정의할 수 없다.

따라서 두 번째 생성자부터는 무조건 다중 정의가 된다.

하지만 이 경우에도 정적 팩토리라는 대안이 있다.

그럼에도 여러 생성자가 같은 수의 매개변수를 받아야 하는 경우를 완전히 피해갈 수는 없으니 그런 상황이 생겼을때의 해결책을 보자.

- 우선 매개변수 수가 같은 다중정의 메서드가 많더라도, 그 중 어느 것이 주어진 매개변수 집합을 처리할 지 명확히 구분된다면 헷갈릴 일이 없다.
	- 두 null이 아닌 타입의 값을 서로 어느쪽으로든 형변환 할 수 없을 때 이를 만족한다.
	- 이 조건을 충족하면 컴파일타임 타입에는 영향을 받지 않게 되고, 혼란을 주는 원인이 사라지는 것이다.

```java
public void print(int number) { ... }
public void print(String text) { ... }
```

### 오토 박싱의 도입에서 야기된 오버로딩 문제

그러나 Java 5부터 오토박싱이 도입되었다. 원래는 모든 기본 타입이 모든 참조 타입과 근본적으로 달랐지만, 오토박싱이 도입된 이후 그렇지 않다.

```java
Set<Integer> set = new TreeSet<>();
List<Integer> list = new ArrayList<>();

for (int i = -3; i < 3; i++) {
	set.add(i);
	list.add(i);
}

for (int i = 0; i < 3; i++) {
	set.remove(i);
	list.remove(i);
}

System.out.println(set + " " + list);
```

- -3부터 2까지의 수를 정렬된 Set과 List에 추가하고 remove() 메서드를 3번 호출했다.
- 그렇다면 결과는 어떻게 예상될까?
	- [-3, -2, -1], [-3, -2, -1] 을 출력할 것이라고 생각한다.
- 그러나 실제로는 [-3, -2, -1], [-2, 0, 2]가 출력된다.

```
[-3, -2, -1, 0, 1, 2] -> remove(0) -> [-2, -1, 0, 1, 2] -> remove(1)
-> [-2, 0, 1, 2] -> remove(2) -> [-2, 0, 2] 
```

- 우리가 원하는 동작은 원소 내용을 기준으로 삭제하는 것이었다.
	- 즉 remove(Object) 인데, for문 안에 타입이 int이기 때문에 index를 기준으로 없애는 remove(int)가 컴파일 시점에 결정되어 실행된 것이다.
	- 참고로 Set 인터페이스에는 int index를 받는 remove 메서드가 없기 때문에 우리가 원하는 동작을 한 것이다.

제네릭이 도입되기 전인 Java 4까지의 List에서는 Object와 int가 근본적으로 달랐기 때문에 문제가 없었다.

하지만 제네릭과 오토박싱이 등장하면서 두 메서드의 매개변수 타입이 더는 근본적으로 다르지 않게 된 것이다.

제네릭과 오토박싱이 등장하면서 List 인터페이스가 취약해졌다. (같은 피해를 입은 API는 거의 없지만, 다중 정의 시 주의를 기울여야 한다는 근거로 충분하다.)

### 람다와 메서드 참조의 도입에서 야기된 오버로딩 문제

박싱타입 뿐만 아니라 람다와 메서드 참조의 도입 또한 다중 정의 시의 혼란을 키웠다.

```java
new Thread(System.out::println).start();

ExecutorService exec = Executors.newCachedThreadPool();
exec.submit(System.out::println);
```

![](https://blog.kakaocdn.net/dn/czLRVh/btsIPXM7gLT/Apext9k4CvPzVJPi4BhxLk/img.png)

- 두 메서드 호출 구조가 비슷하다. 하지만 보다시피 ExecutorService의 메서드에서만 컴파일 에러가 난다.
	- 양쪽 모두 Runnable을 받는 형제 메서드를 다중 정의하고 있다.

```java
public Thread(Runnable target) {  
    this(null, target, "Thread-" + nextThreadNum(), 0);  
}

Thread(Runnable target, @SuppressWarnings("removal") AccessControlContext acc) {  
    this(null, target, "Thread-" + nextThreadNum(), 0, acc, false);  
}

public Thread(ThreadGroup group, Runnable target) {  
    this(group, target, "Thread-" + nextThreadNum(), 0);  
}

public Thread(String name) {  
    this(null, null, name, 0);  
}

//...
```
```java
Future<?> submit(Runnable task);

<T> Future<T> submit(Runnable task, T result);

<T> Future<T> submit(Callable<T> task);
```

- Runnable과 Callable은 Java 멀티쓰레딩에서 작업을 정의하기 위해 사용되는 인터페이스이다.
	- Runnable
		- 단일 메서드 run을 가진 간단한 인터페이스이다. 반환 값이 없으며 예외를 던지지 않는다.
		- 주로 Thread 클래스나 ExecutorService와 함께 사용된다.
	- Callable
		- 제네릭 타입을 사용하여 결과를 반환할 수 있는 인터페이스이다. call 메서드를 가지며 반환 값을 가지고 예외를 던질 수 있다.
-  그런데 Thread 생성자와 submit()의 다중 정의에서의 큰 차이는 submit()에서는 Callable을 받는 다중 정의 메서드가 있다는 것이다.

System.out::println은 void 타입을 반환하기 때문에 Runnable을 만족한다고 여겨진다.

그런데 왜 반환 값이 있는 Callable을 받는 submit()과 혼동되는 것일까? 

- println()도 다중 정의가 되어있는 것이 문제가 된다.
	- 세부적은 내용은 어렵지만 간단하게 정리할 수 있다.
	- 핵심은 **다중 정의된 메서드들이 함수형 인터페이스를 인수로 받을 때 비록 서로 다른 함수형 인터페이스라도 인수 위치가 같다면 혼란이 생긴다는 것**이다.
- Runnable 인터페이스를 받는 메서드와 Callable 인터페이스를 받는 메서드가 다중 정의 되어 있는데 같은 위치의 인수로 받고 있다.
- 그런데 println()도 다중정의 되어 있어 반환값은 비록 void이지만 Runnable 인터페이스에 전달될 지, Callable 인터페이스에 전달될 지 컴파일러가 결정을 하지 못해 문제가 생긴다는 뜻이다. 
	
***결론은 메서드를 다중 정의할 때 서로 다른 함수형 인터페이스라도 같은 위치의 인수로 받아서는 안된다는 것이다.***

## 정리

프로그래밍 언어가 다중 정의를 허용한다고 해서 다중 정의를 반드시 활용하라는 뜻은 아니다.

일반적으로 매개변수 수가 같을 때는 다중 정의를 피하는 것이 좋다.

같은 동작이라도 매개변수에 따라 메서드의 이름을 다르게 지어주거나, 생성자라면 정적 팩토리를 쓰는 방법이 있다.

상황에 따라서는 해당 방법을 쓰지 못하는 경우가 있을텐데, 그 경우에는 헷갈릴 만한 매개변수는 형변환하여 정확하게 다중 정의 메서드가 선택되도록 구현해야 한다.