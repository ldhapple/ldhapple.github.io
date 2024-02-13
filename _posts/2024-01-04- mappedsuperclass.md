---
title: MappedSuperclass
author: leedohyun
date: 2024-01-04 21:13:00 -0500
categories: [Spring, JPA 기본]
tags: [java, Spring, SpringBoot]
---

## @MappedSuperclass

![](https://blog.kakaocdn.net/dn/bpD51F/btsEGLpXn6j/mmhPrAjMBzfMusWxU6FRVk/img.png)

단순하게 공통된 속성을 담는 클래스를 만드는 것이다.

```java
@MappedSuperclass
public abstract class BaseEntity {
	private Long id;
	private String name;

	//getter, setter...
}
```

```java
@Entity
public class Member extends BaseEntity {
	//...
}
```

- 상속관계 매핑이 아니다.
- 엔티티가 아니기 때문에 (BaseEntity)  테이블과 매핑되는 것이 아니다.
- 부모 클래스를 상속받는 자식 클래스에 매핑 정보만 제공한다.
- 따라서 따로 조회가 불가능하다.
	- em.find(BaseEntity.class, ~) X
-  직접 생성해서 사용할 일이 없으므로 추상 클래스로 만드는 것을 권장한다.

### 정리

- 테이블과는 관계가 없고, 단순히 엔티티들이 공통으로 사용하는 매핑 정보를 모으는 역할을 한다.
- 주로 등록일, 수정일, 등록자, 수정자 등과 같은 전체 엔티티에서 공통으로 적용하는 정보를 모을 때 사용한다.
- 참고로 @Entity 클래스는 엔티티나 @MappedSuperclass로 지정한 클래스만 상속이 가능하다.