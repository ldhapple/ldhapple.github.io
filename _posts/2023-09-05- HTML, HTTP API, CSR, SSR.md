---
title: HTML, HTTP API, CSR, SSR
author: leedohyun
date: 2023-09-05 19:13:00 -0500
categories: [JAVA, 스프링 MVC 기본]
tags: [java, Spring, SpringBoot]
---

## 정적 리소스

![](https://blog.kakaocdn.net/dn/ZGO9m/btstfkK5u7T/0M8OdvOVkQVFXkXtUVCJdk/img.png)

정적 리소스를 제공할 때에는 고정된 HTML 파일, CSS, JS, 이미지, 영상 등을 제공하면 된다.

## HTML 페이지

![](https://blog.kakaocdn.net/dn/1EqLQ/btstrPvcBUB/ubt6yhHGgtoXfnAlC3KRJK/img.png)

동적으로 필요한 HTML 파일을 생성해 전달한다.

웹 브라우저는 주어진 HTML을 해석하는 역할을 맡는다.

## HTTP API

![](https://blog.kakaocdn.net/dn/kJVG6/btstrNqB6M6/r688mer6t4ozYD6bzVGgTK/img.png)

HTML이 아니라 데이터를 전달한다.

주로 JSON 형식을 사용한다. 

> 웹 브라우저는 JSON을 그대로 보여주는데?

- HTTP API는 다양한 시스템에서 호출한다.
- 데이터만 주고 받는 것이고 UI 화면이 필요하면 클라이언트가 별도 처리.
- 앱, 웹 클라이언트, 서버 to 서버
	- 웹 클라이언트로 전달하면 자바스크립트의 AJAX 등이 JSON 데이터를 이용해 HTML을 동적으로 만들어 뿌린다거나 할 수 있다.

![](https://blog.kakaocdn.net/dn/CG6So/btstr4eOcCf/UUJ36PiGKrqm6BUuvtsOf0/img.png)

### 정리

- 주로 JSON 형태로 데이터 통신
- UI 클라이언트 접점
	- 앱 클라이언트(아이폰, 안드로이드, PC 앱)
	- 웹 브라우저에서 자바스크립트를 통한 HTTP API 호출
	- React, Vue.js 같은 웹 클라이언트
- 서버 to 서버
	- 주문 서버 -> 결제 서버
	- 기업 간 데이터 통신

백엔드 개발자는 정적 데이터를 어떻게 제공할 것인지, 동적으로 제공되는 HTML 페이지 어떻게 제공할 것인지, HTTP API를 어떻게 제공할 것인지 3가지를 고민하면 된다.

## SSR - 서버 사이드 렌더링

![](https://blog.kakaocdn.net/dn/lXWrU/btstg3vwBhE/d0qk1kwK4dpZg0oqYqLpVK/img.png)

서버는 주문 DB를 조회해 HTML을 JSP나 Thymeleaf 등을 통해 동적으로 생성한다. 

최종적으로 서버에서 최종 HTML 만들어 클라이언트에 전달하는 것이다. 이를 서버 사이드 렌더링이라고 한다.

- HTML 최종 결과를 서버에서 만들어 웹 브라우저에 전달한다.
- 주로 정적인 화면에 사용된다.
- 관련기술 : JSP, 타임리프

## CSR - 클라이언트 사이드 렌더링

![](https://blog.kakaocdn.net/dn/bsnY48/btstqvcVSCw/hwvPuD0TlVBg5GBqAbED0K/img.png)

서버에 요청을 한다. 하지만 이 때 HTML안에는 내용이 없다. 대신 자바스크립트 링크를 준다.

이 후 자바스크립트 요청을 하게 되고 서버가 응답을 한다.

HTTP API를 통해 필요한 데이터를 요청하고 응답받아 그 데이터를 가지고 자바스크립트를 통해 HTML 결과를 렌더링 하는 것이다.

- HTML 결과를 자바스크립트를 사용해 웹 브라우저에서 동적으로 생성해 적용
- 주로 동적인 화면에 사용, 웹 환경을 마치 앱 처럼 필요한 부분부분 변경할 수 있음
- ex) 구글 지도(확대 등 작업을 해도 변함없음), Gmail, 구글 캘린더
- 관련 기술 : React, Vue.js