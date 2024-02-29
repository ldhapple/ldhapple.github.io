---
title: ElementCollection 사용 이유, DataJpaTest (vs SpringBootTest)
author: leedohyun
date: 2024-02-26 20:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

## @ElementCollection

recommtoon을 개발하면서 추천을 위해 Webtoon 정보에 장르가 필요했다.

이번에 네이버웹툰을 새롭게 크롤링하려고 보니 장르에 대한 부분이 태그로 좀 더 세분화되어 있었다. 나는 추천 시스템에 이용할 장르가 필요했고, 선호 웹툰 장르 설문조사를 했을 때 사용한 장르들이 있었기 때문에 해당 장르들만 추출하기로 했다.

그래서 Enum으로 장르들을 모아두고, Enum 클래스에 존재하는 장르들만 필터링해 수집했다.

```java
@Getter  
public enum Genre {  
    ACTION("액션"),  
    DRAMA("드라마"),  
    FANTASY("판타지"),  
    ROMANCE("로맨스"),  
    DAILY("일상"),  
    MUNCHKIN("먼치킨"),  
    SCHOOL("학원물"),  
    COMIC("코믹"),  
    HORROR("공포"),  
    THRILLER("스릴러"),  
    MARTIAL_ARTS("무협");  
  
    private final String koreanName;  
  
    Genre(String koreanName) {  
	  this.koreanName = koreanName;  
    }  
  
  public static Genre isExistGenre(String crawlingGenre) {  
	  for (Genre genre : Genre.values()) {  
		  if (genre.getKoreanName().equals(crawlingGenre)) {  
			  return genre;  
		  }  
	  }  
  
	  return null;  
    }  
}
```

컬럼에는 @Enumerated를 이용해 저장을 하던 중, 내가 조사한 장르들이 하나의 웹툰에 여러 개가 할당되는 것을 발견했다. 

예를 들어 참교육이라는 웹툰은 액션, 학원물, 먼치킨 3가지 장르가 있었던 것이다.

기존의 Enum 방식으로는 Webtoon 테이블에 하나의 장르만 입력할 수 있었다.

그래서 방법을 찾던 중 @ElementCollection을 찾았고, 이 애노테이션에 대해 설명하려 한다.

```java
@Enumberated(value = EnumType.STRING)
@ElementCollction(targetClass = Genre.class)
@CollectionTable(name = "webtoon_genre", joinColumns = @JoinColumn(name = "webtoon_id")
@Column(name = "genre")
private Set<Genre> genres = new HashSet<>(); 
```

코드로 보면 위와 같다.

### 정리

기본적으로는 관계형 DB에는 컬렉션을 저장할 수 없기 때문에 나온 애노테이션이다.

- JPA에서 엔티티에 속하는 컬렉션을 매핑하기 위한 애노테이션.
	- 엔티티 클래스 내에서 컬렉션을 정의하고 매핑한다.
- 주로 값 타입을 매핑하는데 사용된다.
- 컬렉션의 요소를 저장하기 위해 별도의 테이블이 생성된다.
	- 위의 예시에서는 Webtoon_Genre 테이블이 생성됐다.
	- 1 (webtoon) : N (genre) 관계를 이룬다.
- 기본적으로 지연 로딩으로 동작한다. 

```
//Webton_Genre 테이블

- webtoon_id - genre    -
-   12345	 - ACTION   -
-   12345    - SCHOOL   -
-   12345    - MUNCHKIN -
``` 

테이블에는 위와 같이 매핑된다.

> 주의: 값 타입 컬렉션의 데이터 수정

(복습) 값 타입 컬렉션 안의 데이터를 수정할 때 일부만 수정하는 것이 아닌 데이터를 삭제 후 변경된 데이터를 새롭게 추가하는 방식으로 해야 한다.

> 참고: JPA와 동시성 문제

자바 컬렉션 중 동시성 문제에 대해 고려해 ConcurrentHashMap.newKeySet() 을 사용했었다.

하지만 JPA에서는 트랜잭션과 영속성 컨텍스트로 데이터의 일관성을 보장해 동시성 문제가 발생하지 않는다는 것을 알고 HashMap으로 수정하게 되었다.

### 한계

- 값 타입은 엔티티와 다르게 식별자 개념이 없어 값 변경의 추적이 어렵다.
- 값 타입 컬렉션에 변경 사항이 발생할 시, 주인 엔티티와 연관된 모든 데이터를 삭제하고 값 타입 컬렉션에 있는 현재 값을 모두 다시 저장해야 한다.
- 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어 기본 키를 구성한다.

### 대안

일대다 관계를 위한 엔티티를 새로 만들고, 값 타입을 사용하는 방법.

```java
public class Webtoon {
	@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
	@JoinColumn(name = "webtoon_id")
	private Set<GenreEntity> genre = new HashSet<>();
}

public class GenreEntity {
	@Id @GeneratedValue
	private Long id;

	private Genre genre;
}
```

(cascade = CascadeType.ALL, orphanRemoval = true) 옵션을 활용해 주인 엔티티에 의존하면서 한계를 극복할 수 있다. (영속성 전이, 고아객체)

즉 Webtoon 엔티티에서 Genre의 생명 주기를 관리하는 것이다.

> 그러나

내 프로젝트에는 웹툰의 장르라는 특성상 변경될 일이 없기 때문에 복잡도를 낮추고 애노테이션 하나만으로 가능한 우선 @ElementCollection 애노테이션을 사용하기로 했다.

프로젝트를 진행하면서 제약 사항이 생기면 그 때 수정하도록 한다.

## @DataJpaTest

기존에 테스트를 진행할 때, @SpringBootTest, @Transactional을 사용하여 리포지토리 테스트를 했었다.

이번에 스프링 데이터 JPA를 사용하면서 @DataJpaTest라는 애노테이션이 있다는 사실을 알았고, 정리해보려 한다.

### 정리

- 데이터베이스와 관련된 설정을 자동으로 구성
- JPA 엔티티나 리포지토리를 테스트할 때 사용하며 Spring Boot에서 엔티티와 리포지토리에 필요한 Bean을 자동으로 구성해준다.
- 트랜잭션도 관리해준다.
- 내장 데이터베이스를 사용해 테스트의 속도를 향상시킨다.
- JPA 관련 빈만 로드해 테스트가 빠르게 실행된다. 테스트 환경을 최소화해 효율적인 테스트를 할 수 있다.

> vs @SpringBootTest

- 전체 스프링 부트 애플리케이션 컨텍스트를 로드한다.
	- @Controller, @Service, @Repository 등등

따라서 데이터 관련 테스트에 집중할 때에는 @DataJpaTest를 사용해 테스트 성능 향상을 생각해볼 수 있고, Service 계층 등을 활용한 전체적인 통합 테스트를 하고싶다면 @SpringBootTest를 이용하도록 한다.
