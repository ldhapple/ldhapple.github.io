---
title: GithubActions + Docker을 이용한 배포 (Spring + Vue + NginX)
author: leedohyun
date: 2024-03-26 21:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

## 개요

GithubActions와 Docker를 활용해 Spring + Vue 서버를 EC2 서버에 배포하는 방법을 알아보자.

대략적인 흐름은 아래 이미지와 같다.

![](https://ldhapple.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F76e3676b-d188-4ef3-98f1-b4ce8745f301%2Ffcdc4069-ceb9-4d57-89f1-9075aad19a53%2FUntitled.png?table=block&id=2527169b-be76-4ec9-bee7-21d2f9e18fff&spaceId=76e3676b-d188-4ef3-98f1-b4ce8745f301&width=2000&userId=&cache=v2)

## Docker 설치

```
sudo apt-get update

// 필요 패키지 설치
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

// Docker 공식 GPG키 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

// Docker 공식 apt 저장소 추가
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update

// Docker 설치
sudo apt-get install docker-ce docker-ce-cli containerd.io

// Docker 설치 확인
sudo systemctl status docker
```

위 명령어를 통해 EC2 환경에 우선 Docker를 설치한다.

만약 Docker 사용 중 권한 거부 문제가 나타난다면 이 [링크](https://kimjingo.tistory.com/235)를 참고하자.

## DockerFile 작성

배포를 위한 Dockerfile을 작성해보자. 참고로 아직 Docker에 대해 미숙하기 때문에 필요없는 명령어가 포함되어 있을 수 있어 양해를 바랍니다.

### Spring Docker 파일 작성

```dockerfile
FROM openjdk:17-jdk as build  
  
ARG JAR_FILE=build/libs/*.jar  
COPY ${JAR_FILE} app.jar  
  
ENTRYPOINT ["java","-jar","/app.jar"]
```

- openjdk:17-jdk
	- 위 jdk 버전의 이미지를 기반으로 새로운 이미지를 빌드한다.
	- 본인의 개발환경에 맞는 JDK를 포함시키면 된다.
- ARG, COPY
	- build/libs/*.jar에 해당하는 JAR 파일을 app.jar로 복사한다.
	- JAR_FILE은 빌드할 때 전달된 인수로 부터 가져온 JAR 파일을 지정하도록 해준다. 
- ENTRYPOINT
	- 최종 이미지의 ENTRYPOINT를 설정한다.
	- 컨테이너가 시작될 때 실행할 명령을 정의하는 것이다.
	- jar 파일을 실행하기 때문에 Java 애플리케이션을 실행하는 데 사용된다. 

### Vue Docker 파일 작성

```dockerfile
FROM node:lts-alpine as build-stage  
WORKDIR /app  
COPY package*.json ./  
RUN npm install  
COPY . .  
RUN npm run build  
  
FROM nginx:stable-alpine as production-stage  
COPY --from=build-stage /app/dist /usr/share/nginx/html  
EXPOSE 80  
CMD ["nginx", "-g", "daemon off;"]
```

줄 공백을 기준으로 위의 Build Stage와 아래의 Production Stage로 나뉜다.

#### Build Stage

- FROM
	- node:lts-alpine 이미지를 기반으로 빌드 단계용 이미지를 생성한다.
	- 참고로 위에서 사용한 이미지는 Node.js의 LTS 버전을 포함한 Alpine Linux를 사용한다.
- WORKDIR
	- 컨테이너 내부의 작업 디렉토리를 /app 으로 설정한다.
- COPY
	- 현재 디렉토리의 모든 package.json 파일을 컨테이너 내부의 작업 디렉토리로 복사한다.
	- 이는 Docker의 레이어 캐싱 메커니즘을 활용하기 위해 npm install 실행 전에 수행된다.
	- 이를 통해 Docker가 캐시된 레이어를 이용하여 후속 빌드 시 시간을 절약할 수 있도록 해준다.
- RUN npm install
	- package.json에 나열된 종속성을 설치한다.
- COPY . .
	- 호스트 시스템의 모든 파일을 컨테이너 내부의 작업 디렉토리로 복사한다.
	- 여기에 애플리케이션 소스 코드가 포함된다.
- RUN npm run build
	- package.json에 정의된 빌드 스크립트를 실행한다.
	- 애플리케이션을 빌드한다고 보면 된다.   

#### Production Stage

- FROM
	- Alpine Linux에서 실행되는 Nginx. 이를 Production Stage의 기본 이미지를 정의한다.
- COPY --from=build-stage /app/dist /usr/share/nginx/html
	- 빌드 단계에서 빌드된 애플리케이션 파일을 Nginx HTML로 복사한다.
	- Production Stage 컨테이너 내부 디렉토리이다. (/usr/share/nginx/html) 
	- 이렇게 함으로써 Nginx가 빌드 단계에서 생성된 정적 파일을 제공할 수 있다.
- EXPOSE 80
	- 외부 트래픽이 내부에서 실행중인 Nginx 서버에 도달할 수 있도록 컨테이너의 포트 80을 노출시킨다.
- CMD
	- 컨테이너가 실행될 때 실행할 명령을 지정한다.
	- Nginx 서버를 시작하고 foreground (daemon off) 에서 계속 실행하여 Docker가 컨테이너의 수명 주기를 관리할 수 있도록 한다.


> 참고

![](https://blog.kakaocdn.net/dn/bgQ4ST/btsGhm2FKrl/j7wJBCkq7KKs4Wl74wARZk/img.png)

각 도커파일은 위와 같이 해당 서버의 루트 디렉토리에 두면 된다.

위에 사용한 Dockerfile 내용에서 커스텀하여 추가하고 싶다면, 관련 내용을 검색해보자.

## Nginx 설치 및 설정파일

```null
sudo apt update
sudo apt install nginx
```

위와 같이 Nginx를 EC2에 설치하고, Nginx를 사용하기 위한 설정파일을 설정하자.

참고로 Nginx도 직접 설치하지 않고 Docker를 활용해 설치할 수 있다.

### 설정 파일

```
cd /etc/nginx/sites-available
```

위 명령어로 해당 디렉토리로 이동한다.

```
sudo nano (app 이름)
```

Nginx 구성 파일의 이름을 지정하고, 파일을 만든다. nano 명령어를 이용해 텍스트를 편집할 수 있다.

구성 파일의 이름은 일반적으로 도메인이나 애플리케이션의 이름을 딴다.

아래는 구성 파일 내용이다.

```
server {
	listen 80;

	server_name recommtoon.com; # DNS 설정 이전에는 EC2 IP 주소 입력

	location / { 
		proxy_pass http://localhost:8081;
		proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; 
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
		proxy_set_header X-Forwarded-Proto $scheme; 
	} 

	location /api/ { 
		proxy_pass http://localhost:8080; 
		proxy_set_header Host $host; 
		proxy_set_header X-Real-IP $remote_addr; 
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
		proxy_set_header X-Forwarded-Proto $scheme; 
	} 
}
```

- server
	- Nginx 서버 블록을 정의한다.
- listen 80
	- 포트 80에서 들어오는 HTTP 요청을 수신한다.
	- Docker가 80포트를 사용하기 때문에 80포트를 요청을 수신한다.
- location
	- 각 경로 ('/', '/api')에 대한 설정을 정의한다.
- proxy_pass
	- HTTP 요청을 받으면 http://localhost:port 로 해당 요청을 프록시한다.
- proxy_set_header
	- 프록시 요청 헤더를 설정한다. 각 서버에 전달되는 추가 정보를 포함한다. 
- Host $host
	- 프록시 된 요청의 Host 헤더를 $host 변수 값으로 설정한다.
	- 클라이언트가 요청한 서버의 이름을 나타내어 클라이언트가 요청한 원래 호스트 이름을 수신하게 된다.
	- 가상 호스팅 또는 라우팅 목적에서 유용할 수 있다.
- X-Real-IP
	- X-Real-IP 헤더를 클라이언트의 IP 주소로 설정한다.
	- Nginx가 역방향 프록시 또는 로드 밸런서 뒤에 배포되는 경우 클라이언트의 실제 IP 주소를 서버에 전달하는데 사용된다.
- X-Forwarded-For $proxy_add_x_forwarded_for
	- 클라이언트의 IP 주소를 X-Forwarded-For 헤더에 추가한다.
	- 요청이 통과한 프록시 서버 체인을 추적하는데 사용된다.
	- 서버가 전체 클라이언트-서버 간의 통신 경로를 볼 수 있다.
- X-Forwarded-Proto $scheme
	- X-Forwarded-Proto 헤더를 클라이언트가 사용하는 프로토콜 (HTTP / HTTPS)로 설정한다.
	- 원래 요청이 HTTP / HTTPS 에서 이루어졌는지 확인하기 위해 사용된다.  

```
sudo ln -s /etc/nginx/sites-available/(app 이름) /etc/nginx/sites-enabled/
```

파일을 작성한 후 위 명령어로 심볼릭 링크를 생성해 활성화 한다.

위 명령은 sites-available의 새 구성파일을 가리키는 심볼릭 링크를 sites-enabled 디렉토리에 생성한다.

심볼릭 링크의 이름은 파일 이름과 반드시 일치할 필요는 없지만 동일하게 유지하는 것을 추천한다.

> 심볼릭 링크란?

소프트 링크라고도 한다.

파일 시스템의 다른 파일이나 디렉토리에 대한 참조 역할을 하는 특수한 유형의 파일이다.

보통 자주 액세스하는 파일이나 디렉토리에 대해 바로 가기 제공, 임시 별칭 생성, 파일 시스템 구성 촉진 등 다양한 목적으로 사용된다.

복잡한 Nginx 설정에서는 더 나은 구성과 관리를 위해 구성을 여러 파일로 분할하는 것이 일반적이다. 

실제 구성 파일을 다른 위치에 유지하면서 심볼릭 링크를 사용하여 Nginx 구성 파일에 대한 중앙 집중식 디렉터리를 만들 수 있다. 

이를 통해 구성 파일을 더 쉽게 관리하고 업데이트할 수 있는 것이다.


## GithubActions

이제 Dockerfile을 작성했으니 GithubActions를 활용하여 쉽게 배포를 해보도록 한다.

GithubActions는 개발자들이 소프트웨어 개발 워크플로우를 자동화할 수 있게 해주며, 코드 통합부터 배포까지의 전 과정을 GitHub 내에서 관리할 수 있도록 해준다.

워크플로우라는 파일을 yaml 형식으로 작성하게 되면, 이 파일을 통해 push, pull request에 따라 실행되는 작업들을 배정할 수 있다.

복잡한 설정 필요없이 간단하게 워크 플로우를 구성할 수 있는 것이다.

> GithubActions vs Jenkins

Jenkins는 더 최근에 나온 GithubActions보다 참고할 자료가 많다. 하지만 CI/CD를 구성하는게 비교적 어려운 편이다.

사실 어떤 도구를 사용해도 목표는 같기 때문에 자신이 쓰고 싶은 도구를 사용하면 되는 것이다.

그래서 나는 Github에서 소스 코드의 관리와 CI/CD 파이프라인을 모두 처리할 수 있는 GithubActions를 사용하기로 했다.

더해 자료는 적지만 난이도가 비교적 쉬워보였던 점이 선택의 이유이다.

### Workflows 작성

리포지토리의 Actions 탭에서 Workflow file을 작성할 수 있다.

new workflows를 클릭하면 아래와 같이 리포지토리에 추천하는 workflow 템플릿을 추천해준다.

![](https://blog.kakaocdn.net/dn/yFcef/btsGi1QIe29/9kiSwvDniidS3xylkhi0P0/img.png)



#### Spring Repository Workflow 작성

나는 위의 선택지에서 Java with Gradle을 선택해 수정하는 방식으로 진행하였다.

아래는 최종 workflow 파일이다.


```yaml
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.
# This workflow will build a Java project with Gradle and cache/restore any dependencies to improve the workflow execution time
# For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-java-with-gradle

name: Java CI with Gradle  #workflow의 이름

on: # workflow가 언제 실행될 지
  push: # master 브랜치에 push, pull_request 시 실행한다.
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]

jobs: # 작업 (build, dependency-submission)
  build: # Java 프로젝트를 빌드하고, Docker 이미지를 빌드하여 배포
    runs-on: ubuntu-latest # 빌드 작업이 실행될 환경을 지정한다.
    permissions:
      contents: read # 작업이 리포지토리 내용을 읽을 수 있도록 권한을 설정한다.

    steps: # 작업에 포함된 step을 정의한다.
    - uses: actions/checkout@v4 # 소스코드 체크아웃
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: make application.properties # application.properties 파일 설정
      run: |
        cd ./src/main/resources
        touch ./application.properties
        echo "${{ secrets.APPLICATION_PROD }}" > ./application.properties
        
    - name: Grant execute permission for gradlew # Gradle을 사용한 빌드
      run: chmod +x gradlew    

    # Configure Gradle for optimal use in GiHub Actions, including caching of downloaded dependencies.
    # See: https://github.com/gradle/actions/blob/main/setup-gradle/README.md
    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@417ae3ccd767c252f5661f1ace9f835f9654f2b5 # v3.1.0

    - name: Build with Gradle Wrapper
      run: ./gradlew build

    - name: Docker build # Docker 빌드
      run: |
        docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
        docker build -t recommtoon/recommtoon .
        docker push recommtoon/recommtoon

    - name: Deploy # 도커 이미지 EC2 인스턴스에 배포
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }} # EC2 인스턴스 퍼블릭 DNS
        username: ubuntu
        key: ${{ secrets.PRIVATE_KEY }} # pem 키
        port: 22
        # 도커 작업
        script: |
          docker pull recommtoon/recommtoon
          docker stop $(docker ps -a -q)
          docker run -d --log-driver=syslog -p 8080:8080 recommtoon/recommtoon
          docker rm $(docker ps --filter 'status=exited' -a -q)
          docker image prune -a -f

    # NOTE: The Gradle Wrapper is the default and recommended way to run Gradle (https://docs.gradle.org/current/userguide/gradle_wrapper.html).
    # If your project does not have the Gradle Wrapper configured, you can use the following configuration to run Gradle with a specified version.
    #
    # - name: Setup Gradle
    #   uses: gradle/actions/setup-gradle@417ae3ccd767c252f5661f1ace9f835f9654f2b5 # v3.1.0
    #   with:
    #     gradle-version: '8.5'
    #
    # - name: Build with Gradle 8.5
    #   run: gradle build

  dependency-submission:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin' 

    # Generates and submits a dependency graph, enabling Dependabot Alerts for all project dependencies.
    # See: https://github.com/gradle/actions/blob/main/dependency-submission/README.md
    - name: Generate and submit dependency graph
      uses: gradle/actions/dependency-submission@417ae3ccd767c252f5661f1ace9f835f9654f2b5 # v3.1.0
```

위에서 중괄호에 쌓여있는 부분은 Secret 키를 통해 값을 숨길 수 있는 기능을 활용한 것이다. Secret 키에 대해서는 아래에서 설명한다.

주석으로 설명했지만 위 파일에 대해 요약하자면 Gradle을 사용해 Java 프로젝트를 빌드하고 Docker 이미지를 빌드한 후 EC2 인스턴스에 배포하는 과정이다.

- docker build -t recommtoon/recommtoon .
	- 도커파일을 기반으로 이미지를 빌드하고 이미지의 이름과 태그를 지정한다.
-  docker pull recommtoon/recommtoon
	- DockerHub에서 해당 이미지를 가져온다.
	- 로컬 시스템에 해당 이미지가 캐시되어 있지 않은 경우에만 실행된다.
- docker stop $(docker ps -a -q) 
	- 실행중인 모든 컨테이너를 중지한다.
	- 괄호 내부의 명령은 현재 시스템에서 실행중인 모든 컨테이너의 ID를 가져오는 것이다.
- docker run -d --log-driver=syslog -p 8080:8080 recommtoon/recommtoon
	- 이미지를 기반으로 새로운 컨테이너를 실행한다.
	- -d : 옵션을 통해 백그라운드에서 컨테이너를 실행하도록 한다.
	- -log : 컨테이너의 로그를 syslog로 전송한다.
	- -p : 호스트의 포트 8080과 컨테이너의 포트 8080을 매핑한다.  
- docker rm, docker image prune
	- 종료된 모든 컨테이너를 제거한다.
	- 사용하지 않는 모든 이미지를 제거한다.

#### Vue Repository Workflow 작성

Publish Node.js Pacakge를 사용했다.

```yaml
name: Deploy Vue App to EC2 via Docker

on: # master 브랜치에 push할 경우 동작
  push:
    branches:
      - master

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Build and Push Docker image # 도커 이미지 빌드
      run: |
        docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
        docker build -t recommtoon/recommtoonfe .
        docker push recommtoon/recommtoonfe
        
      - name: Deploy to EC2 # EC2 배포
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ${{ secrets.EC2_USERNAME }}
        key: ${{ secrets.EC2_SSH_KEY }}
        script: |
          docker pull recommtoon/recommtoonfe:latest
          docker stop recommtoonfe || true
          docker rm recommtoonfe || true
          docker run -d --name recommtoonfe -p 8081:80 recommtoon/recommtoonfe:latest
```

대부분 위에서 설명한 내용과 비슷하다.

- docker run -d --name recommtoonfe -p 8081:80 ..
	- 호스트의 포트 8081을 컨테이너의 포트 80으로 매핑한다.
	- Nginx 프록시에 의해 Vue 애플리케이션이 8081 포트에서 실행되는데, Docker 컨테이너 내부에서는 기본적으로 80 포트를 사용한다.
	- 따라서 Docker 컨테이너에서 Vue 애플리케이션을 실행하기 위해 매핑을 한 것이다.
- || true
	- 컨테이너가 없어 실패했을 때 에러를 무시하도록 해준다. 

### Secret 키 관리

리포지토리의 Settings에서 좌측의 Secrets and variables 탭의 Actions에서 시크릿 키를 관리할 수 있다.

- Docker_USERNAME, PASSWORD
	- 도커의 id와 password이다.
- EC2_HOST, HOST
	- EC2 서버의 IP 주소를 입력한다.
- EC2_SSH_KEY, PRIVATE_KEY
	- EC2의 시크릿 키 내용을 입력하면 된다.
	- 나는 pem 파일을 받았기 때문에 해당 파일을 열어 값을 입력해주었다.
- EC2_USERNAME
	- EC2의 username을 입력하면 된다.   
- APPLICATION_PROD
	- application.properties의 내용을 입력한다.  

## Swap 메모리 설정

Spring 컨테이너와 Vue 컨테이너를 동시에 가동하는 과정에서 EC2 서버가 멈추는 현상이 발생한다.

멈추는 것 자체는 인스턴스 중지 후 재시작을 해주면 다시 정상 작동되지만, GithubActions workflow를 다시 실행해줘야 하는 번거로움이 생긴다.

이를 Swap 메모리로 해결할 수 있다.

- swap  메모리 추가

```bash
$ sudo dd if=/dev/zero of=/swapfile bs=128M count=16
$ sudo chmod 600 /swapfile
```

EC2는 기본적으로 램 1GB를 가지고 있다. 쉘에 위 명령어를 사용해 2GB의 스왑파일을 생성해준다.

- swap 메모리를 swap 파일로 포맷

```bash
$ sudo mkswap /swapfile
```

- swap 메모리 활성화

```bash
$ sudo swapon /swapfile
$ sudo swapon -s
```

위 명령어를 사용하면 Swap 메모리가 활성된다.

두번째 명령어는 활성화된 스왑 파일의 정보와 크기 등을 보여준다.

- swap 메모리 자동 활성화

```bash
$ sudo vi /etc/fstab 

# 마지막 행에 추가하기
/swapfile swap swap defaults 0 0
```

시스템이 재시작되더라도 자동 활성화 할 수 있는 기능이다.

- 메모리 체크

```bash
$ sudo free -h 
```

현재 메모리에 관한 정보를 확인할 수 있는 명령어이다. 설정한 Swap 메모리가 잘 동작하고 있는지 확인할 수 있다.

## 정리

위와 같은 작업들을 마치고, workflow 파일이 에러없이 정상적으로 실행된다면 EC2 퍼블릭 IP를 http로 접근했을 때 배포가 완료된 것을 볼 수 있다.

참고로 docker ps 명령어를 통해 EC2 인스턴스에서 실행되고 있는 Docker 컨테이너들을 확인할 수 있다.

해당 IP를 DNS에 연결하는 과정과 HTTPS 설정하는 방법은 다음 포스트에서 다룬다.
