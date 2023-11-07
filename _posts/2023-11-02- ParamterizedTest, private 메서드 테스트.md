---
title: ParameterizedTest, private 메서드의 테스트에 관해
author: leedohyun
date: 2023-11-02 19:13:00 -0500
categories: [JAVA, 정리]
tags: [JAVA, 정리]
---

## @ParameterizedTest

JUnit에서 여러 값에 대한 테스트를 작성하기 위해 @ParameterizedTest 라는 애노테이션을 제공한다. 기본적인 사용 방법은 @Test 대신 @ParamaterizedTest 애노테이션을 사용하는 것이다.

이 때 파라미터로 넘겨줄 값들을 지정해주어야 한다. 아래 그 방법들을 보자.

### @ValueSource

```java
@ParameterizedTest  
@ValueSource(strings = {"1999", "28,900", "3001"})  
@DisplayName("1000원 단위 금액을 입력하지 않았을 경우 예외 발생")  
void testIndivisibleValue(String indivisibleInput) {  
	assertThrows(PurchasePriceDivisibleException.class, () -> {  
		PurchasePrice.create(indivisibleInput);  
	});  
}
```

테스트에 주입할 값을 애노테이션에 배열로 지정한다. 테스트를 실행하면 배열을 순회하며 테스트 메서드의 인수에 그 값들을 주입해 테스트한다.

이 때 하나의 테스트에는 하나의 인수만 전달할 수 있다.

- 가능 자료형
	- byte, short, int, long, double, float, char, boolean
	- String, Class

### @CsvSource

@ValueSource는 하나의 인수 (String indivisibleInput) 만 전달 가능했다. 그런데 두 가지 이상의 인수를 전달하고 싶다면?

```java
@ParameterizedTest  
@CsvSource({  
	"0, true, FAIL",  
	"1, false, FAIL",  
	"2, false, FAIL"  
})  
@DisplayName("맞힌 개수가 3개 미만이면 보너스번호와 상관없이 FAIL 이다.")  
void testFailWinningStatus(int matchCount, boolean withBonusNum, LottoResultStatus expected) {  
	LottoResultStatus result = LottoResultStatus.checkResult(matchCount, withBonusNum);  
  
	assertEquals(expected, result);  
}
```

@CsvSource를 활용하면 여러 개의 input 값이 필요한 메서드에 테스트를 할 수 있고, 기댓값까지 한 번에 포함시켜 여러 테스트를 할 수 있게 된다.

이 때 @CsvSource에는 하나의 문자열 내 콤마(,)를 통해 값들을 구분지어준다. delimiter 값을 직접 정의하여 커스텀 구분자를 사용할 수도 있다.

```java
@CsvSource({  
	"0:true:FAIL",  
	"1:false:FAIL",  
	"2:false:FAIL",
	delimiter = ':'  
}) 
```

delimeter 값은 char 형인데 만약 String 형태의 구분자를 사용하고 싶다면 "delimiterString" 을 사용하면 된다. 

### @NullSource, @EmptySource, @NullAndEmptySource

```java
@ParameterizedTest
@DisplayName("null 값 또는 empty 값으로 User 생성 테스트")
@NullAndEmptySource
void createUserExceptionFromNullOrEmpty(String text) {
    assertThatThrownBy(() -> new User(text))
            .isInstanceOf(IllegalArgumentException.class);
}
```

@NullSource는 테스트 메소드에 인수로 null을, @EmptySource는 빈 값을, @NullAndEmptySource는 null과 빈 값을 모두 주입한다.

String text 값에 Null 값이 주입된 것이다.

- 원시 값에는 null 값이 들어갈 수 없으므로 메서드의 인수가 원시 값이라면 @NullSource, @NullAndEmptySource는 사용 불가능하다.
- @NullSource, @EmptySource를 모두 사용한 것과 @NullAndEmptySource는 같다.
- @ValueSource와 함께 사용할 수 있다.

```java
@ParameterizedTest
@DisplayName("null 값 또는 empty 값으로 User 생성 테스트")
@NullAndEmptySource
@ValueSource(strings = {""," "})
void createUserExceptionFromNullOrEmpty(String text) {
    assertThatThrownBy(() -> new User(text))
            .isInstanceOf(IllegalArgumentException.class);
}
```

### @EnumSource

```java
@ParameterizedTest
@DisplayName("6, 7월이 31일까지 있는지 테스트")
@EnumSource(value = Month.class, names = {"JUNE", "JULY"})
void isJuneAndJuly31(Month month) {
    assertThat(month.minLength()).isEqualTo(31);
}
```

Enum 클래스의 모든 값을 사용하려면 @EnumSource 안에 Enum 클래스만 전달해주면 된다. 특정 값만 필요할 경우 value에 Enum 클래스를 넣어주고 names에 선택할 값의 이름을 전달해주면 된다.

이 때 names까지 값을 넣으면 추가로 mode 값을 넣어줄 수 있다.

- INCLUDE
	- names.contains(name)
	- name과 일치하는 모든 Enum값 //default
- EXCLUDE
	- !names.contains(name)
	- name을 제외한 모든 Enum값
- MATCH_ANY
	- patterns.stream().anyMatch(name::matches)
	- 조건을 하나라도 만족하는 Enum 값
- MATCH_ALL
	- patterns.stream().allMatch(name::matches)
	- 조건을 모두 만족하는 Enum 값

### @MethodSource

위의 애노테이션들을 이용해도 전달할 수 없는 복잡한 경우가 있을 수 있다. 이 때 method를 인수로 전달해주어 복잡한 인수를 전달할 수 있다.

```java
@ParameterizedTest
@MethodSource("provideStringsForIsBlank")
void isBlank_ShouldReturnTrueForBlankStrings(String input, boolean expected) {
    assertThat(input.isBlank()).isEqualTo(expected);
}

private static Stream<Arguments> provideStringsForIsBlank() {
    return Stream.of(
            Arguments.of("", true),
            Arguments.of("  ", true),
            Arguments.of("not blank", false)
    );
}
```

provideStringsForIsBlank()라는 메서드를 정의해 값들을 넘겨주고 있다. 규칙을 알아보자.

- @MethodSource에 작성하는 메서드 이름은 인수로 제공하려는 메서드 이름과 같아야 한다.
- 인수로 제공하려는 메서드는 static 이어야 한다.
	- 단, @TestInstance(Lifecycle.PER_CLASS)를 사용하여 클래스 단위 생성주기일 경우 인스턴스 메서드 제공이 가능하다.
- @MethodSource에 메서드 이름을 작성해주지 않을 경우 테스트 메서드 네임과 같은 메서드를 찾아 인수로 제공한다.
- 만약 테스트 호출 당 하나의 인수만 제공하고자 한다면 Arguments로 추상화 할 필요는 없다.

```java
class StringsUnitTest {
    @ParameterizedTest
    @MethodSource("parameterizedtest.StringParams#blankStrings")
    void isBlank_ShouldReturnTrueForBlankStringsExternalSource(String input) {
        assertThat(input.isBlank()).isTrue();
    }
}

// StringParams 클래스는 parameterizedtest 패키지에 있다고 가정
public class StringParams {
    static Stream<String> blankStrings() {
        return Stream.of("", "  ");
    }
}
```

-  정규화된 이름(FQN#methodName의 형식)으로 외부 정적 메소드를 참조할 수 있다.

### @ArgumentsSource

메서드로 선언하지 않고 클래스로 선언할 수도 있다. ArgumentsProvider 인터페이스를 구현한 클래스를 @ArgumentsSource 어노테이션에 선언해주면 된다.

```java
public class LottoNumberArgumentsProvider implements ArgumentsProvider { 

	@Override 
	public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
		 return Stream.of( 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 3, 4, 5, 6)), Rank.FIRST), 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 3, 4, 5, 7)), Rank.SECOND), 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 3, 4, 5, 9)), Rank.THIRD), 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 3, 4, 9, 10)), Rank.FOURTH), 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 3, 8, 9, 10)), Rank.FIFTH), 
			 Arguments.arguments(new Lotto(givenNumbers(1, 2, 8, 9, 10, 11)), Rank.NONE) 
		 ); 
	 } 

	private static List<Number> givenNumbers(int... numbers) { 
		return Arrays.stream(numbers) 
			.mapToObj(Number::new) 
			.collect(Collectors.toList()); 
	} 
}
```

```java
@ParameterizedTest(name = "로또번호 : {0}, 결과 : {1}")  
@ArgumentsSource(LottoNumberArgumentsProvider.class)  
@DisplayName("맞춘 번호에 따라 등수를 반환한다.")  
void findRank(Lotto lotto, Rank rank) { 	
	assertThat(WINNER_LOTTO.findRank(lotto)).isEqualTo(rank);
}
```

## @Nested

비슷한 테스트 메서드를 @Nested 클래스로 묶어 알아보기 쉽게 해준다.

```java
public  class DisplayNameTest { 
	@Nested  
	class testA { 
		@Test  
		public void success() { 
			/* */ 
		} 

		@Test  
		public void fail() { 
			/* */ 
		} 
	} 

	@Nested  
	class testNumber { 

		@Nested  
		class test1 { 
			@Test  
			public void success() { 
				/* */ 
			} 

			@Test  
			public void fail() { 
				/* */ 
			} 
		} 

		@Nested  
		class test2 { 
			@Test  
			public void success() { 
				/* */ 
			} 

			@Test  
			public void fail() { 
				/* */ 
			} 
		} 
	} 
}
```

비슷하게 class로 묶어놓아 단순하게 success(), fail()을 중복으로 사용해도 구분하는데 문제가 없게 된다.

## private 메서드를 테스트?

private 메서드는 테스트가 어렵다. 그냥 할수는 없고 리플렉션을 사용해 테스트하거나, package-private (default) 접근제어자를 사용해 테스트할 수도 있다.

그렇다고 private으로 된 메서드는 테스트를 하지 않아야하나?

아니다. 해야한다. 테스트를 해야하는 모든 로직이 private이 아닐 수는 없다.

### private 메서드를 테스트하고 싶을 경우.

private 메서드를 테스트하고 싶다는 생각이 드는 경우 자체가 사실 객체 분리의 신호탄이 될 수 있다.

객체가 잘 설계된 상태되어 각각의 의미와 역할이 명확하고 추상화가 잘 된 public 메서드를 테스트 함으로써 내부 private 메서드까지 테스트가 되어야 한다.

즉, private 메서드를 따로 테스트하고 싶은 경우가 생긴다면 그 객체의 역할에 대해 생각해보고  미래를 위해 리팩토링을 시도하는 것이 옳다.

### 결론

private 메서드는 테스트하지 않는다.

만약 private 메서드를 테스트해야 하는 경우 아래와 같은 문제점이 있는 지 체크해봐야 한다.

- private 메서드에 dead code가 있다.
- private 메서드가 너무 복잡해 다른 클래스에 속해야 한다.
- 애초에 private 메서드가 아니어야 한다.

즉, public 메서드만을 테스트하는 것이 바람직하다. private을 반드시 테스트 해야 하는 상황이 온다면 높은 확률로 설계가 잘못된 것일 수 있다.

( 테스트 코드 작성을 통해 자신의 코드에 대한 설계가 잘 되었는 지도 확인할 수 있다는 뜻이다. )