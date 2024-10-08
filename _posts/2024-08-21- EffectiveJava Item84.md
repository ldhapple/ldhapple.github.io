---
title: Item84 (프로그램의 동작을 쓰레드 스케줄러에 기대지 말라) + 스프링 스케줄러
author: leedohyun
date: 2024-08-21 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 프로그램의 동작을 쓰레드 스케줄러에 기대지 말라

여러 쓰레드가 실행 중이면 운영체제의 쓰레드 스케줄러가 어떤 쓰레드를 얼마나 오래 실행할지 결정하게 된다.

구체적인 스케줄링 정책은 운영체제마다 다르기 때문에 잘 작성된 프로그램이라면 이 스케줄링 정책에 좌지우지 되어서는 안된다.

***정확성이나 성능이 쓰레드 스케줄러에 따라 달라지는 프로그램이라면 다른 플랫폼에 이식하기 어렵다.***

### 쓰레드의 평균적인 수를 프로세서 수보다 지나치게 많아지지 않도록 하자

견고하고 빠르고 이식성 좋은 프로그램을 작성하는 가장 좋은 방법이다.

쓰레드 스케줄러가 고민할 내용을 주지 않는 것이다.

실행 준비가 된 쓰레드들은 맡은 작업을 완료할 때 까지 계속 실행되도록 만든다. 이런 프로그램이라면 쓰레드 스케줄링 정책이 아주 상이한 시스템이라도 동작이 크게 달라지지 않을 것이다.

여기서 실행 가능한 쓰레드의 수와 전체 쓰레드 수는 구분해야 한다. 전체 쓰레드의 수는 훨씬 많을 수 있고, 대기 중인 쓰레드는 실행 가능하지 않다.

### 쓰레드의 수를 적게 유지하는 방법

실행 가능한 쓰레드 수를 적게 유지하는 주요 기법은 각 쓰레드가 무언가 유용한 작업을 완료한 후 다음 일거리가 생길 때 까지 대기하도록 하는 것이다.

***쓰레드는 당장 처리해야 하는 작업이 없다면 실행되어서는 안된다.***

Executor를 예시로 들면, 쓰레드 풀 크기를 적절히 설정하고 작업은 짧게 유지하면 된다. 단, 너무 짧을 경우 작업을 분배하는 부담이 오히려 성능을 떨어뜨리기 때문에 그 부분은 고려해야 한다.

> 단, 바쁜 대기 상태가 되면 안된다.

공유 객체의 상태가 바뀔 때 까지 쉬지 않고 검사해서는 안된다는 의미이다.

바쁜 대기는 쓰레드 스케줄러의 변덕에 취약할 뿐 아니라 프로세서에 큰 부담을 주어 다른 유용한 작업이 실행될 기회를 박탈한다.

극단적인 예시를 보자.

```java
public class SlowCountDownLatch {
	private int count;

	public SlowCountDownLatch(int count) {
		if (count < 0) {
			throw new IllegalArgumentException(count + " < 0");
		}
		this.count = count;
	}

	public void await() {
		while (true) {
			synchronized(this) {
				if (count == 0) return;
			}
		}
	}

	public synchronized void countDown() {
		if (count != 0) count--;
	}
}
```

- 쓰레드가 특정 조건이 충족될 때 까지 반복적으로 조건을 확인한다.
	- await() 메서드의 while 문을 이용해 count가 0이 될 때까지 count 값을 계속 확인한다.
	- 동기화 블록 안에서 count가 0인지 확인한 후 0이면 return하여 종료한다.
	- count가 0이 아니면 루프는 계속 반복된다.
-  쓰레드가 계속해서 count 값을 확인하기 위해 CPU를 소모한다. 다른 중요한 작업을 할 수 있는 기회를 잃게 되는 것이다.
- 유용한 작업을 하지 않으면서도 CPU를 계속 사용해 성능 문제가 발생한다.

이렇게 하나 이상의 쓰레드가 필요 없이 실행 가능한 상태인 시스템은 의외로 흔하다. 

필요 없이 실행 가능한 상태라는 것은 조건을 체크하면서 CPU 자원을 사용하고 있는데, 이 상태가 실행 가능한 상태인 것이다. 유용한 작업을 하지 않으면서도 상태를 유지한다.

그리고 만약 특정 쓰레드가 다른 쓰레드들과 비교해 CPU 시간을 충분히 얻지 못해 간신히 돌아가는 프로그램을 보더라도 **Thread.yield를 써서 문제를 고쳐보려는 유혹을 떨쳐내야 한다.**

증상이 호전될 수 있지만 이식성은 그렇지 않다. 더해 테스트할 수단도 없다. 차라리 애플리케이션 구조를 바꾸어 동시에 실행 가능한 쓰레드 수가 적어지도록 조치하면 된다.

### 참고: 스프링의 스케줄러

스프링에서도 스케줄러가 존재한다. 다양한 스케줄링 작업을 관리할 수 있고, 시스템 자원을 효율적으로 사용하는데 도움을 준다.

스프링의 쓰레드 스케줄러에 대해 간단히 알아보자.

#### TaskScheduler

- 스프링에서 스케줄링 작업을 관리하기 위한 주요 인터페이스이다.
	- 작업을 특정 시점에 실행하거나, 주기적으로 실행할 수 있도록 도와준다.
- TaskScheduler를 구현한 클래스는 내부적으로 쓰레드 풀을 관리해 작업을 효율적으로 분배하고 실행할 수 있도록 한다.

```java
@Configuration
public class ScheulerConfig {
	@Bean
	public ThreadPoolTaskScheduler taskScheduler() {
		ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
		scheduler.setPoolSize(10);
		scheduler.setThreadNamePrefix("name-");
		return scheduler;
	}
}
``` 

- 위와 같이 쓰레드 풀을 구성하고, 관리할 수 있다.

#### @Scheduled

- 스케줄링 작업을 간단하게 정의할 수 있도록 하는 애노테이션이다.
	- 특정 메서드를 주기적으로 실행하거나, 특정 시간에 실행할 수 있도록 한다.
- 쓰레드 스케줄러에 의존하지 않고 주기적인 작업을 효율적으로 관리할 수 있다.

```java
@Component
public class Tasks {
	
	@Scheduled(fixRate = 5000)
	public void task1() {
		System.out.println("task1..");
	}

	@Scheduled(fixedDelay = 5000)
	public void task2() {
		System.out.println("task delay..");
	}

	@Scheduled(cron = "0 0/1 * * * ?")
	public void taskWithCron() {
		System.out.println("task with cron..");
	}
}
``` 

- fixedRate
	- 작업을 고정된 간격으로 실행한다.
	- 이전 작업이 시작된 시간부터 일정 간격으로 실행된다.
- fixedDelay
	- 이전 작업이 종료된 후 일정 시간 지연 후 작업을 실행한다.
- Cron 표현식을 사용해 작업을 실행할 시간과 간격을 정의한다.
	- 위 Cron 표현식의 뜻은 아래와 같다.
	- 초 ('0') : 작업이 매분 0초에 실행된다.
	- 분 ('0/1') : 작업이 매 1분마다 실행된다. 0/1은 첫 번째 분에서 시작해 매 1분마다 실행하라는 뜻이다.
	- 시 ('*') : 하루 중 모든 시간에서 작업을 실행하라는 뜻이다.
	- 월 ('*') : 매달 작업을 실행하라는 뜻이다.
	- 요일('?') : 요일에 상관하지 않고 작업을 실행하라는 뜻이다. 

이 @Scheduled 애노테이션은 Spring Batch와 같이 사용되기도 한다. Spring Batch는 데이터의 대규모 배치 작업을 효율적으로 관리하고 처리할 수 있도록 해준다.

- 주로 배치 작업을 자동으로 실행하는 데 함께 사용된다.
	- 배치 작업은 일괄 처리 방식으로 데이터를 대량으로 읽고 처리하고 저장하는 등의 역할을 한다.
- 배치 작업은 Job이라고 불리는데 일반적으로 야간 또는 특정 간격으로 실행되기 때문에 이 때 @Scheduled 애노테이션을 사용하게 된다.