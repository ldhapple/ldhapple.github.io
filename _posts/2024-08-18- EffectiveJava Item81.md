---
title: Item81 (wait와 notify보다는 동시성 유틸리티를 애용하라)
author: leedohyun
date: 2024-08-18 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## wait와 notify보다는 동시성 유틸리티를 애용하라

java.util.concurrent 패키지는 직전 아이템에서 다루었던 Executor 실행자 프레임워크, 동시성 컬렉션(Concurrent Collection), 동기화 장치(Synchronizer)의 세 범주로 나눌 수 있다.

이번 아이템에서는 동시성 컬렉션과 동기화 장치에 대해 다루어본다.

우선 이번 아이템의 주제인 wait와 notify에 대해 알아보고 왜 동시성 유틸리티를 사용해야 되는 지 알아보자.

### wait, notify

wait(), notify() 메서드는 동기화된 메서드나 블록 내에서 사용되는 메서드들이다.

쓰레드 간의 통신을 가능하게 하는 도구이지만 사용하는 것이 어렵고 올바르게 사용하지 않으면 교착 상태나 응답 불가 같은 문제가 발생할 수 있다.

- wait()
	- 호출된 쓰레드를 일시적으로 정지시킨다.
	- 이 때 쓰레드는 동기화 락을 놓고 다른 쓰레드가 해당 락을 사용할 수 있도록 한다.
	- wait() 메서드를 호출한 쓰레드는 notify() 또는 notifyAll() 메서드가 호출될 때 까지 대기 상태에 머문다.
	- wait(long timeout)을 이용하면 지정된 시간 동안 대기하다가 자동으로 깨어나게 할 수 있다.
- notify()
	- 대기중인 쓰레드 중 하나를 깨운다.
	- notify() 메서드를 호출하더라도 바로 깨어난 쓰레드가 실행되지는 않는다. 깨어난 쓰레드는 락이 해제될 때 까지 대기하다가 락을 획득하면 실행을 재개한다.
- notifyAll()
	- 모든 대기 중인 쓰레드를 깨운다.
	- 여러 쓰레드가 동일한 조건을 기다리고 있을 때 유용하다. 한 번에 하나의 쓰레드만 락을 획득할 수 있어 깨어난 쓰레드들은 경쟁을 통해 락을 획득한다.

```java
class Example {
	private boolean isAvailable = false;

	public synchronized void produce() throws InterruptException {
		while (!isAvailable) {
			wait(); // 자원이 사용중이라면 대기한다.
		}
		
		//...
		isAvailable = true;
		notify();
	}
}
```

> 한계

- 교착 상태
	- wait()와 notify()를 잘못 이용할 경우 교착상태에 빠질 수 있다.
- 응답 불가
	- wait()를 호출한 쓰레드가 다시 깨워지지 않으면 해당 쓰레드는 영원히 대기 상태에 머물 수 있다.

추가적으로 wait()와 notify()를 올바르게 사용하기란 어렵다. 프로그램의 복잡성을 증가시킨다. 만약 다수의 쓰레드가 복잡한 조건에서 상호작용 한다고 가정한다면 난이도는 더더욱 올라갈 것이다.

이러한 wait와 notify의 사용의 어려움으로 java.util.concurrent 패키지의 고수준 동시성 유틸리티를 사용하는 것이 권장된다.

### 동시성 컬렉션

List, Queue, Map과 같은 표준 컬렉션 인터페이스에 동시성을 가미해 구현한 고성능 컬렉션이다.

높은 동시성에 도달하기 위해 동기화를 각자의 내부에서 수행한다. ***따라서 동시성 컬렉션에서 동시성을 무력화하는 것은 불가능하며, 외부에서 락을 추가로 사용하면 오히려 속도가 느려진다.***

동시성 컬렉션에서 동시성을 무력화하지 못하므로 여러 메서드를 원자적으로 묶어 호출하는 일 역시 불가능하다.

예시를 보자.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("key", 1);

Thread thread1 = new Thread(() -> {
	if (map.containsKey("key")) {
		map.put("key", map.get("key") + 1);
	}
});

Thread thread2 = new Thread(() -> {
	if (map.containsKey("key")) {
		map.put("key", map.get("key") + 1);
	}
});

thread1.start();
thread2.start();
```

- ConcurrentHashMap의 각 메서드인 containsKey(), put(), get()은 개별적으로는 동시성을 보장한다.
- 하지만 여러 메서드를 조합한 작업에서 문제가 발생할 수 있다.
	- map.put("key", map.get("key") + 1); 같은 경우 get()과 put()이 조합되어 있는 상태이다.
	- 함께 조합해서 사용하면 원자성이 보장되지 않아 동시성 문제가 발생할 수 있다.

이렇나 이유로 여러 기본 동작을 하나의 원자적 동작으로 묶는 상태 의존적 수정 메서드들이 추가되었다.

이 메서드들은 유용해 Java 8부터는 일반 컬렉션 인터페이스에도 디폴트 메서드 형태로 추가되었다.

putIfAbsent(key, value) 같은 메서드가 그 예시이다.

### 동기화 장치

동기화 장치는 쓰레드가 다른 쓰레드를 기다릴 수 있도록 해 서로 작업을 조율할 수 있도록 해준다.

가장 자주 사용되는 동기화 장치는 CountDownLatch와 Semaphore다.

CountDownLatch는 일회성 장벽으로 하나 이상의 쓰레드가 또 다른 하나 이상의 쓰레드 작업이 끝날 때 까지 기다리게 한다. 유일한 생성자는 int값을 받으며 이 값이 래치의 countDown 메서드를 몇 번 호출해야 대기 중인 쓰레드들을 깨우는지 결정한다.

설명만 들어도 wait()와 notify()를 직접 사용하는 것 보다 같은 기능을 훨씬 쉽게 구현할 수 있다는 것을 알 수 있다.

코드로 확인해보자. 어떤 동작들을 동시에 시작해 모두 완료하기까지의 시간을 재는 프레임워크를 구축하는 예시이다.

```java
public static long time(Executor executor, int concurrency,
			Runnable action) throws InterruptedException {
			
	CountDownLatch ready = new CountDownLatch(concurrency);
	CountDownLatch start = new CountDownLatch(1);
	CountDownLatch done = new CountDownLatch(concurrency);
	
	for (int i = 0; i < concurrency; i++) {
		executor.execute(() -> {
			ready.countDown(); // 타이머에게 준비 완료를 알림
			try {
				start.await(); // 모든 작업자 쓰레드가 준비될 때 까지 기다림.
				action.run();
			} catch (InterruptedException e) {
				Thread.currentThread().interrupt();
			} finally {
				done.countDown(); // 타이머에게 작업 완료를 알림.
			}
		});
	}
	
	ready.await(); // 모든 작업자가 준비될 때까지 기다림.
	long startNanos = System.nanoTime();
	start.countDown(); // 작업자들을 깨움.
	done.await(); // 모든 작업자가 작업을 마칠때까지 기다림.
	return System.nanoTime() - startNanos;
}
```

- 메서드 하나로 구성되고, 동작들을 실행할 실행자와 동작을 몇 개나 동시에 수행할 수 있는 지를 뜻하는 concurrency를 매개변수로 받는다.
- 타이머 쓰레드가 시계를 시작하기 전 모든 작업자 쓰레드는 동작 수행 준비를 마친다.
- 마지막 작업자 쓰레드가 준비를 마치면 타이머 쓰레드가 시작 방아쇠를 당겨 작업자 쓰레드들이 일을 시작하도록 한다.
- 마지막 작업자 쓰레드가 동작을 마치자마자 타이머 쓰레드는 시계를 멈춘다.

### wait(), notify()를 어쩔 수 없이 사용해야 하는 경우

새로운 코드라면 동시성 유틸리티를 사용하는 것이 무조건 좋다.

하지만 레거시 코드를 다루어야 하는 경우 wait()와 notify()가 포함되어 있을 수 있는데 이 때의 주의할 점을 알아보자.

- 락 객체의 wait() 메서드는 반드시 그 객체를 잠근 동기화 영역 내에서 호출해야 한다.

```java
synchronized (obj) {
	while (<조건이 충족되지 않았음>) {
		obj.wait(); // 락을 놓고, 깨어나면 다시 잡는다.
		//...
	}

	// 조건이 충족됐을 때의 동작...
}
```

위 방식이 wait() 메서드를 사용하는 표준 방식이다.

**wait() 메서드를 사용할 때는 반드시 대기 반복문 관용구를 사용해야 한다. 반복문 밖에서는 절대 호출해서는 안된다.**

**대기 전에 조건을 검사해 조건이 이미 충족되었다면 wait를 건너뛰게 한다. 이는 응답 불가 상태를 예방하는 조치이다.**

만약 조건이 충족되었는데 쓰레드가 notify() 메서드를 호출한 후 대기상태로 빠진다면 쓰레드를 다시 깨우는 것을 보장할 수 없다.

 **또 대기 후에 조건을 검사해 조건이 충족되지 않았다면 다시 대기하도록 하는 것은 안전 실패를 막는 조치이다.**

조건이 충족되지 않았는데 쓰레드가 동작을 이어간다면 락이 보호하는 불변식을 깨뜨릴 수 있다.

조건이 만족되지 않아도 쓰레드가 깨어날 수 있는 상황이 몇 가지 있다.

- 쓰레드가 notify를 호출한 후 대기 중이던 쓰레드가 깨어나는 사이에 다른 쓰레드가 락을 얻어 그 락이 보호하는 상태를 변경한다.
- 조건이 만족되지 않았음에도 다른 쓰레드가 실수로 혹은 악의적으로 notify를 호출한다.
- 깨우는 쓰레드는 지나치게 관대해 대기 중인 쓰레드 중 일부만 조건이 충족되어도 notifyAll()을 호출해 모든 쓰레드를 깨울 수도 있다.
- 대기 중인 쓰레드가 드물게 notify 없이도 깨어나는 경우도 있다.
	- 허위 각성이라는 현상이라고 한다.