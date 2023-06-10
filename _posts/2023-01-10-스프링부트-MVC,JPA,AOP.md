---
title: 웹 MVC 개발 + DB 접근 + AOP
author: leedohyun
date: 2023-01-10 23:13:00 -0500
categories: [JAVA, Spring, Spring-Boot]
tags: [java, Spring, SpringBoot]
---

## 등록 및 조회

- MemberController

```
@Controller  
public class MemberController {  
  
    private final MemberService memberService;  
  
	@Autowired  
    public MemberController(MemberService memberService){  
        this.memberService = memberService;  
    }  
  
    @GetMapping("/members/new")  
    public String createForm(){  
        return "members/createMemberForm";  
    }  
  
    @PostMapping("/members/new") // 등록 
    public String create(MemberForm form){  
        Member member = new Member();  
	    member.setName(form.getName());  
  
	    memberService.join(member);  
  
	   return "redirect:/";  
  }
  
	@GetMapping("/members")  // 조회
	public String list(Model model){  
	    List<Member> members = memberService.findMembers();  
		model.addAttribute("members", members);  
		return "members/memberList";  
	}  
}
```

- MemberForm

```
public class MemberForm {  
    private String name;  
  
    public String getName(){  
        return name;  
    }  
  
    public void setName(String name) {  
        this.name = name;  
    }  
}
```

- createMemberForm.html

```
<!DOCTYPE HTML>  
<html xmlns:th="http://www.thymeleaf.org">  
<body>  
<div class="container">  
 <form action="/members/new" method="post">  
 <div class="form-group">  
 <label for="name">이름</label>  
 <input type="text" id="name" name="name" placeholder="이름을  
입력하세요">  
 </div> <button type="submit">등록</button>  
 </form></div> <!-- /container -->  
</body>  
</html>
```

** form 태그의 method = "post" action = "/members/new"
form을 제출 했을 때 해당 URL에 post방식으로 전송한다.
-> @PostMapping: 데이터를 Form같은 곳에 넣어서 전달할 때 사용.
데이터를 등록할 때 사용하게 된다.
@GetMapping은 주로 조회할 때 사용한다.

# DB 접근

## JdbcTemplate

스프링 JdbcTemplate와 MyBatis 같은 라이브러리는 JDBC API에서 본 반복 코드를 대부분 제거해준다. 하지만 SQL은 직접 작성해야 한다.

- JdbcTemplateMemberRepository

```
public class JdbcTemplateMemberRepository implements MemberRepository{  
  
    private final JdbcTemplate jdbcTemplate;  
  
    @Autowired  
    public JdbcTemplateMemberRepository(DataSource dataSource){  
        this.jdbcTemplate = new JdbcTemplate(dataSource);  
    }  
  
    @Override  
    public Member save(Member member) {  
        SimpleJdbcInsert jdbcInsert = new SimpleJdbcInsert(jdbcTemplate);  
	    jdbcInsert.withTableName("member").usingGeneratedKeyColumns("id");  
  
	    Map<String, Object> parameters = new HashMap<>();  
	    parameters.put("name", member.getName());  
  
	    Number key = jdbcInsert.executeAndReturnKey(new  
	    MapSqlParameterSource(parameters));  
	    member.setId(key.longValue());  
	    return member;  
    }  
  
    @Override  
    public Optional<Member> findById(Long id) {  
        List<Member> result = jdbcTemplate.query("select * from member where id = ?", memberRowMapper());  
	    return result.stream().findAny();  
    }  
  
    @Override  
    public Optional<Member> findByName(String name) {  
        List<Member> result = jdbcTemplate.query("select * from member where name = ?", memberRowMapper());  
	    return result.stream().findAny();  
    }  
  
    @Override  
    public List<Member> findAll() {  
        return jdbcTemplate.query("select * from member", memberRowMapper());  
    }  
  
    private RowMapper<Member> memberRowMapper(){  
        return (rs, rowNum) -> {  
            Member member = new Member();  
		    member.setId(rs.getLong("id"));  
		    member.setName(rs.getString("name"));  
		    return member;  
	  };  
    }  
  }
```

- SpringConfig

```
@Configuration  
public class SpringConfig {  
      
    private DataSource dataSource; 
     
    @Autowired  
    public SpringConfig(DataSource dataSource) {  
         this.dataSource = dataSource;  
    }  
  
    @Bean  
    public MemberService memberService(){  
         return new MemberService(memberRepository());  
    }  
  
    @Bean  
    public MemberRepository memberRepository(){  
         return new JdbcTemplateMemberRepository(dataSource); // new MemoryMemberRepository();로 메모리에 접근했던 부분을 이렇게 이 부분만 수정하면 나머지는 수정 없이 반영 된다.  
    }  
}
```


## 스프링 부트 + JPA

- JPA는 기존의 반복 코드와 기본적인 SQL도 만들어서 실행해준다.
- JPA를 사용하면, SQL과 데이터 중심의 설계에서 객체 중심의 설계로 패러다임을 전환할 수 있다.
- JPA를 사용하면 개발 생산성을 크게 높일 수 있다.


build.gradle의 dependencies에 추가.

```
implementation 'org.springframework.boot:spring-boot-starter-data-jpa' 
runtimeOnly 'com.h2database:h2'
```
 
 application.properties에 추가. (URL은 개인별 DB)

```
spring.datasource.url=jdbc:h2:tcp://localhost/~/test  
spring.datasource.driver-class-name=org.h2.Driver  
spring.datasource.username=sa

spring.jpa.show-sql=true  
spring.jpa.hibernate.ddl-auto=none
```

- show-sql: JPA가 생성하는 SQL을 출력한다.
- ddl-auto: JPA는 테이블을 자동으로 생성하는 기능을 제공하는데 none을 사용하면 해당 기능을 끈다.
	- create를 사용하면 엔티티 정보를 바탕으로 테이블도 직접 생성해준다.

1. @Entity 어노테이션을 통해 Member domain을 테이블과 연결할 수 있다.
2. @Id @GeneratedValue(strategy = GenerationType.IDENTITY) 어노테이션을 통해 자동으로 DB가 생성하는 ID를 매핑할 수 있다.
3. @Column(name = "username") String name; 이면 name을 DB의 username 컬럼에 매핑한다.

- JpaMemberRepository

```
public class JpaMemberRepository implements MemberRepository{  
  
    private final EntityManager em;  
  
	public JpaMemberRepository(EntityManager em){  
        this.em = em;  
    }  
    @Override  
    public Member save(Member member) {  
        em.persist(member);  
	    return member;  
    }  
  
    @Override  
    public Optional<Member> findById(Long id) {  
        Member member = em.find(Member.class, id);  
	    return Optional.ofNullable(member);  
    }  
  
    @Override  
    public Optional<Member> findByName(String name) {  
        List<Member> result = em.createQuery("select m from Member m where m.name = :name", Member.class)  
                .setParameter("name", name)  
                .getResultList();  
  
	   return result.stream().findAny();  
    }  
  
    @Override  
    public List<Member> findAll() {  
        return em.createQuery("select m from Member m", Member.class)  
                .getResultList();  
    }  
}
```

** EntityManaber em: JPA를 이용하려면 주입받아야 함.


## 스프링 데이터 JPA

스프링 데이터 JPA 프레임워크를 사용하면 리포지토리에 구현 클래스 없이 인터페이스만으로 개발을 할 수 있다. 기본 CRUD 기능도 스프링 데이터 JPA가 모두 제공한다.

** CRUD = Create, Remove, Update, Delete

- SpringDataJpaMemberRepository

```
public interface SpringDataJpaMemberRepository extends JpaRepository<Member, Long>, MemberRepository {  
      @Override  
	  Optional<Member> findByName(String name);  
}
```

- SpringConfig

```
@Configuration  
public class SpringConfig {  
  
    private final MemberRepository memberRepository;  
  
  @Autowired  
  public SpringConfig(MemberRepository memberRepository){  
        this.memberRepository = memberRepository;  
  }
}
```


** extends JpaRepository<Member, Long> 에서 Member는 해당 객체
Long은 id, 즉 식별자의 Type.

** 서비스 계층에 @Transactional 어노테이션 추가해야 한다.

- JPA를 통한 모든 데이터 변경은 트랜잭션 안에서 실행해야 한다. 

1. 인터페이스를 통해 기본적인 CRUD 제공
2. findByName(), findByEmail() 등 메서드 선언만으로 기능 제공
3. 페이징 기능 자동 제공



## 스프링 통합테스트

스프링 컨테이너와 DB까지 연결한 통합테스트

- Test.MemberServiceIntegrationTest

```
@SpringBootTest 
@Transactional 
class MemberServiceIntegrationTest { 
	@Autowired MemberService memberService; 
	@Autowired MemberRepository memberRepository; 
	
	@Test public void 회원가입() throws Exception { 
		//Given 
		Member member = new Member(); 	
		member.setName("hello"); 

		//When 
		Long saveId = memberService.join(member); 

		//Then 
		Member findMember = memberRepository.findById(saveId).get();
		assertEquals(member.getName(), findMember.getName());
	} 

	@Test 
	public void 중복_회원_예외() throws Exception { 
		//Given 
		Member member1 = new Member();
		member1.setName("spring"); 
		Member member2 = new Member(); 
		member2.setName("spring"); 

		//When 
		memberService.join(member1);
		IllegalStateException e = assertThrows(IllegalStateException.class, () -> memberService.join(member2)); //예외가 발생해야 한다.
		assertThat(e.getMessage()).isEqualTo("이미 존재하는 회원입니다."); } 
	}
```

** 이제 DB와 연결되었기 때문에 @AfterEach로 clear를 할 필요 없고
스프링 빈으로 객체들을 관리하기 때문에 @Autowired를 사용해 객체를 내려받는다. @BeforeEach로 하나의 객체를 참조할 필요 없다.

- @SpringBootTest: 스프링 컨테이너와 테스트를 함께 실행한다.
- @Transactional: 테스트 케이스에 이 어노테이션이 있으면, 테스트 시작 전에 트랜잭션을 시작하고, 테스트 완료 후에 항상 롤백한다. DB에 데이터가 남지 않으므로 이 어노테이션 덕분에 @AfterEach를 하지 않아도 되는 것이다. (Service 등에 붙으면 롤백하지 않는다.)

** SQL이 작동하고 commit이 되어야 DB에 반영이 되는 구조. 
Test 하는 메서드에 @Commit을 붙이면 commit을 하게 된다.

** 이전에 한 테스트가 순수 자바코드를 이용한 단위 테스트이고 단위 테스트를 잘 만드는 것이 더 좋을 확률이 높다.


## AOP

AOP = Aspect Oriented Programming

- AOP가 필요한 상황
	- 모든 메서드들의 호출 시간을 측정하고 싶다면?
	- 공통 관심 사항 (cross-cutting concern) vs 핵심 관심 사항 (core concern)
	- 회원 가입 시간, 회원 조회 시간을 측정하고 싶다면?

![image](https://user-images.githubusercontent.com/90108877/211929117-0eb886ce-9cbc-49d4-a00e-1d4dcf44a06c.png)

- Aop.TimeTraceAop

```
@Aspect  
public class TimeTraceAop {  
  
    @Around("execution(* hello.hellospring..*(..)")  
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable{  
        long start = System.currentTimeMillis();  
	    System.out.println("START: " + joinPoint.toString());  
	   try{  
	            return joinPoint.proceed();  
	    }finally {  
            long finish = System.currentTimeMillis();  
		    long timeMs = finish - start;  
			System.out.println("END: " + joinPoint.toString() + " " + timeMs + "ms");  
	    }  
    }  
}
```

**  @Aspect 어노테이션 사용해야 함.
@Around 어노테이션으로 Aop 적용 범위 지정
hello.hellospring 이라는 프로젝트 명 하위의 모두에게 적용한다는 뜻
ex) hello.hellospring.service.. 이면 service 하위에만 찍힘.

SpringConfig에서 Bean 등록해야 함.

```
@Bean  
public TimeTraceAop timeTraceAop(){  
    return new TimeTraceAop();  
}
```

