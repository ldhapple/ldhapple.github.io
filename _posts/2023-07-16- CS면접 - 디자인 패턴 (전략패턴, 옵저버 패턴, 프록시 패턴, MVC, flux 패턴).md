---
title: 전략패턴, 옵저버 패턴, 프록시 패턴, MVC, flux 패턴
author: leedohyun
date: 2023-07-16 20:13:00 -0500
categories: [CS, 디자인패턴]
tags: [CS, 디자인패턴]
---

## DI & DIP

### DI (Dependency Injection)

의존성 주입이란 메인 모듈이 직접 다른 하위 모듈에 대한 의존성을 주기보다는 중간에 의존성 주입자(dependency injector)가 이 부분을 책임지고 메인 모듈은 간접적으로 의존성을 주입하는 방식.

인터페이스에만 의존해야 함. -> 직접 구현체를 지정해주면 안됨.

즉 메인 모듈이 하위 모듈을 직접 지정해서 동작하기 않고 의존성 주입자가 이 부분을 책임지고 메인 모듈은 인터페이스에만 의존하도록 하는 방식이다.

이를 통해 메인 모듈과 하위 모듈간의 의존성을 조금 더 느슨하게 만들 수 있으며 모듈을 쉽게 교체 가능한 구조로 만들 수 있다.

의존한다 - A가 B에 의존한다 = B가 변하면 A도 변한다.

```
class BackendDeveloper{
	public void writeJava(){
		System.out.println("자바");
	}
}

class FrontendDeveloper{
	public void writeJavaScript(){
		System.out.println("자바스크립트");
	}
}

public class Project{
	private final BackendDeveloper backendDeveloper;
	private final FrontendDeveloper frontendDeveloper;

	public Project(BackendDeveloper be, FrontendDevelpoer fe){
		this.backendDeveloper = be;
		this.frontendDeveloper = fe;
	}
	
	public void implement(){
		backendDeveloper.writeJava();
		frontEndDeveloper.writeJavaScript();
	}
}

public static void main(String args[]){
	Project a = new Project(new BackendDeveloper(), new FrontendDeveloper());
	a.implement();
}
```

- DI 적용

```
interface Developer{
	void develop();
}

class BackendDeveloper implements Developer{
	@Override
	public void develop(){
		writeJava();
	}
	public void writeJava(){
		System.out.println("java");
	}
}

class FrontendDeveloper implements Developer{
	@Override
	public void develop(){
		writeJavaScript();
	}

	public void writeJavaScript(){
		System.out.println("js");
	}
}

public class Project{
	private final List<Developer> developers;
	
	public Project(List<Developer> developers){
		this.developers = developers;
	}
	public void implement(){
		developers.forEach(Developer::develop);
	}
}

public static void main(String args[]){
	List<Developer> dev = new ArrayList<>();
	dev.add(new BackendDeveloper());
	dev.add(new FrontendDeveloper());
	Project a = new Project(dev);
	a.implement();
}

```

모듈을 만들기 쉽다. 변경에도 용이하다.

### DIP(Dependency Inversion Principle)

![](https://blog.kakaocdn.net/dn/cJTkU6/btsnGZ6Jmhq/UZ5nu8sDid75T1BFT5ghP1/img.png)

![](https://blog.kakaocdn.net/dn/rko9C/btsnZJUTPdM/X58vGGS8iWopK5GBBbhOT1/img.png)

위와 같은 구조에서 아래와 같은 구조로 변했다. 이것이 DIP이다.

의존관계역전원칙.

- 상위 모듈은 하위 모듈에 의존해서는 안된다. 둘 다 추상화에 의존해야 한다.

- 추상화는 세부사항에 의존해서는 안된다. 세부 사항은 추상화에 따라 달라져야 한다.

> 장점

1. 모듈들을 쉽게 교체할 수 있는 구조가 된다.
2. 단위 테스팅과 마이그레이션이 쉬워진다.
3. 애플리케이션 의존성 방향이 좀 더 일관되어 코드를 추론하기 쉬워진다.

> 단점

1. 결국에는 모듈이 더 생기게 되므로 복잡해질 수 있다.
2. 종속성 주입 자체가 컴파일을 할 때가 아닌 런타임에 일어나기 때문에 컴파일을 할 때 종속성 주입에 관한 에러를 잡기가 어려워질 수 있다.


## 전략패턴

전략이라고 부르는 '캡슐화한 알고리즘'을 컨텍스트(행위, 상황) 안에서 바꿔주면서 상호 교체가 가능하게 만드는 디자인패턴.

ex) 로그인이라는 컨텍스트안에 회원로그인, 카카오로그인, 구글로그인 이라는 전략이 있다고 볼 수 있다.

```
interface PaymentStrategy{
	public void pay(int amount);
}

class KakaoPayStrategy implements PaymentStrategy
class BITPayStrategy implements PaymentStrategy

public class Project{
	public static void main(String[] args){
		ShoppingCart cart = new ShoppingCart();
		
		Item A = new Item("snack", 100);
		Item B = new Item("water", 300);
	
		cart.addItem(A);
		cart.addItem(B);
	
		cart.pay(new KakaoPayStrategy("name", "123456", "123", "12/01");
		cart.pay(new BITPayStrategy("lee@example.com", "abcdfse");
	}
}
```

kakaoPay, BITPay 라는 전략들을 cart.pay 라는 컨텍스트안에서 바꿔주면서 상호 교체가 가능하도록 만들었다.

## 옵저버 패턴

주체가 어떤 객체의 상태 변화를 관찰하다가 상태 변화가 있을 때마다 메서드 등을 통해 옵저버 목록에 있는 옵저버들에게 변화를 알려주는 디자인 패턴이다.

객체 그 자체가 주체가 될 수 있다.

트위터의 메인 로직, 그리고 MVC 패턴에도 적용되었다.

내가 어떤 사람을 팔로우 했다면 그 팔로우를 당한 주체가 포스팅을 할 때 팔로워인 나(옵저버)에게 알림을 보내줘야 한다.

```
public class Project{
	interface Subject{
		public void register(Observer obj);
		public void unregister(Observer obj);
		public void notifyObservers();
		public Object getUpdate(Observer obj);
	}

	interface Observer{
		public void update();
	}

	class Topic implements Subject{
		private List<Observer> observers;
		private String message;

		public Topic(){
			this.observers = new ArrayList<>();
			this.message = "";
		}
 
		@Override
		public void register(Observer obj){
			if(!observers.contains(obj)) observers.add(obj);
		}
	
		@Override
		public void unregister(Observer obj){
			observers.remove(obj);
		}

		@Override
		public void notifyObservers(){
			this.observers.forEach(Observer::update);
		}

		@Override
		public Object getUpdate(Observer obj){
			return this.message;
		}

		public void postMessage(String msg){
			System.out.println("Message sended to Topic: " + msg);
			this.message = msg;
			notifyObservers();
		}
	}

	class TopicSubScriber implements Observer{
		private String name;
		private Subject topic;

		public TopicSubscriber(String name, Subject topic){
			this.name = name;
			this.topic = topic;
		}

		@Override
		public void update(){
			String msg = (String) topic.getUpdate(this);
			System.out.println(name + ":: got message >>" + msg);
		}
	}

	public static void main(String[] args){
		Topic topic = new Topic();
		Observer a = new TopicSubscriber("a", topic);
		Observer b = new TopicSubscriber("b", topic);
		Observer c = new TopicSubscriber("b", topic);
		topic.register(a);
		topic.register(b);
		topic.register(c);
	
		topic.postMessage("hello");
	}
}
```

```
Message sended to Topic: hello
a:: got message >> hello
b:: got message >> hello
c:: got message >> hello
```

## 프록시 패턴

프록시 패턴이란 객체가 어떤 대상 객체에 접근하기 전, 그 접근에 대한 흐름을 가로채서 해당 접근을 필터링하거나 수정하는 등의 역할을 하는 계층이 있는 디자인 패턴이다.

![](https://blog.kakaocdn.net/dn/zsi0I/btsnZY5wA6g/KWlz9FaaDhHRCmHJikukJk/img.png)

사이트를 열었는데 어떤 대량의 트래픽이 들어오게 된다면 그것을 막아야 한다.

위와 같이 클라우드 페어가 프록시 서버의 역할을 하고 그 프록시 서버는 비정상적인 대량의 트래픽을 걸러주게 되는 것.

## MVC, MVP, MVVM 패턴

### MVC 패턴

모델, 뷰, 컨트롤러로 이루어진 디자인 패턴

![](https://blog.kakaocdn.net/dn/uGvTG/btsnZIhq6lC/wmdZKkGw1vBQl3NwErvZxk/img.png)

- 모델 - 데이터베이스, 상수, 변수 등을 뜻함. 뷰에서 데이터를 생성하거나 수정할 때 컨트롤러를 통해 모델이 생성 또는 업데이트 된다.

- 뷰 - 사용자 인터페이스 요소를 나타내며 모델을 기반으로 사용자가 볼 수 있는 화면을 의미한다. 모델이 가지고 있는 정보를 따로 저장하지 않으며 변경이 일어나면 컨트롤러에 이를 전달한다.

- 컨트롤러 - 하나 이상의 모델과 하나 이상의 뷰를 잇는 다리 역할을 하며 이벤트 등 메인 로직을 담당한다. 모델이나 뷰의 변경 통지를 받으면 이를 해석하여 각각의 구성 요소에 해당 내용에 대해 알려준다.

> 장점

1. 애플리케이션의 구성 요소를 세 가지 역할로 구분하여 개발 프로세스에서 각각의 구성 요소에만 집중해 개발할 수 있다.
2. 재사용성과 확장성이 용이하다는 장점이 있다.

> 단점

애플리케이션이 복잡해질수록 모델과 뷰의 관계가 복잡해지는 단점이 있다.

### MVP 패턴

C가 P(프레젠터)로 교체된 패턴.

View와 Presenter는 1:1 관계이므로 MVC보다 더 강한 결합을 지닌 디자인패턴이다.

![](https://blog.kakaocdn.net/dn/m37R5/btsnZJ1Iwrf/2fCBY0V1iqDjASei4jGstK/img.png)

### MVVM 패턴

MVC의 C가 VM(view model)로 바뀐 패턴. VM은 뷰를 추상화한 계층이며 VM : V는 1 : N 이라는 관계를 갖는다.

VM은 커맨드와 데이터바인딩을 가진다.

- 커맨드 : 여러 요소에 대한 처리를 하나의 액션으로 처리할 수 있는 기법
- 데이터바인딩 : 화면에 보이는 데이터와 브라우저 상의 메모리 데이터를 일치시키는 방법

대표적인 프레임워크로는 Vue.js가 있다.

![](https://blog.kakaocdn.net/dn/9D2Hq/btsnTDIauuE/FCo3sPbAKzeiQTki0Y50b1/img.png)

## Flux 패턴

단방향으로 데이터 흐름을 관리하는 디자인 패턴.

ex) 페이스북은 "읽은 표시"에 대한 기능 장애를 겪었다. 어떤 페이지에서 메시지를 읽었는데 다른 페이지에서는 메시지를 안읽었다고 뜨는 것이다.

이는 모델과 뷰의 관계가 복잡해지니 버그를 수정하기도 데이터흐름을 알아보기도 어려운 문제였다.

즉 뷰에서 일어난 것이 모델에 영향을 끼치기도 그 반대도 영향을 미치는 로직도 있는 상황에 데이터를 일관성 있게 뷰에 공유하기가 어려웠다.

이를 위한 해결법으로 데이터가 "한방향"으로만 흐르게 flux 패턴이 등장했다.

![](https://blog.kakaocdn.net/dn/H52fW/btsnXh5TY30/bdg746c7WJVskQZ4OUDAJK/img.png)

- Action : 사용자의 이벤트를 담당한다. 마우스 클릭이나, 글을 쓰는 등의 이벤트에 대해 객체를 만들어내 Dispatcher 에게 전달한다.

- Dispatcher : 들어오는 Action 객체 정보를 기반으로 어떠한 "행위"를 할 것인가를 결정한다. 보통 Action객체의 Type을 기반으로 미리 만들어 놓은 로직을 수행하고 이를 Store에 전달한다.

- Store : 애플리케이션 상태를 관리하고 저장하는 계층이다. 도메인의 상태, 사용자의 인터페이스 등의 상태를 모두 저장한다.

- View : 데이터를 기반으로 표출이 되는 사용자 인터페이스이다.

즉 MVC 패턴과 비교해 모델과 뷰가 바로 이어지지 않고, 모델의 변경, 뷰의 변경(액션)을 따로 받아 디스패처에서 관리하고 그에 대한 행위를 결정해 한방향으로만 보내는 것이 핵심.

> 장점

1. 데이터 일관성의 증대
2. 버그를 찾기가 쉬워짐
3. 단위테스팅이 쉬워짐

## Q. 전략패턴과 DI의 차이?

둘 다 무언가를 쉽게 교체하기 위한 것이라는 공통점이 있다.

차이는

전략패턴은 '어떠한 행위를 기반'으로 그 행위에 대해 어떤 인터페이스를 사용할 것이냐 이고.

의존성주입은 행위 기반이 아닌 단지 일부 동작을 구현하고 의존성을 주입하기만 하는 패턴이다.

## Q. 컨텍스트란?

1. 어떤 종류의 상태, 환경을 캡슐화한 것을 뜻함.
2. 작업이 중단되고 나중에 같은 지점에서 계속 될 수 있도록 저장하는 최소 데이터의 집합.

## 예상질문

### Q. 옵저버 패턴을 어떻게 구현하나요?

여러 가지 방법이 있지만 프록시 객체를 사용하곤 합니다. 프록시 객체를 통해 객체의 속성이나 메서드 변화등을 감지하고 이를 미리 설정해 놓은 옵저버들에게 전달하는 방법으로 구현합니다.

### Q. 프록시 서버를 설명하고 사용 사례에 대해 설명해보세요.

프록시 서버란 클라이언트가 자신을 통해서 다른 네트워크 서비스에 간접적으로 접속할 수 있게 해주는 서버를 말합니다. 주로 서버 앞단에 둬서 캐싱, 로깅, 데이터 분석을 서버보다 먼저 하는 서버로 쓰입니다. 이를 통해 포트 번호를 바꿔서 사용자가 실제 서버 포트에 접근하지 못하게 할 수 있으며, 공격자의 DDOS 공격을 차단하는 등의 역할을 할 수 있습니다.

### Q. MVC 패턴을 설명하고 MVVM패턴과의 차이는 무엇인지 말해주세요.

MVC 패턴은 모델, 뷰, 컨트롤러로 이루어진 디자인 패턴입니다. 앱의 구성 요소를 세 가지 역할로 구분하여 개발 프로세스에서 각각의 구성 요소에만 집중해 개발할 수 있다는 점과 재사용성과 확장성이 뛰어나다는 장점이 있습니다. 단점으로는 앱이 복잡해질수록 모델과 뷰의 관계 또한 복잡해지는 단점이 있습니다.

MVVM 패턴은 MVC의 컨트롤러가 뷰모델로 바뀐 패턴입니다. 뷰모델은 뷰를 더 추상화한 계층이며 커맨드와 데이터 바인딩을 가지는 것이 특징입니다. 뷰와 뷰모델 사이의 양방향 데이터 바인딩을 지원하며 UI를 별도의 코드 수정 없이 재사용할 수 있고 단위테스팅이 쉽다는 장점이 있습니다.
