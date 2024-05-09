---
title: Controller 테스트에서 만난 오류 (csrf, HttpMessageNotReadableException)
author: leedohyun
date: 2024-04-16 21:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

## 개요

```java
@Operation(summary = "웹툰 정보 가져오기")  
@ApiResponses(value = {  
  @ApiResponse(responseCode = "200", description = "웹툰 정보 조회 성공", content = @Content(schema = @Schema(implementation = CommentResponseDto.class))),  
  @ApiResponse(responseCode = "400", description = "해당 웹툰이 존재하지 않습니다.", content = @Content(schema = @Schema(implementation = ApiError.class))),  
  @ApiResponse(responseCode = "401", description = "로그인이 필요합니다.", content = @Content(schema = @Schema(implementation = ApiError.class)))  
})  
@PostMapping("/new/{titleId}")  
public ApiSuccess<CommentResponseDto> createComment(@PathVariable String titleId,  
		  @RequestBody CommentRequestDto commentRequestDto,  
		  Authentication authentication) { 
   
  String username = authentication.getName();  
  CommentResponseDto createdComment = commentsService.createComment(commentRequestDto, titleId, username);  
  
  return ApiUtil.success(createdComment);  
}
```

CommentsController의 일부 메서드이다.

이번 포스트에서는 위 메서드의 테스트 코드를 작성하면서 마주했던 문제들에 대해 정리해보고자 한다.

## 문제1 - CSRF 문제

```java
@Test  
@DisplayName("댓글 생성 성공 테스트")  
@WithMockUser("user1")  
void testCreateComment() throws Exception {  
  
  String username = "username";  
  String titleId = "titleId";  
  
  CommentRequestDto commentRequestDto = new CommentRequestDto();  
  commentRequestDto.setContent("content");  
  
  CommentResponseDto commentResponseDto = CommentResponseDto.builder()  
		  .id(0L)  
		  .nickName("nickName")  
		  .likeCount(0L)  
		  .content(commentRequestDto.getContent())  
		  .writeTime("writeTime")  
		  .build();  
  
  given(commentsService.createComment(commentRequestDto, titleId, username)).willReturn(commentResponseDto);  

  mockMvc.perform(post("/api/comments/new/{titleId}", titleId))  
		  .andExpect(status().isOk())  
		  .andExpect(jsonPath("$.response").exists());  
}
```

컨트롤러 테스트에서 나는 @WebMvcTest를 이용하여 코드를 작성하기로 결정했다.

그리고 MockMvc를 사용하여 해당 컨트롤러 메서드 엔드포인트에 대해 테스트 코드를 작성했다.

그런데 위와 같이 작성했을 때 403 error가 발생했다.

분명 나는 이전 포스트에서 정리한 @WithMockUser를 사용해 인증에 대한 문제를 해결하고자 했는데, 403 에러가 발생해서 당황스러웠다.

로그를 확인해도 권한이 잘 생성되어 있는 모습을 확인할 수 있었다.

***검색을 해보니 CSRF 관련 문제였다.***

### CSRF?

- 공격자가 악의적인 코드를 심어놓은 사이트를 만들고, 로그인 된 사용자가 클릭하게 만들어 사용자 의지와 무관한 요청을 발생시키는 공격.

스프링 시큐리티에서는 이를 해결하기 위해 "CSRF 토큰"을 이용해 토큰 값을 비교해 일치하는 경우에만 메서드를 처리하도록 만든다.

```java
mockMvc.perform(post("/api/comments/new/{titleId}", titleId)    
			   .with(csrf()))  
	   .andExpect(status().isOk())  
	   .andExpect(jsonPath("$.response").exists());
```

위와 같이 with(csrf())를 추가해주면 문제는 해결된다.

_csrf 값을 같이 보내주어 해당 문제가 발생하지 않도록 하는 것이다.

## 문제2 - Required request body is missing

위 문제를 해결하니 이번에는 400 에러가 발생하는 모습을 볼 수 있었다.

에러 메시지를 확인하니 "Required request body is missing" 에러를 만날 수 있었다.

나는 Mock의 given에서 RequestDto를 정상적으로 넣어준다고 생각했는데 로그를 보니 body에 아무런 값이 들어가있지 않았다.

![](https://blog.kakaocdn.net/dn/cuVAHi/btsHhrwkpX9/KwrzG2AJuZB18KeyNAAXTK/img.png)

그렇다면 어떻게 값을 넣어줄 수 있을까?

### 해결방법

```java
ObjectMapper objectMapper = new ObjectMapper();  
String requestBody = objectMapper.writeValueAsString(commentRequestDto);

mockMvc.perform(post("/api/comments/new/{titleId}", titleId)    
			   .contentType(MediaType.APPLICATION_JSON)  
			   .content(requestBody)
			   .with(csrf()))  
	   .andExpect(status().isOk())  
	   .andExpect(jsonPath("$.response").exists());
```

위와 같이 commentRequestDto를 ObjectMapper를 이용해 JSON 문자열로 변환해주고, 타입을 JSON으로 변경하여 해결했다.

Spring Mvc Test에서는 자동으로 객체를 JSON으로 변환해 요청 본문에 추가해주지 않는다고 한다.

@RequestBody를 사용하는 메서드의 테스트에서는 직접 객체를 JSON으로 바꿔주는 작업이 필요하다.


## 문제3 - ResponseBody에 아무런 내용이 없음

이제 모든 문제가 해결되고 200 응답을 받아냈다.

그런데 응답 바디값의 검증을 하려했는데 응답 바디에 아무런 값이 들어가지 않았다.

문제는 컨트롤러의 메서드에서 찾을 수 있었다.

컨트롤러의 메서드를 보면 Authentication을 매개변수로 사용하고 있다.

하지만 해당 값이 주어지지 않아 응답은 200이지만 응답 바디에는 제대로 된 값이 들어가지 않는 것이었다.

### 해결

```java
@Test  
@DisplayName("댓글 생성 성공 테스트")  
@WithMockUser("user1")  
void testCreateComment() throws Exception {  
  
  String username = "username";  
  String titleId = "titleId";  
  
  CommentRequestDto commentRequestDto = new CommentRequestDto();  
  commentRequestDto.setContent("content");  
  
  CommentResponseDto commentResponseDto = CommentResponseDto.builder()  
		  .id(0L)  
		  .nickName("nickName")  
		  .likeCount(0L)  
		  .content(commentRequestDto.getContent())  
		  .writeTime("writeTime")  
		  .build();  
  
  given(commentsService.createComment(commentRequestDto, titleId, username)).willReturn(commentResponseDto);  
  
  ObjectMapper objectMapper = new ObjectMapper();  
  String requestBody = objectMapper.writeValueAsString(commentRequestDto);  
  
  mockMvc.perform(post("/api/comments/new/{titleId}", titleId)  
				  .contentType(MediaType.APPLICATION_JSON)  
				  .content(requestBody)  
				  .with(csrf())  
				  .with(user(username)))  
		  .andExpect(status().isOk())  
		  .andExpect(jsonPath("$.response").exists());  
}
```

최종적인 테스트 코드는 위와 같다.

.with(user(username))을 이용해 Authentication에 해당하는 부분을 충족시킬 수 있었다.

이렇게 해주면 Authentication 개체를 매개변수로 기대하는 메서드에서 테스트 실행 중 지정된 사용자 이름으로 해당 개체를 수신하게 된다.

결과적으로 컨트롤러 메서드는 Authentication 객체에서 getName()을 통해 사용자 이름을 검색하고 인증된 사용자에 대해 로직을 수행할 수 있게 되어 응답 바디에 의도한 대로 적절한 데이터가 삽입되게 되는 것이다.