---
title: 스프링 JdbcTemplate
author: leedohyun
date: 2023-10-13 21:13:00 -0500
categories: [Spring, 데이터베이스 접근 기술]
tags: [java, Spring, SpringBoot]
---

# JdbcTemplate

**SQL을 직접 사용하는 경우**에 스프링이 제공하는 JdbcTemplate는 좋은 선택이 될 수 있다. JdbcTemplate는 JDBC를 매우 편리하게 사용할 수 있도록 도와준다.

JPA와 같은 ORM 기술을 사용하면서 동시에 SQL을 직접 작성해야 할 때가 있는데 이런 경우도 JdbcTemplate을 함께 사용하면 된다.

> 주요 기능

- JdbcTemplate
	- 순서 기반 파라미터 바인딩 지원
- NamedParameterJdbcTemplate
	- 이름 기반 파라미터 바인딩 지원 (권장)
- SimpleJdbcInsert
	- INSERT SQL을 편리하게 사용하도록 지원
- SimpleJdbcCall
	- 스토어드 프로시저를 편리하게 호출할 수 있다.

[스프링 JdbcTemplate 사용 방법 공식 메뉴얼](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-JdbcTemplate)   

> 장점

- 설정의 편리함
	- JdbcTemplate은 spring-jdbc 라이브러리에 포함되어 있는데, 이 라이브러리는 스프링으로 JDBC를 사용할 때 기본으로 사용되는 라이브러리이다. 별도의 복잡한 설정 없이 바로 사용할 수 있다.
-  반복 문제 해결
	- JdbcTemplate은 템플릿 콜백 패턴을 사용하여 JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 해준다. (커넥션 조회, 동기화 / 리소스 종료 등등)
	- 개발자는 SQL을 작성하고, 전달할 파라미터를 정의하고, 응답 값을 매핑만 하면 된다.

> 단점

동적 SQL을 해결하기 어렵다.

### 설정 방법

build.gradle

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-jdbc'
	runtimeOnly 'com.h2database:h2'
```

- implementation ~: JdbcTemplate이 들어있는 spring-jdbc가 라이브러리에 포함된다.
- runtimeOnly ~: H2 데이터베이스에 접속하기 위함.

## JdbcTemplate 적용

### 기본적인 사용

> Repository

```java
@Slf4j  
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {  
  
	private final JdbcTemplate template;  
	  
	public JdbcTemplateItemRepositoryV1(DataSource dataSource) {  
		this.template = new JdbcTemplate(dataSource);  
	}  
	  
	@Override  
	public Item save(Item item) {  
		String sql = "insert into item(item_name, price, quantity) values (?,?,?)";  
		KeyHolder keyHolder = new GeneratedKeyHolder();  
		template.update(connection -> {  
			//자동 증가 키  
			PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});  
			ps.setString(1, item.getItemName());  
			ps.setInt(2, item.getPrice());  
			ps.setInt(3, item.getQuantity());  
			return ps;  
		}, keyHolder);  
		  
		long key = keyHolder.getKey().longValue();  
		item.setId(key);  
		  
		return item;  
	}  
	  
	@Override  
	public void update(Long itemId, ItemUpdateDto updateParam) {  
		String sql = "update item set item_name=?, price=?, quantity=? where id=?";  
		template.update(sql,  
				updateParam.getItemName(),  
				updateParam.getPrice(),  
				updateParam.getQuantity(),  
				itemId);  
	}  
	  
	@Override  
	public Optional<Item> findById(Long id) {  
		String sql = "select id, item_name, price, quantity where id = ?";  
		try {  
			Item item = template.queryForObject(sql, itemRowMapper(), id);  
			return Optional.of(item);  
		} catch (EmptyResultDataAccessException e) {  
			return Optional.empty();  
		}  
	}  
	  
	private RowMapper<Item> itemRowMapper() {  
		return ((rs, rowNum) -> {  
			Item item = new Item();  
			item.setId(rs.getLong("id"));  
			item.setItemName(rs.getString("item_name"));  
			item.setPrice(rs.getInt("price"));  
			item.setQuantity(rs.getInt("quantity"));  
			return item;  
		});  
	}  
	  
	@Override  
	public List<Item> findAll(ItemSearchCond cond) {  
		String itemName = cond.getItemName();  
		Integer maxPrice = cond.getMaxPrice();  
		  
		String sql = "select id, item_name, price, quantity from item";  
		//동적 쿼리  
		if (StringUtils.hasText(itemName) || maxPrice != null) {  
			sql += " where";  
		}  
		boolean andFlag = false;  
		List<Object> param = new ArrayList<>();  
		if (StringUtils.hasText(itemName)) {  
			sql += " item_name like concat('%',?,'%')";  
			param.add(itemName);  
			andFlag = true;  
		}  
		if (maxPrice != null) {  
			if (andFlag) {  
				sql += " and";  
			}  
			sql += " price <= ?";  
			param.add(maxPrice);  
		}  
		log.info("sql={}", sql);  
		  
		return template.query(sql, itemRowMapper());  
	}  
}
```

- 우선 JdbcTemplate을 사용하기 위해 dataSource가 필요하다.
	- 생성자를 통해 dataSource를 의존 관계 주입받고 그것을 사용해 JdbcTemplate을 생성한다.
	- 관례상 이 방법을 가장 많이 사용하고 스프링 빈으로 등록하고 주입받아도 무방하다.
- save()
	-  template.update(): 데이터를 변경할 때 사용한다.
		- INSERT, UPDATE, DELETE SQL에 사용한다.
		- template.update()의 반환값은 int이고, 영향 받은 로우수를 반환한다.
	-  데이터를 저장할 때 PK에 auto increment 방식을 사용하기 때문에, PK인 ID값은 개발자가 직접 지정하지 않고 비워두고 저장한다.
		- 그런데 DB가 생성해주는 이 ID값은 데이테베이스가 생성하기 때문에 데이터베이스에 INSERT가 완료되어야 생성된 PK ID값을 확인할 수 있다.
		- item 객체에 id 값을 넣어주어야 하는데 안되는 것이다.
		- KeyHolder와 connection.prepareStatement()를 사용해 id를 지정해주면 INSERT 쿼리 실행 이후 데이터베이스에서 생성된 ID값을 조회할 수 있다.
		- 이 부분은 SimpleJdbcInsert 라는 기능이 있으므로 자세한 설명은 생략한다.
- update()
	- 데이터를 업데이트 한다.
	- ? 에 바인딩할 데이터를 순서대로 전달해야 하는 점만 유의하자.
- findById()
	- template.queryForObject()
		- 결과 로우가 하나일 때 사용.
		- RowMapper는 데이터베이스의 반환 결과인 ResultSet을 객체로 변환한다.
		- 결과가 없으면 EmptyResultDataAccessException 예외가 발생한다.
		- 결과가 둘 이상이면 IncorrectResultSizeDataAccessException 예외가 발생한다.
	- 결과가 없으면 예외를 잡아 Optional.empty()를 반환한다. 
- findAll(cond)
	-  template.query()
		- 결과가 하나 이상일 때 사용한다.
		- 결과가 없으면 빈 컬렉션을 반환한다.
- itemRowMapper()
	- 데이터베이스의 조회 결과를 객체로 변환하는 메서드.
	- Jdbc에서 ResultSet을 사용을 생각하면 된다.
		- 차이는 JdbcTemplate이 루프를 돌려주고 개발자는 RowMapper를 구현해 내부 코드만 채우면 된다.
		- [rs.next() 부분 기억](https://ldhapple.github.io/posts/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%28%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EC%A0%81%EC%9A%A9%29/)

itemRowMapper() 메서드에서 rs를 어디서 가져오는 가에 대한 의문이 생긴다. 이 부분은 RowMapper<> 와 JdbcTemplate가 제공하는 메서드들에 대해 알아보면 된다.



> JdbcTemplateConfig

코드가 동작하도록 구성한다.

```java
@Configuration  
@RequiredArgsConstructor  
public class JdbcTemplateV1Config {  
  
	private final DataSource dataSource;  
	  
	@Bean  
	public ItemService itemService() {  
		return new ItemServiceV1(itemRepository());  
	}  
	  
	@Bean  
	public ItemRepository itemRepository() {  
		return new JdbcTemplateItemRepositoryV1(dataSource);  
	}  
}
```

스프링부트가 커넥션 풀과 DataSource, 트랜잭션 매니저를 스프링 빈으로 자동 등록해주도록 application.properties에 spring.datasource. 설정을 추가한다.

```properties
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
```

> Application

```java
//@Import(MemoryConfig.class)  
@Import(JdbcTemplateV1Config.class)  
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")  
public class ItemServiceApplication {  
  
	public static void main(String[] args) {  
		SpringApplication.run(ItemServiceApplication.class, args);  
	}  
	  
	@Bean  
	@Profile("local")  
	public TestDataInit testDataInit(ItemRepository itemRepository) {  
		return new TestDataInit(itemRepository);  
	}  
  
}
```

@Import를 JdbcTemplateConfig로 바꾸어 준다.

참고로 JdbcTemplate이 실행하는 SQL 로그를 확인하려면 application.properties에 아래를 추가하면 된다.

```properties
logging.level.org.springframework.jdbc=debug
```

#### findAll() 동적쿼리 문제

사용자가 검색하는 값에 따라 실행하는 SQL이 동적으로 달라져야 한다.

- 검색 조건이 없음

```sql
select id, item_name, price, quantity from item
```

- 상품명('itemName')으로 검색

```sql
select id, item_name, price, quantity from item
 where item_name like concat('%', ?, '%')
```

- 최대 가격('maxPrice')으로 검색

```sql
select id, item_name, price, quantity from item
 where price <= ?
```

- 상품명, 최대가격 모두 검색

```sql
select id, item_name, price, quantity from item
 where item_name like concat('%', ?, '%')
  and price <= ?
```

이런식으로 상황마다 SQL을 동적으로 생성해야 한다. 동적 쿼리가 쉬워보이지만 실제 개발하려면 다양한 상황을 고려해야 해 어렵다.

예를 들면 어떤 경우에는 where를 앞에 넣고 어떤 경우에는 and를 넣어야 되는지 모두 계산해야 하고, 각 상황에 맞추어 파라미터도 생성해야 한다. 위의 경우가 간단한 경우이고 실제로는 더 복잡한 동적 쿼리가 필요할 것이다.

위의 코드를 보면 flag에 따라 and를 넣고, 넣지 않고를 결정하는 등, 기능이 크다면 이런 고려사항이 훨씬 복잡해 직접 짜기 어려울 것이다.

이것이 JdbcTemplate의 단점이다.

### 이름 지정 파라미터 도입

위의 기본 사용에서는 ?에 들어갈 파라미터를 순서를 지켜 입력했었다. JdbcTemplate이 파라미터를 순서대로 바인딩하기 때문이다.

그런데 누군가 코드를 수정하면서 순서를 변경했다고 해보자.

```java
template.update(sql,
		itemName,
		price,
		quantity,
		itemId);
```

위와 같이 코드를 작성했었는데 sql문을 누군가 sql = "update item set item_name=?, quantity=?, price=? where id=?" 이렇게 순서를 바꿔 수정한 상황인 것이다.

결과적으로 quantity와 price가 바뀌는 매우 심각한 문제가 발생한다. 이러한 부분은 한 번 잘못하면 코드를 고쳐야 될 뿐 아니라 이미 DB에 잘못들어간 데이터까지 복구해야 하기 때문에 버그를 해결하는데 드는 리소스가 매우 크다.

유지보수 관점에서 모호함을 제거해 코드를 명확하게 만드는 것이 중요하다.

이런 문제를 보완하기 위해 JdbcTemplate은 NamedParameterJdbcTemplate 라는 이름을 지정해 파라미터를 바인딩할 수 있도록 기능을 제공해준다.

> Repository

```java
@Slf4j  
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {  
  
	private final NamedParameterJdbcTemplate template;  
	  
	public JdbcTemplateItemRepositoryV2(DataSource dataSource) {  
		this.template = new NamedParameterJdbcTemplate(dataSource);  
	}  
	  
	@Override  
	public Item save(Item item) {  
		String sql = "insert into item(item_name, price, quantity) values (:itemName, :price, :quantity)";  
		  
		BeanPropertySqlParameterSource param = new BeanPropertySqlParameterSource(item);  
		KeyHolder keyHolder = new GeneratedKeyHolder();  
		template.update(sql, param, keyHolder);  
		  
		long key = keyHolder.getKey().longValue();  
		item.setId(key);  
		  
		return item;  
	}  
	  
	@Override  
	public void update(Long itemId, ItemUpdateDto updateParam) {  
		String sql = "update item set item_name=:itemName, price=:price, quantity=:quantity where id=:id";  
		  
		MapSqlParameterSource param = new MapSqlParameterSource()  
						.addValue("itemName", updateParam.getItemName())  
						.addValue("price", updateParam.getPrice())  
						.addValue("quantity", updateParam.getQuantity())  
						.addValue("id", itemId);  
		  
		template.update(sql, param);  
	}  
	  
	@Override  
	public Optional<Item> findById(Long id) {  
		String sql = "select id, item_name, price, quantity where id = :id";  
		try {  
			Map<String, Long> param = Map.of("id", id);  
			Item item = template.queryForObject(sql, param, itemRowMapper());  
			return Optional.of(item);  
		} catch (EmptyResultDataAccessException e) {  
			return Optional.empty();  
		}  
	}  
	  
	private RowMapper<Item> itemRowMapper() {  
		return BeanPropertyRowMapper.newInstance(Item.class);  
	}  
	  
	@Override  
	public List<Item> findAll(ItemSearchCond cond) {  
		String itemName = cond.getItemName();  
		Integer maxPrice = cond.getMaxPrice();  
		  
		BeanPropertySqlParameterSource param = new BeanPropertySqlParameterSource(cond);  
		  
		String sql = "select id, item_name, price, quantity from item";  
		//동적 쿼리  
		if (StringUtils.hasText(itemName) || maxPrice != null) {  
			sql += " where";  
		}  
		boolean andFlag = false;  
		if (StringUtils.hasText(itemName)) {  
			sql += " item_name like concat('%',:itemName,'%')";  
			andFlag = true;  
		}  
		if (maxPrice != null) {  
			if (andFlag) {  
				sql += " and";  
			}  
			sql += " price <= :maxPrice";  
		}  
		log.info("sql={}", sql);  
		  
		return template.query(sql, param, itemRowMapper());  
	}  
}
```

- NamedParameterJdbcTemplate
	- 마찬가지로 내부에 dataSource가 필요하다.
- SQL에서 ? 대신 ":파라미터이름" 을 받는 것으로 변경되었다. 모든 메서드가 이렇게 변경되었다.

> 파라미터의 전달 방법

변경된 코드를 보면 순서대로  파라미터를 하나하나 입력해주는 대신, 한 번에 파라미터들을 모아 전달한다. 

파라미터를 전달하려면 Map 처럼 key, value 데이터 구조를 만들어 전달해야 한다. key는 :파라미터이름 으로 지정한 파라미터의 이름이고, value는 해당 파라미터의 값이 된다.

이렇게 만든 파라미터를 template.update(sql, param, keyHolder) 처럼 param을 전달하는 것을 볼 수 있다.

이름 지정 바인딩에서 자주 사용하는 파라미터의 종류는 크게 3가지이다.

- Map
- SqlParameterSource
	- MapSqlParameterSource
	- BeanPropertySqlParameterSource

위 리포지토리 코드에서 사용한 방법들을 알아본다.

- Map

```java
Map<String, Long> param = Map.of("id", id);
```

단순히 Map을 직접 사용하는 방법으로 Map.of는 해당 key,value 값을 넣은 Map을 반환한다.

- MapSqlParameterSource

```java
new MapSqlParameterSource()  
		.addValue("itemName", updateParam.getItemName())  
		.addValue("price", updateParam.getPrice())  
		.addValue("quantity", updateParam.getQuantity())  
		.addValue("id", itemId);
```

Map과 유사하지만 SQL 타입을 지정할 수 있는 등 SQL에 좀 더 특화된 기능을 제공한다.

- BeanPropertySqlParameterSource

```java
new BeanPropertySqlParameterSource(item);
```

자바빈 프로퍼티 규약을 통해 자동으로 파라미터 객체를 생성한다. (getXxx() -> xxx)

위에서는 item을 넣어주었으므로 item 내부에는 getItemName(), getPrice()등이 있고, 아래와 같이 데이터를 만들어 객체를 생성한다.

key=itemName, value=상품명

update() 메서드에서는 이 방법을 사용해 updateParam을 넣어주면 itemId를 전달할 수 없어 해당 방법을 사용할 수 없다. 상황에 맞게 사용하자.

> BeanPropertyRowMapper

```java
return ((rs, rowNum) -> {  
	Item item = new Item();  
	item.setId(rs.getLong("id"));  
	item.setItemName(rs.getString("item_name"));  
	item.setPrice(rs.getInt("price"));  
	item.setQuantity(rs.getInt("quantity"));  
	return item;  
});
```

itemRowMapper() 메서드가 기존 위의 코드에서 한 줄로 간단하게 바뀌었다.

```java
return BeanPropertyRowMapper.newInstance(Item.class);
```

BeanPropertyRowMapper는 ResultSet의 결과를 받아 자바빈 규약에 맞추어 데이터를 변환한다.

예를 들면 데이터베이스에서 조회한 결과가 select id, price 라고 하면 아래와 같은 코드를 작성해준다.

```java
Item item = new Item();
item.setId(rs.getLong("id"));
item.setPrice(rs.getInt("price"));
```

데이터베이스에서 조회한 결과 이름을 기반으로 setId() 처럼 자바빈 프로퍼티 규약에 맞춘 메서드를 호출한다.

***별칭***

그런데 select item_name의 경우를 보자. item 객체에는 itemName으로 되어있는데 item_name은 setItem_name() 이라는 메서드가 없기 때문에 이 방법을 사용하기 어렵다.

이 부분은 sql을 수정해주면 된다.

```sql
select item_name as itemName
```

as를 사용해 SQL 조회 결과의 이름을 변경하면 되고 실제로 이 방법은 많이 사용된다. JdbcTemplate 뿐만 아니라 다른 기술에서도 많이 사용된다.

***관례의 불일치***

자바 객체는 카멜 표기법을 사용한다. itemName과 같이 중간에 낙타 봉우리가 올라와있는 듯한 표기법이다.

관계형 데이터베이스에서는 언더스코어를 사용하는 snake_case 표기법을 사용한다. item_name과 같은 것이다.

이렇게 관례로 되어있다보니 BeanPropertyRowMapper는 언더스코어 표기법을 카멜로 자동변환 해준다.

그래서 select item_name으로 조회해도 사실 setItemName()이 문제 없이 작동한다.

따라서 별칭은 컬럼 이름과 객체 이름이 완전히 다른 경우 조회 SQL에서 사용하면 된다.

### SimpleJdbcInsert

JdbcTemplate은 INSERT SQL을 직접 작성하지 않아도 되도록 SimpleJdbcInsert를 제공한다.

> Repository

```java
@Slf4j  
public class JdbcTemplateItemRepositoryV3 implements ItemRepository {  
  
	private final NamedParameterJdbcTemplate template;  
	private final SimpleJdbcInsert jdbcInsert;  
	  
	public JdbcTemplateItemRepositoryV3(DataSource dataSource) {  
		this.template = new NamedParameterJdbcTemplate(dataSource);  
		this.jdbcInsert = new SimpleJdbcInsert(dataSource)  
					.withTableName("item")  
					.usingGeneratedKeyColumns("id");  
					// .usingColumns("item_name", "price", "quantity") //생략 가능  
	}  
	  
	@Override  
	public Item save(Item item) {  
		BeanPropertySqlParameterSource param = new BeanPropertySqlParameterSource(item);  
		Number key = jdbcInsert.executeAndReturnKey(param);  
		item.setId(key.longValue());  
		return item;  
	}
	//...
}
```

- withTableName: 데이터를 저장할 테이블 명 지정
- usingGeneratedKeyColumns: key를 생성하는 PK 컬럼 명을 지정
- usingColumns: INSERT SQL에 사용할 컬럼을 지정, 특정 값만 저장하고 싶을 때 사용하며 생략 가능하다.

SimpleJdbcInsert는 생성 시점에 데이터베이스 테이블의 메타 데이터를 조회한다. 따라서 어떤 컬럼이 있는지 확인할 수 있으므로 usingColumns가 생략 가능한 것이다.

- save()
	- 파라미터를 받아 jdbcInsert.executeAndReturnKey() 메서드를 이용하면 INSERT SQL을 실행하고, 그 반환값으로 key를 받을 수 있다.

## 기능 정리

### 조회

#### 단건 조회 (queryForObject())

하나의 로우를 조회할 때는 queryForObject()를 사용한다. 

- 숫자 조회

```java
int rowCount = jdbcTemplate.queryForObject("select count(*) from t_actor", 
Integer.class);
```

조회 대상이 객체가 아닌 단순 데이터 하나라면 타입을 Integer.class, String.class와 같이 지정해주면 된다.

- 문자 조회, 파라미터 바인딩

```java
String lastName = jdbcTemplate.queryForObject( "select last_name from t_actor where id = ?", 
String.class, 1212L);
```

id = ? 에 바인딩해준다.

- 객체 조회

```java
Actor actor = jdbcTemplate.queryForObject( 
	"select first_name, last_name from t_actor where id = ?", 
	(resultSet, rowNum) -> { 
		Actor newActor = new Actor(); 
		newActor.setFirstName(resultSet.getString("first_name")); 
		newActor.setLastName(resultSet.getString("last_name")); 
		return newActor; 
	}, 
	1212L);
```

결과를 객체로 매핑해야 하므로 RowMapper를 사용해야 한다.

#### 목록 조회 (query())

여러 로우를 조회할 때는 query()를 사용한다.

- 객체

```java
private final RowMapper actorRowMapper = (resultSet, rowNum) -> { 
	Actor actor = new Actor(); 
	actor.setFirstName(resultSet.getString("first_name")); 
	actor.setLastName(resultSet.getString("last_name")); 
	return actor; 
}; 

public List findAllActors() { 
	return this.jdbcTemplate.query("select first_name, last_name from t_actor", 
			actorRowMapper);
}
```

### 변경 (INSERT, UPDATE, DELETE) - update()

데이터를 변경할 때는 update()를 사용한다. 반환 값은 int이고 영향받은 로우 수를 반환한다.

```java
jdbcTemplate.update( "insert into t_actor (first_name, last_name) values (?, ?)", "Leonor", "Watling");
```

### 기타 기능

임의의 SQL을 실행할 때는 execute()를 사용한다.

- DDL

```java
jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```

테이블 생성

- 스토어드 프로시저 호출

```java
jdbcTemplate.update( "call SUPPORT.REFRESH_ACTORS_SUMMARY(?)", Long.valueOf(unionId));
```