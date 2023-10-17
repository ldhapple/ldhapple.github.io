---
title: 커넥션 풀과 DataSource
author: leedohyun
date: 2023-10-07 23:13:00 -0500
categories: [Spring, 데이터베이스 접근 기술 기본]
tags: [java, Spring, SpringBoot]
---

![](https://blog.kakaocdn.net/dn/QgWnD/btsxquoLPFR/TC4kDs9GfRJdsn96tG5mJk/img.png)

데이터베이스 커넥션을 획득할 때 복잡한 과정을 거친다.

1. 애플리케이션 로직이 DB 드라이버를 통해 커넥션을 조회한다.
2. DB 드라이버는 DB와 TCP/IP 커넥션을 연결한다. 이 과정에서 3 way handshake 같은 TCP/IP 연결을 위한 네트워크 동작이 발생한다.
3. DB 드라이버는 TCP/IP 연결을 완료하면 ID, PW, 기타 부가정보를 DB에 전달한다.
4. DB는 위 정보를 통해 내부 인증을 완료하고, 내부에 DB 세션을 생성한다.
5. 이 후 DB는 커넥션 생성이 완료되었다는 응답을 보낸다.
6. DB 드라이버는 커넥션 객체를 생성해 클라이언트에게 반환한다.

이렇게 커넥션을 새로 만드는 것은 복잡하고, 시간도 소요되며, 자원을 사용한다. 고객이 애플리케이션을 사용할 때, SQL을 실행하는 시간 뿐 아니라 커넥션을 새로 만드는 시간도 추가되기 때문에 결과적으로 응답 속도가 느려지고, 사용자에게 부정적인 영향을 끼친다.

이러한 부분을 해결하기 위한 아이디어가 커넥션을 생성해두고 사용하는 방법인 **커넥션 풀** 이다. (쓰레드 풀과 비슷한 느낌인 듯)

## 커넥션 풀

커넥션을 관리하는 풀(수영장) 이라고 생각하면 된다.

> 커넥션 풀 초기화

![](https://blog.kakaocdn.net/dn/RGQHC/btsxh9Mjn0K/q1sgJuIK70kvuH4iYTfq20/img.png)

애플리케이션을 시작하는 시점에 커넥션 풀은 필요한 만큼 커넥션을 미리 확보해 풀에 보관한다. 보통 얼마나 보관할 지는 서비스의 특징과 서버 스펙에 따라 다르지만 기본값은 보통 10개이다.

** 기본값이 10개인 이유?

쓰레드 풀은 보통 200이 기본 값인데 10개로 충분히 커버가 될까?

쓰레드는 고객의 모든 요청을 감당해야하는 것이고, 커넥션 풀은 DB 커넥션을 재사용함에 그 목적이 있다. 생각해보면 DB 작업은 대부분 짧은 시간에 완료된다. 이러한 요청이나 DB 커넥션의 경우 수 많은 고객이 동시에 요청을 보내자하고 요청을 보내지 않는 이상 이러한 수치로 해결가능하다는 뜻이다.

물론 서버의 크기, 작업 등등에 따라 수치를 바꿔야 할 수는 있다.

> 커넥션 풀의 연결 상태

![](https://blog.kakaocdn.net/dn/XadEE/btsxSBlPTmc/rKn7KkpTErgyomiPU5pZAk/img.png)

이러한 커넥션 풀 내부에 있는 커넥션은 이미 TCP/IP로 DB와 커넥션이 연결된 상태이기 때문에 연결과정 필요없이 SQL을 즉시 DB에 전달할 수 있다.

> 커넥션 풀 사용

![](https://blog.kakaocdn.net/dn/yTHn4/btsxtYbimHC/X34rEWpU9FR7Qi2Xlb45zK/img.png)

- 애플리케이션 로직에서 더 이상 새로운 커넥션을 획득할 필요 없다.
- 커넥션 풀을 통해 이미 생성되어 있는 커넥션을 객체 참조로 가져다 쓰면 된다.
- 커넥션 풀에 커넥션을 요청하면 풀은 자신이 가지고 있는 커넥션 중 하나를 반환해준다.

![](https://blog.kakaocdn.net/dn/YbT8P/btsxp53MI0A/zJxA5ht2s7qLzGvZxtxUnK/img.png)

- 애플리케이션 로직은 커넥션 풀에서 받은 커넥션을 사용해 SQL을 데이터베이스에 전달하고 그 결과를 받아 처리한다.
- 커넥션을 사용하고 나면 커넥션을 종료하는 것이 아닌, 다음에 다시 사용할 수 있도록 커넥션 풀에 반환한다. (종료가 아닌 살아있는 상태로 반환)

> 정리

- 적절한 커넥션 풀 숫자는 서비스의 특징과 애플리케이션 서버 스펙, DB 서버 스펙에 따라 다르기 때문에 서능에 따라 조절해야 한다.
- 커넥션 풀은 서버당 최대 커넥션 수를 제한할 수 있다. 따라서 DB에 무한정 연결이 생성되는 것을 막아주어 DB를 보호하는 효과도 있다.
- 커넥션 풀이 주는 이점이 매우 크기 떄문에 실무에서는 항상 기본으로 사용한다.
- 커넥션 풀은 개념적으로 단순해 직접 구현할 수도 있지만, 사용도 편리하고 성능이 뛰어난 오픈소스 커넥션 풀이 많기 때문에 오픈소스를 사용하면 된다.
	- 대표적으로 commons-dbcp2, tomcat-jdbc pool, HikariCP 등이 있다.
	- 성능과 사용의 편리함 측면에서는 hikariCP를 주로 사용한다.
	- 스프링부트 2.0 부터는 기본 커넥션 풀로 hikariCP를 제공한다.
	- 실무에서도 레거시 프로젝트가 아닌 이상 대부분 hikariCP를 사용한다.

## DataSource?

> 커넥션 풀을 이용하려는 시도에서 발생하는 문제

![](https://blog.kakaocdn.net/dn/Cn6ah/btsxp4KBukO/Kc4NYrOmo3L4rKvabI2CcK/img.png)

커넥션을 얻는 방법은 이전 포스트에서 다룬 JDBC DriverManager를 직접 사용하거나, 커넥션 풀을 사용하는 등 다양한 방법이 있다.

![](https://blog.kakaocdn.net/dn/08KND/btsxsSbjduF/Stg8l11Zf5LZkVdQqaOBhK/img.png)

DriverManager를 사용해 커넥션을 획득하다가 커넥션 풀이 좋다고 들어서 커넥션 풀을 사용하는 방법을 도입하려면 어떻게 해야할까?

![](https://blog.kakaocdn.net/dn/Fa7Zc/btsxzYB3esO/CuOOJ5ee9be4WuTXZzqK01/img.png)

예를 들어 애플리케이션 로직에서 DriverManger를 사용해 커넥션을 획득하다가 HikariCP 같은 커넥션 풀을 사용하도록 변경하려면 커넥션을 획득하는 애플리케이션 코드도 함께 변경해야 한다.

의존관계가 DriverManager에서 HikariCP로 변경되고 사용법도 다를 것이기 때문이다.

### DataSource의 등장

![](https://blog.kakaocdn.net/dn/qvCVb/btsxqu3kRnh/QGeVe7xxBpG2AXbKc2k0jk/img.png)

위의 문제를 해결하기 위해 DataSource가 등장했다.

자바에서 제공하는 인터페이스이다.

```java
public interface DataSource {
	Connection getConnection() throws SQLException;
}
```

- javax.sql.DataSource
- DataSource는 커넥션을 획득하는 방법을 추상화하는 인터페이스이다.
- 인터페이스의 핵심 기능은 커넥션 조회 하나이다. (다른 기능도 일부 있지만 중요하지 않다.)

> 정리

- 대부분의 커넥션 풀은 DataSource 인터페이스를 이미 구현해두었다. 따라서 개발자는 커넥션 풀의 코드를 직접 의존하는 것이 아닌, DataSource 인터페이스에만 의존하도록 애플리케이션 로직을 작성하면 된다.
- 커넥션 풀 구현 기술을 변경하고 싶다면 해당 구현체로 갈아끼우기만 하면 된다.
- DriverManager은 DataSource 인터페이스를 사용하지 않는다. DriverManager을 사용하다가 DataSource 기반의 커넥션 풀을 사용하도록 변경하면 관련 코드를 다 고쳐야 한다.
- 하지만 스프링은 DriverManager도 DataSource를 통해 사용할 수 있도록 DriverManagerDataSource라는 DataSource를 구현한 클래스를 제공한다.
- 자바는 DataSource를 통해 커넥션을 획득하는 방법을 추상화했다. 애플리케이션 로직은 DataSource 인터페이스에만 의존하면 된다. 덕분에 DriverManagerDataSource를 통해 DriverManager를 사용하다가 커넥션 풀을 사용하도록 코드를 변경해도 애플리케이션 로직은 변경하지 않아도 된다.

## DataSource 사용

### DriverManager과 DriverManagerDataSource 차이

```java
Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);

DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);  
dataSource.getConnection();
```

DriverManager을 사용해 커넥션을 획득하려면 매번 URL, USERNAME, PASSWORD를 보내야 했다.

하지만 스프링이 제공하는 DriverManagerDataSource를 이용하면 객체를 생성할 때 한번 넘기고 이 후 커넥션을 획득할 때는 getConnection()만 이용하면 된다.

> 설정과 사용의 분리

- 설정 : DataSource를 만들고 필요한 속성들을 사용해 URL, USERNAME, PASSWORD 같은 부분을 입력하는 것을 말한다.
	- 이렇게 설정과 관련된 속성들은 한 곳에 있는 것이 향후 변경에 더 유연하게 대처할 수 있다는 장점이 있다.
- 설정은 신경쓰지 않고, DataSource의 getConnection()만 호출해 사용하면 된다. 

이러한 분리는 별거 아닌 것 처럼 보일 수 있지만 큰 차이를 만들어낸다.

필요한 데이터를 DataSource가 만들어지는 시점에 다 넣어두면 사용하는 곳에서는 URL, USERNAME과 같은 속성들에 의존하지 않아도 된다.

즉 리포지토리는 DataSource에만 의존하고 위와 같은 속성 값은 몰라도 된다.

애플리케이션을 개발하다보면 보통 설정은 한 곳에서 하지만 사용은 수 많은 곳에서 한다. 따라서 객체를 설정하는 부분과 사용하는 부분을 명확하게 분리할 수 있는 점은 큰 장점이다.

### DataSource를 통한 커넥션 풀 이용

```java
@Test  
void dataSourceConnectionPool() throws InterruptedException, SQLException {  
	//커넥션 풀링  
	HikariDataSource dataSource = new HikariDataSource();  
	dataSource.setJdbcUrl(URL);  
	dataSource.setUsername(USERNAME);  
	dataSource.setPassword(PASSWORD);  
	dataSource.setMaximumPoolSize(10); //기본10
	dataSource.setPoolName("MyPool");  
	  
	useDataSource(dataSource);  
	Thread.sleep(1000);  
}  
  
private void useDataSource(DataSource dataSource) throws SQLException {  
	Connection con1 = dataSource.getConnection();  
	Connection con2 = dataSource.getConnection();  
	log.info("connection={}, class={}", con1, con1.getClass());  
	log.info("connection={}, class={}", con2, con2.getClass());  
}
```

- HikariCP 커넥션 풀을 사용한다.
- HikariDataSource는 DataSource 인터페이스를 구현하고 있다.
- 커넥션 풀 최대 사이즈를 10으로, 이름을 MyPool로 지정했다.
- Thread.sleep()?
	- 커넥션 풀에서 커넥션을 생성하는 작업은 애플리케이션 실행 속도에 영향을 주지 않기 위해 별도의 쓰레드에서 작동한다.
	- 별도의 쓰레드에서 동작하기 때문에 테스트가 먼저 종료되어 풀에 커넥션이 생성되는 log가 제대로 찍히지 않을 수 있다.
	- 따라서 Thread.sleep()을 통해 대기시간을 주어 커넥션이 생성되는 로그를 확인한다.
	- 커넥션을 최초로 조회하는 시점에 초기화(set~)가 진행되기 때문에 useDataSource 이후 sleep을 해주는 것이다.

```
#커넥션 풀 초기화 정보 출력 
HikariConfig - MyPool - configuration: 
HikariConfig - maximumPoolSize................................10 
HikariConfig - poolName................................"MyPool" 

#커넥션 풀 전용 쓰레드가 커넥션 풀에 커넥션을 10개 채움 
[MyPool connection adder] MyPool - Added connection conn0: url=jdbc:h2:.. user=SA 
[MyPool connection adder] MyPool - Added connection conn1: url=jdbc:h2:.. user=SA 
[MyPool connection adder] MyPool - Added connection conn2: url=jdbc:h2:.. user=SA 
[MyPool connection adder] MyPool - Added connection conn3: url=jdbc:h2:.. user=SA 
[MyPool connection adder] MyPool - Added connection conn4: url=jdbc:h2:.. user=SA ... 
[MyPool connection adder] MyPool - Added connection conn9: url=jdbc:h2:.. user=SA 

#커넥션 풀에서 커넥션 획득1 
ConnectionTest - connection=HikariProxyConnection@446445803 wrapping conn0: url=jdbc:h2:tcp://localhost/~/test user=SA, class=class com.zaxxer.hikari.pool.HikariProxyConnection 

#커넥션 풀에서 커넥션 획득2 
ConnectionTest - connection=HikariProxyConnection@832292933 wrapping conn1: url=jdbc:h2:tcp://localhost/~/test user=SA, class=class com.zaxxer.hikari.pool.HikariProxyConnection 

MyPool - After adding stats (total=10, active=2, idle=8, waiting=0)
```

위와 같은 로그를 확인할 수 있다.

- HikariConfig
	- HikariCP 관련 설정을 확인할 수 있다. 풀의 이름, 최대 풀 수 등등.
- MyPool connection adder
	- 별도의 쓰레드를 사용해 커넥션 풀에 커넥션을 채우고 있는 것을 알 수 있다.
	- 왜 별도의 쓰레드를 사용할까?
		- 커넥션 풀에 커넥션을 채우는 일은 상대적으로 오래걸리는 일이다.
		- 애플리케이션을 실행할 때 커넥션 풀을 채울 때 까지 대기하고 있다면 애플리케이션 실행 시간이 늦어진다.
		- 따라서 별도의 쓰레드를 사용해 애플리케이션 실행 시간에 영향을 끼치지 않기 위함이다.
- MyPool - After adding stats..
	-  커넥션 풀에서 커넥션을 획득하고 그 결과를 출력했다.
	- 여기서는 커넥션 풀에서 커넥션을 2개 획득하고 반환하지 않았기 때문에 10개의 커넥션 중 2개를 가지고 있는 상태이다.
	- active=2, idle=8은 사용중인 커넥션 2, 대기중 커넥션 8 이라는 뜻이다.
- 만약 사용을 11개 한다면?
	- Test가 계속 종료되지 않는다.
	- 커넥션 풀의 커넥션을 사용하기 위해 대기 상태에 있기 때문이다.
	- Waiting = 1이 생긴다.
	- 따라서 Pool이 다 사용 중이라면 얼마나 기다릴 것인지 설정도 해야 한다.
		- [HikariCP 참고](https://github.com/brettwooldridge/HikariCP)


### DataSource 적용

[기존 코드와 비교](https://ldhapple.github.io/posts/JDBC-%EC%9D%B4%ED%95%B4-%28+%EC%BD%94%EB%93%9C%29/#jdbc-%ED%99%9C%EC%9A%A9)

> MemberRepository

```java
private final DataSource dataSource;  
  
public MemberRepositoryV1(DataSource dataSource) {  
	this.dataSource = dataSource;  
}

//코드가 유지된다는 것을 알기 위한 save.
public Member save(Member member) throws SQLException {  
	String sql = "insert into member(member_id, money) values(?, ?)";  
	  
	Connection con = null;  
	PreparedStatement pstmt = null;  
	  
	try {  
		con = getConnection();  
		pstmt = con.prepareStatement(sql);  
		pstmt.setString(1, member.getMemberId());  
		pstmt.setInt(2, member.getMoney());  
		pstmt.executeUpdate();  
		return member;  
	} catch (SQLException e) {  
		log.error("db error", e);  
		throw e;  
	} finally {  
		close(con, pstmt, null);  
	}  
}

//findById()...
//update()...
//delete()...

private void close(Connection con, Statement stmt, ResultSet rs) {  
  
	JdbcUtils.closeResultSet(rs);  
	JdbcUtils.closeStatement(stmt);  
	JdbcUtils.closeConnection(con);  
}  
  
private Connection getConnection() throws SQLException {  
	Connection con = dataSource.getConnection();  
	log.info("get connection={}, class={}");  
	return con;  
	//return DBConnectionUtil.getConnection();  
}
```

- DataSource 의존관계 주입
	- 외부에서 DataSource를 주입받아 사용한다. 이제 직접 만든 DBConnectionUtil을 사용하지 않는다.
	- getConnection() 부분을 dataSource에서 받아오는 것으로 변경하였다.
	- 이렇게 받아오는 부분만 변경하면 나머지 코드는 그대로 사용하는 것이다.
	- DataSource는 표준 인터페이스이기 때문에 DriverManagerDataSource에서 HikariDataSource로 변경되어도 해당 코드는 변경할 필요 없다.
- JdbcUtils 편의 메서드
	- 스프링은 JdbcUtils를 제공한다.
	- 커넥션을 좀 더 편리하게 닫을 수 있다.

```java
class MemberRepositoryV1Test {  
  
	MemberRepositoryV1 repository;  
	  
	@BeforeEach  
	void beforeEach() {  
		//기본 DriverManager - 항상 새로운 커넥션 획득  
		DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);  
		repository = new MemberRepositoryV1(dataSource);  
	}  
	  
	@Test  
	void crud() throws SQLException {
		//save  
		Member member = new Member("memberV3", 10000);  
		repository.save(member);  
		
		//findById  
		Member findMember = repository.findById(member.getMemberId());  
		assertThat(findMember).isEqualTo(member);  
		  
		//update: money: 10000 -> 20000  
		repository.update(member.getMemberId(), 20000);  
		Member updatedMember = repository.findById(member.getMemberId());  
		assertThat(updatedMember.getMoney()).isEqualTo(20000);
		  
		//delete  
		repository.delete(member.getMemberId());  
		assertThatThrownBy(() -> repository.findById(member.getMemberId()))  
		.isInstanceOf(NoSuchElementException.class);  
		}  
}
```

DriverManagerDataSource를 이용하면, 커넥션 풀을 이용하지 않기 때문에 로그를 보면 save, findById, update 등등을 할 때 마다 새로운 커넥션을 연결하는 것을 볼 수 있다.

```java
@BeforeEach  
void beforeEach() {  
	// //기본 DriverManager - 항상 새로운 커넥션 획득  
	// DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);  
	// repository = new MemberRepositoryV1(dataSource);  
  
	//커넥션 풀링  
	HikariDataSource dataSource = new HikariDataSource();  
	dataSource.setJdbcUrl(URL);  
	dataSource.setUsername(USERNAME);  
	dataSource.setPassword(PASSWORD);  
	  
	repository = new MemberRepositoryV1(dataSource);  
}
```
커넥션 풀에서 커넥션을 끌어와 사용하게 된다.

그런데 로그를 보면 save(), update() 등등 모두 conn0 즉, 하나의 커넥션을 사용하는 것을 볼 수 있다. 

우리가 save() 등 메서드를 작성할 때 finally에 close()를 넣어주었다. 커넥션을 종료하는 것이 아닌 반환하는 메서드이다.

테스트는 순차적으로 실행되기 때문에 반환하고 사용하고의 반복인 것이다. 실제 웹 애플리케이션이라면 여러 쓰레드에서 동시에 요청이 올 때 각각 다른 커넥션을 사용할 것이다.

> DI

DriverManagerDataSource -> HikariDataSource로 변경해도 MemberRepositoryV1의 코드는 전혀 변경하지 않는다. DataSource 인터페이스에만 의존하기 때문이다 (생성자). 이것이 DataSource를 사용하는 장점이다.

(DI + OCP)
