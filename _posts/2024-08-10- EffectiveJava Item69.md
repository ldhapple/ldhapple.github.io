---
title: Item69 (예외는 진짜 예외 상황에만 사용하라)
author: leedohyun
date: 2024-08-10 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 예외는 진짜 예외 상황에만 사용하라

예외를 제대로 활용하면 프로그램의 가독성, 신뢰성, 유지보수성이 높아진다.

하지만 잘못 사용하는 경우 역효과가 난다. 예외를 잘못 사용하는 사례들을 보고 예외를 어떻게 사용해야 하는지 보자.

### 코드가 직관적이지 않고 성능 문제가 발생한다.

```java
try {
	int idx = 0;
	while (true) {
		range[idx++].doSomething();
	}
} catch (ArrayIndexOutOfBoundsException e) {
	//...
}
```

- 위 코드는 배열을 순회하고 있다.
- 무한 루프를 돌다가 배열의 끝에 도달하면 ArrayIndexOutOfBoundsException이 발생하면 끝을 낸다.
-  JVM이 배열에 접근할 때 경계를 넘지 않는지 여부를 검사하는데, 일반적인 반복문 또한 경계에 도달하면 종료한다.
	- 일반적인 for문이라고 가정한다면 for문의 범위 변수로 조건을 걸어, 배열의 경계를 계속 검사했을 것이다. 그런데 JVM 또한 배열의 경계를 검사하기 때문에 두 가지 일을 하는 것이다.
	- 따라서 일을 두 번 반복하지 않기 위해 한 가지 일을 생략한 구조다.

그렇다면 위 코드의 문제는 무엇일까?

- 코드가 전혀 직관적이지 않다. 무슨 일을 의도하는 지 한 눈에 파악할 수 없다.
- 예외를 오용하면 성능 문제가 있을 수 있다.
	- 예외는 예외 상황에만 쓸 용도로 설계되었다. JVM 구현자 입장에서는 명확한 검사만큼 빠르게 만들어야 할 동기가 약하다.
	- 위 예시에서는 예외를 루프를 종료하는 용도로 사용하고 있는데, 알맞지 않다. 예외는 비정상적인 상황을 처리하기 위한 것이다.
	- 예외가 발생할 때 JVM은 예외 객체를 생성한다. 이러한 작업은 비용이 많이 들고 루프를 종료하기 위해 이를 발생시키면 성능이 저하될 수 있다.
	- **JVM 설계자는 예외 처리가 빈번하게 발생하지 않는다고 가정했기 때문에 예외가 이렇게 반복적으로 발생하면 일반적인 조건 검사 방식보다 느릴 수 있다.**
- 코드를 try-catch 블록 안에 넣으면 JVM이 적용할 수 있는 최적화가 제한된다.
	- try-catch 블록 내부에 있으면 JVM은 그 코드에 대해 예외 발생 가능성을 항상 고려해 부가적인 작업을 한다.
	- 따라서 JVM이 코드의 실행을 더 보수적으로 처리해 최적화되지 않는다.
- 배열을 순회하는 표준 관용구는 중복 검사를 하지 않는다. JVM이 최적화를 해준다.
	- 배열의 길이를 미리 계산해 루프 내에서 재계산을 피하는 등의 최적화를 해준다.

```java
for (Instance i : range) {
	i.doSomething();
}
```

> 관용구와 예외 코드의 성능 차이

위의 두 코드의 성능 차이를 보자.

```java
@Test  
void test4() {  
  Instance[] range = new Instance[100_000];  
  
  for (int i = 0; i < 100_000; i++) {  
	  range[i] = new Instance();  
  }  
  
  try {  
	  int idx = 0;  
	  while (true) {  
		  range[idx++].doSomething();  
	  }  
  } catch (ArrayIndexOutOfBoundsException e) {  
	  e.printStackTrace();  
  }  
}  
  
@Test  
void test5() {  
  Instance[] range = new Instance[100_000];  
  
  for (int i = 0; i < 100_000; i++) {  
	  range[i] = new Instance();  
  }  
  
  for (Instance i : range) {  
	  i.doSomething();  
  }  
}
```

- 예외를 사용한 쪽은 349ms
- 관용구를 사용한 쪽은 292ms

책에서는 두 배 이상 성능 차이가 있다고 했는데 doSomething() 코드에서 어떤 동작을 하느냐에 따라 성능 차이가 더 많이 발생할 것 같기는 하다.

결론적으로는 관용구를 사용한 쪽이 성능이 더 좋다.

### 코드가 제대로 동작하지 않을 수도 있다.

예외를 사용한 반복문의 해악은 코드를 헷갈리게 하고 성능을 떨어뜨리는 정도에서 끝나지 않는다.

더 큰 문제는 반복문 안에 버그가 숨어있다면 흐름 제어에 쓰인 예외가 이 버그를 숨겨 디버깅을 어렵게 만든다는 것이다.

반복문에서 호출한 메서드가 내부에서 관련 없는 배열을 사용하다가 ArrayIndexOutOfBoundsException을 일으켰다고 가정해보자.

- **표준 관용구 사용**
	- 예외를 잡지 않고, 스택에 정보를 남긴 후 해당 쓰레드를 즉각 종료시킨다.
- **예외를 사용한 반복문**
	- 버그때문에 발생한 엉뚱한 예외를 정상적인 반복문 종료 상황으로 인식한다.

***따라서 예외는 오직 예외 상황에서만 써야한다. 일상적인 제어 흐름용으로 쓰여서는 절대 안된다.***

표준적이고 쉽게 이해되는 관용구를 사용하고, 성능 개선을 목적으로 과하게 머리를 쓴 기법은 자제해야 한다.

실제로 성능이 좋아지더라도 자바 플랫폼이 꾸준히 개선되고 있어 최적화로 얻은 상대적인 성능 우위가 오래가지 않을 수 있다. 

반면 과하게 영리한 기법에 숨겨진 미묘한 버그의 폐해와 어려워진 유지 보수 문제는 지속된다.

### 잘 설계된 API에서의 예외

**잘 설계된 API는 클라이언트가 *정상적인 제어 흐름*에서 예외를 사용할 일이 없게 해야한다.**

특정 상태에서만 호출 가능한 상태 의존적 메서드를 제공하는 경우, 상태 검사 메서드도 함께 제공해야 한다.

Iterator 인터페이스의 next와 hasNext가 각각 상태 의존적 메서드와 상태 검사 메서드에 해당된다.

```java
List<Instance> range = new ArrayList<>();
for (Iterator<Instance> i = range.iterator(); i.hasNext();) {
	//...
}

try {
	Iterator<Instance> iter = range.iterator();
	while (true) {
		Instance i = iter.next();
	}
} catch (NoSuchElementException e) {
	//...
}
```

- 상태 검사 메서드 덕분에 표준 for 관용구를 사용할 수 있다.
	- for-each도 내부적으로 hasNext를 사용한다.
- 만약 Iterator가 hasNext()를 제공하지 않았다면 그 일을 클라이언트가 처리해야 했다. 

말이 조금 헷갈릴 수 있는데 정리해보자.

- 잘 설계된 API는 클라이언트가 **정상적인 흐름**에서 예외를 사용할 일이 없게 해야된다고 했다.
	- next()는 다음 원소가 있을 때 부를 수 있는 상태 의존적 메서드이다.
	- 만약 hasNext()가 없어, 검사하지 않고 next() 호출을 했다면 다음 원소가 없는 경우 예외가 발생할 것이다.
		- 이런 경우가 있으면 안된다는 뜻이다.
	- 따라서 hasNext()라는 상태 검사 메서드를 제공해 예외가 발생하지 않도록 도와주는 것이다. 
	- 만약 그렇지 않았다면 클라이언트는 예외 발생 가능성에 대비해 try-catch 블록을 사용해 처리해야 했을 것이다.

> 상태 검사 메서드 대신 사용할 수 있는 선택지

null 또는 Optional과 같은 특수한 값을 반환하는 방법이다.

- 외부 동기화 없이 여러 쓰레드가 동시에 접근할 수 있거나 외부 요인으로 상태가 변할 수 있다면 Optional이나 특정 값을 사용한다.
	- 상태 검사 메서드와 상태 의존적 메서드 호출 사이에 객체의 상태가 변할 수 있기 때문이다.
- 성능이 중요한 상황에서 상태 검사 메서드가 상태 의존적 메서드의 작업 일부를 중복 수행한다면 Optional이나 특정 값을 선택한다.
- 다른 모든 경우엔 상태 검사 메서드 방식이 조금 더 낫다.
	- 가독성이 비교적 좋고, 잘못 사용했을 때 발견하기 쉽다.
	- 상태 검사 메서드 호출을 잊은 경우 상태 의존적 메서드가 예외를 던져 버그를 확실히 드러낸다. 반면 특정 값은 검사하지 않고 지나쳐도 발견하기 어렵다.

[옵셔널 참고](https://ldhapple.github.io/posts/EffectiveJava-Item55/)

## 정리

예외는 예외 상황에서만 쓸 의도로 설계되었다. 그 외의 의도로 사용한다면 코드 가독성 문제 및 성능 문제, 그리고 오동작 문제가 생길 수 있다.

정상적인 제어 흐름에서 다른 의도로 절대 사용하지 말자. 

더해 잘 설계된 API에서는 클라이언트가 정상적인 흐름에서 예외를 사용할 일이 없어야 한다.