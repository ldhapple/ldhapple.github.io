---
title: Spring Security + JWT를 이용한 회원가입 및 로그인 (1)
author: leedohyun
date: 2024-03-02 20:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

기본적으로 로그인을 구현하는데 여러 방법이 있다. 쿠키를 사용할 수도 있고, 세션을 사용할 수도 있다.

그 중에서도 이번 프로젝트에서 사용한 로그인 구현 방법에 대해 정리해보려 한다.

## 왜 Spring Security인가?

스프링 시큐리티는 인증과 권한의 역할을 가진다. 그렇다면 인증과 권한이 정확히 뭘까?

예를 들어 로그인에 대한 인증과 권한을 보자.

- 인증
	- 사용자가 로그인 페이지에 자격 증명을 제공해 자신을 인증한다.
	- 즉 말 그대로 사용자 자신을 인증하는데 필요한 것이다.
- 권한 부여
	- 로그인 후 시스템이 사용자에 대한 권한을 확인하고 해당 사용자가 액세스할 수 있는 리소스 및 기능을 결정한다.
	- 예를 들어 로그인 페이지는 전체 사용자가 접근할 수 있어야 한다.
	- 하지만 특정 게시판, 서비스에는 로그인 된 사용자 혹은 어떤 역할로 인증된 사용자만 접근이 가능해야 한다.
	
이러한 인증과 권한 부여에 대해 해결해주는 역할을 한다는 것이다.

그러면 다른 방법에 비해 무슨 장점이 있을까?

- 유연성
	- 다양한 인증 방식과 사용자 정의 보안 설정을 지원한다.
	- 따라서 다양한 요구사항을 쉽게 해결할 수 있고 OAuth, JWT 등 다양한 인증 방식도 지원한다.
- 스프링 프레임워크와의 호환
	- Spring Boot와 사용하면 설정이 간편하고, 보안을 빠르게 구성할 수 있는 장점이 있다.  

그리고 DB에 사용자 password를 저장할 때 문자 그대로를 저장하면 안되는데, 이런 부분의 암호화를 해주는 역할도 한다.

요약하자면 편리하고 쉽게 인증과 권한부여를 하기 위해 사용한다.

> 다양한 웹 보안 문제

- CSRF(사이트 간 요청 위조)
	- 서비스 사용자의 의지와 다르게 공격자의 행위(등록, 수정, 삭제)를 서버에 요청하는 공격을 말한다.
	- 사용자가 서비스에 로그인한 상태에서 CSRF 공격 코드가 삽입된 페이지를 열면 해당 공격 명령을 서버가 신용해 공격에 노출되는 방식이다.
	- 서버가 쿠키를 통한 단순한 사용자 확인 과정을 거친다면 공격 피해 위험이 있다.
- XSS(사이트 간 스크립팅)
	- 웹 사이트 관리자가 아닌 사람이 웹 페이지에 악성 스크립트를 삽입하는 공격
	- 웹 어플리케이션이 사용자로부터 입력받은 값을 제대로 검사하지 않고 사용할 경우 나타난다.
	- 해커가 사용자의 정보(쿠키, 세션 등)을 탈튀하거나, 자동으로 비정상적인 기능을 수행할 수 있도록 한다.
- CORS(교차 출처 자원 공유)
	-  한 출처에서 실행 중인 웹 애플리케이션이 다른 출처의 보호된 자원에 접근할 수 있는 권한을 부여하도록 브라우저에 알려주는 정책.
	- Vue.js와 Spring 서버 간의 통신에 CORS 오류를 만났다.
	- CORS를 활성화하지 않으면 브라우저는 보안 상의 이유로 스크립트에서 시작한 교차 출처 HTTP 요청을 제한한다.
		- 동일 출처 정책.

Spring Security는 기본적인 수준에서 CSRF나 XSS 공격을 방어하기 위한 기능도 제공한다.

## 왜 JWT 토큰인가?

로그인에 대한 부분을 관리할 때, 쿠키, 세션 등을 사용할 수 있다.

하지만 나는 JWT 토큰 방식을 택했다. 장점을 알아보자.

### JWT란?

Json Web Token의 줄임말로 JSON 객체의 정보를 토큰 기반의 인증 시스템을 이용해 안전하게 정보를 전송하는 방법이다.

JWT는 Base64로 인코딩되는 구조를 갖고 있다.

[jwt 홈페이지](https://jwt.io/)를 들어가면 json 파일이 바뀐 결과물도 볼 수 있다.

### JWT 구조

![image](https://user-images.githubusercontent.com/81945553/197397175-c4b39882-e5a4-4eb4-b7f5-e74198ed7d3a.png)

- Header, Payload, Signature(서명)으로 총 세 가지를 포함하는 구조를 가진다.

> Header

alg 및 typ로 구성되어 있는 것을 볼 수 있다.

alg는 알고리즘 방식을 지정하는 것으로 시그니처를 해싱하기 위한 알고리즘을 저장하는 것이다. typ는 토큰의 타입을 지정한다.

> Payload

페이로드에는 토큰에서 사용할 정보의 조각인 Claim이 담겨 있다. 

클레임은 엔티티 및 데이터에 대한 설명으로 등록, 공개, 개인 총 3가지 타입이 존재한다.

- 등록 클레임
	- 필수 X
	- 권장되는 클레임 집합으로 상호 운용 가능한 클레임을 제공한다.
	- ex) 토큰 발행자, 토큰 대상자, 토큰 만료 시간, 토큰 식별자 등등
- 공개 클레임
	- JWT를 원하는대로 정의할 수 있다.
	- 충돌 가능성이 있어 이를 방지하기 위해 IANA JSON Web Token Registry나 네임스페이스를 포함하는 URI를 정의해 사용해야 한다.
- 개인 클레임
	- 당사자 간 개인적으로 정보를 공유하기 위해 만드는 사용자 정의 클레임.  

> Signature

토큰을 인코딩하거나 유효성 검증을 할 때 사용하는 고유 암호화 코드이다.

Signature가 Header와 Payload의 값을 각각 Base64로 인코딩하고, 인코딩한 값을 비밀 키를 이용해 Header에 정의한 알고리즘으로 해싱한다.

그리고 해싱한 값을 Base64로 인코딩하여 생성한다.

### 쿠키, 세션 방식의 장단점

쿠키, 세션 인증 방식은 서버가 클라이언트의 요청에 대한 응답을 작성할 때 인증 정보를 서버에 저장하고 클라이언트 식별자인 JSESSIONID를 쿠키에 담아 유효성을 판단한다.

> 장점

- 구현이 간편하다. 클라이언트에 쿠키를 저장하고 서버에서 읽는 방식으로 구현.
- 세션은 서버측에 저장되기 때문에 클라이언트에서 직접 조작이 불가능하다.
- 쿠키는 클라이언트측에 저장되어 로그인 상태를 유지하기에 편리하다.

> 단점

- 쿠키는 클라이언트 측에 저장되어 보안에 취약하다.
- 세션은 서버측에 저장되어 서버 자원을 소비하기 때문에 서비스가 커지면 서버 부하가 증가한다.
	- 그리고 세션은 서버 간의 상태 공유를 필요로 해 서버의 확장이 어렵다.

### 그래서 왜 JWT?

JWT 인증 방식을 알아보자.

- 클라이언트의 요청이 온다.
- 정보를 Payload에 담아 비밀키를 사용해 Access Token을 발급하여 클라이언트에 전달한다.
- 클라이언트는 Access Token을 저장해두고 서버에 요청할 때 마다 토큰을 Request Header 안에 Authorization을 포함시켜 전달한다.
- 서버는 토큰의 Signature를 비밀키로 복호화하고 위변조 및 유효기간을 확인하여 검증이 완료되면 요청에 응답한다.

> 장점

- header와 payload를 가지고 signature를 생성하여 데이터 위변조를 방지.
- 세션과 다르게 별도의 저장소가 필요하지 않고 무상태가 된다.
- 토큰을 기반으로 OAuth 등의 다른 시스템에 접근 및 공유가 가능하다.

> 단점

- JWT 토큰의 길이가 길어 인증 요청이 많아질수록 네트워크 부하가 심해질 수 있다.
- Payload 자체에는 암호화되지 않아 중요한 정보를 담을 수 없다.
- 토큰을 탈취당하면 대처하기 어렵다.
	- 특정 토큰을 강제 만료하기 어렵다.

이러한 단점에 대해 보완하는 부분은 추후 포스팅에서 다룰 것이다. (Access 토큰의 관리 및 사용이 중요하다.)

요약하자면 서버의 확장이 더 쉽고, 토큰 자체에 필요한 정보를 저장해 매번 다른 저장소에 접근할 필요가 없다.

그리고 클라이언트 측에 저장하는데도 서명과 암호화를 통해 안전하고, 서버에 저장할 필요도 없어 서버 부하도 줄일 수 있다.

## 사용법 및 구현

우선 build.gradle 파일에 의존성을 추가한다.

```
implementation "org.springframework.boot:spring-boot-starter-security"
```

```
implementation 'io.jsonwebtoken:jjwt-api:0.12.3'  
implementation 'io.jsonwebtoken:jjwt-impl:0.12.3'  
implementation 'io.jsonwebtoken:jjwt-jackson:0.12.3'
```

참고로 Spring Security 버전은 6.2.2 이다.

### SecurityConfig 기본 설정

```java
@Configuration  
@EnableWebSecurity  
public class SecurityConfig {  

  @Bean  
  public BCryptPasswordEncoder bCryptPasswordEncoder() {  
      return new BCryptPasswordEncoder();  
  }
  
  @Bean  
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
  
	    //disable
		http  
		        .csrf(AbstractHttpConfigurer::disable)  
			    .formLogin(AbstractHttpConfigurer::disable)  
			    .httpBasic(AbstractHttpConfigurer::disable);  
  
        //경로별 인가  
		http  
		        .authorizeHttpRequests((auth) -> auth  
	                    .requestMatchers("/login", "/", "*/register").permitAll()  
					    .requestMatchers("/admin").hasRole("ADMIN")  
					    .anyRequest().authenticated());
		//세션 설정  
		http  
		        .sessionManagement((session) -> session  
		                .sessionCreationPolicy(SessionCreationPolicy.STATELESS));
                
		return http.build();  
    }  
}
```

- disable
	- csrf
		- 세션 방식에서는 사용자의 인증된 세션을 가지고 공격하는 csrf 방식의 보안이 필요 없다. (JWT에서는 세션이 stateless)
		- 따라서 공격을 방어하는 CSRF 토큰 기능을 비활성화 했다. 
	- formLogin, httpBasic
		- 웹 기반의 폼 로그인 및 HTTP 기본 인증을 비활성화 한 것
	- disable()의 이유는 JWT 토큰을 사용하기 때문이고 불필요한 리소스 낭비를 줄일 수 있다.
	- 대신 보안상 문제가 있을 수 있기 때문에 JWT 토큰의 보안을 잘 구현해야 한다.
- 경로별 인가
	- 필요 시 경로를 수정하면 된다.
	- permitAll() 해당 경로는 모두 접근 가능.
	- hasRole() : 해당 Role을 가진 유저만 접근이 가능.
	- anyRequest().authenticated() : 다른 요청들은 모두 로그인(권한이 있는)이 된 사람만 접근 가능하다.
- 세션 설정 (중요)
	- JWT 방식에서는 항상 세션을 Stateless하게 관리해야 한다.

> BCryptPasswordEncoder

스프링 시큐리티에서 제공하는 클래스 중 하나로 비밀번호를 암호화하는데 사용할 수 있는 메서드를 가진다.

빈으로 등록해두고 주입받아 사용한다.

주로 encode() (password 암호화) 메서드, matches() (password 일치 여부) 메서드를 사용하게 된다.

```java
public String encode(CharSequence rawPassword) {  
    if (rawPassword == null) {  
        throw new IllegalArgumentException("rawPassword cannot be null");  
    } else {  
        String salt = this.getSalt();  
        return BCrypt.hashpw(rawPassword.toString(), salt);  
    }  
}
```

```java
public boolean matches(CharSequence rawPassword, String encodedPassword) {  
    if (rawPassword == null) {  
        throw new IllegalArgumentException("rawPassword cannot be null");  
    } else if (encodedPassword != null && encodedPassword.length() != 0) {  
        if (!this.BCRYPT_PATTERN.matcher(encodedPassword).matches()) {  
            this.logger.warn("Encoded password does not look like BCrypt");  
            return false;  
        } else {  
            return BCrypt.checkpw(rawPassword.toString(), encodedPassword);  
        }  
    } else {  
        this.logger.warn("Empty encoded password");  
        return false;  
    }  
}
```

### 회원 가입

```
/register (요청) -> dto (json, controller) -> Entity -> Repository
```

위와 같은 흐름으로 회원 가입 요청이 오면 회원 데이터를 저장하게 된다.

```java
public Account register(RegisterDto registerDto) {  
	String encodedPassword = bCryptPasswordEncoder.encode(registerDto.getPassword());  
    Mbti mbti = mbtiRepository.findByMbtiType(MbtiType.from(registerDto.getMbtiType()));  
  
    Account account = Account.builder()  
		  .realName(registerDto.getRealName())  
		  .username(registerDto.getUsername())  
		  .nickName(registerDto.getNickname())  
		  .password(encodedPassword)  
		  .gender(registerDto.getGender())  
		  .mbti(mbti)  
		  .role(Role.USER)  
		  .build();  
  
    return accountRepository.save(account);  
}
```

서비스 단에서 Repository를 통해 Account를 저장할 때, PasswordEncoder를 이용해 인코딩해주면 DB에서 사용자가 입력한 비밀번호가 그대로 드러나지 않는 것을 볼 수 있다.

### 로그인

우선 커스텀 필터를 생성해야 한다. 

커스텀 필터를 생성하는 이유를 보자.

![](https://substantial-park-a17.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35178ade-3cd1-4c3b-8e0f-5e20ae225cea%2F0d3452b2-a903-476a-a7bd-6ba01dafd2de%2F%25EC%258A%25A4%25ED%2581%25AC%25EB%25A6%25B0%25EC%2583%25B7_2023-12-27_04.42.09.png?table=block&id=8bd465c6-e9f1-4447-a761-38cd20fc11bf&spaceId=35178ade-3cd1-4c3b-8e0f-5e20ae225cea&width=580&userId=&cache=v2)

기본적으로 서블릿 컨테이너에 존재하는 필터 체인에 DelegatingFilter를 등록한 뒤 모든 요청을 가로챈다.

![](https://substantial-park-a17.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F35178ade-3cd1-4c3b-8e0f-5e20ae225cea%2F569c7a1d-bacc-453d-bac1-e219660f5cd4%2F%25EC%258A%25A4%25ED%2581%25AC%25EB%25A6%25B0%25EC%2583%25B7_2023-12-27_07.39.14.png?table=block&id=479ba9ab-ae91-4f21-968d-9e5b5aecd784&spaceId=35178ade-3cd1-4c3b-8e0f-5e20ae225cea&width=1250&userId=&cache=v2)

이렇게 가로챈 요청들을 SecurityFilterChain에서 처리 후 상황에 따라 거부, 리디렉션, 서블릿으로 요청 전달(성공) 등의 거름망 역할을 한다.

이러한 스프링 SecurityFileterChain에는 여러가지 필터가 있는데, 그 중 UsernamePasswordAuthenticationFilter에서 회원 검증을 한다.

그런데 이 필터는 Form 로그인 방식에서 회원 검증을 한다. 해당 필터가 호출한 AuthenticationManager를 통해 검증하는 방식이다.

이 AuthenticationManager은 UserDetailsService를 통해 유저 정보를 DB에서 조회할 수 있다.

하지만 설정에서 Form 로그인 방식을 disable 했기 때문에 해당 필터가 동작하지 않고, 따라서 로그인을 위한 커스텀 필터가 필요한 것이다.

- 아이디, 비밀번호 검증을 위한 커스텀 필터
- DB에 저장되어 있는 회원 정보를 기반으로 검증하는 로직
- 로그인 성공 시 JWT 토큰을 반환할 핸들러
- 작성한 커스텀 필터를 SecurityConfig에 등록

위와 같은 흐름으로 작성된다.


> 커스텀 필터

```java
@RequiredArgsConstructor  
public class LoginFilter extends UsernamePasswordAuthenticationFilter {  
  
	private final AuthenticationManager authenticationManager;  
	private final JwtUtil jwtUtil;  
  
	@Override  
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {  
  
		  try {  
			  LoginDto loginForm = new ObjectMapper().readValue(request.getInputStream(), LoginDto.class);  
  
//        	  String username = obtainUsername(request);  
//        	  String password = obtainPassword(request);  
			  String username = loginForm.getUsername();  
              String password = loginForm.getPassword();  
  
	          UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(username, password, null);  
  
	          return authenticationManager.authenticate(authToken);  
	      } catch (IOException e) {  
			  throw new RuntimeException(e);  
	      }  
	 }  
  
	  @Override  
	  protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {  
	    
	    CustomUserDetails customUserDetails = (CustomUserDetails) authResult.getPrincipal();  
  
        String username = customUserDetails.getUsername();  
  
        Collection<? extends GrantedAuthority> authorities = authResult.getAuthorities();  
        Iterator<? extends GrantedAuthority> iterator = authorities.iterator();  
        GrantedAuthority auth = iterator.next();  
  
        String role = auth.getAuthority();  
  
        String token = jwtUtil.createJwt(username, role, 60*60*10L);  
  
        response.addHeader("Authorization", "Bearer " + token);  
    }  
  
	  @Override  
	  protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException failed) throws IOException, ServletException {  
		  response.setStatus(401);  
    }  
}
```

- 우선 UsernamePasswordAuthenticationFilter를 상속받는다.
- attemptAuthentication()
	- 로그인 시 id, password를 JSON 형태로 받으므로 loginForm Dto를 통해 요청에서 id와 password를 읽어온다.
	- username과 password를 가지고 token을 발급받는다. (시큐리티에서 검증을 위해 토큰 필요)
	- 토큰을 가지고 AuthenticationManager에 검증을 요청한다. 
- successfulAuthentication()
	- 로그인(검증)에 성공하면 이 메서드가 실행된다.
	- 따라서 JWT 토큰을 이 메서드에서 발급한다.
- unsuccessfulAuthentication()
	- 로그인에 실패하면 이 메서드가 실행된다. 

> UserDetailsService

```java
@Service  
@RequiredArgsConstructor  
public class LoginService implements UserDetailsService {  
  
    private final AccountRepository accountRepository;  
  
    @Override  
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
  
	    Account findAccount = accountRepository.findByUsername(username);  
  
        if (findAccount != null) {  
		    return new CustomUserDetails(findAccount);  
        }  
  
	    return null;  
    }  
}
```

- DB에서 id를 통해 account를 조회한다.
- 해당 account가 존재했을 때 UserDetails에 담아 반환하면 AuthenticationManager가 검증하는 것이다.
- 즉, AuthenticationManager에서 검증받을 수 있도록 커스텀 UserDetailService를 만드는 것이다.

> UserDetails

```java
@RequiredArgsConstructor  
public class CustomUserDetails implements UserDetails {  
  
    private final Account account;  
  
    @Override  
    public Collection<? extends GrantedAuthority> getAuthorities() {  
	    Collection<GrantedAuthority> authorities = new ArrayList<>();  
 
        authorities.add(new GrantedAuthority() {  
			@Override  
		    public String getAuthority() {  
			    return account.getRole().toString(); 
            }  
		});  
  
        return authorities;  
    }  
  
    @Override  
    public String getPassword() {  
	    return account.getPassword();  
    }  
  
    @Override  
    public String getUsername() {  
	    return account.getUsername();  
    }  
  
    @Override  
    public boolean isAccountNonExpired() {  
	    return true;  
    }  
  
    @Override  
    public boolean isAccountNonLocked() {  
	    return true;  
    }  
  
    @Override  
    public boolean isCredentialsNonExpired() {  
	    return true;  
    }  
  
    @Override  
    public boolean isEnabled() {  
	    return true;  
    }  
}
```

- AuthenticationManager에 검증을 위한 데이터를 넘겨주는 일종의 DTO이다.
- LoginForm은 단순하게 username과 password값을 받기 위해 생성한 것이다.
- 이 DTO는 account 객체를 이용해 AuthenticationManager에 전달해 검증할 수 있도록 한다.

> 필터 등록

```java
http  
		.addFilterAt(customLoginFilter, UsernamePasswordAuthenticationFilter.class);
```

SecurityConfig의 filterChain()에 위와 같이 필터를 추가해주면 동작하게 된다.

기본적으로 /login 경로에서 로그인 로직이 이루어지도록 알아서 설정되어 있다.

```java
LoginFilter customLoginFilter = new LoginFilter(authenticationManager(authenticationConfiguration), jwtUtil);  
customLoginFilter.setFilterProcessesUrl("/api/login");
```

만약 로그인 요청 URL을 바꾸고 싶다면 위와 같이 설정할 수 있다.

여기까지 구현했을 때 우선 로그인 로직은 정상적으로 동작하게 된다.

JWT 관련 내용은 다음 포스트에서 다룬다.