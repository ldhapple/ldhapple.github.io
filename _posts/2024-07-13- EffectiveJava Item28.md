---
title: Item28 (배열보다는 리스트를 사용하라)
author: leedohyun
date: 2024-07-13 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 배열보다는 리스트를 사용하라

왜 배열보다는 리스트를 사용하라는걸까? 

리스트에는 사용할 수 있는 API가 많아서? 리스트는 가변 길이로 사용할 수 있어서?

결론부터 말하자면 이유는 리스트와 배열 간의 관계가 아닌 배열과 제네릭 타입에 중요한 차이들이 있기 때문이다. 

그 차이들을 알아보자.

### 공변(covariant)과 불공변(invariant)

결론부터 말하면 배열은 공변이고, 제네릭은 불공변이다.

공변이란 함께 변한다는 뜻으로, Sub라는 클래스가 Super라는 클래스의 하위 클래스라고 가정할 때 Sub[]는 Super[]의 하위 타입이 된다.

반면 불공변은 같이 변하지 않는다는 뜻으로 만약 서로 다른 타입 타입1, 타입2가 있을 때 List<타입1>은 List<타입2>의 하위 타입도 아니고 상위 타입도 아니다.

정의만 봤을 때 공변이 더 유연하게 활용할 수 있는 여지가 있는 것 아닌가 싶다. 그러나 문제를 보자.

```java
Object[] objectArr = new Long[1];
objectArr[0] = "문자열입니다.";
```

- 위 코드는 문법상 허용되는 코드이다.
	- 컴파일 에러가 나지 않는다.
- 그러나 런타임 시 ArrayStoreException이 발생한다.

반면 제네릭의 경우를 보자.

```java
List<Object> objectArr = new ArrayList<Long>();
objectArr.add("문자열입니다.");
```

- 컴파일 시 에러가 발생한다.
	- Object 리스트에 Long형 ArrayList를 할당하는 것부터 안된다.
	- The Long class wraps a value of the primitive type long in an object.

두 결과를 보면 배열이든 리스트든 Long용 저장소에 String을 넣을 수 없는 것은 마찬가지이다.

하지만 배열에서는 그 실수를 런타임에서 알 수 있다는 것과 리스트에서는 컴파일 시 바로 알아채는 것에서 차이가 있다.

컴파일 오류가 가장 좋은 오류이다.

### 배열은 실체화(reify) 된다.

제네릭은 타입 정보가 런타임에는 소거된다. 컴파일 시 타입을 검사했다면 런타임에는 알 수 없다는 뜻이다.

이러한 소거는 제네릭이 지원되기 전의 레거시 코드와 제네릭 타입을 함께 사용할 수 있도록 하는 메커니즘이다.

반면 배열은 위 예시 코드에서도 보았다시피 런타임에도 자신의 원소 타입을 인지하고 확인한다.

이러한 차이로 배열과 제네릭은 잘 어우러지지 못한다. 배열은 제네릭 타입, 매개변수화 타입, 타입 매개변수로 사용할 수 없다.

```java
new List<E>[]
new List<String>[]
new E[]
//전부 불가!
```

이렇게 제네릭 배열을 막아놓은 이유는 타입 안전성 때문이다.

이를 만약 허용한다면 컴파일러가 자동 생성한 형변환 코드에서 런타임에 ClassCastException이 발생할 수 있다. 런타임에 ClassCastException이 발생하는 것을 막겠다는 제네릭의 취지에 벗어나는 것이다.

코드 예시들을 보면서 제네릭 배열을 만약 허용했다면 무슨 일이 발생할 지 알아보자.

```java
List<String>[] strLists = new ArrayList[1]; // 허용 O, 코테풀때 종종 쓰는데?
// 배열의 각 요소가 ArrayList<String> 객체를 참조할 뿐,
// List<String> 배열 자체를 생성하는 것이 아니기 때문에 제네릭 배열 생성을 간접적으로 지원하는 것.
// 반면 stringLists는 직접적으로 제네릭 배열을 생성하는 것. 타입 안정성때문에 허용 X

List<String>[] stringLists = new List<String>[1]; // (1) - 허용 X
List<Integer> intList = List.of(42); // (2)
Object[] objects = stringLists; // (3)
objects[0] = intList; // (4)
String s = stringLists[0].get(0); // (5)
```

- 제네릭 배열을 생성하는 (1)이 허용된다고 가정해보자.
- (2)는 원소가 하나인 Integer List가 생성된다.
- (3)은 (1)에서 생성한 String List 배열을 Object 배열에 할당한다.
- (4)는 (2)에서 생성한 Integer List의 인스턴스를 Object 배열의 첫 원소로 저장한다.
	- 제네릭은 소거되기 때문에 (2)의 Integer List가 List가 되고 (1)의 String List 배열은 List[]가 되기 때문에 가능하다.
	- String List만 저장하겠다고 했던 배열은 Integer List를 저장하고 있다.
- (5)에서 제네릭 소거로 인한 (4)의 허용때문에 String 원소를 기대하지만 Integer 원소를 갖게되는 문제가 발생한다.
	- String - Integer 에서 발생하는 ClassCastException이 발생한다.

따라서 위와 같은 문제가 없으려면 (1)에서 제네릭 배열을 생성하게 두면 안된다.

```java
E, List<E>, List<String>
```

- 위와 같은 타입들을 **실체화 불가 타입**이라고 한다.
	- 실체화되지 않아 컴파일 타임보다 런타임에 타입 정보를 적게 가진다는 뜻이다.
	- 실체화될 수 있는 타입은 비한정적 와일드카드 타입 (?) 뿐이다.
		- 그러나 배열을 비한정적 와일드카드 타입으로 만들어 유용하게 쓰기는 어렵다.

### 배열을 제네릭으로 쓸 수 없어 귀찮아지는 경우들 

**제네릭 컬렉션에서는 보통 자신의 원소 타입을 담은 배열을 반환하는게 불가능**하다.

```java
List<String> stringList = new ArrayList<>(); 
stringList.add("Hello"); 

String[] arr = stringList.toArray(new String[stringList.size()]);
//불가능
```

- 제네릭 타입은 런타임에 소거되기 때문에 toArray() 메서드가 어떤 타입을 반환할 지 알 수 없다.
	- 일반적으로 Object 형태로 반환되고, 이를 컴파일러가 강제로 String으로 캐스팅하려할 때 ClassCastException이 발생할 수 있어 막아둔 것이다.

물론 우회하는 방법도 있긴하다. 추후 아이템에서 설명한다.

그리고 또 **제네릭 타입과 가변인수 메서드를 함께 쓰면 해석하기 어려운 경고 메시지를 받게 된다.**

```java
public static <T> void printElements(List<T> elements) { 
	for (T element : elements) { 
		System.out.print(element + " "); 
	} 
	System.out.println(); 
}

List<String> stringList = Arrays.asList("Hello", "World"); 
List<Integer> integerList = Arrays.asList(1, 2, 3);

printElements(stringList);
```

![](https://blog.kakaocdn.net/dn/dNAcd3/btsIyIbl9Xe/rTVLHAMhSTKkaTPMPQBIc0/img.png)

- 가변인수 메서드를 호출할 때마다 가변인수 매개변수를 담을 배열이 하나 만들어진다.
	- 이 때, 해당 배열의 원소가 실체화 불가 타입이라면 경고가 발생하는 것이다.
	- 이 문제는 @SafeVarargs 애노테이션으로 대처할 수 있다.

**보통 배열로 형변환 시 제네릭 배열 생성 오류나 비검사 형변환 경고가 뜨는 경우의 대부분은 배열인 E[] 대신 List를 사용하면 된다.**

코드가 복잡해지고 성능이 나빠질 수 있지만 타입 안전성과 상호 운용성이 좋아진다.

```java
public class Chooser {
    private final Object[] choiceArray;
    
    public Chooser(Collection choices) {
        choiceArray = choices.toArray();
    }
    
    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray(rnd.nextInt[choiceArray.length)];
    }
}
```

- 위 클래스를 사용하려면 choose() 메서드를 호출할 때마다 반환된 Object를 원하는 타입으로 형변환해야 한다.
	- 다른 타입의 원소가 들어있다면 형변환 오류를 발생시킬 것이다.

제네릭으로 만들어보자.

```java
public class Chooser<T> {
    private final T[] choiceArray;
    
    public Chooser(Collection<T> choices) {
        choiceArray = choice.toArray();
    }

	public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray(rnd.nextInt[choiceArray.length)];
    }
}
```

- 위에서 보았던 대로 제네릭 컬렉션에서는 보통 자신의 원소 타입을 담은 배열을 반환하는게 불가능하다.
	- 따라서 toArray()에서 오류가 발생한다.

소거때문에 무슨 타입인지 알 길이 없기 때문에 그렇다고 했다. 따라서 이런 경우 그냥 배열 대신 제네릭 컬렉션을 사용하면 문제는 해결된다.

```java
public class Chooser<T> {
    private final List<T> choiceList;
    
    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }
    
    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}
```

- 컬렉션은 제네릭 타입을 지원해 런타임에도 요소의 타입 정보를 유지한다.
	- 따라서 타입 안전성을 보장하기 때문에 문제는 해결된다.

## 정리

배열은 공변이고 실체화된다. 반면 제네릭은 불공변이고 타입 정보가 소거된다.

이 차이 때문에 배열은 런타임에는 타입 안전하고 컴파일 타임에는 그렇지 않다. 제네릭은 그 반대이다.

따라서 배열과 제네릭은 섞어 쓰기 어렵다.

따라서 제네릭과 배열을 같이 사용하고 싶다면, 리스트를 사용해보자.