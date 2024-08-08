---
title: Item65 (리플렉션보다는 인터페이스를 사용하라)
author: leedohyun
date: 2024-08-07 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 리플렉션보다는 인터페이스를 사용하라

리플렉션을 사용하면 임의의 클래스에 접근 가능하고, 해당 클래스의 멤버, 필드 등을 가져오고 조작이 가능하다.

심지어 컴파일 당시 존재하지 않던 클래스도 이용할 수 있다.

그러나 리플렉션에는 단점이 있다.

### 리플렉션의 단점

- 컴파일타임 타입 검사가 주는 이점을 누릴 수 없다.
	- 프로그램이 리플렉션 기능을 써서 존재하지 않거나 접근할 수 없는 메서드를 호출하려 시도하면 **런타임 에러**가 발생한다.
- 리플렉션을 이용하면 코드가 지저분하고 장황해진다. 읽기 어렵다.
- 성능이 떨어진다.
	- 리플렉션을 이용한 메서드 호출은 일반 메서드 호출보다 훨씬 느리다.

```java
public class ReflectionTest {  
   public void method() {  
	  int a = 1 + 1;  
   }  
}
```
```java
ReflectionTest rt = new ReflectionTest();  
for (int i = 0; i < 100_000; i++) {  
   rt.method();  
} // 28ms

Method m = ReflectionTest.class.getMethod("method");  
for (int i = 0; i < 100_000; i++) {  
   m.invoke(rt);  
} // 76ms
``` 

- 리플렉션을 사용한 것과 일반 메서드 호출 사이에 2배가 넘는 실행시간의 차이를 보였다.

### 리플렉션의 사용?

리플렉션은 단점을 가지고 있지만, 분명한 목적을 가지고 사용할 수 있는 도구이다.

코드 분석 도구나 의존관계 주입 프레임워크 같은 경우 리플렉션을 써야 하는 경우가 있다.

그러나 이런 도구들마저 리플렉션 사용을 줄이고 있다고 한다. 단점이 명확하기 때문이다.

우리가 작성하는 코드에는 아마 리플렉션은 필요없을 확률이 크다.

### 리플렉션을 사용해야만 하는 경우

만약 리플렉션이 정말 필요한 경우라면, 리플렉션을 제한된 형태로 사용하면 된다.

컴파일타임에 이용할 수 없는 클래스를 사용해야 하는 경우 비록 컴파일타임이라도 적절한 인터페이스나 상위 클래스를 이용할 수는 있다.

**이런 경우 리플렉션은 인스턴스 생성에만 쓰고, 이렇게 만든 인스턴스는 인터페이스나 상위클래스로 참조하면 된다.**

```java
public static void main(String[] args) {

	// 클래스 이름을 Class 객체로 변환
	Class<? extends Set<String>> cl = null;
	
	try {
		cl = (Class<? extends Set<String>>)
				Class.forName(args[0]);
	} catch (ClassNotFoundException e) {
		e.printStackTrace();
	}

	// 가져온 Class 객체로 선언된 생성자를 가져온다.
	Constructor<? extends Set<String>> cons = null;
	try {
		const = cl.getDeclaredConstructor();
	} catch (NoSuchMethodException e) {
		e.printStackTrace();
	}

	// 리플렉션을 통해 인스턴스만 생성하고, 
	// 참조는 상위 클래스인 Set으로 했다.
	Set<String> s = null;
	try {
		s = cons.newInstance();
	} catch (Exception e) {
		// IllegalAccessException (생성자 접근 불가)
		// InstantiationException (인스턴스화 불가)
		// InvocationTargetException (생성자가 예외 발생)
		// ClassCastException (Set의 하위 클래스가 아님)
		e.printStackTrace();
	}

	s.addAll(Arrays.asList(args).subList(1, args.length));
	System.out.println(s);
}
```

- 이 코드에서도 리플렉션의 단점들을 확인할 수 있다.
- 인스턴스만을 리플렉션으로 생성했을 뿐인데, 매우 긴 코드가 나타났다. 리플렉션을 통하지 않았더라면 생성자 호출 하나로 해결된다.
- 또, 리플렉션을 사용하여 수많은 예외를 만날 수 있어 처리해주었다.
	- 리플렉션 없이 생성했다면 컴파일 타임에 잡아낼 수 있는 예외들이다.
	- 하지만 예외가 발생하게 되면 런타임에 발생하게 된다.
- 그러나 리플렉션을 제한된 형태로 사용하여, 성능 문제는 인스턴스 생성시에만 발생하게 되었다.
	- 타입 안전성에서도 안전하며, 코드 가독성 또한 비교적 개선된다.

단점이 많지만, 리플렉션은 런타임에 존재하지 않을 수 있는 다른 클래스, 메서드, 필드와의 의존성을 관리하기에 용이하다.

특히 버전이 여러 개 존재하는 외부 패키지를 다룰 때 유용하다.

## 정리

리플렉션은 여러가지 단점이 있다. 특히 컴파일 타임에 에러를 잡지 못하는 점이 크다.

다만 리플렉션은 단점을 가진 만큼 강력한 기능을 가지고 있다.

특수한 경우가 아니고서야 리플렉션을 사용하는 경우는 드물겠지만, 리플렉션이 꼭 필요한 상황이라면 리플렉션은 단순히 인스턴스 생성용도로만 사용하고, 그 인스턴스를 상위 인터페이스로 참조하는 것이 단점들을 줄일 수 있는 방법이다.