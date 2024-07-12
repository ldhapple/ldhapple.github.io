---
title: Item27 (비검사 경고를 제거하라)
author: leedohyun
date: 2024-07-11 19:13:00 -0500
categories: [JAVA, Effective-Java]
tags: [Java, Effective Java]
---

## 비검사 경고를 제거하라

제네릭을 사용하면 수많은 컴파일러 경고를 보게 된다.

이전 아이템에서 로 타입을 사용했을 때 Coin 타입을 넣었던 것을 생각해보면 된다.

비검사 형변환 경고, 비검사 메서드 호출 경고, 비검사 변환 경고 등등이 있다.

대부분의 비검사 경고는 쉽게 제거할 수 있다.

![](https://blog.kakaocdn.net/dn/d9pLD4/btsIwYlRcjw/5s6DpixDuDl3ubFxfx70OK/img.png)

인텔리제이 기준으로는 위와 같이 무엇이 잘못됐고, 어떻게 수정해야 하는 지 알려준다.

그러나 제거하기 어려운 경고도 있다. 제거하기 어려운 경고는 어떻게 대처해야 할까?

> @SuppressWarnings("unchecked")

경고를 제거할 수는 없지만 타입이 안전하다고 확신할 수 있을 때 이 애노테이션을 달아 경고를 숨길 수 있다.

안전하다고 판단하는 경고를 숨김으로써 진짜 문제를 알리는 새로운 경고가 나왔을 때 대처할 수 있다. 기존 안전한 경고를 숨기지 않으면 새 경고를 놓칠 수 있다.

단, 타입이 안전하다고 정말 확신해야 하고 가능한 좁은 범위(변수 선언, 짧은 메서드 등등)에 적용하도록 해야 한다.

### 제거하기 힘든 비검사 경고

```java
public <T> T[] toArray(T[] a) {
	if (a.length < size) {
		T[] result = (T[]) Arrays.copyOf(elemets, size, a.getClass());
		return result;
	}
	
	System.arraycopy(elements, 0, a, 0, size);
	if (a.legnth > size) {
		a[size] = null;
	}

	return a;
}
```

위 코드는 ArrayList의 toArray() 메서드이다. 해당 부분을 실제로 보면 @SuppressWarnings가 적용되어 있다.

![](https://blog.kakaocdn.net/dn/bL7cx5/btsIy2fDEuj/KbwVrzQTF57lIpx2pfPLi1/img.png)

![](https://blog.kakaocdn.net/dn/2tAMZ/btsIyZQWl5D/ADPjPDnOEx72k3dAaoQaY0/img.png)

해당 부분의 경고를 숨겼다는 것을 알려준다.

T[] 타입이 필요한데 Object[] 타입이 제공되었다는 경고이다.

우리는 이것이 안전하다는 것을 알고 있다. 따라서 **@SuppressWarnings 애노테이션을 이용해 비검사 경고를 무시해주고 안전한 이유를 주석으로 남기면 된다.**

근거를 주석으로 다는 도중 코드가 사실 안전하지 않다는 것을 발견할 수도 있는 것이다.

## 정리

비검사 경고는 런타임에 ClassCastException을 일으킬 수 있는 가능성을 경고하는 것이다. 따라서 모든 비검사 경고는 없애주는게 좋다.

하지만 없애기 힘든 비검사 경고도 존재한다. 그러한 경고 중 타입 안전성이 확실하게 보장된 부분이 있다면 애노테이션을 이용해 경고를 제거하고 주석으로 달면 된다.