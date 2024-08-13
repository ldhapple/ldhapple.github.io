---
title: Item72 (표준 예외를 사용하라) + 커스텀 예외에 대한 내 생각
author: leedohyun
date: 2024-08-12 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 표준 예외를 사용하라

숙련된 프로그래머는 그렇지 못한 프로그래머보다 코드 재사용 빈도가 높다.

예외도 마찬가지다. 재사용하는 것이 좋다. 자바 라이브러리는 대부분 API에서 쓰기에 충분한 수의 예외를 제공한다.

표준 예외를 재사용하면 얻는 이점은 아래와 같다.

- 다른 사람이 API를 익히고 사용하기 쉬워진다.
	- 많은 프로그래머들에게 익숙한 규약을 그대로 따르기 때문이다.
- API 자체도 낯선 예외를 사용하지 않아 가독성이 좋아진다.
- 예외 클래스 수가 적으면 당연히 메모리 사용량도 줄고 클래스를 적재하는 시간 또한 적게 걸린다.

### 표준 예외

많이 쓰이는 표준 예외들을 보자.

- IllegalArgumentException
	- 호출자가 인수로 부적절한 값을 넘길 때 던지는 예외.
- IllegalStateException
	- 대상 객체의 상태가 호출된 메서드를 수행하기에 적합하지 않을 때 던지는 예외.
- NullPointerException
	- null 값을 허용하지 않는 메서드에 null을 건네는 경우 던지는 예외.
- IndexOutOfBoundsException
	- 시퀀스의 허용 범위를 넘는 값을 건네는 경우 던지는 예외.
- ConcurrentModificationException
	- 단일 쓰레드에서 사용하려고 설계한 객체를 여러 쓰레드가 동시에 수정하려고 할 때 던지는 예외.
- UnsupportedOperationException
	- 클라이언트가 요청한 동작을 대상 객체가 지원하지 않는 경우 던지는 예외. 

```java
List<String> list = new ArrayList<>();  
list.add("A");  
list.add("B");  
list.add("C");  
  
List<String> unmodifiableList = Collections.unmodifiableList(list);  
  
  
unmodifiableList.add("D"); 
//UnsupportedOperationException 발생!
```

상황에 부합한다면 자주 쓰이는 표준 예외들을 재사용하도록 하자. 단, 예외의 이름뿐 아니라 예외가 던져지는 맥락 또한 부합할 때만 재사용한다.

> 주의 사항

**Exception, RuntimeException, Throwable, Error는 직접 재사용하지 말자.**

이 클래스들은 추상 클래스로 여기면 된다. 이 예외들은 다른 예외들의 상위 클래스이기 때문에 여러 성격의 예외들을 포괄하는 클래스들이다.

따라서 안정적으로 테스트가 불가능하기 때문에 해당 코드를 직접 재사용하는 것은 추천하지 않는다.

### 예외는 직렬화가 가능하다는 사실을 잊지마라.

더 많은 정보를 제공하기를 원한다면 표준 예외를 확장해도 좋다. 

그러나 예외는 직렬화가 가능하다는 사실을 간과해서는 안된다. 직렬화는 많은 부담이 따르기 때문에 이 부분만으로도 커스텀 예외를 만들지 않을 이유가 될 수 있다.

- 직렬화는 객체를 바이트 스트림으로 변환해 파일제 저장하거나 네트워크를 통해 전송할 수 있도록 하는 것이다.
	- 자바에서는 Serializable 인터페이스를 구현하면 되고, **자바의 표준 예외들은 Serializable을 구현하고 있다.**
- 예외가 직렬화 가능한 이유?
	- 예외 객체가 네트워크 통신이나 파일 시스템에 저장되는 경우가 있기 때문이다.
- 직렬화는 CPU와 메모리 자원을 소모하기 때문에 부담이 된다.

### 커스텀 예외를 사용하는 경우?

커스텀 예외의 대표적인 장점은 예외 클래스의 이름만으로 어떤 예외인지 쉽게 파악할 수 있다는 점이다.

그런데 위에서 커스텀 예외는 분명 부담이 있다는 사실을 알았다. 그리고 표준 예외로도 예외 메시지를 통해 의미를 명확하게 전달할 수 있다.

또한 예외 클래스를 만들다보면 지나치게 많은 커스텀 예외가 많아져 오히려 혼란을 가져올 수도 있고, 위에서 언급했듯 메모리 문제 및 클래스 로딩에도 많은 시간을 소모하게 된다.

그렇다면 커스텀 예외의 장점들을 보자.

- 예외 이름으로 정보를 전달할 수 있다.
	- IllegalArgumentException은 포괄적이지만,  UserNameEmptyException은 의미가 명확하다.
- 예외 정보를 상세하게 제공할 수 있다.

```java
public class IllegalIndexException extends IndexOutOfBoundsException {
	private static final String message = "범위를 벗어났습니다.";

	public IllegalIndexException(List<?> target, int index) {
		super(message + " size: "  + target.size() + " index: " + index);
	}
}
```
```java
public static <T> T getElement(List<T> list, int index) { 
	if (index < 0 || index >= list.size()) { 
		throw new IllegalIndexException(list, index); 
		// 커스텀 예외 발생 
	} 
	return list.get(index); 
}

//예외 발생: 범위를 벗어났습니다. size: 3 index: 5
```

- 예외에 대한 응집도가 향상된다.
	- 클래스를 만드는 것은 관련 정보를 되도록 해당 클래스 내에서 모두 관리하겠다는 의미이다.
	- 커스텀 예외를 만들면 예외에 필요한 메시지, 전달할 정보, 데이터 가공 메서드 등을 한 곳에서 관리할 수 있다.
- 예외 발생 후처리가 용이하다.
	- Spring에서는 ControllerAdvice를 통해 전역적으로 예외 처리를 할 수 있다.
	- 표준 예외는 재사용성이 뛰어나기 때문에 전역적으로 처리했을 경우 처리하지 않아야 할 부분도 처리하게 될 수도 있다.
		- 우리가 작성한 코드가 아닌 라이브러리에서 발생한 예외일 수도 있어 의도치 않게 예외를 처리하게 될 수도 있다.
- 예외 생성 비용을 절감할 수 있다.
	- Java의 예외 생성 비용은 stack trace 때문에 생각보다 크다.
	- stack trace를 만약 사용하지 않는 경우라면 이 비용이 아까울 수 있다.
	- stack trace는 Throwable의 fillInStackTrace() 메서드를 통해 생성되는데, 커스텀 예외는 이를 Override해 stack trace의 생성 비용을 줄일 수 있다.
	- 물론 이 부분은 디버깅을 어렵게 만들기 때문에 상황에 따라 알맞게 판단해야 할 것이다.

> 사견

나는 그동안 커스텀 예외를 사용하는 것이 더 좋다고 생각하고 있었다. 

예외 이름만으로 어디서 어떻게 발생한 예외인지 비교적 쉽게 알 수 있기 때문이다.

이번 아이템을 공부하면서 표준 예외의 장점도 알았고, 커스텀 예외의 장점도 더 잘 알게 되었다.

정답은 없겠지만, 사실 아직은 예외의 직렬화로 인한 성능 문제를 경험해보지 못해서 그런지 커스텀 예외를 만드는 쪽이 더 나은 선택같아 보이기는 하다.

코드의 가독성 부분도 사실 커스텀 예외의 네이밍이 적절하다면 큰 차이가 없다고 생각되고, 직렬화에 대한 부담도 아직은 크게 와닿지 않기 때문인 것 같다.

물론 나중에 그러한 부분들이 체감된다면 커스텀 예외의 장점을 버리고 표준 예외를 재사용하는 쪽으로 의견이 바뀌겠지만, 지금은 커스텀 예외의 관리 용이성, 후처리가 용이하다는 부분에 장점이 더 크다고 느껴진다.