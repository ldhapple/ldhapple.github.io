---
title: Item24 (멤버 클래스는 되도록 static으로 만들어라)
author: leedohyun
date: 2024-07-10 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 멤버 클래스는 되도록 static으로 만들어라

지난 아이템에서 중첩 클래스에 대한 부분을 보았다.

톱 클래스에서만 사용하는 클래스라면 내부에 private static으로 선언해 접근 권한을 최소화하기도 했다.

중첩 클래스는 이렇게 자신을 감싼 바깥 클래스에서만 쓰여야 한다.

이러한 중첩 클래스의 종류는 4가지가 있다.

- 정적 멤버 클래스
- (비정적) 멤버 클래스
- 익명 클래스
- 지역 클래스

이 중 정적 멤버 클래스를 제외하면 나머지는 내부 클래스에 해당한다.

이 4가지의 중첩 클래스는 언제 어떤 경우에 사용해야 할까?

### 정적 멤버 클래스

클래스 내부에 static으로 선언되는 클래스이다.

- 바깥 클래스의 private 멤버에도 접근 가능하다.
- private으로 선언 시 바깥 클래스에서만 접근 가능하다.
- 흔히 바깥 클래스와 함께 쓰일 때만 유용한 public 도우미 클래스로 쓰인다. (ex - builder)
	- private static 멤버 클래스는 바깥 클래스의 메서드에서 활용되어 재사용성에 기여한다.

```java
public class Car {
	private String model;

	public static class Engine {
		private int power;
		private String fuelType;

		public Engine(int power, String fuelType) {
			this.power = power;
			this.fuelType = fuelType;
		}

		public void start() {
			//...
		}

		public void stop() {
			//...
		}

		//...
	}
	//Car Method
}
```

- 바깥 클래스가 표현하는 객체의 구성요소일 때 사용한다.
- 위 코드를 보고 무엇이 장점인지 잘 이해가 가지 않을 수 있다.
	- private static 중첩 클래스의 예시로 Builder가 있다.
	- Builder 클래스는 바깥 클래스의 필드에 접근하여 인스턴스에 필요한 값을 생성해주지만 외부에서는 Builder 클래스에 접근할 수 없어 안전하다.
- 외부에서는 Car.Builder 인스턴스를 통해서만 제어가 가능하다.
	- 코드의 가독성을 높이고 유지보수성을 개선할 수 있다.

### 비정적 멤버 클래스

정적 멤버 클래스와 구문상의 차이는 static이 붙냐 안붙냐의 차이이지만 의미상으로는 차이가 꽤 크다.

- static이 붙지 않은 멤버 클래스이다.
- 바깥 클래스의 인스턴스와 암묵적으로 연결된다.
	- 비정적 멤버 클래스의 인스턴스 메서드에서 클래스명.this를 사용해 바깥 인스턴스의 메서드나 참조를 가져올 수 있다.
	- 따라서 개념상 중첩 클래스의 인스턴스가 바깥 인스턴스와 독립적으로 존재할 수 있다면 정적 멤버 클래스로 만들어야 한다.
- 바깥 인스턴스 없이 생성할 수 없다.

```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable { 
	// HashMap 내부에 있는 KeySet 클래스 
	final class KeySet extends AbstractSet<K> { 

		public Iterator<K> iterator() { 
			return new KeyIterator(); 
		} 

		public int size() { return size; } 
		public boolean contains(Object o) { return containsKey(o); } 

		public boolean remove(Object o) { 
			return removeNode(hash(o), o, null, false, true) != null; 
		} 

		public void clear() { HashMap.this.clear(); } 
	} 

	// ...
	
	transient Set<K> keySet;
	
	// KeySet 비정적 멤버 클래스의 생성자
	public Set<K> keySet() { 
		Set<K> ks = keySet; 
		if (ks == null) { 
			ks = new KeySet(); 
			keySet = ks; 
		} 
		return ks; 
	} 
	//... 
}
```

- HashMap 클래스의 private 멤버에 쉽게 접근할 수 있다.
	- HashMap의 Key를 관리하는 뷰로 동작한다.

```java
Map<String, Integer> map = new HashMap<>();

map.put("a", 10);
map.put("b", 20);

Set<String> keySet = map.keySet();
System.out.println(keySet);
//[a, b]
```

- 실상은 keySet 클래스의 인스턴스지만, 외부에서는 Set 인터페이스를 구현한 일반 클래스처럼 사용되었다.
- 바깥 클래스 없이 생성될 수 없다.
	- map을 통해서만 생성될 수 있다.
- 이러한 컬렉션 뷰의 용도로 많이 사용된다. 

정리하자면 ***주로 클래스의 인스턴스를 감싸 마치 다른 클래스의 인스턴스처럼 보이게 하는 뷰로 사용한다. (어댑터)***

외부 클래스와 강하게 연결되어 외부 클래스의 인스턴스 상태에 따라 독립적으로 사용될 수 없는 경우에 적합하다.

> 주의사항

멤버 클래스에서 바깥 인스턴스에 접근할 일이 없다면 static을 붙이도록 하자.

- 비정적 멤버 클래스는 바깥 인스턴스와 멤버 클래스 관계를 위한 시간과 공간이 소모된다.
- GC가 바깥 클래스의 인스턴스를 수거할 수 없다면 메모리 누수가 발생한다.
- 참조가 눈에 보이지 않아 문제의 원인을 찾기 어려운 경우가 있다.

예시를 들어보자.

Map 인스턴스는 대부분 각각의 키-값 쌍을 표현하는 엔트리 객체들을 갖고 있다.

이 맵 내부의 엔트리 클래스는 맵과 연관되어 있지만 엔트리의 메서드들 (getKey, getValue 등등)은 맵을 사용하지 않는다.

이런 경우 어떻게 선언해야 될까?

**이 경우 private static 멤버 클래스가 적절하다.** 비정적 멤버 클래스로 표현해도 동작은 한다. 하지만 모든 엔트리가 바깥인 맵으로의 참조를 갖게되어 공간과 시간을 낭비한다.

이해를 돕고자 위의 이 맵의 엔트리와 비정적 멤버 클래스로 선언한 keySet과 비교해보자. 엔트리는 맵을 사용하지 않는다고 한 반면, keySet은 HashMap을 사용한다는 차이가 있다.

### 익명 클래스

- 이름이 없는 클래스이다.
- 바깥 클래스의 멤버가 아니다.
- 쓰이는 시점에 선언과 동시에 인스턴스가 만들어진다.

```java
List<Integer> list = Arrays.asList(10, 5, 6, 7, 1, 3, 4);

// 익명 클래스 사용
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1, o2);
    }
});

// 람다 도입 후
Collections.sort(list, Comparator.comparingInt(o -> o));
```

람다가 등장한 이후 익명 클래스는 람다가 역할을 대체했다.

### 지역 클래스

- 지역 변수를 선언할 수 있는 곳이라면 어느 위치던 선언 가능하다.
	- 유효 범위는 지역 변수와 동일하다. 

네 가지 경우 중 가장 드물게 사용된다.

```java
public class OuterClass { 
	private int outerField = 10; 
	
	void outerMethod() { 
		final int localVar = 20;

		// 지역 클래스
		class LocalClass { 
			void localMethod() { 
				// 외부 클래스의 필드와 지역 변수에 접근 가능
				System.out.println("outerField: " + outerField);
				System.out.println("localVar: " + localVar); 
			} 
		} 

		LocalClass lc = new LocalClass(); 
		lc.localMethod(); 
	} 
}
```

- 멤버 클래스와의 공통점
	- 이름이 있고 반복해서 사용 가능하다.
- 익명 클래스와의 공통점
	- 비정적 문맥에서 사용될 때만 바깥 인스턴스 참조 가능하다.
	- 정적 멤버를 가질 수 없다.
	- 가독성을 위해 짧게 작성해야 한다.

## 정리

중첩 클래스는 4가지로 분류할 수 있고 각각의 쓰임이 있다.

멤버 클래스는 어떤 메서드 밖에서도 사용해야 하거나 메서드 안에 정의하기에는 너무 긴 경우에 사용한다.

- static 멤버 클래스
	- 멤버 클래스에서 바깥 **인스턴스**에 접근할 일이 없다면 static을 붙인다.
	- ex) Map의 Entry (Map을 사용하지 않는다.), Builder
- 비정적 멤버 클래스
	- 주로 컬렉션 뷰로 사용된다. (주로 클래스의 인스턴스를 감싸 마치 다른 클래스의 인스턴스처럼 보이게 하는 뷰 / 어댑터)
	- 멤버 클래스가 바깥 클래스를 참조하는 경우 비정적으로 선언. 그 외는 static.
	- ex) HashMap의 KeySet (Map의 메서드들을 사용했었다.)
- 익명 클래스
	- 람다로 대체
- 지역 클래스

***멤버 클래스를 만들때는 바깥 클래스를 참조하는 경우가 아니라면 무조건 static을 붙여 메모리 낭비 문제를 없애자.***