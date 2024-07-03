---
title: Item9 (try-finally 보다 try-with-resources를 사용하라)
author: leedohyun
date: 2024-07-02 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## try-finally 보다 try-with-resources를 사용하라

자바 라이브러리에는 close() 메서드를 호출해 직접 닫아주어야 하는 자원이 많다.

- InputStream
- OutputStream
- java.sql.Connection

위와 같은 자원들이 좋은 예시이다.

이러한 자원 닫기는 클라이언트가 놓치기 쉬워 예측할 수 없는 성능 문제로 이어진다. 이런 자원 중 상당수가 안전망으로 finalizer를 활용하고 있지만 확실성이 없다. (Item 8)

### try - finally를 써서 자원 닫기

```java
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

try {
	String str = br.readLine();
} finally {
	br.close();
}
```

위의 경우 전혀 문제가 없어 보인다. 그러나 만약 자원을 하나 더 사용하게 된다면?

```java
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

try {
	BufferedWriter bw = new BufferedWriter(new OutputStreamReader(System.out));
	try {
		String str = br.readLine();
		bw.write(line);
	} finally {
		bw.close();
	}
} finally {
	br.close();
}
```

자원이 단 하나만 추가되었는데도 코드가 굉장히 지저분하다.

지저분한 것 이외에도 큰 문제가 있다.

- try블록과 finally 블록에서 예외가 모두 발생할 수 있다.
	- 만약 try 블록을 실행하던 중 기기에 문제가 생긴다면 br.readLine() 메서드가 정상적으로 실행되지 못하고 예외를 던지게 될 것이다.
	- 기기에 문제가 생겼기 때문에 finally 블록의 close() 메서드도 예외를 발생시킨다.
	- 이런 상황이라면 finally 블록에서 발생한 예외가 try 블록의 예외를 삼켜 처음 발생한 예외에 대한 정보가 발생하지 않는다.

이렇게 되면 디버깅이 힘들다. 물론 예외 발생 시 이전 예외를 기록하도록 코드를 수정할 수는 있지만 코드가 매우 지저분해질 것이다.

***종합적으로 try-finally 구문은 가독성을 떨어뜨릴 위험성, 그리고 예외 처리 로직에 대한 결함의 문제가 있다.***

### try-with-resources의 사용

위와 같은 try-finally 구문의 문제는 자바 7이 제공한 try-with-resources 덕분에 해결되었다.

***참고로 try-with-resource를 사용하기 위해서는 사용하는 자원이 AutoCloseable 인터페이스를 구현해야 한다.***

```java
public interface AutoCloseable {
	void close() throws Exception;
}
```

자바 라이브러리와 서드파티 라이브러리들의 수많은 클래스와 인터페이스가 이미 AutoCloseable을 구현하거나 확장해두었다.

만약 우리가 자원을 닫아야하는 클래스를 커스텀해야 한다면 AutoCloseable을 반드시 구현해야 할 것이다.

```java
try (BufferedReader br = new BufferedReader(new InputStream(System.in))) {
	BufferedWriter bw = new BufferedWriter(new OutputStreamReader(System.out));
	
	String str = br.readLine();
	bw.write();
}
```

- try 키워드 괄호 안에서 자원을 획득한다.
	- br 획득
	- bw 획득
- try 블록을 벗어나면 자원들이 자동으로 닫힌다. 이 때 자원들은 선언 순서의 역순으로 닫힌다.
	- bw 닫힘
	- br 닫힘
- try 블록 내에서 예외가 발생할 경우 catch 블록으로 전달된다. 이 후 자원들은 예외 발생 여부와 관계없이 닫히게 된다.
	- try문을 중첩하지 않고도 다수의 예외를 처리할 수 있다.

 위와 같이 가독성이 매우 좋아졌다. 심지어 가독성 뿐만 아니라 readLine과 close 모두에서 예외가 발생하는 경우 close 호출 시 발생하는 예외는 숨겨지고 readLine에서의 예외가 기록된다.

> 숨겨진 예외의 처리

```java
static class CustomReader extends BufferedReader {  
  public CustomReader(StringReader reader) {  
	  super(reader);  
  }  
  
  @Override  
  public void close() throws IOException {  
	  super.close();  
	  throw new IOException("close() 호출 예외 발생");
  }  
}
```

```java
try (BufferedReader br = new CustomReader(new StringReader("abc"))) {  
	  throw new IOException("try 구문 내에서 예외 발생");  
} catch (Exception e) {  
	  e.printStackTrace();  
	  System.err.println("예외 발생: " + e.getMessage());  
	  
	  Throwable[] suppressed = e.getSuppressed();  
	  for (Throwable t : suppressed) {  
	  System.err.println("Suppressed 예외: " + t.getMessage());  
  }  
}
```

- CustomReader를 생성했다.
- 그런데 try 구문에서 예외를 발생시켰다.
	- try 블록에서 벗어났기 때문에 자원을 close() 해준다.
- close()할때도 예외가 발생되었다.

![](https://blog.kakaocdn.net/dn/dQbVau/btsIkCQNV2O/billkypr8gkivngEfUW6X0/img.png)

stackTrace를 해보면 위와 같이 Suppressed라고 되며 close() 메서드의 예외는 숨겨진 것을 볼 수 있다.

이렇게 suppressed 상태가 된 예외는 Java 7부터 도입된 getSuppressed 메서드를 통해 가져와서 사용할 수 있다.

```java
Throwable[] suppressed = e.getSuppressed();  
for (Throwable t : suppressed) {  
   System.err.println("Suppressed 예외: " + t.getMessage());  
}
```

## 정리

close()를 통해 자원을 회수해야 하는 객체를 다룰 때는 try-finally보다 try-with-resources를 사용하면 된다. 어떠한 경우에도 코드는 더 짧고 분명해지며 만들어지는 예외 정보도 훨씬 유용하게 사용할 수 있다.

try-with-resources를 사용할 경우 AutoCloseable 인터페이스를 구현해야 한다는 점은 잊으면 안된다.