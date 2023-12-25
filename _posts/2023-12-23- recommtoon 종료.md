---
title: 프리티어 기간 만료에 따른 배포 서버 종료하기
author: leedohyun
date: 2023-12-23 12:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

![](https://blog.kakaocdn.net/dn/EIeLM/btsCxbkD5gy/COI8X4fitxFLPjUmjkQuLK/img.png)

recommtoon.com 사이트를 통해 나의 웹 서비스를 처음 만들고, AWS EC2를 이용한 배포까지 모두 처음 경험한 지 1년이 지났다.

위와 같이 AWS 프리티어 기간이 곧 만료된다고 메일이 오게 되었다.

일단 스프링 학습이 거의 다 끝났고, 새로운 프로젝트를 하기까지 얼마 남지 않았다. 완성하려면 당연히 더 걸릴 것이다.

그리고 예상치 못한 추가적인 비용이 해당 사이트에서 더 발생하면 안되기 때문에 우선 나는 서버를 중단하기로 한다. 중단 방법을 정리해본다.

# AWS 프리티어의 종료

순서는 아래와 같다.

- EC2 중지 후 종료 및 삭제
- 탄력적(Elastic) IP 삭제
- 보안 그룹 삭제
- 키페어 삭제
- DB 삭제(개인 선택)
- Route 53(도메인)

## EC2 인스턴스 중지 및 종료/삭제


AWS에 접속해 로그인을 진행한 후 EC2 서비스를 접속하여 인스턴스를 확인한다.

![](https://blog.kakaocdn.net/dn/rIO6e/btsCzrs9nfk/suKZ7s3HM3VBO17UbOorK0/img.png)

중지를 해주고, 종료를 눌러주면 된다.

![](https://blog.kakaocdn.net/dn/bpuNhI/btsCEIAF7Ps/x1TJSfvnlaedRXoSx7Hrm0/img.png)

종료를 누르면 위와 같은 안내 팝업이 나타난다.

아래의 탄력적 IP와 관련된 내용도 포함되어 있다.

종료를 누르면 인스턴스가 종료가 되어도 인스턴스 목록에 몇 분 노출될 수 있다. 조금 지나면 목록에서도 사라지게 되며 목록에서 사라지지 않았더라도 종료되었기 때문에 접속이 불가능하기 때문에 상관없다.

![](https://blog.kakaocdn.net/dn/bIfpal/btsCxIvElN5/6beSS18KCko0qo7dltbbQK/img.png)

recommtoon.com에 접속해보면 503 Error가 발생하는 것을 볼 수 있다.

## 탄력적 IP 삭제

탄력적 IP를 사용하지 않고 유동 IP를 사용했다면 추가 금액은 발생하지 않는다.

하지만 프리티어에서 탄력적 IP를 무료로 1개 사용할 수 있어, 대부분 프리티어로 배포 연습 겸 사용했다면 탄력적 IP를 사용하게 된다.

좌측 메뉴에서 [네트워크 및 보안] -> [탄력적 IP] 메뉴로 들어간다.

![](https://blog.kakaocdn.net/dn/coVo8J/btsCDg5lVPo/FFbeg4Q17n01K8QKSesVd1/img.png)

그리고 탄력적 IP 주소 릴리스를 해주면 된다.

![](https://blog.kakaocdn.net/dn/TlJI0/btsCzFrjCiM/kVx0ZTDDLFcrlsSjpYeZB0/img.png)

버튼을 누르게 되면 위와 같은 안내 팝업이 나타난다. 릴리스 해주면 된다.

릴리스 해주면 탄력적 IP주소 목록에서 사라지는 것을 볼 수 있다.

## 보안 그룹 삭제

좌측 메뉴에서 [네트워크 및 보안] -> [보안 그룹] 메뉴로 들어간다.

![](https://blog.kakaocdn.net/dn/6xOyn/btsCBMQYW2Y/2KVJE9IlLnMPnNZXBMbW40/img.png)

보안 그룹 삭제를 해주면 된다.

- 보안 그룹 삭제는 RDS 등을 사용했다면 바로 삭제가 되지 않을 수 있다.
- RDS 등을 삭제 후 다시 삭제하면 된다.
- 따로 사용한게 없다면, 바로 삭제가 가능하다.

![](https://blog.kakaocdn.net/dn/bajnA2/btsCypWX1qK/ZDdUv9QpoMppIYJhEdJZ0k/img.png)

나는 삭제를 누르니 위와 같은 안내 팝업이 나타났다.

두 보안 그룹 전부 삭제가 되지 않는다. 이유의 링크를 들어가니 네트워크 인터페이스가 2개 나타났고, 네트워크 인터페이스도 삭제를 시도했다.

![](https://blog.kakaocdn.net/dn/b2zFBp/btsCzIn0Ww8/7D6pGZkMrmMjOFaFkJglrK/img.png)

이번엔 네트워크 인터페이스를 삭제할 수 없다고 나타났다.

링크를 타고 들어가니 로드 밸런서가 나타났고, 로드 밸런서 삭제를 진행했다.

로드 밸런서의 삭제 후 보안 그룹 삭제를 시도하니 기본 보안 그룹을 제외하고는 삭제가 되었다.

## 키 페어 삭제

좌측 메뉴에서 [네트워크 및 보안] -> [키 페어] 메뉴로 들어간다.

![](https://blog.kakaocdn.net/dn/bBCVV4/btsCCz45U2U/YqHTPrUeAkP8kUTUgF6tb1/img.png)

키 페어도 마찬가지로 삭제를 진행해주면 된다.

## Route 53 삭제

프리티어를 사용하면서 도메인도 구매하여 이용하였다.

메일을 보면 따로 연장하지 않을 시, 자동으로 삭제가 된다고 한다.

Route 53 메뉴로 들어가 직접 삭제해주어도 된다.

![](https://blog.kakaocdn.net/dn/cDEy0g/btsCzqgGR62/viErkZymcIVE6Mftmi4nGk/img.png)

recommtoon.com 링크로 들어가면 아래와 같이 도메인에 대한 정보들이 나타나고 삭제 요청도 가능하다.

![](https://blog.kakaocdn.net/dn/ROiaC/btsCBJ7KA3S/K5kjKKHnwr8M7PPl7IZsC1/img.png)

메뉴에 들어간 김에 삭제까지 완료했다.

## 정리

![](https://blog.kakaocdn.net/dn/tNNX4/btsCG103eSE/UeWnkjDCB4TNImiAubkqT0/img.png)

종료된 인스턴스, 기본 보안 그룹만 남아있게 되었다.

### 마무리

![](https://blog.kakaocdn.net/dn/nuDw8/btsCyIhCYIn/Bn0dLruJ17sgeyC97tef5K/img.png)
![](https://blog.kakaocdn.net/dn/lLZL6/btsCxGEJbHY/874osie9STnba9uAYYar10/img.png)

운영 기간 약 1년 동안 사용자는 총 897명이 집계되었다.

웹툰 추천 사이트이기 때문에 유의미한 데이터를 쌓아 추천 시스템의 개선을 바랬지만 사용자들이 접속 후 추천 시스템을 이용하는 과정이 별로 진행되지 않았다. 내가 생각하는 이유는 아래와 같다.

- 모바일 UI가 준비되지 않아 PC로만 진행이 가능 했다.
- 로그인을 하는 과정이 있어 번거롭게 느낄 수 있다.
- 그 외 다른 이유들..

첫 개인 프로젝트로써 데이터를 쌓아 그 데이터를 이용하는 과정도 경험해보면 더 좋았겠지만 서버를 구축하고, 배포를 하며 아무것도 몰랐던 웹 개발에 대해 조금 친숙해지게 된 것 같다.

이 경험을 바탕으로 계획중인 다음 개인 프로젝트 및 팀 프로젝트는 더 열심히 해봐야 겠다.

- [데모 영상 유튜브](https://youtu.be/fnjMRiy3E1A)
- [recommtoon github](https://github.com/ldhapple/Webtoon_recommender_web)