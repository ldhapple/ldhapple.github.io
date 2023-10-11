---
title: equals, hashCode, @Data
author: leedohyun
date: 2023-10-09 18:13:00 -0500
categories: [JAVA, 정리]
tags: [JAVA, 정리]
---

equals and hashcode를 클래스에 정의해두면 테스트 코드를 작성할 때 객체의 값이 같으면 isEqualTo를 적용시킬 수 있었다. @Data 애노테이션에 포함된다.

어떻게 동작하는지 무슨 원리인지 잘 모르고 사용했기 때문에 정리해보려고 한다.

## equals, hashcode?

equals와 hashcode는 모두 Java 객체의 부모 객체인 Object 클래스에 정의되어 있다.

```java
@Override  
public int hashCode() {  
	return super.hashCode();  
}  
  
@Override  
public boolean equals(Object obj) {  
	return super.equals(obj);  
}
```

기본적으로 Override 함수를 호출하면 위와 같은 내용으로 되어있다.

### equals

```java
public boolean equals(Object obj) {
	return (this == obj);
}
```

보면 알 수 있듯 기본적으로는 2개의 객체가 동일한지 검사하기 위해 사용된다. equals가 구현된 방법을 보면 2개의 객체가 참조하는 것이 동일한 지 확인한다.

즉, 2개의 객체가 가리키는 곳이 동일한 메모리 주소일 경우에만 동일한 객체가 된다.

하지만 동일한 값의 객체가 메모리 상에 여러 개 띄워져있는 경우가 있다. 우리가 의도하는 바는 값이 같은 객체는 같은 객체인데, 프로그램은 두 객체가 서로 같은 메모리 주소를 갖고있지 않기 때문에 서로 다른 객체라고 인식한다.

우리가 의도한대로 동등성을 체크하기 위해, 값으로 객체를 비교하도록 equals 메서드를 오버라이딩 해주는 것이다.

### hashCode()

```java
public native int hashCode();
```

- native : 메서드가 JNI(Java Native Interface)라는 native code를 이용해 구현되었음을 의미한다.
	- 메서드에만 적용가능한 제어자로 C/C++ 등 Java가 아닌 언어로 구현된 부분을 JNI를 통해 Java에서 이용하고자 할 때 사용된다.
	- 일반 개발자는 사실 어디에도 사용할 수 없다.

이 메서드는 실행 중 객체의 유일한 integer 값을 반환한다. Object 클래스에서는 heap에 저장된 객체의 메모리 주소를 반환하도록 되어 있다.

hashCode는 HashTable과 같은 자료구조를 사용할 때 데이터가 저장되는 위치를 결정하기 위해 사용된다.

### 그래서 equals와 hashCode의 관계가..?

동일한 객체는 동일한 메모리 주소를 갖는다는 것을 의미하므로, 동일한 객체는 동일한 해시코드를 가져야 한다.

따라서 우리가 equals() 메서드를 오버라이드한다면 hashCode() 메서드도 함께 오버라이드 되어야 우리가 의도하는 대로 되는 것이다.

정리하자면

- Java 프로그램을 실행하는 동안 equals에 사용된 정보가 수정되지 않았다면, hashCode는 항상 동일한 정수값을 반환해야 한다.
- 두 객체가 equals()에 의해 동일하다면, 두 객체의 hashCode() 값도 일치해야 한다.
- 두 객체가 equals()에 의해 동일하지 않다면, 두 객체의 hashCode()도 일치하지 않아도 된다.

> 참고

equals가 true이면 hashCode(obj1) == hashCode(obj2) 이어야 하지만, 해시코드가 서로 같다고, equals가 true일 필요는 없다.

다만 위와 같은 경우는 다른 객체에 대해 동일한 hashCode를 생성한 것이므로 hashTable을 생성하는데 불이익을 받을 수 있음을 인지해야한다.

## 예시

```java
public class Member {
	private Integer id;
	private String name;
	private int num;
}
```

이렇게 Member 클래스가 있다고 가정하자. 각 Member는 id를 고윳값으로 가진다고 가정한다.

```java
@Test
void equaltest() {
	Member member1 = new Member();
	Member member2 = new Member();

	member1.setId(1);
	member2.setId(1);
	
	System.out.println(member1.equals(member2));
	assertThat(member1).isEqualTo(member2);
``` 

당연히 false가 나오게 된다. 우리가 의도한 바는 고유번호인 Id가 같기 때문에 같은 객체로 인식해야 한다. 하지만 현재는 따로 정의해주지 않았기 때문에 서로 다른 메모리 주소를 가리키고 있어 false가 나오는 것이다.

```java
@Override  
public boolean equals(Object obj) {  
	if (obj == null) {  
		return false;  
	}  
	  
	if (obj == this) {  
		return true;  
	}  
	  
	if (getClass() != obj.getClass()) {  
		return false;  
	}  
	  
	Member m = (Member) obj;  
	return (this.getId() == m.getId());  
}
```

해결하기 위해 Member 클래스에 위와 같이 equals 메서드를 오버라이드 해주었다. 테스트도 성공이 되고 true를 반환하는 것을 볼 수 있다.

하지만 Member를 HashSet과 같은 자료구조에 저장하면 또 다른 문제가 발생한다.

> hashCode() 오버라이드의 필요성

HashTable이나 HashSet, HashMap과 같은 자료구조는 자료를 저장하기 위해 HashCode를 이용한다.

Member를 그대로 hashSet에 저장해보자.

```java
@Test  
void test() {  
	Member member1 = new Member();  
	Member member2 = new Member();  
	  
	member1.setId(1);  
	member2.setId(1);  
	  
	HashSet<Member> set = new HashSet<>();  
	set.add(member1);  
	set.add(member2);  
	  
	System.out.println(set);  
}
```

![](https://blog.kakaocdn.net/dn/pE6TN/btsydVK0BAb/EyzUSVZwf2WB9ZEFdVlXEk/img.png)

equals를 재정의해서 hashSet에는 중복된 객체이기 때문에 하나만 들어가야 되는데 2개가 들어간 모습을 볼 수 있다.

hashCode() 메서드는 해당 메모리 주소값을 반환한다고 하였고, 그렇기 때문에 서로 다른 해시값을 반환한 것이다.

hashCode도 재정의해보자.

```java
@Override  
public int hashCode() {  
	final int PRIME = 31;  
	int result = 1;  
	result = PRIME * result + getId();  
	return result;  
}
```

![](https://blog.kakaocdn.net/dn/SyaBo/btsycWwDcMC/6cCmitajmVLfCZkCs5TZQ1/img.png)

재정의 한 이후 객체가 하나만 저장되는 모습을 볼 수 있다. @뒤의 값이 해시값과 관련있는 것을 볼 수 있다.

## @Data

@Data 애노테이션 안에는 @EqualsAndHashCode가 들어있다.

이 부분에서 사실 @Data를 사용하지 않는 것이 권장된다.

여기서 hashCode()는 객체의 값을 통해 해시코드를 생성한다. 

```java
@Data
public class MemberV1 {
	private Integer id;
	private String name;
	private int num;
}

@Data
public class MemberV2 {
	private Integer id;
	private String name;
	private int num;
}
```

이렇게 값이 같은 서로 다른 객체가 있다고 하자.

```java
@Test  
void test() {  
	Member member1 = new Member();  
	MemberV2 member2 = new MemberV2();  
	  
	member1.setId(1);  
	member2.setId(1);  
	  
	System.out.println(member1.hashCode());  
	System.out.println(member2.hashCode());  
}
```

![](https://blog.kakaocdn.net/dn/dhE9V1/btsycT0YNHK/XXeAtt2ktxzck3MvpFeyzK/img.png)

hashCode의 값이 같다. Member객체와 MemberV2 즉 서로 다른 객체임에도 불구하고 값이 같다는 이유로 hashCode가 같은 것이다.

동일한 객체임을 확인하는 equals()에서는 false가 나온다.

이러한 문제말고도 JPA를 사용했을 때 양방향 연관관계 필드 포함 시 문제가 발생하는데 이 부분은 JPA 학습 후 따로 포스팅하도록 한다.