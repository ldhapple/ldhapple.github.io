---
title: AWS EC2 서버 개설 (RDS, Elasticache)
author: leedohyun
date: 2024-03-25 21:13:00 -0500
categories: [사이드 프로젝트, recommtoon.com]
tags: [recommtoon.com, SpringBoot]
---

## 개요

Recommtoon 사이트를 어느정도 개발하고, 배포를 할 때가 됐다.

서버를 구입할 비용은 없으므로 서버로는 IaaS의 한 종류인 AWS EC2를 사용하고 도메인 또한 AWS Route53을 이용하기로 결정했다.

그리고 백엔드 단에서 MySQL, Redis를 이용하여 구현하였으므로 RDS를 이용해 MySQL을, Elasticache를 통해 Redis 환경을 구축하고자 한다.

Docker을 통해 EC2에서 설치하여 사용하는 방식을 사용할 수도 있지만 각각 장점이 있기 때문에 위 방법을 선택했다.

구축 과정에 대해 정리한다.

## RDS

- RDS
	- Amazon Relational Database Service
	- 클라우드에서 관계형 DB를 간편하게 설정 및 운영, 확장을 할 수 있도록 도와준다.
	- 다양한 데이터베이스 엔진을 선택하여 이용할 수 있다.

그렇다면 EC2에 DB를 직접 설치해 사용하는 방식에 비해 장점은 뭘까?

우선은 두 환경 모두 VPC 보안 환경에서 데이터베이스를 구축할 수 있고, 확장성도 뛰어나다.

그러나 DB를 직접 관리하는 EC2 방법에 비해 RDS는 개발자의 관리 책임을 줄일 수 있다.

![](https://blog.kakaocdn.net/dn/v9ABZ/btrvgl0qHul/355y9meVyk80w936HsuWr0/img.png)

AWS에서 제공하는 지침서를 보자.

우선 On premises 방식은 자체적으로 서버를 구축하여 하는 방식이기 때문에 당연히 모든 관리와 책임을 User가 지게 된다.

- EC2 vs RDS (RDS를 사용했을 때 어떤 책임이 줄어드는 지 보자)
	- 데이터베이스 규모 확장
	- 가용성, 내구성 확보
	- 데이터 백업
	- 데이터베이스 설치 및 관리
	- OS 설치 및 관리
	- 기반 구축

위 책임들을 EC2와 비교했을 때 AWS에서 담당해준다고 보면 된다.

### RDS 고가용성

RDS는 고가용성(high availability) 아키텍처를 지원한다. 데이터베이스가 잠시라도 셧다운되면 기업의 모든 업무가 마비될 수 있기 때문에, 데이터베이스는 본질적으로 고가용성 아키텍처를 따른다.

- 싱글 AZ(Availability Zone) 배포
	- 상용화 환경이 아닌 개발 환경에서 DB를 사용하거나 경험 차원에서 RDS를 사용하는 경우 싱글 AZ 환경에서 RDS 인스턴스를 사용해도 된다.
	- 적당한 스토리지의 하나의 인스턴스를 사용한다.
- 멀티 AZ(Availability Zone) 배포
	-  기업에서 데이터를 잃어버려서는 안되고, 서비스에 대한 장애가 있어서는 안될 경우 멀티 AZ 환경에서 DB를 배포해야 한다.
	- 마스터 DB라고 부르는 기본 DB가 모든 트래픽을 처리하고, 스탠바이 DB는 기본 DB가 셧다운되는 상황을 대비해 가동 가능 상태에서 대기한다.
	- 스탠바이 DB는 대기 상태에 있는 한 작동시킬 수 없고, 트래픽 분산도 불가능하다. 다만 기본 DB의 데이터는 스탠바이 DB에 복제된다.
	
RDS는 장애 발생 시 다양한 유형의 대응 방식을 자동으로 적용해, 호스트 인스턴스 셧다운, 스토리지 장애, 기본 인스턴스의 네트워크 연결 단락, AZ 자체 셧다운 등의 장애 상황에 대처하도록 도와준다.

### RDS 확장성 구현

RDS에서 데이터베이스의 확장을 하는 방법은 2가지가 있다.

- 인스턴스 타입 변경

단순하게 인스턴스 타입을 변경하여 스케일업, 스케일 다운이 쉽게 가능하다. 다만 이 방법은 인스턴스를 변경하는 과정에서 약간의 가동 정지시간이 발생할 수 있어 이를 고려해야 한다.

- 읽기 사본 활용

읽기 사본이란 마스터 데이터베이스의 Read-Only 복사본을 의미한다. 마스터 데이터베이스와 동기화 상태를 유지하며 read-only 쿼리의 부담을 줄여준다.


### 선택

성능 및 안정성, 편리성의 측면에서 RDS가 장점을 가진다. 다만 그런만큼 비용이 비싸다.

따라서 DB에 대해 잘 알고, 백업, 리플리카 등을 구성하는데 어려움이 없는 경우라면 EC2 환경에서 DB를 직접 설치해 사용해도 무방하다.

## Elasticache

RDS와 같은 관계형 데이터베이스이다.

다만 Redis나 Memcached같은 캐시 기술을 관리할 수 있도록 하는 서비스이다. RDS와 대부분의 장점은 비슷하다.

Elasticache를 사용하면 데이터베이스 읽기 작업에 특화되어 읽는 작업에 대한 부하를 줄일 수 있다.

> RDS와의 차이

RDS는 보안그룹을 설정하면 AWS 외부에서도 접근이 가능하다.

하지만 Elasticache는 보안 그룹을 설정해도 동일한 VPC에 속한 EC2 인스턴스에서만 접근이 가능하다는 차이가 있다.

## EC2 구축

이번 포스트에서 EC2, RDS, Elasticache의 구축에 대해 다룰 것이다. 먼저 EC2부터 보자.

![](https://blog.kakaocdn.net/dn/XwO76/btsGfKiw1oq/0mAjwwfR9hrYbfFkChK5K1/img.png)

우선 Ubuntu 환경의 프리티어 이미지를 고른다.

![](https://blog.kakaocdn.net/dn/u5w7l/btsGfgPBTlx/1Wk17Ijal0p6itHCAzMtLk/img.png)

인스턴스는 t2.micro를 선택하고, 키페어는 존재한다면 그것을 사용하고 없다면 새 키페어를 생성해주면 된다.

파일은 다시 발급받을 수 없기 때문에 잘 저장해야 한다.

![](https://blog.kakaocdn.net/dn/bBb36N/btsGh7cNgDc/oKSSnbsrsg7TWbxYSDAHKK/img.png)

보안그룹 또한 기존에 존재하고 사용할 수 있다면 사용하고, 없으면 보안 그룹을 생성해주면 된다.

그리고 인스턴스를 중지했을 때 IP 변경을 방지하기 위해 탄력적 IP를 해당 인스턴스에 할당해준다.

이후 pem키 파일과 호스트 IP 주소를 이용해 mobaxterm, putty 같은 도구로 서버에 연결할 수 있다.

## RDS 구축

![](https://velog.velcdn.com/images/wonizizi99/post/23934f87-0baa-47fe-92b4-6e94a4224960/image.png)
![](https://velog.velcdn.com/images/wonizizi99/post/5f24e429-5cc4-4e26-a443-2046e838ec32/image.png)

자신에게 맞는 DB를 선택한다.

![](https://velog.velcdn.com/images/wonizizi99/post/efe345e3-6798-4563-a3df-37d50d06372e/image.png)

![](https://velog.velcdn.com/images/wonizizi99/post/f235f058-1912-4107-b775-09801a859704/image.png)

- 마스터 사용자의 id와 암호는 추후 MySQL 접속에 이용되어 기억해두어야 한다.
- 
![](https://velog.velcdn.com/images/wonizizi99/post/ed1d8211-cbbf-4d3f-8de1-83c118fab6e7/image.png)
![](https://velog.velcdn.com/images/wonizizi99/post/9314fab9-5540-4934-82ba-ac367b014d55/image.png)

![](https://velog.velcdn.com/images/wonizizi99/post/321fb9ae-1292-421a-9dd9-22b73afbf958/image.png)

- 퍼블릭 액세스는 예로 지정한다. 아니요를 선택할 경우 퍼블릭 IP 주소가 할당되지 않아 외부에서 접속이 불가능하다.

![](https://velog.velcdn.com/images/wonizizi99/post/11977960-9524-48f2-93c6-911a6f5309fc/image.png)

- 백업 설정 활성화
	- 연습용일 경우 데이터가 중요하지 않다면 비활성화한다.
	- 스냅샷 생성으로 인해 메모리를 낭비할 수 있기 때문이다.

### RDS 파라미터 그룹 세팅

RDS - 파라미터 그룹 - 편집

![](https://velog.velcdn.com/images/wonizizi99/post/ed531f78-9fee-4acf-ae2f-3e0d6a8f3a68/image.png)

character를 검색하면 나오는 파라미터를 utf8mb4로 설정.

![](https://velog.velcdn.com/images/wonizizi99/post/ecc8e1a5-0f70-4982-a0fa-815d55125b7c/image.png)

time_zone을 검색하면 나오는 파라미터 값을 서울로 설정.

![](https://velog.velcdn.com/images/wonizizi99/post/7fc30b84-74fe-40f8-a998-8e280cbb0822/image.png)

collation을 검색하여 나오는 파라미터의 값을 uf8mb4_general_ci로 변경

![](https://velog.velcdn.com/images/wonizizi99/post/63f90bb7-e911-49ad-851b-424d0cb75836/image.png)

max_connections 수정.

### MySQL EC2 설치 및 확인
```
sudo apt update
sudo apt install mysql-server

sudo ufw allow mysql
sudo systemctl start mysql

mysql -h RDS엔드포인트 -u admin -p
```

위 명령어로 mysql을 EC2에 설치하고, 접근해볼 수 있다.

## Elasticache 구축

![](https://velog.velcdn.com/images/yes_jihyeon/post/c2dc9f77-f6b3-4e4c-b9dd-d4648572037e/image.png)
![](https://velog.velcdn.com/images/yes_jihyeon/post/0221b7db-ecbd-4a6d-b3cc-c0da4ad639d4/image.png)

프리티어 기준으로 t2.micro 노드 1개에 대해 무료로 제공된다.

클러스터 모드는 동적으로 노드 갯수가 늘어나 요금이 발생할 가능성이 있어 비활성화 해준다.

![](https://velog.velcdn.com/images/yes_jihyeon/post/1009ec92-1c26-40de-878c-bda99de9947a/image.png)
![](https://velog.velcdn.com/images/yes_jihyeon/post/b9b47941-a6e2-4841-9b3c-0e3d62c142b0/image.png)

복제본의 갯수를 2개로 하면 Free-tier의 경우 750(t2.micro 무료 제공시간)/3(master + 복제본2)시간으로 소모된다.

따라서 비용이 부과될 수 있어서 0으로 설정하는 것을 권장한다.

```
sudo apt-get update
sudo apt-get install build-essential wget

wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz

cd redis-stable
make distclean 
make

src/redis-cli -c -h yourcachecluster.yourregion.cache.amazonaws.com -p 6379

set a "hello"
```

이후 Ubuntu 기준 위 명령어를 통해 redis를 설치하고 확인할 수 있다.

yourcachecluster.yourregion.cache.amazonaws.com 부분에는 본인 Elasticache의 기본 엔드포인트 주소를 입력하면 된다.