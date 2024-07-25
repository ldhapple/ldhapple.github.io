---
title: Item42 (익명 클래스보다는 람다를 사용하라)
author: leedohyun
date: 2024-07-22 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

이번 아이템부터는 람다와 스트림에 대해 다룬다. 스트림이 for 반복문보다 느리고 박싱/언박싱 과정에서 조금 성능 하락이 있을 수 있다고 알고 있지만 코드 가독성 및 유지 보수성에 도움을 주기 때문에 나도 능숙하게 다뤄보고자 코딩 테스트 문제를 연습할 때 종종 쓰곤 했다.

이번 아이템들을 정리해보면서 다른 부분에 비해 비교적 미숙하게 사용만 해봤던 람다와 스트림에 대해 보다 깊게 파악해보고자 한다.

## 익명 클래스보다는 람다를 사용하라

예전의 Java에서는 함수 타입을 표현하기 위해 추상 메서드를 하나만 담는 인터페이스를 사용했다.

이렇게 만들어진 인터페이스의 인스턴스는 특정 함수 혹은 동작을 나타내는데에 사용됐다.

```java
interface Temp {
	int apply(int x);
}
```
```java
Temp temp = new Temp() {
	@Override
	public int apply(int x) {
		return x * x;
	}
}

int result = temp.apply(7);
```

많이 본 구조일 것이다. 1997년 JDK 1.1이 등장하면서 함수 객체를 만드는 주요 수단은 익명 클래스가 되었다.

위 구조와 비슷한 익명 클래스에 대해 자세히 알아보자.

### 익명 클래스

익명 클래스란 이름이 없는 클래스이다. 왜 이름이 없다고 하는지 알아보자.

```java
public class Point {
	private final int x;
	private final int y;
	
	public Point(int x, int y) {
		this.x = x;
		this.y = y;
	}

	public void checkPoint() {
		System.out.println("Check Point " + x + " " + y); 
}

Point point = new Point(1, 7);
```
```java
Point anonymousPoint = new Point(1, 5) {
	@Override
	public void checkPoint() {
		System.out.println("Anonymous Check Point " + (x + y) + " " + y);
	}

	//public Point() { ... } --> X
}	
```

- 일반적인 클래스는 new 키워드를 사용해 인스턴스를 생성한다.
	- new Point(1, 3) 인스턴스는 클래스 이름인 Point를 가진다. (이름을 가진다.)
- 반면 익명 클래스는 인스턴스 생성을 위해 마찬가지로 new 키워드를 쓰지만 이후 등장하는 중괄호로 인해 이름이 없다고 판단한다.
	- 해당 인스턴스는 Point 클래스를 상속받는 형태를 띈다. 따라서 Point 클래스의 메서드를 재정의할 수 있다.
	- 익명 클래스 구조에서 Point 라는 클래스는 해당 인스턴스의 이름이 아닌 단지 상속 받을 클래스의 이름인 것이다. 
	- 따라서 익명 클래스 구조를 이용해 인스턴스를 만들면 해당 인스턴스는 이름이 없는 클래스의 인스턴스인 것이다.
	- 이러한 익명 클래스는 이름을 가지지 않아 내부에 생성자 선언을 할 수 없다.

즉, 익명 클래스는 클래스 선언과 동시에 인스턴스화를 진행하는 것이다.

이러한 익명 클래스를 사용하는 예시를 보자.

```java
Collections.sort(words, new Coparator<String>() {
	@Override
	public int compare(String s1, String s2) {
		return Integer.compare(s1.length(), s2.length());
	}
}
```

Comparator 인터페이스가 정렬을 담당하는 추상 전략을 뜻하고, 문자열을 정렬하는 구체적인 전략을 익명 클래스로 구현한 것이다.

그러나 이러한 익명 클래스 방식은 보다시피 코드가 길어 가독성이 좋지 않아 자바에서의 함수형 프로그래밍이 적합하지 않았다.

그래서 더 깔끔한 코드로 함수형 프로그래밍을 가능하게 하는 람다가 Java8부터 등장하게 되었다.

### 람다

위에서 본 익명 클래스, 지금은 함수형 인터페이스라고 부르는 인터페이스의 인스턴스를 람다식을 사용해 만들 수 있다.

람다는 함수나 익명 클래스와 개념은 비슷하지만 코드가 간결하다. 메서드로 전달할 수 있는 함수 객체를 단순화 한 것이며 이름을 가지지 않고 매개변수, 바디, 반환 형식 등을 가질 수 있다.

```java
Collections.sort(words, (str1, str2) -> Integer.compare(str1.length(), str2.length());
```

- 익명 클래스로 구현했던 부분이 람다식을 활용하면 매우 간단해진다.
- 코드 길이가 줄었고, 가독성이 매우 좋아졌다.
- 매개변수의 타입이나 반환값의 타입은 컴파일러가 추론한다.
	- 상황에 따라 컴파일러가 추론하지 못하는 경우가 있는데 그 때는 작성자가 직접 명시해주어야 한다.
	- 단, 타입 추론 규칙은 매우 방대하고 복잡하므로 우선 **타입을 명시해야 코드가 더 명확한 경우를 제외하고는 람다의 모든 매개변수 타입은 생략하는 것을 원칙으로 한다.** (혹은 오류 발생 시)
	- 참고로 이러한 추론은 보통 제네릭을 통해 하게 되므로 로 타입과 같이 타입을 제공하지 않으면 추론할 수 없다. 따라서 로 타입을 쓰지말라는 아이템 26과 제네릭을 쓰라는 아이템 29는 람다와 함께할 때 더 중요해진다.

```java
Collections.sort(words, comparingInt(String::length);
words.sort(comparingInt(String::length));
```

- 람다 자리에 비교자 생성 메서드를 사용하면 코드를 더 간결하게 쓸 수도 있다.

```java
public enum Operation {
	PLUS("+", (x, y) -> x + y),
	//...

	private final String symbol;
	private final DoubleBinaryOpeartor op;

	//...
}
```

- 열거형에서도 기존에 추상 메서드를 사용해 하나하나 작성했던 것을 함수형 인터페이스와 람다를 활용해 더 간단히 표현할 수도 있다.

> 람다의 단점

메서드나 클래스와 달리 람다는 이름이 없고 문서화를 하지 못한다.

따라서 코드 자체로 동작이 명확히 설명되지 않거나 코드 줄 수가 많아진다면 람다를 쓰지 말아야 한다. (한 줄일 때 가장 좋고 길어야 세 줄 안에는 끝내야 한다.)

### 람다와 this

우선 람다의 제한적인 부분들을 알아보자.

- 함수형 인터페이스에만 쓰일 수 있다.
- 추상 클래스의 인스턴스를 만들 때 람다를 쓸 수 없다. -> 익명 클래스 사용해야 한다.
- 추상 메서드가 여러 개인 인터페이스의 인스턴스를 만들 때에도 익명 클래스를 사용해야 한다.
- 람다는 자신을 참조할 수 없다.

나머지 내용은 어떻게 보면 당연한 내용이다. 추상 메서드가 여러 개라면 람다를 어떻게 작성해야 할 지도 모르겠다.

그런데 자신을 참조할 수 없다는 부분은 무슨 말일까?

```java
public class Test {
	public static void main(String[] args) {
		Function<Integer, Integer> factorial = (n) -> {
			if (n == 0) return 1;
			return n * this.apply(n - 1);
		}
	}
}
```

- 위 코드는 컴파일 오류가 발생한다.
- 람다 표현식 내에서 자기 자신인 factorial을 직접 참조할 수 없다.
- 람다에서의 this 키워드는 바깥 인스턴스를 가리킨다.
	- Test를 참조한다.

따라서 이렇게 자기 자신을 참조해야 하는 경우는 익명 클래스를 사용하면 된다. 익명 클래스는 this 키워드가 자기 자신을 참조한다.

### 람다와 익명 클래스의 차이

표현 방식, 그리고 자기 자신의 참조에 대한 부분을 제외하고도 람다와 익명 클래스에는 차이가 있다.

- 람다
	- 컴파일러가 invokeDynamic (동적 메서드 호출) 이라는 명령어를 사용해 람다 표현식을 처리한다.
	- 그리고 런타임에 실제 구현체를 생성한다.
	- ***람다는 익명 클래스와 달리 새로운 클래스 파일을 생성하지 않는다.***
		- 따라서 JVM이 런타임에 최적화해줄 수 있다.
		- 수명이 짧은 람다식이 계속 클래스 파일을 생성한다면 효율이 매우 떨어질 것이다.
- 익명 클래스
	- 컴파일러가 invokeSpecial 이라는 명령어로 생성자를 호출해 새로운 클래스를 생성한다.
	- ***.class 파일로 생성된다.***
		- JVM이 로드하는 과정을 거쳐야 한다.

이렇게 내부적으로 동작 방식이 다르다. 왜 다를까?

> 내부 동작 방식이 다른 이유

람다를 만약 단순히 익명 클래스로 치환하여 동작하는 경우 익명 클래스처럼 람다식마다 클래스가 생기고 새로운 인스턴스를 만들게 될 것이다.

- 람다 표현식은 기본적으로 불변 객체로 취급되어 쓰레드 안전성을 확보하기 쉽다.
	- 반면 익명 클래스는 각 인스턴스가 독립적인 상태를 가질 수 있어 멀티 쓰레드 환경에서 추가적인 노력이 필요할 수 있다.
- 람다는 런타임에 동적으로 생성되기 때문에 클래스 파일을 만들지 않아 메모리 사용을 절약할 수 있다.
- 기존에도 추상 메서드가 하나인 인터페이스를 많이 사용해 람다와 호환을 유지할 수 있다.
	- 즉 기존에 존재하던 개념 (추상 메서드가 하나인 인터페이스)을 Functional Interface로 이어 사용하기 때문에 그 부분에서도 장점을 가진다.

위 내용들을 종합해보면 람다와 익명 클래스의 **내부 동작이 다른 이유는 람다가 함수형 프로그래밍을 지원하고 최적화된 성능을 제공하기 위함이다.**

## 정리

자바 8 이후 작은 함수 객체를 구현하는데 적합한 람다가 나왔다. 이를 통해 자바에서도 함수형 프로그래밍을 할 수 있게 되었다.

이러한 람다는 익명 클래스와 내부적으로 클래스를 생성하지 않고, 하고의 차이가 있다. 따라서 람다와 익명 클래스를 그저 비슷한 개념이라고 혼동해서는 안된다.

익명 클래스는 함수형 인터페이스가 아닌 타입의 인스턴스를 만들 때만 사용해야 된다.