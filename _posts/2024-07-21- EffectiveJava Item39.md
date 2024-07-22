---
title: Item39 (명명 패턴보다 애너테이션을 사용하라)
author: leedohyun
date: 2024-07-21 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 명명 패턴보다 애너테이션을 사용하라

명명 패턴이란 무엇일까?

```java
int studnetAge;
String firstName;

public static final int MAX_HEIGHT = 150;

public void caculateScore() { ... }

public class UserAccount { }

package com.example.project;
```

- 위와 같이 우리가 적용하고 있는 Camel Case, 클래스명, 패키지명의 네이밍 방법 및 규칙들이 포함된다.
- 또 예시로는 JUnit은 버전 3까지 테스트 메서드 이름을 test로 시작하게끔 했다.
	- 이러한 일종의 규약도 명명 패턴에 포함된다.

이러한 명명 패턴은 가독성 및 유지보수성에 큰 도움이 되지만 단점이 있다.

- 오타가 나면 안된다.
- 올바른 프로그램 요소에서만 사용되리라 보증할 방법이 없다.
	- ex) 메서드가 아닌 클래스 이름을 TestSafetyMechanisms로 JUnit에 던졌는데, JUnit은 클래스 이름에 관심이 없다. 메서드 이름이 아니기 때문에 그냥 무시하고 테스트를 수행하지 않는다.
- 프로그램 요소를 매개변수로 전달할 마땅한 방법이 없다.
	- ex) 특정 예외를 던져야만 성공하는 테스트 메서드가 있다. 기대하는 예외 타입을 테스트에 매개변수로 전달해야 한다. 예외의 이름을 테스트 메서드 이름에 붙일 수도 있지만 가독성이 좋지 않다.

이러한 부분을 애노테이션이 해결해줄 수 있다.

### 애노테이션을 써보자.

```java
//마커 애노테이션 타입 선언
@Retenstion(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
}
```

- @Test 애노테이션 타입 선언 자체에도 두 가지의 다른 애노테이션들이 달려있다.
	- @Retension: @Test가 런타임에도 유지되어야 한다는 표시. 이 애노테이션을 생략하면 테스트 도구는 @Test 애노테이션을 인식할 수 없다.
	- @Target: @Test가 반드시 메서드 선언에서만 사용되어야 한다는 것을 알려준다. 따라서 클래스 선언, 필드 선언 등 다른 프로그램 요소에는 달 수 없다.
- 이와 같이 아무 매개변수 없이 단순히 대상에 마킹한다는 뜻을 가진 애노테이션을 마커 애노테이션이라고 한다.

```java
public class SampleInstance {
	@Test
	public static void test1() { }
	
	public static void test2() { }
	
	@Test
	public static void test3() {
		throw new RuntimeException("실패");
	}

	@Test
	public void test4() { } // 정적 메서드가 아니기 때문에
```

- @Test 애노테이션을 붙인 메서드는 테스트 도구가 테스트한다.
	- 애노테이션을 붙이지 않은 메서드는 테스트 도구가 무시한다.
- 인스턴스 메서드인 test4()는 잘못 사용한 경우이다.
	- 보통 테스트 프레임워크가 테스트 메서드를 정적 메서드로 제한한다.
	- 상태를 가지지 않아 테스트 간의 간섭을 피하고 독립성을 보장한다.
	- 정적 메서드는 인스턴스를 생성하지 않아도 되기 때문에 테스트 프레임워크의 구현을 단순화한다.
- 애노테이션이 SampleInstance 클래스에 직접적인 형향을 끼치지 않는다. 단순히 이 애노테이션에게 관심있는 프로그램에게 추가 정보를 제공할 뿐이다. 

물론 이렇게 애노테이션만 선언하고 사용한다고 해서 원하는 동작이 이루어지지는 않는다. 

해당 애노테이션을 어떻게 사용할지의 로직을 갖고있는 프로그램이 필요하다. (ex - 테스트 프레임워크)

```java
import java.lang.reflect.*; //자바 리플렉션 API 사용

public class RunTests {
	public static void main(String[] args) throws Exception {
		int tests = 0;
		int passed = 0;

		Class<?> testClass = Class.forName(args[0]); //테스트 할 클래스 이름을 받아옴
		for (Method m : testClass.getDeclaredMethods()) { //클래스 모든 메서드 순회
			if (m.isAnnotationPresent(Test.class)) { //메서드에 @Test가 붙어있다면
				tests++;
				try {
					m.invoke(null); //정적 메서드인 경우 invoke(null)로 메서드 호출
					passed++;
				} catch (InvocationTargetException wrappedExc) {
					Throwable exc = wrappedExc.getCause();
					System.out.println(m + " 실패: " + exc);
				} catch (Exception exc) {
					System.out.println("잘못 사용한 @Test: " + m);
				}
			}
		}
		System.out.printf("성공: %d, 실패: %d\n",
							passed, tests - passed);
	}
}			
```

- 우리가 사용하는 테스트 프레임워크는 이와 비슷하게 동작한다.
	- 물론 실제로는 더 많은 기능과 복잡한 로직이 있다.
- 리플렉션을 사용해 클래스와 메서드, 애노테이션 정보를 가져와 애노테이션이 붙은 메서드를 호출했다.

> 자바 리플렉션 API

- 런타임에 클래스, 메서드, 필드, 애노테이션 등에 대한 정보를 동적으로 검사하고 조작할 수 있는 기능을 제공한다.
	- 자바 프로그램이 자기 자신을 검사하고 수정할 수 있다.
	- 프레임워크나 라이브러리 개발에서 쓰인다.

간단하게 몇 가지 기능만 확인해보자.

```java
Class<?> clazz = Class.forName("com.example.MyClass");
```

- 클래스의 이름, 구현된 인터페이스 등등을 가져올 수 있다.

```java
Constructor<?> constructor = clazz.getConstructor(); 
Object instance = constructor.newInstance();
```

- 클래스의 생성자를 가져올 수 있다. 
- 그 생성자를 사용해 객체를 생성할 수 있다.

```java
Method method = clazz.getMethod("myMethod"); 
method.invoke(instance);
```

- 클래스의 메서드 정보를 가져올 수 있다. 
- 이를 사용해 메서드를 호출할 수도 있다.

```java
Field field = clazz.getField("myField"); 
field.set(instance, "newValue");
```

- 클래스의 필드 정보를 가져올 수 있다. 
- 필드 값을 세팅할 수도 있다.

```java
if (method.isAnnotationPresent(MyAnnotation.class)) { 
	MyAnnotation annotation = method.getAnnotation(MyAnnotation.class); 
}
```

- 클래스나 메서드, 필드 등에 선언된 애노테이션을 가져올 수 있다.

> 리플렉션의 장단점

- 장점
	- 동적으로 객체를 생성하고 조작할 수 있다.
	- 클래스, 메서드, 필드 등의 구조를 분석하고 조작할 수 있어 코드 분석 도구나 프레임워크 개발에 유용하다.
- 단점
	- 리플렉션을 사용하면 일반 메서드 호출보다 느리다. 메서드, 필드 등에 동적으로 접근하는 데에 추가적인 비용이 발생한다. (성능 문제)
	- 리플렉션을 사용하면 컴파일 타임에 타입 검사를 피할 수 있다. (안전성 문제) 

### 매개변수를 받는 애노테이션 사용법

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElemenetType.METHOD)
public @interface ExceptionTest {
	Class<? extends Throwable> value();
}
```

- Class<? extends Throwable> value()
	- 이 메서드의 선언으로 애노테이션에 매개변수를 받을 수 있다.
	- 매개변수는 value 메서드의 매개변수로 전달된다.

```java
m.getAnnotation(ExceptionTest.class).value();
```

- 테스트 프레임워크에서는 위와 같은 방식으로 매개변수 값을 받아와 사용할 수 있는 것이다.

```java
Class<? extends Throwable>[] value();
```

- 위와 같이 애노테이션에 배열을 매개변수로 받도록 할 수 있다.

## 정리

애노테이션을 만들 때는 적절한 보존 정책(@Retenstion)과 적용 대상(@Target)을 명시해야 한다.

애노테이션을 통해 소스 코드에 추가 정보를 제공할 수 있다. 명명 패턴도 가능한 일이지만 오타의 가능성이 문제고, 활용도가 떨어진다. 따라서 애노테이션으로 해결할 수 있는 일이라면 애노테이션을 사용해보자.

물론 도구 제작자를 제외하고는 일반 프로그래머가 애노테이션 타입을 직접 정의하는 일은 거의 없다. 하지만 만들어놓은, 제공되는 애노테이션들을 적극 활용하자. (스프링 AOP, Test 등등)