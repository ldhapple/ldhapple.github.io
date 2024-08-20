---
title: Item80 (스레드보다는 실행자, 태스크, 스트림을 애용하라) + Callable, Runnable, Executor
author: leedohyun
date: 2024-08-18 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 스레드보다는 실행자, 태스크, 스트림을 애용하라

이전 아이템에서 동기화에 대해 다루었다. 필요하면서도, 제대로 구현하기 위해서는 외계인 메서드 등을 고려하며 안전 실패나 응답 불가 문제를 없애야 한다.

이렇게 까다로운 동시성 프로그래밍에 도움이 되는 패키지가 등장했다.

java.util.concurrent 패키지이다. 이 패키지는 실행자 프레임워크 (Executor Framework)라는 인터페이스 기반의 유연한 태스크 실행 기능을 담고 있다.

Java의 멀티쓰레딩과 동시성을 다루는 클래스와 인터페이스들에 대해 정리해보자.

### Runnable

- Runnable은 Java에서 쓰레드를 실행할 수 있는 작업을 정의하는 가장 기본적인 인터페이스이다.
	- 실행할 코드 블록을 포함하는 객체를 만드는 데 사용된다.

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

```java
class RunnableExample implements Runnable {
	@Override
	public void run() {
		System.out.println("Runnable");
	}
}

RunnableExample runnable = new RunnableExample();
Thread thread = new Thread(runnable);
thread.start(); //별도의 쓰레드에서 run() 메서드 수행
```

- Runnable 인터페이스는 반환 값이 없는 단순 작업을 정의할 때 사용된다.
- run() 메서드 하나만을 가지고 있어, 이 메서드에 실행할 코드를 작성한다.
- Runnable을 구현한 클래스를 쓰레드에 전달해 병렬로 작업을 수행할 수 있다.

그러나 Runnable은 결과를 반환할 수 없다는 한계가 있다. 

반환값을 얻기 위해서는 공용 메모리나 파이프 등을 사용해야 하는데, 이 작업은 매우 번거롭다.

이에 Java 5 이후 제네릭을 사용해 결과를 받을 수 있는 Callable 인터페이스가 추가되었다.

### Callable

- Runnable과 유사하게 쓰레드에서 실행할 작업을 정의하는 인터페이스이다.
	- 그러나 **반환 값이 있고 예외를 던질 수 있다**는 점에서 Runnable과 다르다.

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
```java
class CallableExample implements Callable<Integer> {
	@Override
	public Integer call() throws Exception {
		System.out.println("callable");
		return 1;
	}
}
```

- Callable 인터페이스는 제네릭 타입을 사용해 반환할 결과의 타입을 정의한다.
- Callable은 Future 객체와 함께 사용되어 작업의 완료 상태를 비동기적으로 처리할 수 있다.

### Future

- Callable의 구현체는 작업이다.
- 이 작업은 가용 가능한 쓰레드가 없어 실행이 미루어질 수 있고 작업 시간이 오래 걸릴 수도 있다.
- 따라서 실행 결과를 바로 받지 못하고 미래의 어느 시점에 얻을 수 있는데, 미래에 완료된 Callable의 반환값을 구하기 위해 사용되는 것이 Future이다.
- 정리하면 Future은 비동기 작업의 결과를 나타내는 인터페이스로, 비동기 작업의 완료 여부를 확인, 결과를 가져오거나 작업을 취소할 수 있다.

Future의 메서드를 보자.

- get()
	- 비동기 작업이 완료될 때까지 대기한 후 결과를 반환한다.
- get(long timeout, TimeUnit unit)
	- 지정된 시간 동안만 대기하고, 작업이 완료되면 결과를 반환한다.
- cancel(boolean mayInterruptIfRunning)
	- 작업을 취소한다.
- isDone()
	- 작업이 완료되었는지 확인한다.
- isCancelled()
	- 작업이 취소되었는지 확인한다.

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Callable<Integer> task = new CallableExample();
Future<Integer> future = executor.submit(task);

if (future.isDone()) {
	System.out.println("Task 완료");
} else {
	System.out.println("Task 진행중");
}

Integer result = future.get(); //비동기 작업이 완료될 때 까지 기다림.

executor.shutdown();
```

### Executor

- 비동기 작업을 실행하기 위한 기본적인 인터페이스로 이 인터페이스는 작업을 실행하는 메커니즘을 추상화한다.
- 사용자가 직접 쓰레드를 생성하고 관리할 필요없이 작업을 실행할 수 있도록 해준다.
- 쓰레드 풀은 Executor의 한 가지 구현 방식으로 볼 수 있다.

```java
public interface Executor {  
    void execute(Runnable command);  
}
```

- 복잡한 쓰레드 관리를 추상화해 개발자가 비동기 작업을 간단하게 실행할 수 있도록 해준다.

### ExecutorService

- Executor 인터페이스를 확장한 것으로 비동기 작업을 관리하고 제어하는 기능들을 제공한다.
- 작업 제출, 작업 취소, 작업 완료 여부 확인 등의 작업 관리 기능을 제공한다.

ExecutorService의 주요 메서드에는 아래와 같은 것들이 있다.

- submit(Runnable task)
	- Runnable 작업을 제출한다.
- submit(Callable task)
	- Callable 작업을 제출하고 Future를 반환한다.
- invokeAll(Collection ? extends Callable tasks)
	- 모든 Callable 작업이 완료될 때까지 기다리며 결과를 반환한다.
	- 최대 쓰레드 풀 크기만큼 작업을 동시에 실행한다.
 - shutdown()
	 - 더 이상 새로운 작업을 받지 않고, 이미 제출된 작업이 완료되면 Executor를 종료한다.
	 - 이 메서드를 호출하기 전까지 계속해서 다음 작업을 대기하므로 작업이 완료되었다면 명시적으로 호출해주어야 한다.
- shutdownNow()
	- 실행중인 작업을 모두 중단하고 즉시 Executor를 종료한다. 

```java
ExecutorService executor = Executors.newFixedThreadPool(3);

Callable<Integer> task = new CallableExample();
Future<Integer> future = executor.submit(task);

Integer result = future.get();

executor.shutdown();
```

### Executors

- 정적 메서드를 통해 다양한 유형의 ExecutorService 인스턴스를 생성할 수 있도록 해주는 유틸리티 클래스이다.
- 주요 역할은 다양한 쓰레드 풀을 생성하고 관리하는 것이다.
- newFixedThreadPool(int n)
	- 고정된 수의 쓰레드를 가진 쓰레드 풀을 생성한다.
- newCachedThreadPool()
	- 필요한 만큼 쓰레드를 생성하고, 사용하지 않는 쓰레드를 캐싱하는 쓰레드 풀을 생성한다.
- newSingleThreadExecutor()
	- 단일 쓰레드를 사용하여 작업을 순차적으로 실행하는 ExecutorService를 생성한다.
- newScheduledThreadPool(int corePoolSize)
	- 일정 시간 간격 혹은 특정 시점에 작업을 실행할 수 있는 쓰레드 풀을 생성한다.  

```java
ExecutorService executor = Executors.newFixedThreadPool(3);
```

### ExecutorService를 사용하기 까다로운 경우

작은 프로그램이나 가벼운 서버라면 Executors.newCachedThreadPool이 일반적으로 좋은 선택이다.

특별하게 설정할 것이 없고, 일반적인 용도에 저합하게 동작한다.

하지만 CachedThreadPool은 무거운 프로덕션 서버에는 알맞지 않다. 요청받은 작업들이 큐에 쌓이지 않고 즉시 쓰레드에 위임되어 실행된다.

가용한 쓰레드가 없다면 새로 하나를 새성하며 만약 서버가 아주 무거울 경우 CPU 이용률이 100%로 치닫고 새로운 작업이 도착하는 족족 또 다른 쓰레드를 생성하며 상황을 악화시킨다.

따라서 이런 경우 newFixedThreadPool을 선택해 쓰레드 개수를 고정하거나 완전히 통제할 수 있는 ThreadPoolExecutor를 직접 사용하는 편이 낫다.

### 쓰레드를 직접 다루지 말자.

작업 큐를 손수 만드는 일은 삼가고, 쓰레드를 직접 다루는 것도 일반적으로는 삼가야 한다.

쓰레드를 직접 다루면 Thread가 작업 단위와 수행 메커니즘 역할을 모두 수행하게 된다.

반면 Executor 프레임워크에서는 작업 단위와 실행 매커니즘이 분리된다.

작업 단위를 나타내는 추상 개념이 Task(Runnable, Callable)이다.

그리고 이 Task를 실행하는 일반적인 메커니즘이 ExecutorService인 것이다.

태스크 수행을 실행자 서비스에 맡기면 원하는 태스크 수행 정책을 선택 가능하고, 생각이 바뀌었을 때 변경 또한 쉽다.

## 정리

쓰레드를 직접 다루는 것 보다는 이를 도와주는 패키지를 사용하는 것이 좋다.

쓰레드를 직접 다루었을 경우 쓰레드에 모든 책임이 있어 다루기 까다롭다. 반면 ExecutorService를 사용할 경우 작업의 생성, 등록, 실행 등의 모든 작업이 수월해진다.