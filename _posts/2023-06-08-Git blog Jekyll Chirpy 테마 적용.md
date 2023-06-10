---
title: Git blog Jekyll Chirpy 테마 적용
author: leedohyun
date: 2023-06-07 23:13:00 -0500
categories: [Git, Blog]
tags: [git, blog]
---

## 테마 적용

https://jekyllthemes.org/ 사이트에서 chirpy 테마를 다운로드한다.

> zip 설치

본인이 github 블로그로 만들기 위해 저장한 로컬 github repo 폴더에 압축을 푼다.

같은 파일이 존재하면 모두 덮어쓴다.

> 스킨 설치

이제 받은 파일들을 설치한다.

그 전에 Chirpy를 초기화 시켜 빌드가 가능한 상태로 만들어야 한다.

지금 받은 zip파일은 개발자가 본인에게 맞게 수정한 상태이기 때문에 github에서 빌드 자체가 되지 않는다.

```
bash tools/init.sh
```

명령어를 사용하라고 readme파일에 적혀있지만 해당 명령어는 linux 전용이므로 윈도우에서는 직접 손으로 해야한다.

1. Gemfile.lock 파일 삭제
2. .travis.tml 파일 삭제
3. _posts 디렉토리 삭제
4. docs 디렉토리 삭제
5. .github/workflows 폴더에서 pages-deploy.yml.hook 파일을 제외한 나머지 파일 삭제
6. pages-deploy.yml.hook 파일명을 pages-deploy.yml 로 변경

존재하지 않는 파일이 있다면 건너 뛰어도 무방하다.

이제 cmd에 관리자모드로 접근하여 클론했던 repo 폴더로 이동 후

```
bundle install
```

명령어를 입력하면 테마 적용이 완료된다.

> 블로그 설정하기

현재 블로그 스킨은 개발자의 이름으로 되어 있다. 

자기에게 맞게 수정하는 작업을 해야한다.

로컬의 repo 폴더에 들어가서 _config.yml 파일을 연다.

- 기본 설정

```
timezone: Asia/Seoul
lang: ko
title: 제목
tagline: 제목 아래 설명 (서브타이틀)
url: 'https://username.github.io'
github: username: github_username
avatar: assets/img/profile.jpg (이미지 주소를 넣는 것을 추천)
```

```
bundle exec jekyll serve
```
명령어를 통해 로컬에서 블로그를 테스트해볼 수 있습니다.

_post 디렉토리 밑에 게시글 파일을 넣고 테스트했는데 태그 및 아카이브 등등은 적용이 됐지만 main화면에 글이 보이지 않을 경우

1. _layout/home.html 파일을 연다.
2. 아래쪽 괄호안에 for post in posts 코드가 있는데 for post in site.posts로 변경한다.

설정을 다 한 후 push하면 됩니다.

> git add . 오류

git add 명령어를 입력했는데

```
LF will be replaced by CRLF the next ~
```

오류가 발생하는 경우가 있다.

이는 git이 파일을 업데이트할 때, LF(Line Feed) 줄바꿈 문자가 
CRLF(Carriage Return Line Feed) 줄바꿈 문자로 대체될 것임을 나타내는 경고이다.

주로 Windows 운영체제에서 나타나는 경고이다.

이는 git이 CRLF대신 LF를 사용하도록 강제로 설정을 변경해주면 해결된다.

```
git config --global core.autocrlf true
```

명령어를 통해 해결할 수 있다.


> build 과정에서의 오류

push를 했더니 build과정에서

The process '/opt/hostedtoolcache/Ruby/3.2.2/x64/bin/bundle' failed with exit code 16

오류가 발생해서 실패했다.

github Actions에서 확인해보니

![image](https://blog.kakaocdn.net/dn/dwtK3D/btsjiQF8duZ/hXFIcMuAEnC8tjpYWQK5IK/img.png)

```
bundle lock --add-platform x86_64-linux
```

명령어를 입력하고 다시 시도하라고 떠서 입력하고 다시 시도해보았다.

해당 오류는 해결됐다.

![image](https://blog.kakaocdn.net/dn/byE7Gh/btsjnabV2kw/KdYH1y9ckARFiq18p9tqtK/img.png)

이러한 오류도 발생해서 찾아보았다.

assets/dist 디렉토리 내에 js 파일이 존재하지 않는다는 뜻인데

로컬 repo에 찾아보니 _javascript 디렉토리 내에 저 파일들이 존재했다.

404.html과 index.html 파일을 열어보면

```
<script defer src = "/assets/js/dist/misc.min.js"></script>
```

이런 코드들이 있다.

```
<script defer src = "/_javascript/misc.min.js"></script>
```


1. 이렇게 js파일을 참조하는 코드를 올바른 경로로 수정한다.
2. jekyll은 js파일을 처리하는 디렉토리가 정해져있고 _javascript 디렉토리는 해당하는 디렉토리가 아니기 때문에 _config.yml 파일에 코드를 수정해준다.

```
include:
 - _javascript
```

코드를 추가해준다.



## 마무리

build는 약 5분정도 소요되며

