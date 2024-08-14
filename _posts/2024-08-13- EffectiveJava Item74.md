---
title: Item74 (메서드가 던지는 모든 예외를 문서화하라)
author: leedohyun
date: 2024-08-13 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 메서드가 던지는 모든 예외를 문서화하라

메서드가 던지는 예외는 해당 메서드를 올바르게 사용하는 데 아주 중요한 정보이다.

따라서 각 메서드가 던지는 예외 하나하나를 문서화하는 데 충분한 시간을 들여야 한다.

### 검사 예외 문서화

**검사 예외는 항상 따로따로 선언하고, 각 예외가 발생하는 상황을 JavaDoc의 @throws 태그를 사용해 정확히 문서화해야 한다.**

```java
/*
 * @param filePath 파일 경로
 * @param connetion 데이터베이스 연결 객체
 * @throws IOException 파일을 읽는 중 오류가 발생할 경우
 * @throws SQLException 데이터베이스 작업 중 오류가 발생할 경우
*/
public void fileSave(String filePath, Connection con) 
		throws IOException, SQLException {
	
	BufferedReader reader = null;

	try {
		reader = new BufferedReader(new FileReader(filePath));
		String line;

		while ((line = reader.readLine()) != null) {
			saveDatabase(con, line);
		}
	} finally {
		if (reader != null) {
			reader.close();
		}
	}
}
```

주의할 점은 공통 상위 클래스 하나로 뭉뚱그려 선언하면 안된다. 극단적인 예시로는 메서드가 Exception이나 Throwable을 던지는 경우가 있을 것이다.

이러면 각 사용자에게 예외에 대한 힌트를 주지 못하고, 같은 맥락에서 발생할 여지가 있는 다른 예외까지 삼켜버릴 수도 있다. 이러면 API 사용성이 크게 떨어진다.

> main 메서드

main 메서드는 이 규칙에서 예외이다.

main은 오직 JVM만이 호출해 Exception을 던지도록 선언해도 괜찮다. JVM은 예외를 출력하고 프로그램을 종료하기 때문이다.

### 비검사 예외 문서화

비검사 예외의 문서화는 Java 언어가 요구하지는 않는다. 하지만 검사 예외와 마찬가지로 문서화해두면 좋다.

비검사 예외는 일반적으로 프로그래밍 오류를 뜻한다고 했다. 따라서 자신이 일으킬 수 있는 오류들이 무엇인지 문서화해 알려준다면 프로그래머들은 해당 오류가 나지 않도록 작성할 것이다.

- public 메서드라면, 필요한 전제조건을 문서화해야 한다. 그 수단으로 가장 좋은 것이 비검사 예외들을 문서화하는 것이다.
- 인터페이스 메서드에서 비검사 예외를 문서화하는 것은 특히 중요하다.
	- 정리하는 해당 조건이 인터페이스의 일반 규약이 되고, 모든 구현체가 그 규약을 참고해 일관되게 동작하도록 하기 때문이다.

주의할 점은 **메서드가 던질 수 있는 예외를 각각 @throws 태그로 문서화하되, 비검사 예외는 메서드 선언의 throws 목록에 넣으면 안된다.**

JavaDoc 기준으로는 @throws와 실제 throws절을 사용한 것을 시각적으로 구분해주기 때문에 확실하게 구별해야 한다.

![](https://user-images.githubusercontent.com/59357153/161410380-5a5b0f13-d5b3-465b-bb3a-cf6374b51430.png)
![](https://user-images.githubusercontent.com/59357153/161410698-3caf8495-e547-47ec-9465-8a67605b47ec.png)

해당 메서드를 사용하는 클라이언트는 비검사 예외와 검사 예외를 바로 구분할 수 있다.

### 클래스 설명에 예외 추가

검사, 비검사 모든 예외를 문서화하라고 조언하지만, 현실적으로 불가능한 경우가 있다.

클래스를 수정하면서 새로운 비검사 예외를 던지게 되어도 소스 호환성과 바이너리 호환성이 그대로 유지된다는 것이 가장 큰 이유이다.

예를 들어 다른 사람이 작성한 클래스를 사용하는 메서드가 있다고 가정하자.

그리고 발생 가능한 모든 예외를 문서화했다.

그런데 이후 외부 클래스가 새로운 비검사 예외를 던지도록 수정된다면, 아무 수정도 하지 않은, 이미 문서화를 마친 우리 메서드는 문서에 언급되지 않은 새로운 비검사 예외를 전파하게 된다.

따라서 이런 경우를 대비해 **한 클래스에 정의된 많은 메서드가 같은 이유로 같은 예외를 던지는 경우라면, 그 예외를 각각의 메서드가 아닌 클래스 설명에 추가할 수도 있다.**

```java
/*
 * 클래스의 모든 메서드는 인수로 null이 넘어오면 NullPointerException을 던진다.
*/
public class ExampleClass {
	public void doSomething(Instance i) {
		throw new NullPointerException();
	}
	
	//...
}
```

NullPointerException이 가장 흔한 사례이다.

## 정리

메서드가 던질 수 있는 모든 예외를 문서화하는 것이 좋다.

발생 가능한 예외를 문서화하면 그 메서드를 사용하는 사람들이 문제를 미리 피할 수 있고, 효과적으로 사용할 수 있다.