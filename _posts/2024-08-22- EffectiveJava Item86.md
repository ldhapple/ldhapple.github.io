---
title: Item86 (Serializable을 구현할지는 신중히 결정하라)
author: leedohyun
date: 2024-08-22 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## Serializable을 구현할지는 신중히 결정하라

어떤 클래스의 인스턴스를 직렬화할 수 있도록 하려면 간단하게 Serializable을 구현해주면 된다.

너무 쉽게 적용할 수 있기 때문에 프로그래머가 특별히 신경 쓸 것이 없다는 오해가 생길 수 있다. 하지만 진실은 그렇지 않다.

직렬화를 구현한다는 것은 아주 값비싼 일이다.

### Serializable을 구현하면 릴리스 뒤에 수정하기 어렵다.

클래스가 Serializable을 구현하면 직렬화된 바이트 스트림 인코딩도 하나의 공개 API가 된다.

따라서 이 클래스가 널리 퍼지는 경우 공개 API 처럼 영원히 지원해주어야 하는 것이다.

그런데 커스텀 직렬화 형태를 설계하지 않고 자바의 기본 방식을 사용한다면 직렬화 형태는 최소 적용 당시 클래스의 내부 구현 방식에 영원히 묶여버린다.

기본 직렬화 형태에서는 클래스의 private와 package-private 인스턴스 필드들마저 API로 공개하는 꼴이 되는 것이다. **즉, 캡슐화가 깨진다.**

```java
class Member implements Serializable {
	private String name;
	private int age;

	public Member(String name, int age) {
		this.name = name;
		this.age = age;
	}
	
	public String getName() {
		return name;
	}

	public int getAge() {
		return age;
	}
}
```
```java
Member member = new Member("Hong", 25);

FileOutputStream fileOut = new FileOutputStream("member.ser");
ObjectOutputStream objOut = new ObjectOutputStream(fileOut);

objOut.writeObject(member);
objOut.close();
fileOut.close();

System.out.println("member.ser 파일에 직렬화 완료");

FileInputStream fileIn = new FileInputStream("member.ser");
ObjectInputStream objIn = new ObjectInputStream(fileIn);

Member deserializedMember = (Member) objIn.readObject();
objIn.close();
fileIn.close();

System.out.println("역직렬화 객체 이름: " + deserializedMember.getName());
System.out.println("역직렬화 객체 나이: " + deserializedMember.getAge());
```

- name과 age라는 private 필드를 가지고 있었음에도 불구하고 역직렬화하면 리플렉션 등을 이용해서 private 필드에 쉽게 접근할 수 있다.
- 즉, 직렬화된 상태를 갖고 있는 상태라면 필드로의 접근을 막을 수 없는 상태가 된다.
- 만약 뒤늦게 클래스 내부 구현을 바꾸게 되었다고 가정하면 원래의 직렬화 형태와 달라진다.
	- 한쪽은 구 버전 인스턴스를 직렬화하고 다른 쪽은 신 버전 클래스로 역직렬화하게 되면 실패가 발생할 것이다.

ObjectOutputStream.putFields와 ObjectInputStream.readFields를 이용한다면 원래의 직렬화 형태를 유지하면서 내부 표현을 바꿀 수 있긴 하다. 하지만 어렵고, 소스코드도 지저분해진다.

**따라서 직렬화 기능 클래스를 만들고자 한다면, 길게 보고 감당할 수 있을 만큼 고품질의 직렬화 형태도 함께 설계해야 한다.**

> 직렬화가 클래스 개선에 방해되는 또 다른 경우

직렬화된 클래스는 고유 식별 번호를 부여받는다.

static final long serialVersionUID 라는 필드로 직접 명시하지 않으면 시스템이 런타임에 암호 해시 함수를 적용해 자동으로 클래스 안에 생성해 넣는다.

이 값을 생성하는 데에는 클래스 이름, 구현한 인터페이스들, 컴파일러가 자동으로 생성해 넣은 것을 포함한 대부분의 클래스 멤버들이 고려된다.

나중에 이들 중 하나라도 수정한다면 직렬 버전 UID 값도 변하게 되고, 쉽게 호환성이 깨진다.

따라서 UID를 직접 관리하는 것이 좋다.

![](https://blog.kakaocdn.net/dn/bzwfSd/btsJdnkJhX1/Ktsii42uKmnOskRfryCJL1/img.png)

위 사진은 예시 Member 클래스에 아무런 필드를 하나 추가한 상태로 다시 역직렬화를 해본 결과로, 위 문제를 확인할 수 있다.

### 버그와 보안 구멍이 생길 위험이 높아진다.

객체는 생성자를 사용해 만드는 것이 기본이다.

그런데 직렬화는 그렇지 않다. 언어의 기본 메커니즘을 우회하는 객체 생성 기법인 것이다.

기본 방식을 따르든 재정의해 사용하든 역직렬화는 일반 생성자의 문제가 그대로 적용되는 숨은 생성자이다.

이 생성자는 전면에 드러나지 않아 **생성자에서 구축한 불변식을 모두 보장해야 하고 생성 도중 공격자가 객체 내부를 들여다 볼 수 없도록 해야 한다는 사실**을 떠올리기 어렵다.

즉, 기본 역직렬화를 사용하면 불변식 깨짐과 허가되지 않은 접근에 쉽게 노출된다.

### 해당 클래스의 신버전을 릴리스할 때 테스트할 것이 늘어난다.

직렬화 가능 클래스가 수정되면 신버전 인스턴스를 직렬화한 후 구버전으로 역직렬화할 수 있는지, 그리고 그 반대도 가능한지 검사해야한다.

따라서 테스트해야 할 양이 직렬화 가능 클래스의 수와 릴리스 횟수에 비례해 증가하게 된다.

위에서도 언급했지만 클래스를 처음 제작할 때 커스텀 직렬화 형태를 잘 설계했다면 이런 테스트 부담을 줄일 수 있을 것이다.

### Serializable을 사용하는 경우

객체를 전송하거나 저장할 때 자바 직렬화를 이용하는 프레임워크용도로 만든 클래스라면 어쩔 수 없이 사용해야 한다.

아니면 Serializable을 반드시 구현해야 하는 다른 클래스의 컴포넌트로 쓰일 클래스일 때도 어쩔 수 없다.

하지만, Serializable 구현에 따르는 비용이 적지 않기때문에 클래스를 설계할 때마다 그 이득과 비용을 잘 조율해야 한다.

### 상속용으로 설계된 클래스 대부분은 Serializable을 구현하면 안된다.

인터페이스 또한 마찬가지이다. 대부분 Serializable을 확장해서는 안된다.

만약 그렇지 않는다면 그런 클래스를 확장하거나 그런 인터페이스를 구현하는 쪽에 큰 부담을 지우게 된다.

예외가 하나 있다면, Serializable을 구현한 클래스만 지원하는 프레임워크를 사용하는 상황일 것이다.

상속용으로 설계된 클래스 중 Throwable과 Component는 Serializable이 구현되어 있다. 

그 중 Component는 GUI를 전송하고 저장하고 복원하기 위해 구현했지만, Swing과 AWT가 널리 쓰이던 시절에도 현업에서 이러한 용도로 사용되지 않았다.

따라서 웬만해서는 Serializable을 구현할 이유가 없는 것이다.

## 정리

Serializable은 선언하기 쉽지만, 그렇다고 쉽게 사용해서는 안된다.

정말 안전한 환경이 아니라면 사용하지 않는 것이 낫다. 상속할 수 있는 클래스라면 주의할 점이 더더욱 많다.

직렬화를 사용하고 싶거나, 사용해야 할 경우 아래와 같은 부분을 인지하고 직렬화를 사용하는 것이 권장된다.

- 외부 저장소로 저장되는 데이터는 짧은 만료시간의 데이터를 제외하고 자바 직렬화 사용을 지양해야 한다.
- 역직렬화시 반드시 예외가 생긴다는 것을 생각하고 개발한다.
- 자주 변경되는 비즈니스에 관련된 데이터에 자바 직렬화를 사용하지 말자
- 긴 만료 시간을 가지는 데이터는 JSON 등 다른 포맷을 사용하자.