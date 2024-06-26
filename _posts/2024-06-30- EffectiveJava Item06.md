---
title: Item6 (불필요한 객체 생성을 피하라)
author: leedohyun
date: 2024-06-30 21:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 불필요한 객체 생성을 피하라

똑같은 기능의 객체를 매번 생성하는 것 보다 객체 하나를 재사용하는 편이 좋을 때가 많다.

특히 불변 객체는 언제든 재사용이 가능하다.

우선 Java String이 어떻게 불필요한 객체 생성을 피하는지 알아보자.

### String의 불필요한 객체 생성 방지

```java
String str = "cool"; // 1
String str2 = new String("cool"); // 2
```

보통 우리는 1번 방식을 사용하게 된다. 만약 2번 방식을 사용하면 실행될 때 마다 String 인스턴스를 새로 만들게 된다.

> String str = new String("cool");

이러한 문장이 만약 반복문이나 빈번하게 호출되는 메서드 내부에 있는 문장이라면 쓸데없는 String 인스턴스가 수없이 많이 만들어질 수도 있다.

메모리의 힙 영역에 수많은 인스턴스들이 공간을 차지하게 된다.

> String str = "cool";

반면 1번 방식은 새로운 인스턴스를 매번 만드는 대신 하나의 String 인스턴스를 사용한다.

![](https://blog.kakaocdn.net/dn/b62c1o/btsIj6JSIbv/zkocdpQHyYYyRJsi289ih0/img.webp)

JVM 내에서 같은 문자열 리터럴을 사용하는 모든 코드가 같은 객체를 재사용하는 것이 보장된다. (Java에서 문자열 리터럴은 내부적으로 문자열 풀(String Pool)에 저장되어 재사용된다.)

***참고로 String Pool에서 관리되어 객체를 재사용함이 보장되는 것은 한가지 전제가 필요하다. 객체가 [불변]이어야 한다.***

만약 String이 가변이라면 같은 참조를 가지는 객체값이 변경이 가능하다는 의미이고 같은 참조지만 다른 값을 가지는 경우가 생긴다는 뜻이다.

그렇다면 String Pool에서 관리되어 재사용되어야 하는 객체가 재사용되지 못함을 의미한다.

하지만 String은 불변으로 설계되었기 때문에 무의미한 인스턴스들을 만들지 않고 객체를 공유해 사용할 수 있게 되어 있다.

### 생성자 대신 정적 팩토리 메서드를 이용해 불필요한 객체 생성을 방지하자

정적 팩토리 메서드를 제공해 불필요한 객체 생성을 피할 수 있다.

생성자는 호출될 때 마다 새로운 객체를 만들 수 밖에 없다. 그러나 정적 팩토리 메서드는 그렇지 않다.

예시로 Java Boolean 클래스를 보자.

```java
public final class Boolean implements java.io.Serializable,  
  Comparable<Boolean>, Constable  
{
	public static final Boolean TRUE = new Boolean(true);  
  
	public static final Boolean FALSE = new Boolean(false);
	
	public Boolean(String s) {  
	    this(parseBoolean(s));  
	}

	public static Boolean valueOf(String s) {  
	    return parseBoolean(s) ? TRUE : FALSE;  
    }
}
```

Boolean 생성자와 valueOf 팩터리 메서드를 보자.

valueOf() 팩토리 메서드는 새로운 객체를 만드는 것이 아닌 만들어 둔 객체를 반환한다. 즉 객체를 재사용한다.

꼭 불변 객체뿐만이 아니라 가변 객체라 하더라도 사용 중에 변경이 되지 않음이 보장된다면 재사용이 가능하다.

### 생성 비용이 비싼 객체는 캐싱해서 재사용하자.

생성 비용이 비싸다는 의미는 인스턴스를 생성하는데 드는 메모리와 같은 자원이 높다는 의미이다.

생성 비용이 비싼 객체의 예시로는 정규표현식을 사용하는 Pattern 인스턴스가 있다.

```java
static boolean isMatch(String s) {
	return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

문자열이 해당 정규표현식의 조건을 충족하는지 쉽게 확인할 수 있다. 그러나 성능이 중요한 상황에서는 반복해서 사용하기 어렵다.

***이 메서드가 내부에서 Pattern 인스턴스를 만든다. 이는 한 번 쓰고 버러져 바로 가비지 컬렉션의 대상이 된다.*** 

Pattern은 입력받은 정규표현식에 해당하는 유한 상태 머신을 만들기 때문에 인스턴스 생성 비용이 높다. 

인스턴스 생성 비용은 높은데 바로 가비지 컬렉션의 대상이 되어 더더욱 비효율적이다.

따라서 Pattern 객체를 만들어 컴파일하고 재사용하는 것이 권장되는 것이다.

```java
private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

static boolean isMatch(String s) {
	return ROMAN.matcher(s).matches();
}
```

![](https://blog.kakaocdn.net/dn/bJGxot/btsIiUcihbf/Qh1cqPoTGF1zkhmDX5PjTK/img.png)

단순하게 한 번 호출을 해도 컴파일을 해놓은 것과 안해놓은 부분의 차이 때문인지 캐싱을 했을 때와 안했을 때를 단순 실행 시간으로 측정해보아도 많이 차이가 나는 것을 볼 수 있다. (약 20배)

성능도 성능이지만 정규표현식에 네이밍을 줄 수 있다는 것 또한 장점이다.

> 유한 상태 머신

일종의 수학적 모델로 주어지는 모든 시간에 처해 있을 수 있는 유한 개의 상태를 가지고, 주어지는 입력에 따라 어떤 상태에서 다른 상태로 전이시키거나 출력과 같은 액션이 일어나도록 하는 장치를 나타낸 모델이다.

- 유한 상태 머신의 동작 과정
	- Pattern.compile("hello")
		- 정규표현식 "hello"를 컴파일하며 이 과정에서 유한 상태 머신이 생성된다.
		- 문자열의 각 문자와 상태 전이가 정의된다.
	- Matcher matcher = pattern.matcher("hello, how are you?") ;
		- 입력 문자열과 매칭할 Matcher 객체를 생성한다.
		- 내부적으로 유한 상태 머신을 사용해 패턴 매칭을 수행한다.
	- 유한 상태 머신 동작
		- 입력 문자열을 처음부터 끝까지 순회하며 패턴과 일치하는 부분을 찾는다.
		- "h" 문자로 시작하는지 확인하고 "h"가 입력 문자열의 첫 문자와 일치하지 않으면 다음 문자로 넘어간다. "h"가 보이면 "h"를 본 상태로 전이된다.
		- "e" 문자를 확인하고, 일치하면 "he"를 본 상태로 전이된다. 만약 "e"가 아닌 다른 문자를 본다면 아직 아무 문자도 확인하지 못한 상태로 전이된다.
		- 위 과정을 반복해 입력 문자열을 끝까지 탐색한다.

일반적인 탐색과 다르게 모든 상태와 전이를 정의해놓은 상태에서 일치하는 지 확인하기 때문에 생성 비용이 높은 것이다.

하지만 이렇게 정의해놓기 때문에 생성 이후부터는 속도가 빨라 컴파일러 시점에서 생성하는 것이 권장된다.

### 오토 박싱에서의 불필요한 객체 생성

오토 박싱은 프로그래머가 기본 타입과 박싱된 기본 타입을 섞어 쓸 때 자동으로 상호 변환해주는 기술이다.

기본 타입과 그에 대응하는 박싱된 기본 타입의 구분을 흐려주지만, 완전히 없애주지는 않는다.

```java
Integer multiplication = 0;  
for (int i = 1; i <= 10000000; i++) {  
  multiplication += i;  
}
```

```java
int multiplication = 0;  
for (int i = 1; i <= 10000000; i++) {  
  multiplication += i;  
}
```

![](https://blog.kakaocdn.net/dn/baAOLE/btsIktxSHCy/w3aYLQCYpzrYHKfbrcsTaK/img.png)

두 코드의 실행 시간 차이이다.

의미상으로는 크게 차이가 없으나 성능면에서 큰 차이가 발생한다.

단순히 박싱된 기본 타입을 잘못 사용했다고 해서 의미 없는 인스턴스가 수없이 많이 생겨 성능 저하가 발생한 것이다.

***박싱된 기본 타입보다는 기본 타입을 사용하고 의도치 않은 오토 박싱이 숨어들지 않도록 주의해야 한다.***

## 정리

이번 아이템인 불필요한 객체 생성을 피하자는 객체 생성은 비싸니까 무조건 피해야 한다는 의미가 아니다. 불필요한 객체 생성은 성능에 안좋은 영향이 있으니 주의해야 한다가 더 나은 표현일 것 같다.

요즘 JVM에서는 별다른 일을 하지 않는 작은 객체를 생성하고 회수하는 일은 크게 부담되지 않는다. 오히려 프로그램의 명확성, 간결성, 기능을 위해서라면 객체를 추가로 생성하는 것이 좋은 일이다.

그리고 아주 무거운 객체가 아닌 이상 단순히 객체 생성을 피하고자 본인만의 객체 풀을 만드는 것은 권장되지 않는다. JVM의 가비지 컬렉터는 최적화가 잘 되어있기 때문에 가벼운 객체를 다룰 때는 직접 만드는 것 보다 훨씬 나을 것이다.

아이템 50을 미리 보면 ***새로운 객체를 만들어야 한다면 기존 객체를 재사용하지 마라*** 라는 것이 있다. 방어적 복사를 다룬다.

이번 아이템과 대조적이다. 방어적 복사가 필요한 상황에서 객체를 재사용했을 때의 피해가 필요 없는 객체를 반복 생성했을 때의 피해보다 훨씬 크다.

방어적 복사에 실패하면 언제 터져 나올지 모르는 버그와 보안 문제들로 이어지지만 불필요한 객체 생성은 단순히 코드 형태와 성능에만 영향을 끼친다.

> 방어적 복사

내부 객체를 반환할 때 객체의 복사본을 만들어 반환하는 것.

이렇게 복사한 외부의 객체를 변경하더라도 원본 내부 객체가 변경되지 않는다.