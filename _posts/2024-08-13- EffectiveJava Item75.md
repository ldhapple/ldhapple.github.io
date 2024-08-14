---
title: Item75 (예외의 상세 메시지에 실패 관련 정보를 담으라)
author: leedohyun
date: 2024-08-13 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 예외의 상세 메시지에 실패 관련 정보를 담으라

예외를 잡지 못해 프로그램이 실패하면 그 예외의 Stack Trace 정보를 자동으로 출력한다.

이 스택 추적 정보는 예외 객체의 toString 메서드를 호출해 얻는 문자열로, 보통은 예외의 클래스 이름 뒤에 상세 메시지가 붙는 형태이다.

이러한 정보는 프로그래머가 얻을 수 있는 유일한 정보인 경우가 많다. 만약 실패를 재현하기 어려운 경우 더더욱 그렇다.

**따라서 [아이템 72](https://ldhapple.github.io/posts/EffectiveJava-Item72/)에서도 살짝 언급했듯 예외의 toString 메서드에 실패 원인에 관한 정보를 가능한 많이 담아 반환하는 일은 아주 중요하다.**

![](https://blog.kakaocdn.net/dn/oVJpG/btsI49TKaDJ/kszH3Ha6GKAznjUnf1Txxk/img.png)

이렇게 예외 메시지가 없다면, 실패에 대한 원인을 분석하는 작업이 필요하다. 사후 분석을 위해 실패 순간의 상황을 정확히 포착하여 예외의 상세 메시지에 담아야 한다.

### 예외에 관여된 모든 매개변수와 필드의 값을 담자

실패 순간을 포착하기 위해서는 발생한 예외에 관여한 모든 매개변수와 필드의 값을 실패 메시지에 담아야 한다.

예를 들면 IndexOutOfBoundsException의 상세 메시지에는 범위의 최솟값과 최댓값, 그리고 그 범위를 벗어났다는 인덱스 값을 담아주어야 한다.

```java
public class InvalidRangeException extends RuntimeException {  
	private final int lowerBound;  
	private final int upperBound;  
	private final int index;  
	  
	public InvalidRangeException(int lowerBound, int upperBound, int index) {  
		super(MessageFormat.format("최소 범위: {0}, 최대 범위: {1}, 인덱스: {2}", lowerBound, upperBound, index));  
		this.lowerBound = lowerBound;  
		this.upperBound = upperBound;  
		this.index = index;  
	}  
}
```

![](https://blog.kakaocdn.net/dn/b7XZSb/btsI5MwKziq/p0Hew3G1Y7oLnsJndoxx31/img.png)

실패에 관한 많은 정보를 제공해줄 수 있다. 어떤 부분이 잘못되었는 지 쉽게 파악 가능하다.

그런 경우가 거의 없겠지만 최솟값이 최댓값보다 큰 경우에도 문제를 쉽게 파악할 수 있다.

> 보안과 관련된 정보는 주의해라

예외와 관련된 정보를 제공하라고 했지만, 보안과 관련된 정보는 조심해서 다루어야 한다.

문제를 진단하고 해결하는 과정에서 스택 추적 정보는 많은 사람들이 보게 된다. 따라서 상세 메시지에 비밀번호같은 보안과 관련된 부분을 담아서는 안된다.

### 정보를 장황하게 작성할 필요는 없다.

관련 데이터를 모두 담으라 했지만, 너무 장황하게 작성할 필요는 없다.

문제를 해결하려는 사람은 스택 추적뿐만 아니라 관련 문서와 소스코드를 함께 살펴보기 마련이다.

보통 스택 추적에는 예외가 발생한 파일 이름, 줄 번호, 스택에서 호출한 다른 메서드들의 파일 이름과 줄 번호까지 정확히 기록되어 있다.

따라서 문서와 소스코드에서 제공되는 정보까지 길게 늘어뜨려 제공할 필요는 없다.

예외의 상세 메시지는 보통 문제를 분석하는 프로그래머와 엔지니어라는 사실을 잊지말자.

단, 최종 사용자에게 보여줄 오류 메시지와 혼동해서는 안된다. 사용자에게는 친절하고 자세한 안내 메시지를 보여주어야 한다.

### 상세 메시지를 미리 생성하는 방법을 고려하라

현재 IndexOutOfBoundsException은 String을 받도록 구현되어 있다.

```java
public IndexOutOfBoundsException(String s) {  
    super(s);  
}

public IndexOutOfBoundsException(int index) {  
    super("Index out of range: " + index);  
} //Java 9 이후 등장
```

실패를 적절하게 포착하기 위해서는 필요한 정보를 예외 생성자에서 모두 받아 상세 메시지를 미리 생성하는 방법도 좋다.

```java
public IndexOutOfBoundsException extends RuntimeException {
	private final lowerBound;
	private final upperBound;
	private final index;

	public IndexOutOfBoundsException(int lowerBound, int upperBound, int index) {
		super(String.format(
				"최솟값: %d, 최댓값: %d, 인덱스: %d",
				lowerBound, upperBound, index));
				
		this.lowerBound = lowerBound;
		this.upperBound = upperBound;
		this.index = index;
	}
}
```

Java 9 이후 등장한 index 값을 받는 IndexOutOfBoundsException을 보면 최솟값과 최댓값까지는 받지 않는다.

자바 라이브러리에서는 이 책의 저자가 제안하는 관여된 모든 매개변수와 필드의 값을 모두 담으라는 의견을 적극적으로 수용하지 않는다는 뜻이다.

그러나 이 책의 저자는 프로그래머가 던지는 예외는 실패를 잘 포착하게 되고, 더불어 고품질의 상세 메시지를 만들어내는 코드를 예외 클래스 안으로 모아 클래스 사용자가 메시지를 만드는 작업을 중복하지 않아도 된다는 장점이 있다고 어필한다.

나도 이 저자의 의견에 동의한다. 예외를 던지면서 같은 오류라도 메시지를 다르게 작성할 수 있는데, 고품질의 예외 메시지를 만들어내면서 그런 경우를 방지해줄 수 있다. 가독성 향상 및 오류 포착에 매우 도움이 될 것 같다.

### 예외의 접근자 메서드

```java
public class InvalidInputException extends Exception {
	private final String input;
	private final String expectedFormat;
	
	public InvalidInputException(String input, String expectedFormat) {
		super(String.format("Invalid Input: %s, Expected Format: %s", input, expectedFormat);
		//...
	}

	public String getInput() {
		return input;
	}

	public String getExpectedFormat() {
		return expectedFormat;
	}
}
```
```java
public void inputFunc(String input) throws InvalidInputException {
	String expectedFormat = "\\d+";

	if (!input.matches(expectedFormat)) {
		throw new InvalidInputException(input, expectedFormat);
	}

	//...
}

try {
	//...
} catch (InvalidInputException e) {
	System.err.println("Invalid Input: " + e.getInput());
	//...
}
```

아이템 70에서도 언급했듯 예외도 결국 객체이기 때문에 정보를 담을 수 있고, 예외 상황에서 벗어나기 위해 필요한 정보를 알려주는 메서드를 제공하는 것이 좋다.

포착한 실패 정보는 예외 상황을 복구하는데 유용하기 때문에 비검사 예외보다는 검사 예외에서 더 빛을 발할 수 있다.

비검사 예외의 상세 정보에 프로그램적으로 접근하기를 원하는 프로그래머는 드물것이다. 

하지만 toString이 반환한 값에 포함된 정보를 얻어올 수 있는 API를 제공하자는 일반 원칙을 따른다는 관점에서 비검사 예외라도 상세 정보를 알려주는 접근자 메서드를 제공하기를 권한다.

## 정리

모든 예외의 상세 메시지에 예외에 관여된 모든 매개변수와 필드 정보를 제공하자. 

실패에 관한 많은 정보를 제공해 어떤 부분이 잘못되었는 지 쉽게 파악 가능하다.

상세 메시지를 예외 클래스 내에서 미리 생성해 관리하는 방법도 좋다. 그리고 관련된 정보를 알려주는 접근자 메서드도 제공하면 좋다.