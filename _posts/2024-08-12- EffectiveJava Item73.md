---
title: Item73 (추상화 수준에 맞는 예외를 던지라)
author: leedohyun
date: 2024-08-12 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 추상화 수준에 맞는 예외를 던지라

수행하려는 일과 관련 없어 보이는 예외가 나타나면 당황스럽다.

이는 메서드가 저수준 예외를 처리하지 않고 바깥으로 전파하는 경우 종종 발생하는 일이다.

이러한 일은 프로그래머를 당황시키는 정도에 그치지 않고, 내부 구현 방식을 드러낸다는 치명적인 문제를 가지고 있다. 

상위 레벨의 API를 오염시킨다. 만약 다음 릴리스에서 구현 방식을 바꾼다면 다른 예외가 발생해 기존 클라이언트 프로그램을 깨뜨릴 가능성 또한 존재한다.

### 상위 계층에서는 예외 번역을 하자.

위에서 언급한 문제를 피하려면 **상위 계층에서 저수준 예외를 잡아 자신의 추상화 수준에 맞는 예외로 바꾸어 던져야 한다.** 이를 예외 번역이라고 한다.

```java
try {
	//...
} catch (LowerLevelException e) {
	throw new HigherLevelException(...);
}
```

AbstractSequentialList를 예시로 예외 번역에 대해 자세히 알아보자.

```java
public E get(int index) {
    try {
        return listIterator(index).next();
    } catch(NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: " + index);
    }
}
```

- get() 메서드는 리스트 내의 지정한 인덱스 위치의 원소를 반환하는 메서드이다.
- listIterator(index)를 통해 리스트의 해당 위치로 이동하고 next()를 호출하여 그 위치에 있는 요소를 반환한다.
	- 이 때 지정된 인덱스가 리스트의 범위를 벗어날 경우 NoSuchElementException을 던질 수 있다.
	- Iterator나 ListIterator가 더 이상 요소를 반환할 수 없을 때 발생하는 예외이다.
- 저수준 예외인 NoSuchElementException을 잡아 IndexOutOfBoundsException으로 변환해 다시 던진다.
	- 더 의미있고, 해당 메서드에 어울리는 예외로 바뀌었다.

### 저수준 예외가 디버깅에 도움이 되는 경우

저수준 예외가 디버깅에 도움이 되는 경우가 있을 것이다.

이런 경우 예외 연쇄를 사용하면 된다. 예외 연쇄란 저수준 예외를 고수준 예외에 실어 보내는 것을 뜻한다.

별도의 접근자 메서드를 통해 언제든 저수준 예외를 꺼내 볼 수 있다.

```java
try {
	//...
} catch (LowerLevelException cause) {
	throw new HigherLevelException(cause);
	// 저수준 예외를 고수준 예외에 실어 보낸다.
}
```

```java
class HigherLevelException extends Excpetion {
    HigherLevelException(Throwable cause) {
        super(cause);
    }
}
```

- 고수준 예외의 생성자는 예외 연쇄용으로 설계된 상위 클래스의 생성자에 원인을 건네주어 Throwable 생성자까지 건네지게 한다.
- 대부분의 표준 예외는 위와 같은 예외 연쇄용 생성자를 갖추고 있다.
	- 만약 예외 연쇄용 생성자를 갖추고 있지 않은 예외라도 Throwable의 initCause 메서드를 이용해 원인을 직접 설정할 수 있다.

```java
public class CustomException extends Exception { 
	public CustomException(String message) { 
		super(message); 
	} 
}
```
```java
try { 
	String str = null; 
	str.length(); // NullPointerException 발생 
} catch (NullPointerException e) { 
	// CustomException으로 변환하여 던짐 
	CustomException customException = new CustomException(); 
	customException.initCause(e); // 원인을 설정 
	throw customException; 
}
```

- 위와 같이 initCause() 메서드를 이용해 원인을 직접 설정할 수 있고 위와 같은 경우에는 Caused by: java.lang.NullPointerException 문구를 통해 원인을 파악할 수 있게 된다.

결론적으로 예외 연쇄는 문제의 원인을 getCause() 메서드 등으로 프로그램에서 접근할 수 있도록 해주고, 원인과 고수준 예외의 스택 추적 정보를 잘 통합해주는 역할을 한다.

### 예외 번역을 남용하지 말자

무턱대고 예외를 전파하는 것보다 예외 번역이 우수한 방법인 것은 맞다. 하지만 남용해서는 안된다.

가능하면 **저수준 메서드가 반드시 성공하도록 하여 아래 계층에서는 예외가 발생하지 않도록 하는 것이 최선이다.**

때로는 상위 계층 메서드의 매개변수 값을 넘기기 전 validator를 통해 미리 검사하는 방법으로 이 목적을 달성할 수도 있다.

만약, 아래 계층에서 예외를 피할 수 없는 경우 상위 계층에서 그 예외를 처리해 API 호출자에게 전파하지 않는 방법도 있다.

이렇게 처리할 경우 발생한 예외를 로깅을 통해 기록해두면 클라이언트 코드와 사용자에게 문제를 전파하지 않으면서, 프로그래머는 로그를 분석해 추가적인 조치를 취할 수 있다.

## 정리

아래 계층의 예외를 예방하거나, 스스로 처리할 수 없고 그 예외를 상위 계층에 그대로 노출했을 때 혼란을 준다면 예외 번역, 예외 연쇄를 사용하자.

상위 계층에서 맥락에 맞는 예외를 던지면서도 그 원인을 쉽게 추적할 수 있게 해준다.