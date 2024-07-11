---
title: Item25 (톱레벨 클래스는 한 파일에 하나만 담아라)
author: leedohyun
date: 2024-07-10 20:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 톱레벨 클래스는 한 파일에 하나만 담아라

소스 파일 하나에 톱레벨 클래스가 여러 개 있어도 컴파일러에서 문제를 일으키지는 않는다.

하지만 아무런 득이 없고, 위험을 감수해야 한다.

> 톱 레벨 클래스

함수, 클래스 또는 무언가로 감싸지지 않은 모든 구문을 톱 레벨이라고 한다.

즉, 톱 레벨 클래스는 중첩 클래스가 아닌 바깥 클래스를 뜻한다.

### 톱레벨 클래스가 여러개일 때 위험성

```java
public class Main {

    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
}
```

```java
//Utensil.java에 두 톱레벨 클래스 존재
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```

- 이 경우 Main을 실행하면 pancake를 정상적으로 출력한다.

그런데 우연히 똑같은 두 클래스를 담은 Dessert.java라는 파일이 생긴다면?

```java
//Dessert.java에 두 톱레벨 클래스 존재
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```

![](https://blog.kakaocdn.net/dn/pjXjc/btsIwO3noNa/4gkiYgoB6OMOK4PRVRh4BK/img.png)

- 다시 실행해보면 위와 같은 에러를 만날 수 있다.
	- 운이 좋은 상황이다. 컴파일러가 동작한 순서가 좋았다.
	- Main.java 컴파일
	- System.out.println(Utensil.NAME + Dessert.NAME) -> Utensil.java 컴파일
	- 두 번째 인수인 Dessert.java 컴파일 할 때 중복을 알게 된 것이다.
- 만약 컴파일러의 순서가 아래와 같다면?
	- javac Main.java / javac Main.java Utensil.java
	- 문제를 일으키지 않고 pancake를 출력한다.

이렇게 컴파일러에 **어떤 순서로 소스파일을 건네느냐에 따라 동작이 달라지는 문제가 발생한다.**

### 해결 방안

아주 쉽게도 하나의 소스파일에 두 개 이상의 톱 레벨 클래스를 두지 않으면 된다.

위 예시로라면 Utensil과 Dessert를 각각 다른 소스 파일로 분리하면 된다.

> 굳이굳이 여러 톱레벨 클래스를 한 파일에 담고 싶다면?

정적 멤버 클래스를 사용하는 방안을 고민해볼 수 있다.

```java
public class Main {

    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
    
    private static class Utensil{
        static final String NAME = "pan";
    }
    
    private static class Dessert{
        static final String NAME = "caks";
    }
}
```

다른 클래스에 딸린 부차적인 클래스라면 정적 멤버 클래스로 만드는 것이 더 나을 것이다.

## 정리

소스 파일 하나에는 반드시 톱 레벨 클래스는 하나만 있어야 한다.

그렇지 않으면 컴파일러가 예기치 못한 동작을 하는 일은 없을 것이다.