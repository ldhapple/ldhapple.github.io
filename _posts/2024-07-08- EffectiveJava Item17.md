---
title: Item17 (변경 가능성을 최소화하라)
author: leedohyun
date: 2024-07-08 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 변경 가능성을 최소화하라

불변 클래스란 그 인스턴스의 내부 값을 수정할 수 없는 클래스이다.

불변 인스턴스에 간직된 정보는 고정되어 객체가 파괴되는 순간까지 절대 달라지지 않는다.

자바 플랫폼 라이브러리에는 String, 기본 타입 박싱클래스, BigInteger 등이 불변으로 설계되어 있다.

### 객체를 불변으로 만드는 방법

- **객체의 상태를 변경하는 메서드 (변경자)를 제공하지 않는다.**
	- setter 제공 X
- **클래스를 확장할 수 없도록 한다.**
	- 하위 클래스에서 부주의하게 객체의 상태를 변하게 만들 수 있다.
	- 상속을 막는 간단한 방법은 final이고 생성자를 private으로 만드는 정적 팩토리 메서드 사용도 상속을 막는 방법 중 하나이다.
	- 정적 팩토리 메서드를 사용하면 다음 릴리스에 캐싱을 적용하는 등의 장점도 갖는다.
- **모든 필드를 final로 선언한다.**
	- 불변의 의도를 명확하게 드러낸다.
- **모든 필드를 private으로 선언한다.**
	- 필드가 참조하는 가변 객체를 클라이언트에서 직접 접근해 수정하는 일을 막아준다.
	- 기술적으로는 기본 타입 필드나 불변 객체는 public final로만 선언해도 되지만, 이전 아이템에서 보았듯 내부 표현을 마음대로 바꾸기 힘들다. 
- **자신 외에는 내부의 가변 컴포넌트에 접근할 수 없도록 한다.**
	- 클래스 내부에 가변 객체를 참조하는 필드가 하나라도 있다면 클라이언트에서 그 객체의 참조를 얻을 수 없도록 해야 한다.
		- 방어적 복사를 수행하자.

### 불변 객체로 만들었을 때의 장점

그렇다면 왜 객체를 불변으로 만들어야 할까? 복소수 예시 클래스를 통해 알아보자.

```java
public final class Complex {
	private final double re;
	private final double im;

	public Complex(double re, double im) {
		this.re = re;
		this.im = im;
	}

	public double realPart() { return re; } //실수부 반환
	public double imaginaryPart() { return im; } //허수부 반환

	public Complex plus(Complex c) {
		return new Complex(re + c.re, im + c.im);
	}

	public Complex minus(Complex c) {
		return new Complex(re - cr.re, im - c.im);
	}

	public Complex times(Complex c) {
		return new Complex(re * c.re - im * c.im, re * c.im + im * c.re);
	}

	public Complex dividedBy(Complex c) {
		double tmp = c.re * c.re + c.im * c.im;
		return new Complex((re * c.re + im * c.im) / tmp,
							(im * c.re - re * c.im) / tmp);
	}

	@Override
	public boolean equals(Object o) {
		if (o == this) return true;
		if (!(o instanceof Complex)) return false;
	
		Complex c = (Complex) o;

		return Double.compare(c.re, re) == 0
			&& Double.compare(c.im, im) == 0;
		//복습: 오류 발생할 수 있는 비교연산자 대신, Compare 사용!
	}

	@Override
	public int hashCode() {
		return 31 * Double.hashCode(re) + Double.hashCode(im);
		//복습: 핵심 필드 전부 사용!
	}
	
	@Override
	public String toString() {
		return "(" + re + " + " + im + "i)";
	}
```

- 사칙 연산 메서드들의 특이점
	- 인스턴스를 반환할 때 자신을 수정하지 않고 새 인스턴스를 만들어 반환한다.
	- final로 선언되어 있기 때문에 자신을 수정할 수 없다.
	- 이렇게 피연산자에 함수를 적용해 결과를 반환하지만, 이후 피연산자 자체는 그대로인 프로그래밍 패턴을 함수형 프로그래밍이라고 한다.
	- add같은 동사대신 plus같은 전치사를 사용한 이유도 존재한다. 해당 메서드가 객체의 값을 변경하지 않는다는 사실을 강조한다.

구현을 보면 어색하고, 번거롭다고 느낄 수 있다. 그러나 불변 객체는 확실히 장점이 있다. 장점들을 알아보자.

- **불변 객체는 단순하다.**
	- 불변 객체는 생성된 시점의 상태를 파괴될 때까지 상태를 유지한다. 믿고 사용할 수 있다.
	- 가변 객체는 임의의 복잡한 상태에 놓일 수 있다. 어디서 바뀌는 지 체크할 수 없을 수도 있다.
- **불변 객체는 근본적으로 쓰레드 안전하다.**
	- 쓰레드 안전이기 때문에 따로 동기화할 필요가 없다.
	- 여러 쓰레드가 동시에 사용해도 훼손되지 않는다. 
	- 클래스를 쓰레드 안전하게 만드는 가장 쉬운 방법이기도 하다.
- **불변 객체는 안심하고 공유할 수 있다.**
	- 쓰레드 안전이기 때문에 다른 쓰레드가 영향을 미칠 수 없다.
	- 안심하고 공유할 수 있기 때문에 불변 클래스인 경우 한 번 만든 인스턴스를 최대한 재활용하는 것이 권해진다.
		- 재활용 방법 중 하나는 자주 쓰이는 값을 public static final로 제공하는 것이다.
		- public static final Complex ZERO = new Complex(0, 0);
- **불변 객체는 방어적 복사가 필요 없다.**
	- 값이 바뀌지 않고 원본과 항상 같기 때문이다.
	- 복사 자체가 의미가 없으니 불변 클래스는 clone 메서드나 복사 생성자를 제공하지 않는 것이 좋다.
- **불변 객체는 자유롭게 공유할 수 있음은 물론, 불변 객체끼리는 내부 데이터를 공유할 수 있다.**
	-  BigInteger 클래스를 예시로 들 수 있다. 아래에 설명해놓았다.
-  **객체를 만들 때 다른 불변 객체들을 구성 요소로 사용하면 이점이 많다.**
	- 구조가 아무리 복잡해도 불변식을 유지하기 수월하다.
	- Map의 키와 Set의 원소로 사용해도 무방하다. 원래 내부 값이 바뀌면 불변식이 허물어지는데, 그 걱정을 하지 않아도 되기 때문이다.
- **불변 객체는 그 자체로 실패 원자성을 제공한다.** 
	- 예외가 발생한 후에도 그 객체는 여전히 메서드 호출 전과 같은 유효한 상태여야 한다는 성질을 뜻한다.
	- 상태가 절대 변하지 않으니 불일치 상태에 빠질 가능성이 없다.

> 불변 클래스의 캐싱

불변 클래스는 한 번 만든 인스턴스를 최대한 재활용하면 좋다고 했다. 따라서 캐싱해 활용해도 좋다. 기본 박싱 타입이 그 예시이다.

```java
public final class Integer extends Number  
        implements Comparable<Integer>, Constable, ConstantDesc {
	
	//...

	public static Integer valueOf(int i) {  
	    if (i >= IntegerCache.low && i <= IntegerCache.high)  
	        return IntegerCache.cache[i + (-IntegerCache.low)];  
		return new Integer(i);  
	}
	
	//...
	private static class IntegerCache {  
	    static final int low = -128;  
		static final int high;  
		static final Integer[] cache;  
		static Integer[] archivedCache;
		//...
	}
}
```

- Integer 객체를 캐싱하여 사용하고 있다. IntegerCache 객체는 내부 private 중첩 클래스로 선언되어 있다.
- Integer 클래스는 -128에서 127 사이의 정수 값에 대해 불변 객체를 캐싱하여 최적화를 한다.

> 불변 객체의 내부 데이터 공유 (BigInteger)

BigInteger 클래스는 내부에서 값의 부호(sign)와 크기(magnitude)를 따로 표현한다.

```java
public class BigInteger extends Number implements Comparable<BigInteger> {
	//...
	final int signum; //부호
	final int[] mag; //크기
	
	//...
	public BigInteger negate() {  
	    return new BigInteger(this.mag, -this.signum);  
	}
}
```

- 크기가 같고 부호만 반대인 새로운 BigInteger를 생성하는 negate() 메서드를 볼 수 있다.
- 배열은 이 때 가변이지만, 복사하지 않고 원본 인스턴스와 공유해도 무방하다.
	- 참고로 배열의 final 키워드는 배열의 참조값이 변경되지 않는다는 의미이지, 배열 내 요소가 변경되지 않는 다는 의미가 아니기 때문에 가변이다.
	- 따라서 새로운 BigInteger 인스턴스도 원본 인스턴스가 가리키는 내부 배열을 그대로 가리킨다.

그런데 컬렉션은 내부 객체의 변경 위험때문에 값을 복사하곤 한다. 마찬가지로 BigInteger 클래스의 필드인 배열 내부의 요소를 변경할 수 있는데 왜 그대로 공유해도 된다고 하는걸까? 

BigInteger의 불변성을 보면 된다.

- BigInteger는 msg 배열의 내용을 수정하는 메서드를 제공하지 않는다.
	- 값을 변경하는 모든 작업에도 현재 인스턴스를 수정하는 대신 새 인스턴스를 반환한다.
- BigInteger의 생성자는 private이다. mag와 signum을 변경하여 새 인스턴스를 직접 생성할 수도 없다.

결과적으로 내부 배열을 공유하지만 공유하는 두 BigInteger 인스턴스 모두 배열을 수정할 수 없으므로 내부 데이터를 공유해도 된다는 것이다. 

### 불변 객체, 단점도 있다.

- 값이 다르면 반드시 독립된 객체로 만들어주어야 한다.

당연한 소리이긴 하다. 값을 변경할 수 없으니 값이 다른 것을 의도한다면 독립된 객체를 만들어주어야 한다.

값의 가짓수가 많다면 이 객체들을 모두 만드는데 큰 비용을 치뤄야 한다.

만약 새 독립된 객체를 완성하기까지 단계가 많고, 중간 단계에서 만들고 버려지는 중간 객체들까지 존재한다면 성능 문제가 커질 수 있다.

> 객체 생성에 단계를 거치는 성능 문제 해결 방법

- 다단계 연산들을 예측하여 기본 기능으로 제공하는 방법
	- 각 단계마다 객체를 생성하지 않고 복잡한 연산을 예측해 제공해준다.
- 클래스를 public으로 제공한다.
	- ex) String 클래스

## 정리

클래스는 꼭 필요한 경우가 아니라면 불변이어야 한다. 장점이 훨씬 많고, 단점도 특정 상황에서의 잠재적인 성능 저하일 뿐이다.

불변으로 만들 수 없는 클래스라도 변경할 수 있는 부분을 최소한으로 줄여야 한다. 꼭 변경해야 할 필드를 제외한 모두를 final로 선언하는 것이 좋다.

생성자는 불변식 설정이 모두 완료된, 초기화가 완벽히 끝난 상태의 객체를 생성해야 한다. 확실한 이유가 없다면 생성자와 정적 팩터리 외에는 그 어떤 초기화 메서드도 public으로 제공해서는 안된다.

***종합하면 다른 합당한 이유가 없다면 모든 필드는 private final 이어야 한다.***