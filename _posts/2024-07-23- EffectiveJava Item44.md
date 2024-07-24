---
title: Item44 (표준 함수형 인터페이스를 사용하라)
author: leedohyun
date: 2024-07-23 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 표준 함수형 인터페이스를 사용하라

자바가 람다를 지원함에 따라 API를 작성하는 모범 사례도 크게 바뀌었다.

예를 들면 상위 클래스의 기본 메서드를 재정의해 원하는 동작을 구현하는 템플릿 메서드 패턴의 매력은 크게 줄었다.

```java
// 템플릿 메서드 패턴
abstract class Test {
	public final void execute() {
		prepare();
		executeTest();
		cleanUp();
	}

	protected abstract void executeTest();
	
	// prepare(), cleanUp()...
}

class extendTest extends Test {
	@Override
	protected void executeTest() {
		System.out.println("execute Test");
	}
}
```

```java
Test test = new Test();
test.execute();
```

```java
//람다 사용 (함수 객체를 받는 생성자 사용)
class Test {
	private final Consumer<Test> executeTest;
	
	public Test(Consumer<Test> executeTest) {
		this.executeTest = executeTest;
	}

	public final void execute() {	
		prepare();
		executeTest.accept(this);
		cleanUp();
	}
	
	//...
}
```
```java
Test test = new Test(t -> System.out.println("execute Test");
test.execute();
```

- 과거의 템플릿 메서드 패턴대신 같은 효과의 함수 객체를 받는 정적 팩토리나 생성자를 제공하여 대체한다.
	- 이를 일반화해서 말하면 함수 객체를 매개변수로 받는 생성자와 메서드를 더 많이 만들어야 한다고 할 수 있다.
- 위의 예시에서는  동작을 정의하기 위해 Consumer 함수형 인터페이스를 사용했다.
	- 값을 반환해야 하는 경우라면 Function 등을 쓸 수 있을 것이다.
	- 주의할 점은 이렇게 람다를 이용하도록 생성자 등을 제공하는 방식으로 구현한다면 함수형 매개변수 타입을 올바르게 선택해야 한다는 점이다.

그런데 위의 예시에서는 람다로 사용했을 때의 장점을 크게 느낄 수 없다. 오히려 경우에 따라 더 번거로워 보인다.

이번에는 LinkedHashMap을 이용한 예시를 보자.

### LinkedHashMap -> 람다 (불필요한 함수형 인터페이스를 만들지 말자)

LinkedHashMap의 내부를 보면 removeEldestEntry()라는 메서드가 존재한다. 

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
   ...
   // HashMap의 Put 메서드에서 마지막에 afterNodeInsertion을 호출한다.
   afterNodeInsertion(evict);
}

void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    // removeEldestEntry을 통해서 오래된 데이터를 지울 것인지에 대한 여부를 체크한다.
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// LinkedHashMap 현재 내부 구현 -> 항상 false를 리턴한다.
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

- 현재 removeEldestEntry() 메서드는 항상 false를 반환하도록 되어 있다.
- 이러한 removeEldestEntry() 메서드를 재정의하면 캐시로 사용할 수 있다.

우선 람다를 사용하지 않는 버전을 보자.

```java
class OverrideLinkedHashMap extends LinkedHashMap<String, Integer> {

    private final int maxSize;

    OverrideLinkedHashMap(int maxSize) {
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > maxSize;
    }
}
```
```java
@Test
void Eldest_Entry_Removal_Override_Test() {
    OverrideLinkedHashMap overrideLinkedHashMap = new OverrideLinkedHashMap(1);
    overrideLinkedHashMap.put("1번", 1);
    overrideLinkedHashMap.put("2번", 2);

    assertThat(overrideLinkedHashMap.size()).isEqualTo(1);
    assertThat(overrideLinkedHashMap.get("1번")).isEqualTo(null);
}
```

- removeEldestEntry() 메서드를 재정의해 생성자에서 받은 maxSize 값보다 현재 size가 크다면 true를 반환하도록 했다.
	- 맵에서 가장 오래된 원소를 제거한다. 즉, 생성자에서 받은 maxSize 값만큼의 개수만큼 최근 원소를 유지한다.
	- 가장 최근의 원소들을 저장하므로 사이즈가 한정된 캐시로 사용할 수 있는 것이다.

Test 코드를 보면 알 수 있다시피 동작은 문제없이 된다. 하지만 람다를 활용하면 더 깔끔하다.

```java
@FunctionalInterface
public interface EldestEntryRemovalFunction<K, V> {
    boolean remove(Map<K, V> map, Map.Entry<K, V> eldest);
}
```

- 함수형 인터페이스를 선언했다.
- 그런데 remove에 Entry뿐 아니라 Map을 넘긴다. 이유가 뭘까?
	 - removeEldestEntry() 선언을 보면 Map.Entry를 받아 boolean을 반환해야 할 것 같지만 꼭 그렇지만은 않다.
	- size()를 호출해 맵 안의 원소 수를 알아내는데, 이는 인스턴스 메서드이기에 가능한 방식이다.
	- 그러나 생성자에 넘기는 함수 객체는 해당 Map의 인스턴스 메서드가 아니다. 팩토리나 생성자를 호출할 때는 맵의 인스턴스가 존재하지 않는다.
	- 따라서 맵은 자기 자신도 함수 객체에 건네주어야 하기 때문에 Map도 같이 넘기는 것이다.

```java
public class FunctionalInterfaceLinkedHashMap<K, V> extends LinkedHashMap<K, V> {  
	private final EldestEntryRemovalFunction<K, V> removalFunction;  
	  
	public FunctionalInterfaceLinkedHashMap(EldestEntryRemovalFunction<K, V> removalFunction) {  
		this.removalFunction = removalFunction;  
	}  
	  
	@Override  
	protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {  
		return removalFunction.remove(this, eldest);  
	}  
}
```
```java
@Test
void Eldest_Entry_Removal_Interface_Test() {
    FunctionalInterfaceLinkedHashMap fiLinkedHashMap = new FunctionalInterfaceLinkedHashMap(
            (map, eldest) -> map.size() > 1
    );
    fiLinkedHashMap.put("1번", 1);
    fiLinkedHashMap.put("2번", 2);

    assertThat(fiLinkedHashMap.size()).isEqualTo(1);
    assertThat(fiLinkedHashMap.get("1번")).isEqualTo(null);
}
```

- 새로운 함수형 인터페이스를 만들고 활용하여 함수 객체를 받는 생성자를 제공하도록 바꾸었다.
- 해당 클래스를 처음 보는 사람이 보았을 때 어떤 동작을 하는 지 비교적 파악하기 쉽다.
- 가장 큰 부분은 람다를 사용하면 넘기는 함수 객체로 remove의 조건을 유연하게 바꿀 수 있다.
	- 위 예시는 map.size() > 1 정도의 경우라 와닿지 않을 수 있지만 단순히 크기로만 조건을 다는 것이 아닌 복잡한 조건이라면?
	- 만약  람다를 사용하지 않았을 경우에는 조건이 maxSize의 수가 바뀌는 정도의 변경이 아닌 이상 새로운 클래스를 만들어 LinkedHashMap을 확장하는 새로운 클래스가 필요할 것이다.

람다 등장 전의 방식보다 람다를 사용한 방식의 장점은 알았다. 그러나 한 가지 주의할 점이 있다.

위 예시에서는 알맞게 사용할 수 있도록 새로운 함수형 인터페이스를 정의했다. 하지만 그럴 필요 없다. 이미 자바 표준 라이브러리에 같은 모양의 인터페이스가 준비되어 있다.

***필요한 용도에 맞는 인터페이스가 있다면 직접 구현하지 않고 표준 함수형 인터페이스를 활용하는 것이 좋다.***

- API가 다루는 개념의 수가 줄어들어 익히기 더 쉽다.
- 표준 함수형 인터페이스들은 유용한 디폴트 메서드를 많이 제공해 다른 코드와의 상호 운용성도 좋아진다.

```java
class BiPredicateLinkedHashMap<K, V> extends LinkedHashMap<K, V> {

    private final BiPredicate<Map<K, V>, Map.Entry<K, V>> removalBiPredicate;

    BiPredicateLinkedHashMap(BiPredicate<Map<K, V>, Map.Entry<K, V>> removalBiPredicate) {
        this.removalBiPredicate = removalBiPredicate;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return removalBiPredicate.test(this, eldest);
    }
}
```

- 표준 함수형 인터페이스 BiPredicate를 사용하여 같은 동작을 쉽게 구현할 수 있다.

### 표준 함수형 인터페이스 정리

표준 함수형 인터페이스의 개수는 모두 43개이다.

모두 기억하기는 어렵고, 기본 인터페이스 6개를 기억하자. 나머지는 기본 인터페이스를 통해 유추할 수 있을 것이다.

정리하기 전 우선 이 부분을 인지하자. 기본 인터페이스들은 모두 참조 타입용이다. -> 원시 타입을 직접 사용할 수 없다.

- UnaryOperator
	- 반환값과 인수의 타입이 같은 함수를 뜻한다. 인수가 1개이다.
- BinaryOperator
	- 반환값과 인수의 타입이 같은 함수를 뜻한다. 인수가 2개이다. 
- Predicate
	- 인수 하나를 받아 boolean을 반환하는 함수를 뜻한다.
- Function
	- 인수와 반환 타입이 다른 함수를 뜻한다.
- Supplier
	- 인수를 받지 않고 값을 반환(혹은 제공)하는 함수를 뜻한다.
- Consumer
	- 인수 하나를 받고 반환값은 없는 함수를 뜻한다.
   
#### UnaryOperator

반환값과 인수의 타입이 같은 함수. 인수가 1개이다.

- UnaryOperator
	- T apply(T t)
- IntUnaryOperator
	- int applyAsInt(int value)
- LongUnaryOperator
	- long applyAsLong(long value)
- DoubleUnaryOperator
	- double applyAsDouble(double value)

```java
UnaryOperator<String> unaryOperator = String::toLowerCase;
unaryOperator.apply("ABDJ"); //인수 1개, 반환값과 인수의 타입이 같다.
```

#### BinaryOperator

반환값과 인수의 타입이 같은 함수. 인수가 2개이다.

- BinaryOperator
	- T apply(T t1, T t2)
- IntBinaryOperator
	- int applyAsInt(int value1, int value2)
- LongBinaryOperator
	- long applyAsLong(long value1, long value2)
- DoubleBinaryOperator
	- double applyAsDouble(double value, double value2)

```java
BinaryOperator<BigInteger> binaryOperator = BingInteger::add;
binaryOperator.apply(BigInteger.ONE, BigInteger.TWO); 
//인수 2개, 반환값과 인수의 타입이 같다.
```

#### Predicate

인수를 받아 boolean을 반환하는 함수를 뜻한다.

- Predicate
	- boolean test(T t)
- BiPredicate
	- boolean test(T t, U u)
- IntPredicate
	- boolean test(int value)
- LongPredicate
	- boolean test(long value)
- DoublePredicate
	- boolean test(double value)

```java
BiPredicate<String, Integer> predicate = (str, num) -> str.length() <= num;
predicate.test("abc", 3);
```    

#### Function

인수 타입과 반환 타입이 다른 함수를 뜻한다.

- Function
	- R apply(T t)
- BiConsumer
	- R apply(T t, U u)
- PFunction
	- R apply(p value)
- PtoQFunction
	- Q applyAsQ(p value)
- ToPFunction
	- P applyAsP(T t)
- ToPBiFunction
	- P applyAsP(T t, U u)

여기서 P와 Q는 기본 자료형(Primitive Type)을 뜻한다.

```java
Function<int[], List> func = Arrays::asList;
int[] arr = {1, 2, 3};
List<Integer> list = func.apply(arr);
```     

> 로 타입 쓰지 말라며? (기본 티입과 박싱 타입)

위 Function 예시에서 반환 타입으로 로(raw) 타입을 썼다. Function의 인자로 Integer List를 넣으면 컴파일 오류가 나타난다.

이유를 알아보자. 

```java
public static <T> List<T> asList(T... a)
```

- Arrays::asList가 제네릭 메서드로 정의되어 있는데 이 때 이 메서드는 가변인수를 받아 제네릭 리스트를 반환한다.
- 하지만 제공되는 인자는 원시 타입 배열이다. 제네릭은 원시 타입 배열을 직접적으로 처리할 수 없다.

따라서 Function에서 컴파일러는 int[]를 제네릭 T로 간주하고, 이로 인해 반환타입도 사실 아래와 같아야 한다.

```java
Function<int[], List<int[]>> func = Arrays::asList;
```

- 원시 타입 배열을 제네릭 타입으로 처리할 수 없다.
	- 따라서 로 타입으로 제한을 우회한 것이다.

근데 이렇게 하면 우리가 원하는 결과가 아니다. 우리는 int 배열의 원소들을 원소로 가지는 Integer List를 반환받기를  원한다. 위 처럼 했을 경우 List의 첫 번째 원소로 int 배열이 통째로 들어갈 뿐이다.

```java
Function<Integer[], List<Integer>> func = Arrays::asList;
```

위 코드는 우리가 의도한대로 정상적으로 동작한다. 이유가 뭘까?

- 제네릭 가변 인수는 참조 타입 배열은 직접적으로 처리할 수 있지만 원시 타입 배열은 직접 처리할 수 없다.
	- 제네릭은 타입이 소거된다고 했다. 런타임에는 로 타입(Function)으로 변환된다. 원시 타입 배열은 이를 처리할 수 있는 타입 정보가 없는 것이다.
	- 따라서 제네릭 타입 파라미터는 항상 참조 타입이어야 한다.

```java
Function<int[], List<int[]>> 에서 int[]는 제네릭 타입 파라미터로 사용된다.
근데 컴파일 시 타입이 소거된다고 했다. 그렇다면 Function의 원시 타입만 남는 것이다.
int[]에 대한 타입 정보가 사라졌다.
```

- 참고로 여기서 int[]를 막아놓지 않은 이유는 제네릭 타입이 기본적으로 참조 타입을 기대하는데, 배열 자체는 결국 참조 타입이기 때문이다.

```java
Function<Integer[], List<Integer>> 에서 Integer[]는 타입 소거 후에도 일관성을 유지한다.
제네릭은 참조 타입을 지원한다.
```

- 여기서 타입 소거 후에도 일관성을 유지한다는 말은 타입 정보를 유지한다는 말이 아니다.
- Integer 배열은 배열 요소가 참조 타입이다.
	- 배열 자체가 참조 타입 객체를 포함하고 있다.
	- 제네릭 타입 파라미터로 사용될 때, 타입 소거가 되어도 배열의 참조 타입 구조는 유지된다. 
	- 배열이 객체의 참조를 포함하고 있기 때문에 객체의 참조 타입으로 일관성을 유지할 수 있는 것이다.
- 반면 원시 타입 배열은?
	- 배열 자체가 원시 타입 값을 직접 포함하고 있다.
	- 제네릭 타입 파라미터로 사용될 수 없고, 타입 소거 후에 일관성을 유지할 수 없다.

즉, 정리하자면 제네릭 타입에 원시 타입을 섞어 쓰면 안된다. 제네릭의 타입 소거 메커니즘이 원시 타입 배열과 호환되지 않는다.

#### Supplier

인수가 필요하지 않고 값을 반환한다.

- Supplier
	- T get()
- BooleanSupplier
	- boolean getAsBoolean()
- IntSupplier
	- int getAsInt()
- LongSupplier
	- long getAsLong()
- DoubleSupplier
	- double getAsDouble()

```java
Supplier<String> supplier= () -> "Hello";
Supplier<Point> supplier = () -> new Point(1, 7);
Supplier<Integer> supplier = () -> {
	Random random = new Random();
	return random.nextInt(100);
}
```

#### Consumer

인수를 받고 반환값은 없는 함수를 뜻한다.

- Consumer
	- void accept(T t)
- BiConsumer
	- void accept(T t, U u)
- IntConsumer
	- void accept(int value)
- LongConsumer
	- void accept(long value)
- DoubleConsumer
	- void accept(double value)  
- ObjIntConsumer
	- void accept(T t, int value)
- ObjLongConsumer
	- void accept(T t, long value)
- ObjDoubleConsumer
	- void accept(T t, double value)     

```java
Consumer consumer = System.out::print;
cousumer.accept("hello");
```


### 함수형 인터페이스를 직접 구현해야 하는 경우

대부분의 경우 제공되는 함수형 인터페이스를 그대로 사용하는 것이 나을 것이다. 그렇다면 언제 함수형 인터페이스를 직접 구현해야 할까?

- 표준 함수형 인터페이스 중 필요한 용도에 맞는 것이 없는 경우

```java
@FunctionalInterface
public interface TriPredicate<T, Q, R> {
	boolean test(T t, Q q, R r);
}
```

- 구조적으로 똑같은 표준 함수형 인터페이스가 있지만 아래 조건을 따르는 경우
	- 자주 쓰이며 이름 자체가 용도를 명확하게 설명해주는 경우
	- 반드시 따라야하는 규약이 있는 경우
	- 유용한 디폴트 메서드를 제공할 수 있는 경우

```java
@FunctionalInterface
public interface Comparator<T> {
      
    // 반드시 따라야 하는 규약
    // Compares its two arguments for order. 
    // Returns a negative integer, zero, or a positive integer as 
    // the first argument is less than, equal to, or greater than the second.
    
    int compare(T o1, T o2);
    
    // 유용한 디폴트 메서드
    default Comparator<T> reversed() {
        return Collections.reverseOrder(this);
    }
}
```

> 주의할 점

함수형 인터페이스를 구현할 때는 항상 @FunctionalInterface 애노테이션을 사용해야 한다.

- 해당 클래스의 코드나 설명 문서를 읽을 사람에게 해당 인터페이스가 람다를 사용하기 위한 것임을 알려줄 수 있다.
- 해당 인터페이스가 추상 메서드를 하나만 가지고 있어야 컴파일되도록 해준다.
- 유지보수 과정에서 실수로 메서드를 추가하지 못하게 막아준다.

## 정리

API를 설계할 때 람다를 고려하자. 템플릿 메서드보다 생성자나 팩토리에 함수객체를 제공하는 방식과 같은 것이 그 예시이다.

가독성과 유연함이 증가될 수 있다.

단, 이 경우 표준 함수형 인터페이스를 사용하자. 기본 제공되는 인터페이스만으로도 충분하다. 흔치는 않지만 직접 구현해야 확실하게 더 나은 경우에만 직접 구현하자.