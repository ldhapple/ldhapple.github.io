---
title: Item8 (finalizer와 cleaner 사용을 피하라)
author: leedohyun
date: 2024-07-02 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## finalizer와 cleaner 사용을 피하라

자바에서는 두 가지의 객체 소멸자를 제공한다. finalizer와 cleaner이다.

finalizer는 예측할 수 없고, 상황에 따라 위험할 수 있어 일반적으로 불필요하다. 오동작, 낮은 성능, 이식성 문제의 원인이 되기도 한다. 쓰임새가 몇 있긴하지만 기본적으로 사용하면 안된다.

그래서 Java 9부터 finalizer를 사용 자제(deprecated) API로 지정하고 cleaner를 그 대안으로 소개했다.

cleaner는 finalizer보다는 덜 위험하지만 여전히 예측할 수 없고, 느리고, 일반적으로 불필요하다.

***즉 두 소멸자 모두 예측이 불가능하고, 느리고 일반적으로 불필요하다.***

### finalizer와 cleaner는 즉시 수행을 보장하지 않는다.

객체에 접근할 수 없게 된 후 finalizer나 cleaner가 실행되기까지 얼마나 걸릴 지 알 수 없다.

- 만약 파일 닫기를 finalizer나 cleaner에 맡긴다면?
	- 시스템이 동시에 열 수 있는 파일 개수는 한정적이다.
	- 그런데 finalizer나 cleaner가 즉시 수행되지 않기 때문에 언제 그 파일 닫기가 처리될 지 알 수 없다.
	- 이렇게 파일이 계속 열려있는 상태에서 새로운 파일을 열지 못하는 경우가 생기는 것.

### finalizer와 cleaner가 얼마나 빠르게 수행될 지 알 수 없다.

두 소멸자가 얼마나 신속하게 수행될지는 가비지 컬렉터 알고리즘에 달려있다. 즉 가비지 컬렉터의 구현마다 다르다는 뜻이다.

finalizer나 cleaner 수행 시점에 의존하는 프로그램이 있다고 가정한다면, 개발자가 test한 JVM 환경에서는 완벽하게 동작하던 프로그램이 수행 시점이 변경되어 클라이언트 시스템에서는 엄청난 오류를 발생시킬 수 있다.

### 우선순위가 낮다.

finalizer 쓰레드는 우선순위가 낮아 실행될 기회를 얻지 못할 수 있다.

클래스에 finalizer를 달아두면 그 인스턴스의 자원 회수가 지연될 수 있다. 해당 인스턴스의 처리가 우선순위에서 밀려 계속 쌓이다보면 결국 OutOfMemoryError까지 발생시킬 수 있는 것이다.

cleaner는 자신을 수행할 쓰레드를 제어할 수 있어 조금 낫긴 하지만 여전히 백그라운드에서 수행되며 가비지 컬렉터의 통제하에 있기 때문에 즉시 수행되리라는 보장은 없다.

### 수행 여부조차 보장되지 않는다.

수행 시점뿐 아니라 수행 여부조차 보장하지 않는 점도 문제다.

이제 접근할 수 없는 일부 객체의 종료 작업을 전혀 수행하지 않고 프로그램이 중단될 수 있다는 뜻이다. 따라서 프로그램의 생애 주기와 관계 없는 상태를 영구적으로 수정하는 작업에서는 절대 finalizer나 cleaner에 의존해서는 안된다.

ex) 데이터베이스의 영구 락 해제

```java
@Override
protected void finalize() thorws Throwable {
	//락 해제
}
```

- 어떤 객체가 락을 가지고 있는데 그 객체가 소멸될 때 finalizer에 락 해제를 맡긴 것이다.
	- 이 경우 락 해제가 실행되지 않을 수 있고 그렇게되면 분산 시스템 전체가 서서히 멈출 수 있다.

### System.gc()나 System.runFinalization() 메서드에 현혹되지 말자

이러한 메서드는 finalizer와 cleaner가 실행될 가능성을 높여줄 수는 있으나 여전히 보장하지 않는다.

System.runFinalizersOnExit와 Runtime.runFinalizersOnExit 메서드는 보장해준다는 명목하에 나타났었다.

하지만 Java 9에서 deprecated 되었고, Java 11에서는 삭제되었다.

- 이 메서드들은 JVM이 종료될 때 정의된 소멸자가 있는 모든 객체에 대해 소멸자를 실행하려고 했다.
	- 그런데 이 경우 아직 사용중이거나 완전한 종료 프로세스에 필요한 객체 또한 포함되어 문제가 발생할 수 있었다.
-  소멸자가 실행되는 순서가 일관적이지 않았다.
- JVM 종료 중 언제든 소멸자가 실행될 수 있어 여러 소멸자가 동일한 락 또는 리소스를 획득하려고 시도하는 등의 교착상태가 발생할 수 있었다.
- 종료 시점에 많은 수의 소멸자를 동시에 실행하게 되어 성능 문제가 발생할 수 있었다.
	- 소멸자를 사용해 객체를 생성하고 삭제하는 프로그램에서는 더더욱 문제가 발생한다.

### finalizer 동작 중 발생한 예외는 무시된다.

예외도 무시되고, 처리할 작업이 남아있어도 그 순간 종료된다.

무시된 예외로 객체는 작업을 마무리하지 못한 채로 남을 수 있다. 이런 상태에서 만약 다른 쓰레드가 작업을 마무리하지 못한 객체를 그대로 사용하게 된다면 예기치 못한 에러를 일으킬 수 있다.

보통의 경우 잡지 못한 예외가 쓰레드를 중단시키고 에러 내역을 출력하겠지만 finalizer에서 발생했을 경우 아무런 경고조차도 출력하지 않는다.

cleaner는 그나마 자신의 쓰레드를 통제하기 때문에 이러한 문제는 발생하지 않는다.

### finalizer와 cleaner의 성능문제

try - with - resources로 닫는 것과 finalizer를 사용하는 것의 성능 차이를 보면 아래와 같다.

```java
static class Resource implements AutoCloseable {  
	  @Override  
	  public void close() {...}
 }  
  
static class ResourceWithFinalize {  
	  @Override  
	  protected void finalize() throws Throwable {  
		  try {  
		  } finally {  
			  super.finalize();  
		  }  
	  }  
}
```

![](https://blog.kakaocdn.net/dn/bmELK3/btsIkgAmUrF/cxaKaWjaLIvnTWmwFsdkKk/img.png)

위 객체들을 1000번 생성하고 파괴한 결과이다. 엄청난 성능 차이를 보인다. finalizer를 사용한 객체를 생성하고 파괴할 때 훨씬 느리다. 가비지 컬렉터의 효율을 떨어뜨리기 때문이다.

cleaner도 마찬가지로 클래스의 모든 인스턴스를 수거하는 형태로 사용한다면 성능은 finalizer와 비슷하다.

### 공격에 노출될 수도 있다.

finalizer를 사용한 클래스는 finalizer 공격에 노출되어 심각한 보안 문제를 발생시킬 수 있다.

생성자나 직렬화 과정에서 예외가 발생하면 이 생성되다 만 불완전한 객체에서 악의적으로 하위 클래스의 finalizer가 수행될 수 있게 된다.

```java
class Product {
	private int price;
	private int stock;

	public Product(final int price, final int stock) {
		if (stock < 0) {
			throw new IllegalArgumentException("생성 불가");
		}
		this.price = price;
		this.stock = stock;
	}

	void sell(final int count) {
		this.stock -= count;
		System.out.printf("%d개 판매 완료", count);
	}
}
```
```java
class Attack extends Product {
	public Attack(final int price, final int stock) {
		super(price, stock);
	}

	@Override
	protected void finalize() throws Throwable {
		this.sell(10);
	}
}
```

```java
Product product = null;

try {
	product = new Attack(1_000, -5);
	product.sell(50);
} catch (Exception e) {
	System.out.println("예외 발생");
}

System.gc();
sleep(2000);
```

![](https://blog.kakaocdn.net/dn/bXRGvs/btsIkEHx0wG/6QKRXCu24NIRhtMkCc2hWK/img.png)

- 개수가 음수인 물건은 생성을 못하도록 막아놨기 때문에 예외가 발생했다.
	- 그런데 그 하위 클래스에서 finalizer가 수행될 수 있도록 만들어놨기 때문에 판매 로직은 정상 작동 되는 현상을 볼 수 있다.

실제로 이런 구현은 없겠지만 악의적으로 이렇게 공격할 수 있다는 뜻이다.

> 어떻게 방지할 수 있을까?

객체 생성을 막으려면 생성자에서 예외를 던지는 것만으로 충분하다. 그러나 finalizer가 있다면 그렇지도 않다는 것을 위에서 보았다.

- final 클래스를 만들어 하위 클래스가 만들어지는 것을 방지
- final이 아닌 클래스를 원한다면 아무 일도 하지 않는 finalize 메서드를 만들고 final로 선언하면 된다.

## 그럼 어떻게 사용해야 할까?

finalizer와 cleaner에 문제가 많다는 것을 알았고, 그렇다면 그의 대안이 필요하다.

### AutoCloseable 구현으로 대체

AutoCloseable을 구현하고 클라이언트에서 인스턴스를 다 썼을 때 close() 메서드를 호출하면 된다.

일반적으로 try-with-resources를 사용해 예외가 발생해도 제대로 종료되도록 해야 한다.

```java
class Resource implements AutoCloseable {
	private boolean isClosed = false;

	public void 작업() {
		if (isClosed) {
			throw new IllgalStateException("이미 닫힌 객체입니다.");
		}
		//...
	}

	@Override
	public void close() throws Exception {
		if (!isClosed) {
			isClosed = true;
			//...
			//리소스 종료에 필요한 작업..
		}
	}
}
```

close() 구현 시 각 인스턴스는 자신이 닫혔는지 추척하는 것이 좋다. close 메서드에서 이 객체는 더 이상 유효하지 않음을 필드에 기록하고 다른 메서드는 이 필드를 검사해 객체가 닫힌 후에 불렸다면 예외를 던지는 방식이다.

## finalizer와 cleaner의 사용

그럼에도 불구하고 finalizer와 cleaner는 적절한 쓰임새가 있다.

### 자원의 소유자가 close() 메서드를 호출하지 않는 것에 대비하는 안전망 역할

cleaner나 finalizer가 즉시, 그리고 호출된다는 것 자체에 보장은 없다.

하지만 클라이언트가 하지 않은 자원 회수를 늦게라도 해주는 것이 아예 안하는 것 보다 낫다.

이러한 안전망 역할의 finalizer를 작성할 때에는 그럴만한 값어치가 있는 지 판단하는 것이 중요하다.

자바 라이브러리의 일부 클래스는 안전망 역할의 finalizer를 제공하고 있다.

- FileInputStream, FileOutputStream, ThreadPoolExecutor

### 네이티브 피어(native peer)와 연결된 객체에서 사용

네이티브 피어는 자바 객체가 아니기 때문에 가비지 컬렉터가 그 존재를 인지하지 못한다. 따라서 자바 피어를 회수할 때 네이티브 객체까지 회수하지 못한다.

이런 경우 cleaner나 finalizer가 처리해줄 수 있다.

단 성능 저하를 감당할 수 있고, 네이티브 피어가 심각한 자원을 가지고 있지 않는 경우에만 해당한다.

네이티브 피어가 사용하는 자원을 즉시 회수해야 한다면 close() 메서드를 사용해야 한다.

> 네이티브 피어

일반 자바 객체가 네이티브 메서드를 통해 기능을 위임한 네이티브 객체를 의미한다. 일반적으로 C나 C++로 작성된 코드와 상호 작용하는 경우를 의미한다. 

성능 향상이나 라이브러리와의 통합을 위해 사용한다.

### Cleaner의 사용

Cleaner를 안전망으로 활용하는 것은 조금 까다롭다. 아래 코드를 보자.

```java
public class Room implements AutoCloseable {
	private static final Cleaner cleaner = Cleaner.create();
	
	//청소가 필요한 자원(Room을 참조해서는 안된다.)
	private static class State implements Runnable {
		int numJunkPiles; //Room 안의 쓰레기 수

		State(int numJunkPiles) {
			this.numJunkPiles = numJunkPiles;
		}
	
		@Override
		public void run() {
			System.out.println("방 청소");
			numJunkPiles = 0;
		}
	}
	
	private final State state;
	private final Cleaner.Cleanable cleanable;

	public Room(int numJunkPiles) {
		state = new State(numJunkPiles);
		cleanable = cleaner.register(this, state);
	}
	
	@Override
	public void close() {
		cleanable.clean();
	}
}
```

- State가 Room을 참조해서는 안되는 이유는 순환참조 때문이다.
	- Room 객체가 State 클래스를 통해 참조되는 경우 State 객체가 Room 객체를 참조하고 있고, Room 객체는 State 객체를 참조하고 있을 때 가비지 컬렉터가 객체들을 회수할 수 없게 된다.
	- State 클래스가 정적 중첩 클래스인 이유이기도 하다.
		- static이 아닌 중첩 클래스는 자동으로 바깥 객체의 참조를 갖게 된다.
- State는 cleaner가 방을 청소할 때 수거할 자원을 담는다.
	- 내부의 run() 메서드는 cleanable에 의해 딱 한번 호출될 것이다.
- run() 메서드가 호출되는 상황은 둘 중 하나이다.
	- Room의 close 메서드를 호출할 때
		- cleanable의 clean()을 호출하면 이 메서드 내부에서 run()을 호출하게 된다.
	- 가비지 컬렉터가 Room을 회수할 때 까지 close를 호출하지 않는 경우 cleaner가 run() 메서드 호출

이렇게 구현된 Cleaner는 오로지 안전망으로만 쓰였다. 클라이언트가 모든 Room의 생성을 try-with-resources 블록으로 감쌌다면 자동 청소는 필요하지 않다.

```java
try (Room myRoom = new Room(7)) {
	System.out.println("방 + 쓰레기");
}
```

위 코드를 실행하면 정상적으로 방을 생성한 후  '방 청소'를 출력하게 될 것이다.

```java
new Room(77);
System.out.println("방 + 쓰레기");
```

방을 생성하지만 '방 청소'가 출력되지 않는 경우가 생긴다. Cleaner가 동작되는 것에 대한 보장이 없기 때문이다.

## 정리

finalizer나 cleaner는 성능문제, 불확실성 (순서나 실행보장) 등의 문제로 사용하지 않는 것을 권장한다.

대신 close()를 사용하고 두 가지의 소멸자는 close()에 대한 안전망 역할이나 중요하지 않은 네이티브 자원 회수용으로 사용하는 것이 좋다. (물론 이 경우도 성능이나 불확실성에 주의해야 한다.)