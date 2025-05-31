---
title: ReentrantLock
author: leedohyun
date: 2025-05-25 12:13:00 -0500
categories: [JAVA, 정리]
tags: [Java]
---

멀티스레드 환경에서는 공유 자원에 대해 경쟁 상태를 관리해주어야 한다. 대표적인 방법으로 Java에서 synchronized 키워드를 예시로 들 수 있다. 관련해서 찾아보다가 ReentrantLock 이라는 것을 알게 되었고 정리해보려고 한다.

## ReentrantLock

java.util.concurrent.locks 패키지에 포함된 락 클래스이다.

Re-entrant 단어와 같이 재진입 가능한 락으로 한 스레드가 이미 획득한 락을 중복해서 획득할 수 있다. 같은 스레드가 락을 여러 번 걸어도 블로킹되지 않고 대신 락을 건 횟수 만큼 unlock()도 여러 번 호출해야 락이 완전히 해제될 수 있다.

> 락을 여러 번 걸게되는 이유?

락을 여러 번 거는 상황은 개발자가 의도적으로 여러 번 거는 경우보다, 내부적으로 중첩 호출이 발생했을 때 자연스럽게 발생할 수 있다.

만약 재진입이 불가능하게 설계된다면 쉽게 데드락이 생길 수 있기 때문에 쓰이지 않게 된다.

```java
private final ReentrantLock lock = new ReentrantLock();

public void transfer() {
	lock.lock();
	try {
		withdraw();
	} finally {
		lock.unlock();
	}
}

public void withdraw() {
	lock.lock(); // 같은 락 다시 획득
	try {
		//출금 로직
	} finally {
		lock.unlock();
	}
}
```

### 내부 동작 원리

ReentrantLock은 내부적으로 AbstractQueuedSynchronizer를 기반으로 동작한다.

락을 시도한 스레드는 AQS의 큐에 들어가고, 선착순 또는 비공정하게 락을 획득하게 된다. 재진입 횟수는 동일 스레드가 몇 번 lock을 호출했는지 카운트하여 관리하게 된다.

> AQS??

AQS는 스레드 간 동기화를 위한 공통 로직을 제공하는 추상 클래스이다. 내부에 FIFO 큐를 두고 스레드들이 락을 기다리게 한다.

락을 획득하려는 스레드는 CAS(Compare And Swap)로 락을 획득하게 되고 만약 실패한다면 AQS의 대기 큐에 추가된다.

락을 보유한 스레드가 unlock()을 호출하면 다음 노드가 깨어나 락 획득을 시도한다.

만약 동일 스레드가 다시 lock()을 호출하는 경우 AQS의 state 값을 증가시켜 재진입 횟수를 누적 관리하는 방식으로, unlock()이 호출될 때 마다 state를 감소시켜 0이 되면 실제 락이 해제 처리되는 방식이다.

### synchronized와의 차이

synchronzied는 자바에서 가장 기본적인 락 개념이다. 더 세밀하고 유연한 동기화 제어가 필요할 때 ReentrantLock을 사용하게 된다.

우선 synchronized는 키워드를 사용해 선언하고 블록을 만들 수 있다. 반면 ReentrantLock은 객체를 생성하고 명시적으로 사용할 수 있다.

| 구분 | synchronized  | ReentrantLock |
|--|--|--|
| 재진입 가능 여부 | O | O |
| 락 해제 방식 | 자동 해제| 명시적 해제 (unlock() 호출해야 함)|
|공정성 설정|불가능, 기본적으로 비공정|가능 (new ReentrantLock(true))|
|타임아웃 제어|불가능|가능 (tryLock(long time, TimeUnit unit)|
|인터럽트 대응|불가능|가능 (lockInterruptibly())|
|조건 변수 지원|wait(), notify()|Condition 객체로 다중 조건 지원|

- 단순한 동기화 블록이 필요하거나, 실수로 락 해제를 빠뜨리는게 걱정된다면 synchronized를 사용하는 것이 더 낫다.
- 반면 ReentrantLock을 사용하는 경우는 아래와 같은 경우들이 있을 수 있다.
	- 락 획득 시도 실패 후 다른 처리 로직을 넣고 싶다. (tryLock())
	- 락을 오래 기다린다면 interrupt로 중단하고 싶다. (lockInterruptibly())
	- 여러 조건으로 나누어 wait/notify를 하고 싶다.

unlock()으로 명시적인 해제가 필수이기 때문에 항상 try-finally로 감싸는 것이 안전하다.

> 공정성, 비공정성

공정성은 락을 획득할 때 요청 순서대로 보장하는 것이다. 예측이 가능하고 특정 스레드가 Starvation에 빠질 일이 없다는 것이 장점이지만 컨텍스트 스위칭이 많아져 성능 저하 가능성이 생긴다는 단점이 있다.

비공정성은 요청 순서를 무시해 락 획득 기회가 더 자주 생겨 성능이 좋을 수 있다. 하지만 오래 기다린 스레드가 계속 밀릴 수 있어 Starvation이 발생할 수 있다.

공정 락은 실시간성과 응답 보장이 중요한 시스템 (은행 거래 처리 순서)에서 사용되고, 비공정 락은 지연이 적고 고성능이 우선일 때 사용하면 된다.

> ReentrantLock을 사용하는 경우

특정 유저의 동시 요청들을 하나만 성공시키고, 나머진 실패시켜야 하는 경우를 생각해보자.

만약 synchronized 키워드를 사용한다면 특정 메서드나 객체에 고정적으로 락을 걸게 된다. 이렇게 되면 모든 유저의 발급 요청이 하나의 락에 의해 직렬 처리되고 병목이 생긴다. 특정 유저의 요청들만 순차처리하면 되는데, 모든 유저의 요청들이 순차처리 되는 것이다.

반면 ReentrantLock을 사용하게 된다면 Map을 만들어 user별로 락을 분리해 처리할 수 있다.

```java
public void doSomething(Long userId) {
	String key = "key" + userId;
	ReentrantLock lock = lockMap.computeIfAbsent(key, k -> new ReentrantLock());
	lock.lock();

	try {
		// do...
	} finally {
		lock.unlock();
	}
}
```

정리하자면 동일 유저의 요청은 직렬화하면서도, 전체 시스템으로 봤을 땐 병렬성을 유지하기 위함이라고 볼 수 있다.

** 대신 lockMap과 같이 저렇게 관리하는 경우, 사용자 수가 많을 경우 무한히 커지므로 일정 시간 사용하지 않은 락을 제거하는 관리도 필요하다.

### ReentrantLock 메서드

|메서드  |설명  |
|--|--|
|lock()  |락을 무조건 획득할 때 사용된다. 락이 해제될 때까지 대기한다.|
|unlock()|락을 해제한다.|
|trylock()|락을 획득할 수 있다면 true, 아니면 false를 반환한다. 락 대기를 중단하고 다른 로직을 수행할 수 있다.|
|tryLock(long time, TimeUnit unit)|지정 시간까지 락 획득을 시도하고 실패하면 false를 반환한다.|
|isLocked()|현재 락이 걸려 있는지 여부를 확인한다. 디버깅/로깅 용도로 활용된다.|
|isHeldByCurrentThread()|현재 스레드가 락을 보유 중인지 확인한다. 안전하게 unlock하기 위해 사용된다.|
|getHoldCount()|현재 스레드가 락을 획득한 횟수를 반환한다.|


### ReentrantLock이 사용되는 곳

- ConcurrentLinkedQueue 등의 내부 구현에서 부분적으로 사용된다.
	- JDK의 고수준 동시성 도구는 내부적으로 AQS -> ReentrantLock 기반의 구조.
- Spring의 스케줄러, 캐시, 트랜잭션 커밋 등에 사용된다.
	- 트랜잭션 커밋 처리 시 내부적으로 커밋 상태 관리에 ReentrantLock을 활용하기도 한다. (구현체에 따라 다르다.)
	- Redisson Redis 분산락 라이브러리도 ReentrantLock과 유사한 구조이다.