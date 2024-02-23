---
title: 순수 JPA + Querydsl
author: leedohyun
date: 2024-01-26 20:13:00 -0500
categories: [Spring, QueryDSL]
tags: [java, Spring, SpringBoot]
---

## 기본

```java
@Repository  
public class MemberJpaRepository {  
  
	private final EntityManager em;  
	private final JPAQueryFactory queryFactory;  
	  
	public MemberJpaRepository(EntityManager em) {  
		this.em = em;  
		this.queryFactory = new JPAQueryFactory(em);  
	}  
	  
	public void save(Member member) {  
		em.persist(member);  
	}  
	  
	public Optional<Member> findById(Long id) {  
		Member findMember = em.find(Member.class, id);  
		return Optional.ofNullable(findMember);  
	}  
	  
	public List<Member> findAll() {  
		return em.createQuery("select m from Member m", Member.class)  
				.getResultList();  	
	}  
	  
	public List<Member> findAll_Querydsl() {  
		return queryFactory  
				.selectFrom(member)  
				.fetch();  
	}  
	  
	public List<Member> findByUsername(String username) {  
		return em.createQuery("select m from Member m where username = :username", Member.class)  
				.setParameter("username", username)  
				.getResultList();  
	}  
	  
	public List<Member> findByUsername_Querydsl(String username) {  
		return queryFactory  
				.selectFrom(member)  
				.where(member.username.eq(username))  
				.fetch();  
	}  
}
```

기본 기능을 순수 JPA 코드와 Querydsl을 사용한 코드로 작성해보았다.

> 참고

JPAQueryFactory를 스프링 빈으로 등록해 주입받아 사용해도 된다.

동시성 문제는 걱정하지 않아도 된다.

스프링이 주입해주는 엔티티 매니저는 실제 동작 시점에 실제 엔티티 매니저를 찾아주는 프록시용 가짜 엔티티 매니저이다. 이 가짜 엔티티 매니저는 실제 사용시점에 트랜잭션 단위로 실제 엔티티 매니저를 할당해준다.

## 동적 쿼리

- 조회 최적화용 DTO

```java
@Data  
public class MemberTeamDto {  
	private Long memberId;  
	private String username;  
	private int age;  
	private Long teamId;  
	private String teamName;  
	  
	@QueryProjection  
	public MemberTeamDto(Long memberId, String username, int age, Long teamId, String teamName) {  
		this.memberId = memberId;  
		this.username = username;  
		this.age = age;  
		this.teamId = teamId;  
		this.teamName = teamName;  
	}  
}
```

- 회원 검색 조건 객체

```java
@Data  
public class MemberSearchCondition {  
  
	private String username;  
	private String teamName;  
	private Integer ageGoe;  
	private Integer ageLoe;  
}
```

참고로 페치 조인은 엔티티로 조회할 때만 사용한다. 연관된 엔티티도 함께 조회하는 기능이기 때문이다.

DTO로 조회할 때는 일반 조인을 사용해야 하고, DTO는 특정 엔티티에 종속적인 것이 아니기 때문에 여러 테이블을 조인하고 각 테이블에 있는 원하는 값만 별도로 조회할 수 있다.

한 번의 쿼리로 조인한 테이블의 정보를 조회한다.

### Builder 사용

```java
public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {  
  
	BooleanBuilder builder = new BooleanBuilder();  
  
	if (StringUtils.hasText(condition.getUsername())) {  
		builder.and(member.username.eq(condition.getUsername()));  
	}  
	if (StringUtils.hasText(condition.getTeamName())) {  
		builder.and(team.name.eq(condition.getTeamName()));  
	}  
	if (condition.getAgeGoe() != null) {  
		builder.and(member.age.goe(condition.getAgeGoe()));  
	}  
	if (condition.getAgeLoe() != null) {  
		builder.and(member.age.loe(condition.getAgeLoe()));  
	}  
  
	return queryFactory  
			.select(new QMemberTeamDto(  
					member.id,  
					member.username,  
					member.age,  
					team.id,  
					team.name  
			))  
			.from(member)  
			.leftJoin(member.team, team)  
			.where(builder)  
			.fetch();  
}
```

### Where절 파라미터 사용

```java
public List<MemberTeamDto> search(MemberSearchCondition condition) {  
	return queryFactory  
			.select(new QMemberTeamDto(  
					member.id,  
					member.username,  
					member.age,  
					team.id,  
					team.name
			))  
			.from(member)  
			.leftJoin(member.team, team)  
			.where(usernameEq(condition.getUsername()),  
					teamNameEq(condition.getTeamName()),  
					ageGoe(condition.getAgeGoe()),  
					ageLoe(condition.getAgeLoe()
			))  
			.fetch();  
}  

private BooleanExpression usernameEq(String username) {  
	return StringUtils.hasText(username) ? null : member.username.eq(username);  
}  

private BooleanExpression teamNameEq(String teamName) {  
	return StringUtils.hasText(teamName) ? null : team.name.eq(teamName);  
}  

private BooleanExpression ageGoe(Integer ageGoe) {  
	return ageGoe == null ? null : member.age.goe(ageGoe);  
} 
 
private BooleanExpression ageLoe(Integer ageLoe) {  
	return ageLoe == null ? null : member.age.loe(ageLoe);  
}
```

> 조회 API 컨트롤러

```java
@GetMapping("/api/members")
public List<MemberTeamDto> searchMember(MemberSearchCondition condition) {
	return memberJpaRepository.search(condition);
}
```

- 쿼리파라미터에서 condition을 지정한다.