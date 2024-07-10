---
title: Item20 (추상 클래스보다는 인터페이스를 우선하라.)
author: leedohyun
date: 2024-07-09 18:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 추상 클래스보다는 인터페이스를 우선하라.

자바는 추상 클래스와 인터페이스라는 일종의 타입을 제공하는 다중 구현 메커니즘을 제공한다.

자바 8부터는 인터페이스도 디폴트 메서드를 제공할 수 있게 되어 두 메커니즘 모두 인스턴스 메서드를 구현 형태로 제공할 수 있다.

둘의 차이점을 보자.

- 차이점
	- 추상 클래스
		- 추상 클래스가 정의한 타입을 구현하는 클래스는 반드시 추상 클래스의 하위 클래스여야 한다.
		- 이 때문에 새로운 타입을 정의하는데 큰 제약이 있다. 자바는 단일 상속만 지원하기 때문이다.
	- 인터페이스
		- 인터페이스가 선언한 메서드를 정의하고 규약을 잘 지켰다면 추상 클래스와는 다르게 어떤 클래스를 상속했더라도 같은 타입으로 취급한다. 
		
간단하게 차이점을 보았다. 비슷한데도 훨씬 제약이 줄어든 것이 인터페이스임을 확인할 수 있었다.


## 인터페이스의 장점

그렇다면 인터페이스의 장점을 자세히 알아보자.

### 기존 클래스에도 새로운 인터페이스를 쉽게 구현해 넣을 수 있다.

인터페이스가 요구하는 메서드가 없을 경우 추가하고, 클래스 선언에 implements 구문만 추가하면 쉽게 넣을 수 있다.

추상 클래스라고 생각해보자.

두 클래스가 같은 추상 클래스를 확장하길 원한다면, 그 추상 클래스는 계층 구조상 두 클래스의 공통 조상이어야 한다.

새로 추가된 추상 클래스의 모든 자손이 이를 상속하게 되는 것이다.

### 인터페이스는 믹스인(mixin) 정의에 안성맞춤이다.

아래 믹스인 설명이 어려울 수 있는데, 가장 간단한 예시가 Comparable이다.

```java
class Point implements Comparable {
	private int y;
	private int x;

	@Override  
	public int compareTo(Object o) {  
	    return this.y - o.y;  
	}
}
```

Comparable 인터페이스를 구현하면 우리는 구현 클래스 인스턴스의 순서를 정해 비교할 수 있음을 알고 있다. 클래스에 특정 행위를 제공한다는 것이 이런 것이다.

반면 추상 클래스를 생각해보자.

기존 클래스에 덧씌울 수 없다. 클래스는 두 개 이상의 부모를 가질 수 없고, 클래스 계층 구조가 정해진 상태에서 믹스인을 삽입하기에 합리적인 위치가 없다.

> 믹스인

클래스가 구현할 수 있는 타입으로, 믹스인을 구현한 클래스에 원래 '주된 타입' 외에도 특정 선택적 행위를 제공한다고 선언하는 효과를 제공한다.

### 인터페이스로는 계층 구조가 없는 타입 프레임워크를 만들 수 있다.

타입을 계층적으로 정의할 경우 수많은 개념을 구조적으로 잘 표현할 수 있다. 하지만 현실에는 계층을 엄격하게 구분할 수 없는 경우가 많다.

예를 들어 Singer 인터페이스와 Songwriter 인터페이스가 있다고 가정해보자.

```java
public interface Singer {
	AudioClip sing(Song s);
}

public interface SongWirter {
	Song Compose(int chartPoisition);
}
```

그런데 현실에는 딱 가수와 작곡가로 나뉘지 않는다. 가수와 작곡을 동시에 하는 SingerSongwriter가 존재한다.

```java
public interface SingerSongWriter extends Singer, SongWirter {
	AudioClip strum();
	void actSensitive();
}
```

인터페이스를 활용하면 이렇게 유연하게 Singer와 SongWriter 모두를 확장하고 새로운 메서드까지 추가하는 제 3의 인터페이스를 정의할 수도 있다.

만약 비슷한 구조를 클래스를 통해 구현했다면?

```java
class Singer {
    public AudioClip sing(Song s) {
        //...
    }
}

class SongWriter {
    public Song compose(int chartPosition) {
        //...
    }
}

class Performer {
    public void perform() {
        //...
    }
}

class SingerSongWriter extends Singer {
    private SongWriter songwriter;

    public SingerSongWriter() {
        this.songwriter = new SongWriter();
    }

    @Override
    public AudioClip sing(Song s) {
        return super.sing(s);
    }

    public Song compose(int chartPosition) {
        return songwriter.compose(chartPosition);
    }
}

class SingerSongWriterPerformer extends SingerSongWriter {
    private Performer performer;

    public SingerSongWriterPerformer() {
        super();
        this.performer = new Performer();
    }

    @Override
    public AudioClip sing(Song s) {
        return super.sing(s);
    }

    @Override
    public Song compose(int chartPosition) {
        return super.compose(chartPosition);
    }

    public void perform() {
        performer.perform();
    }
}
```

- 하나만 추가했는데도 각각의 클래스를 구현하는데 번거로운 구조가 만들어졌다.
	- 심지어 모든 조합을 전부 구현한 것도 아니다. 모든 조합을 구현ㅎ한다면 복잡한 구조가 만들어질 것이다.

### 래퍼 클래스 + 인터페이스는 기능을 향상시키는 안전하고 강력한 수단이 된다.

타입을 추상 클래스로 정의해두면 그 타입에 기능을 추가하는 방법은 상속뿐이다. 상속해서 만든다면 활용도가 떨어지고, 깨지기 쉽다.

반면 복습해보자면 이전 [아이템18 (상속보다는 컴포지션을 사용하라)](https://ldhapple.github.io/posts/EffectiveJava-Item18/)의 예시에서 인터페이스를 통해 엔진을 갈아끼우는 방식으로 상속 대신 사용했었다.

- 엔진을 갈아끼우는 방식을 통해 자동차를 상속하는 대신 엔진 인터페이스를 구현한 구현체를 갈아끼워 해당 엔진에 맞는 차량으로 만들었다.
- 더해 컴포지션을 활용해 상속의 문제였던 상위 클래스 구현에 따라 하위 클래스가 오작동할 수 있던 문제를 해결했었다.
	- 떠올리기 쉽도록 엔진 인터페이스만 다시 삽입한다. (안떠오른다면 다시 해당 포스트로)

```java
interface Engine { 
	void run(Vehicle vehicle); 
} 

class ElectricEngine implements Engine { 

	@Override 
	public void run(Vehicle vehicle) { 
		if (vehicle.getEnergy() >= 10) { 
		vehicle.decreaseEnergy(10); 
			System.out.println("ElectricCar. Remaining energy: " + vehicle.getEnergy()); 
		} else { 
			System.out.println("ElectricCar 운전 불가"); 
		} 
	} 
}
```

### 인터페이스의 디폴트 메서드 사용? => 골격 구현 활용

인터페이스의 메서드 중 구현 방법이 명백한 것이 있다면 그 구현을 디폴트 메서드로 제공하면 좋다. 디폴트 메서드를 제공할 때는 상속하려는 사람을 위한 설명을 문서화해야 한다. 

그러나 일단 인터페이스의 디폴트 메서드에는 제약 사항이 있다.

- Object 메서드인 equals와 hashcode는 디폴트 메서드로 제공해서는 안된다.
	- 인터페이스를 구현하는 클래스들에 공통된 메서드로 보이지만, 클래스마다 구체적인 구현이 필요하기 때문에 디폴트 메서드로 제공하는 것은 바람직하지 않다.
- 인터페이스는 인스턴스 필드를 가질 수 없다.
	- 인터페이스는 메서드 시그니처만을 정의하고 상태를 갖지 않는다.
	- 상태를 갖게 되면 다중 상속을 지원하는 인터페이스의 특성상 상태 충돌 문제가 발생할 수 있다.
- 본인이 만든 인터페이스가 아니라면 디폴트 메서드를 추가할 수 없다.
	- 자신이 만들지 않은 인터페이스에 디폴트 메서드를 추가하면 해당 인터페이스를 사용하던 기존 코드에 영향을 미칠 수 있다.

그래서 이를 극복하기 위해 인터페이스와 추상 골격 구현 클래스를 함께 제공하는 식으로 인터페이스오 추상 클래스의 장점을 모두 취하는 방법이 있다.

#### 추상 골격 구현 클래스의 활용

- 인터페이스로 타입을 정의하고, 필요하면 디폴트 메서드도 제공한다.
- 그리고 추상 골격 구현 클래스가 나머지 메서드들을 구현한다.

이렇게 했을 경우 단순히 골격 구현을 확장하는 것만으로 해당 인터페이스를 구현하는 데 필요한 일이 대부분 완료된다. 템플릿 메서드 패턴이다. 아래 예시를 보고 이해해보자.

```java
interface MyCollection<E> {
	boolean add(E element);
	boolean remove(E element);
	boolean contains(E element);
	int size();
}
```
```java
abstract class AbstractMyCollection<E> implements Collection<E> {
	@Override
	public abstract boolean add(E element);

	@Override
	public abstract boolean remove(E element);

	@Override
	public abstract boolean contains(E element);
	
	@Override
	public abstract int size();

	public boolean isEmpty() {
		return size() == 0; //기반 메서드를 사용해 구현
	}

	public void clear() {
		Iterator<E> iter = iterator()'
		while (iter.hasNext()) {
			iter.next();
			iter.remove();
		}
	}
}
```

- 인터페이스의 메서드들은 다른 메서드들의 구현에 사용되는 기반 메서드들이다.
	- 이러한 기반 메서드들은 추상 골격 구현 클래스에서는 추상 메서드가 된다.
- 그리고 추상 골격 구현 메서드에서 기반 메서드들을 사용해 직접 구현할 수 있는 메서드들을 모두 디폴트 메서드로 제공한다. 

이제 이렇게 구현하면 어떻게 활용할 수 있는지 보자.

```java
class MyArrayList<E> extends AbstractMyCollection<E> {
	private List<E> list = new ArrayList<>();

	@Override
	public boolean add(E element) {
		return list.add(element);
	}

	//...
	@Override
	public int size() {
		return list.size();
	}

	@Override
	public Iterator<E> iterator() {
		return list.iterator();
	}
}
```

- 추상 골격 구현 클래스인 AbstractMyCollection 클래스는 인터페이스의 일반적인 구현을 추상 메서드로 제공하고, 공통된 메서드들을 디폴트 메서드로 제공했다.
- 이를 통해 추상 골격 구현 클래스를 상속받는 하위 클래스들은 인터페이스의 일관성을 유지하면서도 공통된 동작을 제공받아 재사용할 수 있는 것이다.

정리해보면 인터페이스의 디폴트 메서드에는 제약사항이 있으므로 추상 골격 구현 클래스를 활용해 디폴트 메서드를 우회해서 제공하는 방식을 사용할 수 있다. 

이를 통해 인터페이스의 장점과 디폴트 메서드의 제약 해결을 모두 얻을 수 있다.

주의할 점은 이러한 **추상 골격 구현은 기본적으로 상속해 사용하는 것을 가정하기 때문에 아이템 19의 설계 및 문서화 지침을 모두 따라야 한다.**

## 정리

***다중 구현용 타입으로 추상 클래스보다 인터페이스가 적합하다.*** 추상 클래스는 단일 상속이라는 제약 속에서 새로운 타입을 정의하는데 제약이 있다.

인터페이스를 사용하면 기존 클래스에서도 추가해 사용하기 쉽고, 믹스인 정의에도 좋다. 더해 상속만으로는 구현하기 힘든 현실적인 계층 구조도 인터페이스를 통해 유연하게 만들 수 있다.

단, 인터페이스의 디폴트 메서드 활용에는 제약 사항이 있으므로 디폴트 메서드를 활용하고 싶다면 추상 클래스에 디폴트 메서드를 제공하는 방식을 사용해보자.
