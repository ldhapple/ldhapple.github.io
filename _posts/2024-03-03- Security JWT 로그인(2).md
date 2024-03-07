---
title: Spring Security + JWT를 이용한 회원가입 및 로그인 (2)
author: leedohyun
date: 2024-03-03 20:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

이제 JWT를 어떻게 구현하는 지 알아보자. 버전은 0.12.3 버전이다.

## JWT (단일 토큰)

- 로그인 -> 성공 -> JWT 발급
- 접근 시 -> JWT 검증

위와 같이 로그인이 성공했을 때 토큰을 발급할 뿐 아니라 로그인 이후 어떤 URL에 접근할 때 JWT 토큰에 관한 검증도 필요하다.

따라서 JWT에 관해 발급과 검증을 담당하는 클래스가 필요하다.

### 암호화 키 저장

- application.properties

```properties
spring.jwt.secret=sdfjlskfsjelfijasflksjfldifjslfkj
```

암호화 키는 하드코딩 방식으로 구현 내부(일부 클래스)에 탑재하는 것을 지양하기 때문에 변수 설정 파일에 저장한다.

### JwtUtil

- 토큰 Payload에 저장될 정보
	- username(id)
	- role
	- 생성일
	- 만료일

```java
@Component  
public class JwtUtil {  
  
    private SecretKey secretKey;  
  
    public JwtUtil(@Value("${spring.jwt.secret}") String secret) {  
	    secretKey = new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), Jwts.SIG.HS256.key()
			    .build()
			    .getAlgorithm());  
    }  
  
    public String getUsername(String token) {  
	    return Jwts.parser().verifyWith(secretKey).build()
			    .parseSignedClaims(token).getPayload()
			    .get("username", String.class);  
    }  
  
    public String getRole(String token) {  
  	    return Jwts.parser().verifyWith(secretKey).build()
			    .parseSignedClaims(token).getPayload()
			    .get("role", String.class);  
    }  
  
    public Boolean isExpired(String token) {  
	    return Jwts.parser().verifyWith(secretKey).build()
			    .parseSignedClaims(token).getPayload()
			    .getExpiration().before(new Date());  
    }  
  
    public String createJwt(String username, String role, Long expiredMs) {  
	    return Jwts.builder()  
		    .claim("username", username)  
		    .claim("role", role)  
		    .issuedAt(new Date(System.currentTimeMillis()))  
		    .expiration(new Date(System.currentTimeMillis() + expiredMs))  
		    .signWith(secretKey)  
		    .compact();  
    }  
}
```

- SecretKey (생성자)
	- String key는 JWT 기반에서 사용하지 않는다.
	- 따라서 위에서 만들어놓은 암호화 키를 사용하여 새롭게 시크릿 키 객체를 생성해 사용한다.
- getUsername, getRole, isExpired
	- 검증을 진행할 때 필요한 메서드들이다.
	- 토큰의 payLoad에서 해당 정보들을 가져오게 된다.
- createJwt()
	- 토큰을 발급하는 메서드이다.
	- 토큰에 저장될 정보들을 담아 토큰을 생성하게 된다. 

### JwtUtil 이용

이전 포스트에도 코드를 그냥 두었지만 다시 봐보자.

```java
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
```

로그인이 성공했을 때 실행되는 메서드라고 했다.

- Authentication.getPrincipal()을 이용해 UserDetails를 받는다.
	- Account 정보를 이용할 수 있게 된다.
	- AuthenticationManager를 통해 검증할 때 UserDetails를 통해 검증한다고 했다. 그 검증 성공한 정보를 다시 반환받는 것이라고 생각하면 된다.
- Account 정보를 이용해 username, role을 받아 해당 정보를 바탕으로 JWT 토큰을 발급하는 것이다.

> 인증 방식

응답 헤더에 Authorization에 Bearer를 붙이는 방식을 볼 수 있다.

이는 HTTP 인증 방식이 RFC 7235 정의에 따라 특정 인증 헤더 형태를 가져야하기 때문이다.

```
Authorization: 타입 + 인증 토큰

//예시
Authorization: Bearer + 인증토큰(String)
```

### JWT 검증 필터

이제 로그인에 성공하면 JWT 토큰을 발급해 응답 헤더에 담아 클라이언트에 보낸다.

이제 이 JWT를 검증하기 위해 커스텀 필터를 만들어 검증을 하도록 한다.

커스텀 필터를 통해 요청 헤서 Authorization 키에 JWT가 존재하는 경우 JWT를 검증하고 강제로 SecurityContextHolder에 세션을 생성한다.

이 세션은 Stateless 상태로 관리되어 해당 요청이 끝나면 소멸된다.

```java
@RequiredArgsConstructor  
public class JwtFilter extends OncePerRequestFilter {  
  
    private final JwtUtil jwtUtil;  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
	    String authorization = request.getHeader("Authorization");  
  
        if (authorization == null || !authorization.startsWith("Bearer ")) {  
		    filterChain.doFilter(request, response);  
  
            return;  
        }  
  
	    String token = authorization.split("  ")[1];  
  
        if (jwtUtil.isExpired(token)) {  
		    filterChain.doFilter(request, response);  
  
            return;  
        }  
  
	    String username = jwtUtil.getUsername(token);  
        String role = jwtUtil.getRole(token);  
  
        Account account = Account.builder()  
		    .username(username)  
		    .role(Role.from(role))  
		    .build();  
  
        CustomUserDetails customUserDetails = new CustomUserDetails(account);  
  
        Authentication authToken = new UsernamePasswordAuthenticationToken(customUserDetails, null, customUserDetails.getAuthorities());  
  
        SecurityContextHolder.getContext().setAuthentication(authToken);  
  
        filterChain.doFilter(request,response);  
    }  
}
```

- 요청 헤더에서 Authorization의 키 값을 가져온다.
- 해당 키 값을 검증하고 만약 존재하지 않거나 정상적인 값이 아니라면 메서드를 종료시킨다.
- filterChain.doFilter()
	- 필터는 여러 개가 존재하는데, 만약 검증에 실패한다면 request, response를 담아 다음 필터로 넘기는 것이다.
- Bearer라는 접두사를 제거해 토큰을 획득한다.
	- 토큰이 만료되면 마찬가지로 다음 필터로 넘기고 메서드를 종료한다.

이후 토큰이 검증되고, 만료 시간도 검증이 되었다면, UsernamePasswordAuthenticationToken (스프링 시큐리티 인증 토큰)을 만든다.

- SecurityContextHolder.getContext().setAuthentication()
	- 세션에 사용자를 등록한다.
	- 이후 다음 필터로 넘기는 것이다.

이렇게 생성한 필터를 SecurityConfig에 등록해주면 된다.

## 단일 토큰의 문제점

위의 구현까지 보면 현재 일종의 Access 토큰 (사용자 인증에 사용) 만을 발급하고 있고, 이는 문제가 있다.

우선 세션 방식에 대비해서는 상태를 저장하지 않기 때문에 별도의 저장소가 필요하지 않고, 서버의 부하를 줄일 수 있다는 점에 대해서는 장점을 가지는 것은 맞다.

하지만 아래의 문제들을 보자.

- 보안 문제
	- Access 토큰은 보통 쿠키에 저장된다.
	- 그런데 Access 토큰이 탈취되었을 경우, 만료 시간동안 사용자의 인증 정보를 이용할 수 있다.
	- 그렇다고 만료시간을 짧게하면 사용자 경험이 저하된다.
- 토큰 갱신 문제
	- 이러한 단일 토큰이 만료되었을 경우 사용자는 다시 로그인을 해 토큰(권한)을 얻어야 한다.
	- 토큰을 갱신하는 로직이 없어 사용자 경험이 저하된다.
- 로그아웃
	- 로그아웃이 되면 발급한 토큰을 무효화해야 한다. 그리고 다시 로그인할 때 토큰을 새로 발급해야 한다.
	- 그러나 서버 측에서 특정 토큰을 즉시 무효화하기 어렵다.  

## Access 토큰 + Refresh 토큰 구현

위와 같이 단일 토큰 방식에는 문제점이 있기 때문에 보통 Access 토큰과 Refresh 토큰을 사용해 구현하게 된다.

- Access 토큰
	- 짧은 유효기간을 가지며, 사용자 인증에 사용된다.
	- 주로 리소스 접근 권한을 검증할 때 사용한다.
- Refresh 토큰
	- Access 토큰보다 보통 긴 유효기간을 가진다.
	- Access 토큰이 만료되었을 때 새로운 Access 토큰을 발급받기 위해 사용된다. 

우선 생각해야 할 부분이 있다.

Refresh 토큰을 만약 Access 토큰처럼 사용자의 응답 헤더에 넣어 쿠키를 사용하는 방식으로 사용한다면, Refresh 토큰과 Access 토큰을 같이 탈취당할 수 있기 때문에 보안에 관련한 문제가 해결되지 않는다.

이를 해결하기 위해 Redis를 사용한다.

### Redis 사용

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis:3.2.3'
```

```properties
spring.data.redis.host=localhost  
spring.data.redis.port=6379
```

```java
@Component  
@RequiredArgsConstructor  
public class RedisUtil {  
  
    private final RedisTemplate<String, String> redisTemplate;  
  
    public void saveRefreshToken(String username, String refreshToken, long duration) {  
	    redisTemplate.opsForValue().set(username, refreshToken, duration, TimeUnit.MILLISECONDS);  
    }  
  
    public String getRefreshToken(String username) {  
	    return redisTemplate.opsForValue().get(username);  
    }  
  
    public void deleteRefreshToken(String username) {  
	    redisTemplate.delete(username);  
    }  
}
```

Redis에 Refresh 토큰을 저장하고, 조회하고, 삭제하는 기능을 하는 RedisUtil 클래스를 작성했다.

### Jwt

그리고 JwtUtil에서는 Refresh 토큰을 생성하는 메서드를 추가한다.

```java
public String createRefreshToken(String username, Long expirationMs) {  
   return Jwts.builder()  
		  .claim("username", username)  
		  .issuedAt(new Date())  
		  .expiration(new Date(System.currentTimeMillis() + expirationMs))  
		  .signWith(secretKey)  
		  .compact();  
}
```

### Refresh 토큰 사용

우선 로그인이 성공했을 때 AccessToken을 발급하고, Refresh 토큰은 Redis에 저장하는 로직이 있어야 한다.

```java
@Override  
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,  
                                        Authentication authResult) throws IOException, ServletException {  
    
    CustomUserDetails customUserDetails = (CustomUserDetails) authResult.getPrincipal();  
  
    String username = customUserDetails.getUsername();  
  
    Collection<? extends GrantedAuthority> authorities = authResult.getAuthorities();  
    Iterator<? extends GrantedAuthority> iterator = authorities.iterator();  
    GrantedAuthority auth = iterator.next();  
  
    String role = auth.getAuthority();  
  
    String accessToken = jwtUtil.createAccessToken(username, role, 60 * 60 * 10L);  
    String refreshToken = jwtUtil.createRefreshToken(username, 60 * 60 * 10000L);  
  
    redisUtil.saveRefreshToken(username, refreshToken, 60 * 60 * 10000L);  
  
    response.addHeader("Authorization", "Bearer " + accessToken);  
}
```

![](https://blog.kakaocdn.net/dn/xdYJG/btsFEhtppPa/052Lud65kNarAuzkin5sQ0/img.png)

redis 내부를 보면 아이디가 key 값인 refresh 토큰이 정상적으로 저장되어 있는 것을 볼 수 있다.

그리고 Refresh 토큰은 Access 토큰이 만료되었을 때, 갱신의 목적으로 사용된다. 따라서 프론트 서버에서 갱신에 사용할 컨트롤러가 필요하다.

```java
@RestController  
@RequestMapping("/api/auth")  
@RequiredArgsConstructor  
public class AuthController {  
  
    private final JwtUtil jwtUtil;  
    private final RedisUtil redisUtil;  
  
    @PostMapping("/refresh")  
    public ResponseEntity<?> refreshAccessToken(HttpServletRequest request) {  
    
	    String authHeader = request.getHeader("Authorization");  
	    String accessToken = authHeader.split("  ")[1];  
	    String username = jwtUtil.getUsername(accessToken);  
	      
	    String storedRefreshToken = redisUtil.getRefreshToken(username);  
      
	    if (jwtUtil.isExpired(storedRefreshToken)) {  
		    return ResponseEntity.status(HttpStatus.FORBIDDEN).body("만료된 Refresh 토큰입니다.");  
	    }  
  
	    String newAccessToken = jwtUtil.createAccessToken(username, "USER", 60 * 60 * 10L);  
  
	    HttpHeaders responseHeaders = new HttpHeaders();  
	    responseHeaders.set("Authorization", "Bearer " + newAccessToken);  
	    return ResponseEntity.ok().headers(responseHeaders).body("Access 토큰이 갱신되었습니다.");  
	}  
  
	@PostMapping("/logout")  
    public ResponseEntity<?> logout(HttpServletRequest request) {  
	    String authHeader = request.getHeader("Authorization");  
  
        if (authHeader != null && authHeader.startsWith("Bearer ")) {  
		    String token = authHeader.split("  ")[1];  
            String username = jwtUtil.getUsername(token);  
  
            redisUtil.deleteRefreshToken(username);  
            return ResponseEntity.ok().body("로그아웃이 완료되었습니다.");  
        }  
  
	    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("로그인이 되어있지 않습니다.");  
    }  
}
```

- refresh
	- 헤더에서 만료되기 전 Access 토큰을 이용해 user을 식별한다.
	- 식별한 username을 통해 redis에서 Refresh 토큰을 조회한다.
	- Refresh 토큰이 만료되지 않았다면 새로운 Access 토큰을 발급한다.
- logout
	- 로그아웃을 할 때는 refresh 토큰을 삭제해주어야 한다.
	- 따라서 프론트 서버에서 로그아웃을 구현할 때 이 api를 호출해주면 된다.
	

### 문제점

간단하게 redis를 사용해 refresh 토큰을 저장하고 이를 access 토큰을 갱신하는데 사용했다.

하지만 이 방법에도 문제가 있다.

우선 요청헤더에서 사용자를 직접 식별하고 있는데, 요청헤더라는 것이 조작할 수 있는 정보이기 때문에 보안상 좋지 않다.

거기에 더해 Access 토큰이 결국 만료시간이 짧더라도 공격을 받을 수 있는 쿠키에 담겨진다는 것이다.

만료시간이 조금이라도 남은 토큰이 탈취되면 서버는 위조된 요청을 받을 수 있는 가능성이 존재하는 것이다.

따라서 이 부분도 개선이 필요하다.

개선에 대한 부분은 다음 포스트에서 다룬다.