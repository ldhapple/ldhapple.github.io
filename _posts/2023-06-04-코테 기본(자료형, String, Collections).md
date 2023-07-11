---
title: 코테 기본(자료형, 문자열, Collection 등)
author: leedohyun
date: 2023-06-03 23:13:00 -0500
categories: [JAVA, Data-Structure]
tags: [java, algorithm, data-structure]
---

## List

```
List<Integer> list = new ArrayList<>();
list.get(i);
list.add(k);
list.stream().distinct().mapToInt(Integer::intValue).toArray(); 
//distinct()로 중복제거.
```

## HashMap

```
HashMap<Integer, String> map = new HashMap<Integer, String>();
map.get(key);
map.put(1, "apple");
```

## HashSet

```
LinkedHashSet<Integer> set = new LinkedHashSet<>();
set.stream().mapToInt(Integer::intValue).toArray();
```

- 정렬

```
List<Integer> list = new ArrayList<>(set);
Collections.sort(list);
```

- 원소에 접근

```
HashSet<Integer> set = new HashSet<>();

for(Integer i : set)하거나

Integer[] arr = set.toArray(new Integer[set.size()]);
이렇게 배열로 바꾸어서 index로 접근 가능.
```

- 중복된 원소만 구하기

```
set.retainAll(set_B); //set과 set_B의 공통된 요소만 남기고 다 지운다.
```


## Stream

```
Arrays.stream(numList).filter(value -> value % n == 0).toArray();
```

## ArrayList를 배열로

```
arrlist.stream().mapToInt(Integer::intValue).toArray();
arrlist.stream().mapToInt(x -> x).toArray();
```

## queue, stack, deque

- queue

```
FIFO.
Queue<Integer> queue = new LinkedList<>();
[1,2,3,4,5]가 있으면
queue.poll() 했을 때 -> [2,3,4,5] // queue.poll() == 1
queue.add(7)하면 -> [2,3,4,5,7]
queue.peek() == 2
queue.offer(7) 
// add와 같이 추가하는 함수이지만 add는 추가할 수 없는 경우 예외를 발생시키고
offer는 추가할 수 없을 때 false를 반환한다.
```

```
Queue<int[]> queue = new LinkedList<>();
queue.offer(new int[]{ i, num});
//이렇게 배열을 queue에 집어넣을 수 있는데
단순히 집어넣는 num값과 함께 몇번째인지 나타내는 i같은 값을
넣어야 한다면 이런식으로 활용가능하다.

map과 같이 key, value 느낌으로 활용 가능.
//백준 프린터큐
```

- stack

```
FILO.
Stack<Integer> stack = new Stack<>();
[1,2,3,4,5]가 있으면
stack.pop() 했을 때 -> [1,2,3,4] // stack.pop() == 5
stack.add(7) 하면 -> [1,2,3,4,7]
stack.peek() == 7
```

- deque

```
stack으로도 queue로도 활용할 수 있음.
양방향 접근 가능.

Deque<Integer> deque = new ArrayDeque<>();
deque.add() // addLast()가 기본
addFirst() addLast() 존재

deque.peek() //

LinkedList<Integer> deque = new LinkedList<>();
//이렇게 덱을 이용하면 index도 접근할 수 있고, deque의 기능도 사용 가능.
ex) deque.indexOf(num);

```

## 문자열 접근

```
String str;
char[] ch = str.toCharArray();
str.charAt(i);
str.substring(int a, int b);
str.replaceAll("[aeiou]", "");
str.equals(str2);
str.indexOf("c");
str.split(" ");
str.trim(); -> 문자열 앞 뒤의 공백을 제거, 중간 공백은 제거 X
str.contains("bc"); -> "abcd" 에서 "bc"를 포함하고 있으므로 true 반환
str.toLowerCase().replaceAll("[abcdefghijklmnopqrstuvwxyz]", ""); -> 알파벳 전부 제거
String[] strarr = str.replaceAll("[a-zA-Z]", " ").split(" ");
String str1 = Integer.toString(1234);
String str1 = String.valueOf(char, int, ...);
int a = Integer.ParseInt(str1);
//아스키코드 - 'A' = 65 / 'a' = 97 / 'z' = 127 / '0' = 48 ---> <result - '0'>
str.startsWith(another_str); // str이 another_str로 시작하는지 확인
//str=abc / another_str = abcd / true
//str=abc / another_str = acb / false
//즉 charAt(0~n)까지 계속 비교해준다고 보면 됨.
//str.indexOf(another_str) == 0 으로도 체크 가능, 
//abc / ab라고 할 때 startsWith()는 ab를 abc로 시작한다고 체크하지 못함. 하지만 abc에서 ab를 indexOf로 할 경우 0이 나오게 됨.
str.endsWith(another_str); //str이 another_str로 끝나는지 확인
```

## replaceAll 정규식
```
. 개행문자를 제외한 아무 문자 .nd -> end, and, bnd, nnd
[abc]  a,b,c중 아무것이나  [ae]nd -> and, end
[^abc] a,b,c를 제외하고   [^ae]nd -> bnd, cnd
[a-g] a~g 사이의 모든 문자  [1-9][0-9] -> 10, 25, 88, ...
a*  a 0개 이상    1[0-9]*  -> 1,10,164...
a+  a 1개 이상    1[0-9]+ -> 1,10,164...
a?   a 0개 또는 1개  1[0-9]? -> 1,11,15,19...
a{5} a 5개    [a-c]{3}  -> aaa, aab, aac, bbc, ccc
a{2,} a 2개 이상
a{2,4} a 2개 이상 4개 이하
ab|cd ab 또는 cd 
^a 문자열의 처음문자가 a
a$ 문자열의 마지막 문자가 a
\\. 특수문자는 [\\.] 이런식으로 앞에 \\ 붙여줘야 함.

str.replaceAll(~); 로 하면 적용안되고
str = str.replaceAll(~); 로 해야 적용.
```

## isStringNumber(String s)

String으로 숫자를 받은 경우 true반환 아닌경우 false반환.

```
public static boolean isStringNumber(String s)  
{  
	try{  
		Double.parseDouble(s);  
		return true;  
	}  
	catch(NumberFormatException e)  
	{  
		return false;  
	}  
}
```

## 숫자, 알파벳별 count

```
count[arr[i]]++;
countAlp[arr[i] - 'a']++;
```

## 각 자리 수 접근

```
while(num!= 0)
{
      if(num% 10 == 3 || num% 10 == 6 || num% 10 == 9)
      {
          count++;
      }
      num= num/10;
}
```

## 진법 변환
```
Integer.parseInt(String s, int radix) // s를 radix진법으로 변환
Integer.toString(int i, int radix)
Long.parseLong, Long.toString
```

```
//bin1, bin2는 이진수가 String으로 주어짐. 이진수의 합을 이진수로 나타냄을 가정
Integer.toBinaryString(Integer.parseInt(bin1, 2) + Integer.parseInt(bin2, 2)); 

// 이진수의 합을 십진수로.
Integer.toString(Integer.parseInt(bin1, 2) + Integer.parseInt(bin2, 2)); 
Integer.toBinaryString(25); -> 25를 이진수로 변환.
```

##  표기

- %02d
1:2:3을 표기할때 01:02:03으로 표기하고 싶다면 %02d 이용.

- double, float
System.out.printf("#%d", test); %d대신 %f를 사용한다.
소수점 아래자리 6자리까지 출력하고 싶다면, %.6f를 사용하면 된다.

## while문 활용

```
int N = Integer.parseInt(br.readLine());

while(N-- > 0) // N--가 0보다 클때까지, 즉 N이 0이 될때까지.
{
	~~~~~~
}
```

## Math 연산

```
Math.pow(int N, int n) // N의 n승
Math.ceil(double n) //n의 올림
Math.floor(double n) //n의 내림
Math.round(double num) // 소수 num의 첫째자리 반올림 (1.5 -> 2)
Math.round(num * 10) / 10.0; // 소수점 둘째자리 반올림 (1.45 -> 1.5)
-> String.format("%.1f", num) 같은 형식으로 원하는 소숫점 자리까지 출력 가능.
Math.abs(double n) // n의 절댓값
```

## 그 외 주의

Math.pow(2,N) 등 Math. 연산은 소수형태로 결과가 나오기 때문에 int형으로 변환해야 하는 경우 int를 붙여주어야 함.

