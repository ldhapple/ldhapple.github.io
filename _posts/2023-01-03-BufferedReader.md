---
title: BufferedReader, BufferedWriter, StringTokenizer
author: leedohyun
date: 2023-01-02 23:13:00 -0500
categories: [JAVA, Data-Structure]
tags: [java, algorithm, data-structure]
---

## BufferedReader

Scanner를 사용하면 space, enter 모두를 경계로 인식해 입력 데이터를 가공하기 편하다.
Buffered는 enter만 경계로 인식하고 받은 데이터를 String으로 고정시키기 때문에 입력 데이터를 가공하는 작업이 필요하다.

단, 많은 양의 데이터를 입력 받을 때 Buffered를 통해 입력받는 것이 빠르기 때문에 사용하게 된다.

### 사용법

> n*n 크기의 2차원 배열을 입력받고 값 집어넣기



```
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
int n = Integer.parseInt(br.readLine());
int map[][] = new int[n][n];

for(int i = 0; i < n; i++)
{
	StringTokenizer st = new StringTokenizer(br.readLine());
	for(int j = 0; j < n; j++){
		map[i][j] = Integer.parseInt(st.nextToken());
	}
}
```

이런 경우 BufferedReader 객체를 매 줄 입력받을 때 생성해주어야 StringTokenizer를 사용하기 편하다.
