---
title: Item78 (공유 중인 가변 데이터는 동기화해 사용하라)
author: leedohyun
date: 2024-08-15 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

이번 아이템부터는 동시성에 대해 다룬다.

쓰레드는 여러 활동을 동시에 수행할 수 있도록 해주는데 이를 위한 동시성 프로그래밍은 단일 쓰레드 프로그래밍보다 어렵다.

잘못될 수 있는 일들이 늘어나고 무엇보다 **문제를 재현해내기가 어렵다.** 

그렇다고 언제까지 피할 수 없다. 내 것으로 만들어야 한다. 동시성 프로그램을 명확하고, 정확하게 만들고 잘 문서화하는 방법들을 알아보자.

## 공유 중인 가변 데이터는 동기화해 사용하라

synchronized 키워드는 해당 메서드나 블록을 한 번에 한 쓰레드씩 수행하도록 보장한다.

많은 프로그래머가 동기화를 **배타적 실행**, 즉 한 쓰레드가 변경하는 중이라서 상태가 일관되지 않은 순간의 객체를 다른 쓰레드가 보지 못하도록 막는 용도로만 생각한다.

```java
public class Counter {
	private int count = 0;

	public synchronized void increment() {
		count++;
	}
	
	public synchronized int getCount() {
		return count;
	}
}
```
```java
Counter counter = new Counter();

Thread t1 = new Thread(() -> {
	for (int i = 0; i < 1000; i++) {
		counter.increment();
	}
});

Thread t2 = new Thread(() -> {
	for (int i = 0; i < 1000; i++) {
		counter.increment();
	}
});

t1.start();
t2.start();

try {
	//메인 쓰레드가 각각의 두 쓰레드가 종료될 때 까지 대기
	//두 쓰레드 종료 후 count 출력 위함
	t1.join();
	t2.join();
} catch (InterruptedException e) {
	e.printStackTrace();
}

System.out.println(counter.getCount());
```

- 위 코드를 synchronized 키워드 없이 실행하면 count 값은 1819가 나오고, 위 코드 그대로 수행하면 2000의 값이 나온다.
	- 값을 증가시킬 때 동시에 같은 값을 읽고 하나의 증가만 반영된 경우가 존재하기 때문이다.

한 객체가 일관된 상태를 가지고 생성되고, 이 객체에 접근하는 메서드는 그 객체에 락을 건다.

락을 건 메서드는 객체의 상태를 확인하고 필요할 경우 수정한다. 즉, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화시킨다.

동기화를 제대로 사용하면 어떤 메서드도 이 객체의 상태가 일관되지 않은 순간을 볼 수 없다.

따라서 synchronized 키워드를 통한 동기화는 배타적 실행의 용도가 맞는 것처럼 보인다.

틀린말은 아니지만 동기화에는 중요한 기능이 더 있다.

### 동기화는 쓰레드 사이의 안정적인 통신에 필요하다.

동기화 없이는 한 쓰레드가 만든 변화를 다른 쓰레드에서 확인하지 못할 수 있다.

동기화는 일관성이 깨진 상태를 볼 수 없게 해주는 것은 물론, 동기화된 메서드나 블록에 들어간 쓰레드가 같은 락의 보호하에 수행된 모든 이전 수정의 최종 결과를 보게 해준다.

```java
private static boolean stop = false;

public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		int i = 0;
		while (!stop) {
			i++;
		}
		System.out.println("t1 쓰레드가 멈췄습니다. 최종값은 " + i);
	});

	t1.start();

	Thread.sleep(1000);

	stop = true;
	System.out.println("stop = true");

	t1.join();
	System.out.println("main 종료");
}
```

![](https://blog.kakaocdn.net/dn/b9ksa2/btsI7mdRgWE/unzXZ2XKvkixh85T8y53Vk/img.png)

- 위 코드는 stop = true를 출력한다.
- 그러나 t1 쓰레드는 stop의 변경을 즉시 감지하지 못해 실행을 멈추지 않는다.
	- main 종료 문구는 t1 쓰레드가 완료되지 않아 출력되지 않는다.
	- t1 쓰레드는 stop이 여전히 false인 것으로 간주하고 무한루프를 돈다.
	- main 쓰레드에서 값을 변경했음에도 t1 쓰레드에서는 그것을 인지하지 못한 것이다.

동기화하지 않아 메인 쓰레드가 수정한 값을 백그라운드 쓰레드(t1)가 언제쯤에 보게될 지 보장할 수 없다.

#### 동기화가 없는 경우

동기화가 없다면 VM에서는 아래와 같이 최적화할 수 있다.

```java
while (!stop) {
	i++;
}
```
```java
if (!stop) {
	while (true) {
		i++;
	}
}
```

OpenJDK 서버 VM이 실제로 적용하는 끌어올리기(hoisting) 라는 최적화 기법이다.

동기화 없이 stop 변수를 사용하고, 컴파일러가 이를 반복문 밖으로 끌어올리면 stop의 변경이 반영되지 않기 때문에 t1 쓰레드는 무한 루프에 빠진다.

이를 최적화라고 부르는 이유는 우선 최적화는 프로그램의 실행 속도를 높이는 것이 목표이다.

그런데 최적화는 특정 상황에서만 유효하다. 동기화가 없는 상태에서 stop 변수에 대해 컴파일러의 최적화는 단일 쓰레드 환경에서는 유효하지만, 다중 쓰레드 환경에서는 예기치 않은 동작을 일으킬 수 있는 것이다.

즉, 최적화가 반드시 좋은 결과를 보장하는 것이 아님을 알아야 한다.

#### 해결방법 1

```java
private static boolean stop = false;

private static synchronized void requestStop() {
	stop = true;
}

private static synchronized boolean stop() {
	return stop;
}

public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		int i = 0;
		while (!stop()) {
			i++;
		}
		System.out.println("t1 쓰레드가 멈췄습니다. 최종값은 " + i);
	});

	t1.start();

	Thread.sleep(1000);

	requestStop();
	System.out.println("stop = true");

	t1.join();
	System.out.println("main 종료");
}
```

![](https://blog.kakaocdn.net/dn/bemyWE/btsI5P2IU98/SNn8r8Csgtrhf6U5g176WK/img.png)

- 쓰기 메서드와 읽기 메서드 모두를 동기화해야 한다.
	- 쓰기 메서드만 동기화해서는 충분하지 않다.
	- 쓰기와 읽기 모두 동기화되지 않으면 동작이 보장되지 않는다.
- 기대한대로 문구를 출력하고 종료한다.
- 문제가 되었던 stop 필드를 동기화해 접근한 결과이다.

#### 해결방법 2

반복문에서 매번 동기화하는 비용이 크지는 않지만 속도가 더 빠른 대안이 있다.

volatile 키워드를 사용하는 것이다.

```java
private static volatile boolean stop = false;

public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		int i = 0;
		while (!stop) {
			i++;
		}
		System.out.println("t1 쓰레드가 멈췄습니다. 최종값은 " + i);
	});

	t1.start();

	Thread.sleep(1000);

	stop = true;
	System.out.println("stop = true");

	t1.join();
	System.out.println("main 종료");
}
```

![](https://blog.kakaocdn.net/dn/MzjJL/btsI5OvZVJW/bFLsAmxNNChfj2X8nFW7tk/img.png)

- volatile 키워드를 사용하면 동기화를 생략해도 된다.
- volatile 한정자는 배타적 수행과는 상관없지만 항상 **가장 최근에 기록된 값을 읽게 됨을 보장한다.**
- volatile 키워드를 적용한 변수는 L1, L2등의 캐시를 참고하지 않고 직접 메모리를 참조하도록 한다.

![volatile](https://github.com/woowacourse-study/2022-effective-java/raw/main/11%EC%9E%A5/%EC%95%84%EC%9D%B4%ED%85%9C_78/images/volatile.png)

> 주의 사항

**volatile 키워드는 원자성을 보장하지는 않는다.**

volatile 키워드를 사용한 변수에 대한 복합 연산은 여전히 원자성이 보장되지 않는다.

예를 들면 ++ 증가 연산자는 코드상으로 하나지만 실제로는 해당 변수에 두 번 접근한다. 먼저 값을 읽고, 그런다음 증가한 새로운 값을 저장하는 방식이다.

만약 두 번째 쓰레드가 이 두 접근 사이를 비집고 들어와 값을 읽어가면 잘못된 결과를 계산해내는 것이다. 이를 **안전 실패**라고 한다.

따라서 volatile 키워드는 변수의 변경이 boolean 플래그의 전환 등의 단일 연산으로 이루어질때 적합하다.

이런 경우의 대안은 두 가지이다.

- volatile 키워드를 제거하고 synchronzied를 사용한다.
- java.util.concurrent.atomic.AtomicLong 같은 라이브러리를 사용한다.
	- 해당 패키지는 락 없이도 쓰레드 안전한 프로그래밍을 지원하는 클래스들이 담겨있다.
	- volatile 키워드는 통신과 배타적 실행 중 통신만 지원하지만, 이 패키지는 배타적 실행(원자성)까지 지원한다.
	- 성능도 동기화 버전보다 더 우수하다.

### 하지만..

이번 아이템에서 언급한 문제들을 피하는 가장 좋은 방법은 애초에 가변 데이터를 공유하지 않는 것이다.

불변 데이터만 공유하거나 아무것도 공유하지 말자. 즉, ***가변 데이터는 단일 쓰레드에서만 사용하자.***

이런 정책이 정해졌다면, 그 사실을 문서에 남겨 유지보수 과정에서도 정책이 계속 지켜지도록 해야한다.

**한 쓰레드가 데이터를 다 수정한 후 다른 쓰레드에 공유할 때는 해당 객체에서 공유하는 부분만 동기화해도 된다.** 그러면 그 객체를 다시 수정할 일이 생기기 전 까지 다른 쓰레드들은 동기화 없이 자유롭게 값을 읽어갈 수 있다.

이러한 객체를 **사실상 불변**이라고 한다.

그리고 다른 쓰레드에 **사실상 불변** 객체를 건네는 행위를 **안전 발행**이라고 한다.

안전 발행하는 방법은 클래스 초기화 과정에서 객체를 정적 필드, volatile 필드, final 필드, 혹은 보통의 락을 통해 접근하는 필드에 저장하거나 동시성 컬렉션에 저장하는 방법 등이 있다.

## 정리

여러 쓰레드가 가변 데이터를 공유하면 그 데이터를 읽고 쓰는 동작은 반드시 동기화해야 한다.

동기화하지 않으면 한 쓰레드가 수행한 변경을 다른 쓰레드가 보지 못할 수 있다.

공유되는 가변 데이터를 동기화하는 데 실패하면 응답 불가 상태에 빠지거나 안전 실패로 이어질 수 있다. 문제는 디버깅 난이도가 매우 높다는 것이다.

간헐적이거나 특정 타이밍에만 발생할 수 있고, VM에 따라 현상이 달라지기도 한다.

배타적 실행(원자성)이 필요하지 않고, 쓰레드끼리의 통신만 필요하다면 volatile 한정자만으로 동기화할 수 있다. (물론 사용이 까다롭긴 하다.)