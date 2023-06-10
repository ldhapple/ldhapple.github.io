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

## Stream

```
Arrays.stream(numList).filter(value -> value % n == 0).toArray();
```

## ArrayList를 배열로

```
arrlist.stream().mapToInt(Integer::intValue).toArray();
arrlist.stream().mapToInt(x -> x).toArray();
```

## queue, stack

- queue

```
FIFO.
[1,2,3,4,5]가 있으면
queue.poll() 했을 때 -> [2,3,4,5] // queue.poll() == 1
queue.add(7)하면 -> [2,3,4,5,7]
queue.peek() == 2
```

- stack

```
FILO.
[1,2,3,4,5]가 있으면
stack.pop() 했을 때 -> [1,2,3,4] // stack.pop() == 5
stack.add(7) 하면 -> [1,2,3,4,7]
stack.peek() == 7
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
\\. 
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

## 그 외 주의

Math.pow(2,N) 등 Math. 연산은 소수형태로 결과가 나오기 때문에 int형으로 변환해야 하는 경우 int를 붙여주어야 함.

