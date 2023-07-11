---
title: CS면접 - JSON, XML
author: leedohyun
date: 2023-07-09 23:13:00 -0500
categories: [CS]
tags: [CS, 개발자 필수지식]
---

##  데이터 교환형식 (JSON, XML)

### JSON

JavaScript 객체 문법으로 구조화된 데이터 교환의 한 형식.

객체 문법 말고도 단순 배열, 문자열도 표현 가능.

JavaScript 객체 문법 - key와 value로 구성됨. 이미 존재하는 키를 중복선언하면 나중에 선언한 해당 키에 대응한 값이 덮어쓰이게 된다.

```
//json 파일

[{
	"name" : "lee",
	"name" : "king", //얘만 남게 된다.
	"age" : 30
},
{
	"name" : {
		"firstname" : "lee"
		"lastname" : "do"
	},
	"age" : 20}
]
//대괄호로 묶으면 json array. 배열형태로 만들 수 있음.
```

```
//json파일 js에서의 접근

const fs = require('fs') //모듈1
const path = require('path') //모듈2
const a = fs.readFileSync(path.join(__dirname, "a.json") //파일 읽기
const b = JSON.parse(a) //js에서 사용하기 위해 json파일을 js object로 변환 python이라면 JSON.loads() 함수로 dictionary 변환
console.log(b[1].name.firstname) //접근
```

### JSON을 데이터 교환형식(양식)으로 사용하는 이유

여러 언어에서 사용되어 지는데 그 이유는 '독립적' 이기 때문이다.

json in js = js object
json in python = dict

이렇게 객체나 해시테이블, 딕셔너리 등으로 변환되어 쓰이게 된다.

예를들어 python, java, js같은 언어는 계속 업데이트 되는데 이렇게 언어가 업데이트 되었다고 json의 어떠한 부분을 바꿀 필요가 없다. 그리고 어떠한 언어에서 사용하던 그 언어에 맞게 변환되어 사용되어지기 때문에 이를 독립적이라고 한다.

### json 예시
```
const a = {
	"지브리OST리스트": [{
			"name" : "마녀 배달부 키키"
			"song" : "따스함에 둘러쌓인다면"
		},
		{
			"name" : "하울에 움직이는 성",
			"song" : "~~~~~~" //String 아닌 number가 들어가도 된다.
		}
	]
}
```

```
const a = {
	"name" : "lee"
	"favorite" : {
		"아령" : {
			"weight" : 10
			"feature" : "육각형"
		},
		"바나나" : {
			"color" : "green"
		}
	}
}
```

### JSON의 타입

문자열, boolean, number, 객체, 배열, json array, null 이 들어갈 수 있다.

JS와 다르게 method 가 들어갈 수 없다.

### 직렬화, 역직렬화

직렬화 - 외부의 시스템도 가져와서 사용할 수 있도록 변환하는 것.

```
const a = fs.readFileSync(path.join(__dirname, "a.json")
const b = JSON.parse(a) // 역직렬화 -> json을 js object로 만듦.
const c = JSON.stringify(b) // 직렬화 -> json을 String으로 만듦. 
//js에서 한 작업을 Python등에 가져가서 사용하기 위해. js Object 형태로는 사용 불가
```

### JSON의 활용

json은 프로그래밍 언어와 프레임워크 등에 독립적이므로, 서로 다른 시스템간에 데이터를 교환하기에 용이하다.

주로 API의 반환형태, 시스템을 구성하는 설정파일에 활용된다.

ex) npm install package 라고 했을 때 package의 버전 등을 나타내는 설정파일을
json 파일로 만든다.


## XML

마크업 형태를 쓰는 데이터 교환형식

### 마크업

마크업은 태그 등을 이용하여 문서나 데이터의 구조를 나타내는 방법이다.
(속성부여 가능)

1. 프롤로그
2. 루트요소 - 딱 한개만 존재
3. 하위요소

```
<?xml version = "1.0" encoding="UTF-8"?> //프롤로그
<OSTList> //루트요소
 <OST> //하위요소
   <name>마녀 배달부 키키</name> <song>따스함에 둘러쌓인다면</song>
 </OST>
 <OST>
   <name> ~~~~~~ </name> <song> ~~~~~ </song>
 </OST>
</OSTList>
```

### XML의 활용

sitemap.xml로 쓰인다.
내가 만든 사이트를 상위에 노출시키려면 내 사이트의 페이지 정보가 최대한 많이 검색사이트에 제공되어져야 하는데 이 때 검색사이트의 크롤링봇이 알아서 내 사이트의 정보를 잘 긁어갈 수 있도록 sitemap을 제공해야 한다. 이 때 xml파일 형식을 사용한다.

여러 언어에서도 독립적으로 사용된다.

## HTML과 XML의 차이

1. HTML의 용도는 데이터를 표시하는 것이고, XML은 데이터를 저장 및 전송한다.
2. HTML은 미리 정해진 태그를 사용하지만, XML은 태그를 정의하여 사용한다.
3. XML은 대/소문자를 구분하지만 HTML은 구분하지 않는다.


## JSON과 XML의 차이

1. JSON과 비교했을 때, 닫힌 태그가 계속해서 들어가기 때문에 JSON에 비교하면 무겁다.
2. JavaScript Object로 변환하기 위해서 JSON보다 더 많은 노력이 필요하다.

```
</OST> // 닫힌태그
Json.parse() // json은 변환과정이 이거 하나.
```