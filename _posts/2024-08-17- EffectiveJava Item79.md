---
title: Item79 (과도한 동기화는 피하라)
author: leedohyun
date: 2024-08-17 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 과도한 동기화는 피하라

이전 아이템에서 동기화의 중요성을 알아보았지만, 동기화가 과도하면 오히려 역효과가 날 수 있다.

성능을 떨어뜨리고, 교착 상태에 빠뜨리거나 예측 불가능한 동작을 낳기도 한다.

### 동기화 메서드나 동기화 블록 안에서 제어를 클라이언트에 맡기면 안된다.

응답 불가와 안전 실패를 피하려면 동기화 메서드나 동기화 블록 안에서의 제어를 클라이언트에 양도하면 안된다.

예를 들면 동기화 된 영역 안에서는 재정의할 수 있는 메서드는 호출하면 안된다. 그리고 클라이언트가 넘겨준 함수 객체를 호출해서도 안된다.

동기화된 영역을 포함한 클래스의 관점에서 보면 이러한 메서드들은 외계인이다. 그 메서드가 무슨 일을 할지 알 수 없고 통제할 수도 없다.

```java
public class ObservableSet<E> extends ForwardingSet<E> {
	public ObservableSet(Set<E> set) {
		super(set);
	}

	private final List<SetObserver<E>> observers = new ArrayList<>();

	public void addObserver(SetObserver<E> observer) {
		synchronized(observers) {
			observers.add(observer);
		}
	}

	public boolean removeObserver(SetObserver<E> observers) {
		synchronized(observers) {
			return observers.remove(observer);
		}
	}

	private void notifyElementAdded(E element) {
		synchronized(observers) {
			for (SetObserver<E> observer : observers) {
				observer.added(this, element);
			}
		}
	}

	@Override
	public boolean add(E element) {
		boolean added = super.add(element);
		if (added) {
			notifyElementAdded(element);
		}
		return added;
	}

	@Override
	public boolean addAll(Collection<? extends E> c) {
		boolean result = false;
		for (E element : c) {
			result |= add(element);
		}
		return result;
	}
}
```

- Set을 감싼 래퍼 클래스이다.
- 클라이언트는 Set에 원소가 추가되면 알림을 받을 수 있는 구조이다.
- SetObserver는 BiConsumer과 구조가 같은 커스텀 함수형 인터페이스이다.

```java
ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

set.addObserver((s, e) -> System.out.println(e));

for (int i = 0; i < 100; i++) {
	set.add(i);
}
```

- 실제로 프로그램은 잘 동작할 것 같고, 위 코드도 0부터 99까지를 출력하게 된다.

```java
set.addObserver(new SetObserver<>() {
	public void added(ObservableSet<Integer> s, Integer e) {
		System.out.println(e);
		if (e == 23) {
			s.removeObserver(this);
		}
	}
});

for (int i = 0; i < 100; i++) {
	set.add(i);
}
```

- removeObserver 메서드에 함수 객체 자신을 전달해야 하는데, 람다는 자신을 참조할 수단이 없기 때문에 익명 클래스를 사용한다.
- 똑같이 Set에 추가된 정수값 원소들을 출력하다가 그 값이 23일 경우 자기 자신을 구독해지(관찰자 제거)하는 코드이다.
- 결과가 이상하다.
	- 0부터 23까지 출력한 후 구독 해지를 하고 종료가 되어야 할 것 같다.
	- 하지만 **ConcurrentModificationException을 던진다.**
	- Observer의 added 메서드 호출이 일어난 시점이 notifyElementAdded가 Observer들의 리스트를 순회하는 도중이기 때문이다.
	- added 메서드는 removeObserver 메서드를 호출하고, 이 메서드는 다시 remove 메서드를 호출한다.
	- 이 과정에서 리스트에서는 원소를 제거하려는데, 리스트를 순회하는 도중이기 때문에 발생하는 예외인 것이다.
		- 즉, **notifyElementAdded()는 옵저버 리스트를 여전히 순회하고 있는데, added()의 remove 메서드가 리스트에서 요소(옵저버)를 제거하려고 하기 때문인 것이다.**

notifyElementAdded() 메서드에서 수행하는 순회는 동기화 블록 안에 있다. 

동기화는 동시에 수정이 일어나지 않도록 보장한다.  하지만 ***자신이 콜백을 거쳐 되돌아와 수정하는 부분 까지는 막지 못하는 것***이 이 문제의 이유이다.

이번엔 removeObserver()를 다른 쓰레드에서 호출하도록 해보자.

```java
set.addObserver(new SetObserver<>() {
	public void added(ObservableSet<Integer> s, Integer e) {
		System.out.println(e);
		if (e == 23) {
			ExecutorService exec = Executors.newSingleThreadExecutor();
			try {
				exec.submit(() -> s.removeObserver(this)).get();
			} catch (ExecutionException | InterruptedException ex) {
				throw new AssertionError(ex);
			} finally {
				exec.shutdown();
			}
		}
	}
});
```

- 이 방식은 예외는 발생하지 않지만, **교착상태에 빠진다.**
- 백그라운드 쓰레드가 removeObserver()를 호출하면 Observer를 잠그려 시도한다. 하지만 락을 얻을 수 없다.
	- 메인 쓰레드가 이미 락을 쥐고 있다.
	- 그리고 그 메인 쓰레드는 백그라운드 쓰레드가 removeObserver()를 해내기를 기다리고 있다.

Observer가 자신을 해지하는데 백그라운드 쓰레드를 이용하는 예시가  억지스럽지만, 이 문제는 경우에 따라 충분히 발생 가능한 상황이다.

실제로 동기화된 영역 안에서 외계인 메서드를 호출해 교착상태에 빠지는 사례는 자주 있다고 한다.

> 만약 불변식이 깨진 경우라면?

위의 예시들과 같은 상황이지만, 여기에 불변식이 임시로 깨진 경우라면 문제는 더 크다.

Java 언어의 락은 재진입을 허용한다. ConcurrentModificationException을 던진 첫 예시를 보자.

외계인 메서드를 호출하는 쓰레드는 이미 락을 가지고 있어 그 락이 보호하는 데이터에 대해 개념적으로 관련 없는 다른 작업이 진행중에 있음에도 다음 락도 획득 가능하다. 

- added() 메서드가 호출 될 때, 그 메서드를 호출하는 쓰레드는 이미 observers 리스트에 대한 락을 획득하고 있다.
- added() 메서드가 호출되는 동안 이 메서드는 removeObserver()에 접근하는데, observers 리스트에 대한 락을 가지고 있어 접근이 가능하다.
- Java의 재진입은 하나의 쓰레드가 이미 특정 객체에 대한 락을 획득한 상태에서 다시 동일한 객체에 대한 락을 요청하면, 즉시 그 락을 획득할 수 있도록 한다.
- 따라서, 같은 observers 리스트에 대해 added()와 removeObserver()는 다른 개념의 동작이지만 둘 다 observers에 대한 락을 획득 가능하다는 뜻이다.

이러한 재진입 가능 락은 객체 지향 멀티쓰레드 프로그램을 쉽게 구현할 수 있도록 해주지만, 교착상태가 될 상황을 안전 실패(데이터 훼손)로 변모시킬 수 있다는 것이 문제이다.

이를 해결할 방법은 외계인 메서드 호출을 동기화 블록 바깥으로 옮기면 된다.

```java
private void notifyElementAdded(E element) {
	List<SetObserver<E>> snapshot = null;
	synchronized(observers) {
		snapshot = new ArrayList<>(observers);
	}

	for (SetObserver<E> observer : snapshot) {
		observer.added(this, element);
	}
}
```

- notifyElementAdded 메서드에서 observers 리스트를 복사해 사용하면 락 없이도 안전하게 순회 가능하다.
- 외계인 메서드를 동기화 블록 바깥으로 옮겨 예외 발생과 교착상태를 막을 수 있다.
- 동기화 블록 내에서 observers 리스트에 대한 동시 접근을 제어한다. 그리고 블록 내에서 복사본을 만들어 동기화 블록 바깥에서 복사본을 순회한다.
- observers를 순회하고 있는 도중 변경이 발생하는 것이 문제였는데, 복사본을 통해 안전하게 순회하고 외계인 메서드인 added()는 동기화 블록 바깥에서 호출되어 리스트의 수정 작업이 안전하게 이루어진다.
	- 원본 리스트가 순회 도중 변경되어도 복사본에는 영향이 없어 안전하다.

이보다 나은 방법은 Java의 동시성 컬렉션 라이브러리인 CopyOnWriteArrayList를 사용하는 것이다.

ArrayList를 구현한 클래스로 내부를 변경하는 작업이 항상 복사본을 만들어 수행하도록 구현되어 있다.

내부의 배열이 절대 수정되지 않아 순회 시 락이 필요 없어 매우 빠르다.

이 때문에 다른 용도로 사용되면 느릴 수 있지만 수정할 일이 드물고, 순회만 빈번하게 발생하는 관찰차 패턴의 리스트 용도로는 최적이다.

```java
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
	observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
	return observers.remove(observer);
}

private void notifyElementAdded(E element) {
	for (SetObserver<E> observer : observers) {
		observer.added(this, element);
	}
}
```

### 동기화 영역에서는 가능한 한 일을 적게 하라

외계인 메서드는 얼마나 오래 실행될 지 알 수 없는데, 동기화 영역 내에서 호출된다면 그동안 다른 쓰레드는 보호된 자원을 사용하지 못하고 대기해야 한다.

락을 얻고 공유 데이터를 검사하고 필요하면 수정하고 락을 놓는다.

오래 걸리는 작업이라면 가변 데이터임에도 동기화 영역 바깥으로 옮길 수 있는 방법을 찾아보아야 한다.

### 동기화의 비용

Java의 동기화 비용은 빠르게 낮아져 왔다. 하지만 과도한 동기화는 여러 비용이 든다.

- 락을 얻는데 드는 CPU 시간
- 경쟁하느라 낭비하는 시간
	- 병렬로 실행할 기회를 잃고 모든 코어가 메모리를 일관되게 보기 위한 지연 시간
- 가상머신의 코드 최적화를 제한하는 점

이 중 경쟁하느라 낭비하는 시간이 가장 중요하다.

따라서 가변 클래스를 작성할 때 동기화에 신경써야 한다. 두 선택지가 있다.

- 동기화를 전혀 하지 않고, 해당 클래스를 동시에 사용해야 하는 클래스가 외부에서 알아서 동기화하도록 한다.
- 동기화를 내부에서 수행해 쓰레드 안전한 클래스로 만든다.
	- 이 방법은 클라이언트가 외부에서 객체 전체에 락을 거는 것보다 동시성을 월등히 개선할 수 있을때만 선택한다.

java.util은 첫 번째 방식을 취했다. 그리고 java.util.concurrent는 두 번째 방식을 택했다.  

### 참고: 우리가 동기화를 거의 사용하지 않는 이유

Spring 프로젝트에서 synchronized와 같은 동기화 메커니즘을 사용한 것을 본 적이 없는 것 같다.

그 이유는 DB 트랜잭션, 그리고 Spring의 동시성 관리 도구때문이다.

- 우선 데이터베이스는 트랜잭션을 통해 ACID 특성을 보장한다.
	- 데이터베이스 레벨에서 작업의 원자성을 보장한다.
	- 여러 클라이언트가 동시에 동일한 데이터를 수정하려할 때 데이터 일관성을 유지한다.
	- 트랜잭션 격리 수준에 따라 작업을 순차적으로 처리해 동시성 문제를 방지한다.
	- 즉, 데이터베이스에서 직접 동시성을 관리하는 것이다.
- Spring은 Executor, @Async, TaskExecutor 같은 동시성 관리 도구를 제공한다. 
- 또 Sptring 애플리케이션은 보통 무상태로 설계된다.
	- 각 요청이 서로 독립적이고 공유 상태를 최소화한다.
	- 애플리케이션 상태를 서버 메모리가 아닌, 주로 DB나 외부 캐시에 저장해 동기화 문제를 피하는 것이다.

## 정리

교착상태와 데이터 훼손을 피하기 위해서는 동기화 영역 안에서 외계인 메서드를 호출해서는 안된다.

가변 클래스를 설계할 때 그 클래스 내에서 스스로 동기화해야할 지 고민하자.