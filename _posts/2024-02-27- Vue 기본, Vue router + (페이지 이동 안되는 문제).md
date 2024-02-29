---
title: Vue 기본 구조 (+ Vue router 페이지 이동이 안됐던 문제), Axios
author: leedohyun
date: 2024-02-27 20:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

## Vue 기본 구조

- public
	- index.html
	- 기타 정적 자원(css, img)

```html
//index

<!DOCTYPE html>  
<html lang="">  
<head>  
  <meta charset="utf-8">  
 <meta http-equiv="X-UA-Compatible" content="IE=edge">  
 <meta name="viewport" content="width=device-width,initial-scale=1.0">  
 <link rel="icon" href="<%= BASE_URL %>favicon.ico">  
 <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bootstrap/5.3.3/css/bootstrap.min.css"  
  integrity="sha512-jnSuA4Ss2PkkikSOLtYs8BlYIeeIK1h99ty4YfvRPAlzr377vr3CXDb7sb7eEEBYjDtcYj+AjBH3FLv5uSJuXg=="  
  crossorigin="anonymous" referrerpolicy="no-referrer"/>  
 <title><%= htmlWebpackPlugin.options.title %></title>  
</head>  
<body>  
<noscript>  
  <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled.  
        Please enable it to continue.</strong>  
</noscript>  
<div id="app"></div>  
<script src="https://cdnjs.cloudflare.com/ajax/libs/bootstrap/5.3.3/js/bootstrap.min.js"  
  integrity="sha512-ykZ1QQr0Jy/4ZkvKuqWn4iF3lqPZyij9iRv6sGqLRdTPkY69YX6+7wvVGmsdBbiIfN/8OdsI7HABjvEok6ZopQ=="  
  crossorigin="anonymous" referrerpolicy="no-referrer"></script>  
<!-- built files will be auto injected -->  
</body>  
</html>
```

내 프로젝트에서는 부트스트랩을 사용하기 위한 script 코드들을 추가했었다.

애플리케이션의 진입점으로 Vue 앱이 마운트되는 HTML 파일이다.

- src
	- assets: css, img 등 기타 정적 파일
	- components: Vue 컴포넌트들을 담는다. 각 컴포넌트는 Vue 파일로 작성된다.
		- ex) Card, Header, Footer
	-  views: 라우팅에 따라 렌더링되는 페이지를 나타내는 Vue 파일
	- router: Vue Router 관련 파일을 포함한다.
	- store: VueX 스토어 관련 파일들을 포함한다. 애플리케이션의 상태 관리를 위한 파일들이 포함된다.
	- App.vue: 애플리케이션의 최상위 컴포넌트로, 모든 페이지에 공통적으로 렌더링되는 요소들이 들어간다.
	- main.js: Vue 앱의 진입점으로 Vue 인스턴스를 생성하고 라우터, 스토어 등을 연결한다.

```js
//main.js

import { createApp } from 'vue'  
import App from './App.vue'  
import router from './router'; // 라우터 설정 import  

createApp(App).use(router).mount('#app')
```

라우터를 사용하기 위한 코드를 작성했다.

- package.json: 의존성 및 스크립트 등을 관리한다.
- vue.config.js: Vue CLI 설정파일. 빌드 프로세스와 번들링 설정 등이 포함된다.



## Vue Router

페이지 기반 네비게이션을 관리하는 라이브러리이다.

```
npm install vue-router
```

명령어로 설치할 수 있다.

다양한 URL 경로에 대해 각각의 컴포넌트를 연결하고, 이를 통해 단일 페이지 애플리케이션(SPA)을 개발할 수 있다. 페이지 이동할 때 url이 변경되면, 변경된 요소의 영역에 컴포넌트를 갱신한다.

- 페이지를 새로고침하지 않고도 URL에 따라 동적으로 컴포넌트를 렌더링할 수 있다.
- URL 경로와 컴포넌트 간의 매핑을 간단하게 정의할 수 있습니다. 이를 통해 애플리케이션의 각 페이지를 명확하게 구성할 수 있다.
- 중첩된 URL 경로에 대해 중첩된 컴포넌트를 렌더링할 수 있습니다. 이를 통해 복잡한 페이지 구조를 구현할 수 있다.
- HTML5 History API를 사용하여 브라우저의 주소 표시줄을 업데이트하고, 애플리케이션의 히스토리를 관리한다. 
	- 이를 통해 사용자가 뒤로 가기와 앞으로 가기 버튼을 사용하여 애플리케이션의 이전 상태로 되돌아갈 수 있다.

이러한 특성을 통해 사용자 경험의 향상을 가져올 수 있다.

### 내가 맞닥뜨린 문제.

```html
<button type="button" class="btn btn-outline-light me-2" @click="$router.push('/login')">Login</button>
```

이렇게 push 메서드를 통해 Login 페이지로 넘어가려고 했었다.

그런데 URL만 바뀔 뿐 페이지가 렌더링되지 않는 문제가 있었다.

라우터의 사용 설정, 경로는 모두 문제가 없었다.

```js
import { createRouter, createWebHistory } from 'vue-router';  
import LoginPage from '@/pages/LoginPage.vue';
import RegisterPage from "@/pages/RegisterPage.vue";
  
const router = createRouter({  
  history: createWebHistory(),  
    routes: [  
        {  
			path: '/login',  
            name: 'LoginPage',  
            component: LoginPage,  
        },  
        {  
			path: '/register',  
            name: 'RegisterPage',  
            component: RegisterPage,  
        },  
  ],  
});  
  
export default router;
```
```js
//main.js
import { createApp } from 'vue'  
import App from './App.vue'  
import router from './router'; // 라우터 설정 import  
createApp(App).use(router).mount('#app')
```

오랜 시간 문제를 해결하려고 노력했는데, 결과적으로는 허무했다.

내가 Vue에 대해 정확히 파악하지 못해 일어난 일이었다.

나는 App.vue 파일이 단순하게 기본 경로 즉 홈의 역할을 한다고 생각해 Home.vue라는 이름으로 바꾸고 App.vue 파일은 없는채로 사용했다.

그러나 위의 설명에도 있고 설정에서도 ./App.vue 경로가 있듯 App.vue는 중요한 역할을 한다. 물론 꼭 App.vue일 필요는 없을 것 같다. main.js의 설정을 바꾸면 되지 않을까 싶긴하다.

여튼 App.vue 파일을 봐보자.

```html
<template>  
 <MobileHeader v-if="isMobile"/>  
 <PcHeader v-if="!isMobile"/>  
 <RouterView/>
 <MobileFooter v-if="isMobile"/>  
</template>

//...
```

핵심은 RouterView이다. Vue Router에서 제공하는 컴포넌트로 라우팅된 컴포넌트를 표시해준다.

즉 위의 Header, Footer는 기본적으로 유지되도록 할 수 있는 것이고, RouterView를 통해 전체 페이지를 다시 로드하지 않고 동적으로 컴포넌트를 업데이트한다.

이러한 실수를 하는 사람은 드물 것 같지만 혹시나 해서 정리한다..


## Vue Cors 설정

CORS는 오류이다.

프론트에서 백에 데이터를 요청할 때 프론트와 백의 origin(url)이 달라 발생하는 오류이다.

서버와 API 통신을 하기위해 설정이 필요하다.

```js
const { defineConfig } = require('@vue/cli-service')  
module.exports = defineConfig({  
  devServer: {  
  proxy: {  
	  '/api': {  
			  target: 'http://localhost:8080'  
		  }  
		}  
	 }  
  }  
)
```

vue.config.js 파일에 위와 같이 설정했다.

프록시를 설정해주는 것이다.

### 프록시 설정

CORS는 요청하는 주소와 서버 주소가 다르기 때문에 발생한다고 했다.

따라서 주소가 동일해야 정상적으로 응답받을 수 있다는 것이다.

이러한 문제를 해결하기 위해 proxy 설정을 해 api 주소로 가는 모든 요청을 웹 페이지 주소로 가도록 바꿔서 CORS 문제를 해결하는 것이다.

- 로그인 API : localhost:8080/api/login
- 웹 주소 : localhost:3000

웹 주소에서 API 주소로 로그인 처리를 하는 API를 요청한다고 가정하자.

- proxy 설정 전 : 웹 페이지 -> 서버
- proxy 설정 이후 : 웹 페이지 -> proxy -> 서버

즉 localhost:3000/login 이라는 URL로 요청을 했을 때 이 요청이 localhost:8080/api/login 이라는 요청 URL로 바뀌는 것이다.

위 설정 파일의 코드는 '/api' 로 시작하는 모든 요청을 proxy 서버를 통해 바꿔주는 것이다.

### Axios

웹 브라우저와 Node.js를 위한 Promise 기반의 HTTP 클라이언트이다.

이를 사용해 비동기 방식으로 HTTP 요청을 보낸다.

- HTTP 요청과 응답을 자동으로 변환하고 JSON 데이터를 자동으로 파싱한다.
- HTTP 요청을 취소할 수 있는 기능을 제공한다. 요청이 오래 걸리는 경우 유용하다.
- 요청과 응답을 인터셉트하여 처리할 수 있는 기능을 제공한다.
	- 요청을 보내기 전 인증 토큰 추가 작업 등

 ```
 npm install axios
 ```

명령어를 통해 설치하여 사용할 수 있다.

하나의 사용 예시를 보자.

```html
<input type="email" class="form-control" id="floatingInput" placeholder="name@example.com"
             @keyup.enter="submit()" v-model="state.form.email">

<input type="password" class="form-control" id="floatingPassword" placeholder="Password" 
			@keyup.enter="submit()" v-model="state.form.password">
```

- v-model을 사용해 입력 필드의 값을 state.form. 에 바인딩 했다.
- @keyup.enter: Enter 키를 누를 때 submit() 메서드를 호출한다.

```vue
export default {
  setup() {
    const state = reactive({
      form: {
        email: "",
        password: ""
      }
    })

    const submit = () => {
      axios.post("/api/account/login", state.form).then((res) => {
        window.alert("로그인하였습니다.");
      }).catch(() => {
        window.alert("로그인 정보가 존재하지 않습니다..");
      });
    }

    return {state, submit}
  }
}
```

- reactive 함수를 사용해 반응 상태 객체 state를 생성해 컴포넌트의 데이터를 보관할 수 있도록 한다.
- axios를 사용해 HTTP POST 요청을 보낸다.
	- proxy를 통해 (스프링) 서버의 API 컨트롤러를 통한 요청이 이루어 질 것이다.
- submit() 메서드는 폼을 제출할 때 수행된다.
	- 백엔드 API에 로그인 요청을 보내고 응답을 처리한다.
	- 로그인이 실패하면 경고 메시지를 표시한다. 


> 비동기 방식

작업이 순차적으로 진행되지 않고, 요청을 보낸 후 결과를 기다리지 않고 다른 작업을 계속할 수 있는 방식을 말한다. 

이는 특히 네트워크 요청이나 파일 읽기/쓰기 등의 I/O 작업을 처리할 때 유용하다.

- Non blocking: 요청을 보낸 후 해당 요청이 완료될 때까지 대기하지 않고 다른 작업을 수행할 수 있다.
- 효율성: 비동기 방식은 여러 작업을 동시에 처리할 수 있기 때문에 시스템 자원을 효율적으로 활용할 수 있다.
- 응답 시간 단축 : 비동기 방식을 사용하면 작업을 병렬로 처리하여 응답 시간을 단축할 수 있다.
- 이벤트 : 비동기 작업은 이벤트 기반으로 동작할 수 있습니다. 예를 들어, 네트워크 요청의 완료를 알리는 이벤트를 받아서 처리할 수 있다.