----
title: Item15 (클래스와 멤버의 접근 권한을 최소화하라)
author: leedohyun
date: 2024-07-07 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

자바에는 클래스와 인터페이스 설계에 사용하는 강력한 요소가 많이 있다. 

이번 아이템부터는 클래스와 인터페이스에 대해 다룬다.

자바의 강력한 요소들을 적절히 활용해 클래스와 인터페이스를 쓰기 편하고 견고하며 유연하게 만드는 방법들을 소개한다.

이번 장의 아이템들을 읽어본 후 개인적인 생각으로는 이번 아이템들이 나같이 자바에 미숙한 사람들에게 비교적 큰 도움이 되는 부분이라고 생각됐다.

## 클래스와 멤버의 접근 권한을 최소화하라

어설프게 설계된 컴포넌트는 무엇이고 잘 설계된 컴포넌트란 무엇일까?

### 잘 설계된 컴포넌트

- 클래스 내부 데이터와 내부 구현 정보를 외부 컴포넌트로부터 잘 숨김
- 구현과 API를 완벽하게 분리해, 오직 API를 통해서만 다른 컴포넌트와 소통한다.
	- 서로의 내부 동작 방식에 영향을 받지 않는다.

***정보 은닉, 캡슐화***의 개념이 잘 설계된 컴포넌트에는 녹아 들어있다.

> 정보 은닉

다른 객체에게 자신의 정보를 숨기고 자신의 연산만을 통해 접근을 허용하는 것.

- 시스템 개발 속도를 높여준다.
	- 각 컴포넌트를 병렬로 개발할 수 있기 때문.
- 시스템 관리 비용을 낮춰준다.
	- 각 컴포넌트를 더 빨리 파악할 수 있고, 교체 부담도 적다.
	- 비교적 독립적이기 때문.
- 성능 최적화에 도움이 된다.
	- 최적화할 컴포넌트를 정하면 다른 컴포넌트에 영향을 주지 않고 해당 컴포넌트만 최적화 할 수 있기 때문.
- 소프트웨어 재사용성을 높인다.
- 큰 구조의 시스템 개발에 유리하다.
	- 각 컴포넌트들의 동작을 개발 후 개별로 그때그때 확인해볼 수 있기 때문.

자바는 이러한 장점들을 가진 정보 은닉을 제공하기 위해 클래스, 인터페이스, 멤버의 접근성을 제어하는 접근 제한자를 활용한다.

***정보 은닉의 기본 원칙은 모든 클래스와 멤버의 접근성을 가능한 한 좁히는 것***이다.

### 정보 은닉의 구현 방법

정보 은닉의 기본 원칙은 모든 클래스와 멤버의 접근성을 가능한 한 좁히는 것이라고 했다. 그렇다면 어떻게 구현해야 할 지 알아보자.

#### 톱 레벨 클래스, 인터페이스

톱 레벨 클래스란 가장 바깥의 클래스라는 뜻이다.

- package-private (default): 해당 패키지 내에서만 이용 가능
- public: 공개 API
	- 클라이언트에 영향을 준다.

기본적으로 톱 레벨 클래스나 인터페이스같은 경우 패키지 안에서만 이용할 수 있도록 되어 있다. 패키지 외부에서 쓸 이유가 없다면 기본을 유지하자.

API가 아닌 내부 구현이 되어 클라이언트에 피해를 주지 않고 언제든 수정 가능하다.

```java
public class Calculator {
	public int add(int a, int b) {
		return a + b;
	}

	public int multiply(int a, int b) {
		return a * b;
	}
} 
```

위와 같은 공개 API가 있다고 하자. 여기서 add 메서드의 인자 타입을 단순히 long으로 바꾼다면?

```java
public long add(long a, long b) {
	return a + b;
}

Calculator calc = new Calculator();
int result = calc.add(5, 3); //컴파일 에러 발생
```

public으로 선언해 공개 API로 만든다면 위와 같은 일이 벌어져 계속 관리해주어야 하는 일이 생긴다.

따라서 public일 필요가 없는 클래스의 접근 수준을 package-private으로 고치자.

> 바깥 클래스 하나에서만 접근 가능하도록 하고 싶다면?

기본 값인 package-private 제어자는 같은 패키지의 모든 클래스가 접근할 수 있다. 경우에 따라 이조차도 정보 은닉의 범위가 좁다고 생각할 수 있다.

만약 **톱 클래스 하나에서만 접근하도록 하고 싶다면 private static으로 클래스를 중첩시키는 것**도 방법이다. 

```java
class A {
	//...
}

class B {
	//...
}
```
```java
class A {
	private int a;

	private static class B {
		//...
	}
}

B b = new B(); //접근 불가
A.B //톱 클래스를 통해 접근 가능
```

#### 접근 제어자 설정의 기본 원칙

- private
	- 멤버를 선언한 톱 레벨 클래스에서만 접근 가능
- package-private
	- 멤버가 소속된 패키지 안의 모든 클래스에서 접근 가능
	- 접근 제한자를 명시하지 않았을 때 적용되는 패키지 접근 수준
- protected
	- 멤버를 선언한 클래스의 하위 클래스에서도 접근 가능
- public
	- 모든 곳에서 접근 가능

클래스의 ***공개 API를 세심하게 설계하고 그 외의 모든 멤버는 private으로 만들어야 한다.*** 그런 다음 오직 같은 패키지의 다른 클래스가 접근해야 하는 멤버에 한해 package-private으로 만들어 주면 된다. 

이 과정에서 권한을 풀어주는 일을 자주 하게 된다면 컴포넌트를 더 분해해야 하는 것은 아닌지 의심해봐야 한다.

#### 공개 API

public 클래스의 protected 멤버는 공개 API가 된다는 사실을 잊으면 안된다. 공개 API가 되면 계속 관리해주어야 한다.

따라서 private과 package-private이 아닌 경우를 세심하게 설계해야 한다.

protected와 public은 공개 API에 영향을 줄 수 있기 대문에 내부 동작 방식을 API문서에 적어 사용자에게 공개하는 경우도 생긴다.

> private과 package-private의 예외 사항

private과 package-private은 기본적으로 공개되지 않는 내부 구현용이지만, 예외적으로 Serializable (byte 코드 직렬화)을 직접 구현한 클래스에서는 공개 API가 될 수 있다.

#### public 클래스의 인스턴스 필드 (인스턴스 필드의 가변성 문제)

public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다.

필드가 가변 객체를 참조하거나, final이 아닌 인스턴스 필드를 public으로 선언하면 그 필드에 담을 수 있는 값을 제한할 힘을 잃게 된다.

```java
public class Data {
	public List<Integer> data = new ArrayList<>();
	
	public void addData(int value) {
		data.add(value);
	}

	public List<Integer> getData() {
		return data;
	}
}
```
```java
Data data = new Data();

data.addData(1);
data.addData(2);

List<Integer> dataList = data.getData();
dataList.data.add(3); //public 필드에 직접 접근해 데이터 추가
```

- 클라이언트 코드에서 직접 접근해 수정이 가능하다.
- 불변식을 보장할 수 없게 된다.
- 더해 필드가 수정될 때 락 획득과 같은 다른 작업을 할 수 없게되어 public 가변 필드를 갖는 클래스는 일반적으로 스레드 안전하지도 못하다.


> 정적 필드에서의 문제

정적 필드에서도 public 클래스의 인스턴스 필드를 public으로 선언했을 때와 비슷한 문제점이 있다.

다만 예외가 있다.

해당 클래스가 표현하는 추상 개념을 완성하는 데 꼭 필요한 구성 요소로써의 상수라면 public static final 필드로 공개해도 좋다. (물론 이 때 가변 객체의 참조는 안된다.)

> Array와 List 불변으로 리턴하는 방법

public static final 배열 필드나 이런 필드를 반환하는 접근자 메서드를 제공하면 안된다. 길이가 0이 아닌 배열은 위 처럼 모두 변경이 가능하다.

이를 불변으로 리턴하는 법을 알아보자.

- public 배열을 private으로 접근 제한하고, public 불변 리스트를 추가한다.

```java
private static final Instance[] PRIVATE_VALUES = {...};
public static final List<Instance> VALUES = 
	Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

- 배열을 private으로 만들고, 그 복사본을 반환하는 public 메서드를 만든다. (방어적 복사)

```java
private static final Instance[] PRIVATE_VALUES = {...};
public static final Instance[] values() {
	return PRIVATE_VALUES.clone();
}
```

#### 멤버 접근성을 못좁히는 경우?

멤버 접근성을 최대한 좁혀야 한다는 사실은 알았다. 그런데 이런 접근성을 좁히지 못하게 방해하는 제약이 하나 있다.

상위 클래스의 메서드를 재정의할 때는 그 접근 수준을 상위 클래스에서보다 좁게 설정할 수 없다. 

```java
class A {
	public void method1() {
		System.out.println("method1");
	}
}

class B extends A {
	@Override
	private void method1() {
		System.out.println("B method1");
	} // 컴파일 에러!
}
```

클래스가 인터페이스를 구현하는 것은 이 규칙의 특별한 예시이다. 인터페이스가 정의한 모든 메서드를 구현 클래스에서는 public으로 선언해야 한다.

```java
interface A {
	void method1();
	public void method2();
}

class AImpl implements A {
	@Override
	void method1() {
		System.out.println("method1");
	} // 컴파일 에러!

	@Override
	public void method2() {
		System.out.println("method2");
	}
}
```

#### 테스트 목적의 접근성 넓히기?

코드 테스트 목적으로 클래스나 인터페이스, 멤버의 접근 범위를 넓히려고 할 수 있다. 그러나 클래스의 ***private 멤버를 package-private으로 풀어주는 것 까지는 허용할 수 있지만, 그 이상(public, protected)은 안된다.***

만약 이렇게 풀어줘야 하는 경우 테스트의 목적에 대해 다시 생각해보고, 클래스의 구조나 설계가 잘못 되었는지를 되돌아봐야 한다.

#### Java 9의 모듈 시스템

자바 9부터 모듈 시스템이라는 개념이 도입되었다.

모듈은 패키지의 묶음이다. 모듈에 속하는 패키지 중 공개할 것을 선언하는 파일이 생성되었다.

이 파일에 존재하지 않는 패키지라면 public, protected 멤버라도 모듈 외부에서 접근이 불가능하다. 

다른 모듈에서는 접근할 수 없으나 같은 모듈이라면 패키지가 달라도 접근할 수 있는 접근 제어자가 생긴 것이다.

## 정리

정보 은닉에는 다양한 장점이 있다. 특히 각 컴포넌트에 일종의 독립성을 챙겨주어 개발 효율성이나 재사용성, 관리에 있어 큰 이점을 가져다 준다. 클라이언트나 다른 클래스 코드의 영향을 받지 않고 그 컴포넌트에 집중해 개발할 수 있다.

이러한 정보 은닉을 위해서는 최대한 접근성을 줄여야 한다. (private)

클래스, 인터페이스, 멤버는 의도한 설계가 아닌 이상 private으로 유지하는 것이 바람직하다. 공개 API를 정의하는 메서드는 public으로 선언해주면 된다.