---
title: Bean Validation
author: leedohyun
date: 2023-09-26 20:13:00 -0500
categories: [Spring, 스프링 MVC 활용]
tags: [java, Spring, SpringBoot]
---

검증 기능을 이전 Validation 포스트처럼 매번 코드로 작성하는 것은 번거롭다. 특히 특정 필드에 대한 검증 로직은 대부분 빈 값인지 아닌지, 특정 크기를 충족하는 지 등의 일반적인 로직이다.

```java
public clas Item {
	private Long id;

	@NotBlank
	private String itemName;

	@NotNull
	@Range(min = 1000, max = 1000000)
	private Integer price;

	@NotNull
	@Max(9999)
	private Integer quantity;
	//...
}
```

위처럼 검증 로직을 모든 프로젝트에 적용할 수 있도록 공통화하고, 표준화 한 것이 Bean Validation이다. 애노테이션 하나로 검증 로직을 매우 편리하게 사용할 수 있다.

## Bean Validation

Bean Validation은 특정 구현체가 아닌 Bean Validation 2.0(JSR-380)이라는 기술 표준이다.

검증 애노테이션과 여러 인터페이스의 모음이다. 마치 JPA가 기술 표준이고 그 구현체로 하이버네이트가 있는 것과 같다.

Bean Validation을 구현한 기술 중 일반적으로 사용하는 구현체는 하이버네이트 Validator이다.

- [검증 애노테이션 모음](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/#section-builtin-constraints)
- [공식 메뉴얼](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/)

## Bean Validation - 스프링 X

build.gradle에 아래의 의존관계를 추가해야 한다.

```
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

검증 애노테이션

- @NotBlank : 빈 값 + 공백만 있는 경우를 허용하지 않는다.
- @NotNull : null을 허용하지 않는다.
- @Range(min = 1000, max = 1000000) : 범위 안의 값이어야 한다.
- @Max(9999) : 최대 9999까지만 허용한다.

![](https://blog.kakaocdn.net/dn/ZqboL/btsvYSZjYTZ/WA92HNOsUOQaSQdAekVBp1/img.png)

위가 빈 검증기를 Java에서 사용하는 방법이다. 검증 오류가 발생한 객체, 필드, 메시지 정보 등 다양한 정보를 제공한다.

이걸 어떻게 사용할까 싶은데, 스프링 MVC는 개발자를 위해 빈 검증기를 스프링에 완전히 통합해두었다.

> 참고

```
javax.validation.constraints.NotNull
org.hibernate.validator.constraints.Range
```

javax.validation 으로 시작하면 특정 구현에 관계없이 제공되는 표준 인터페이스이고, org.hibernate.validator로 시작하면 하이버네이트 validator 구현체를 사용할 때만 제공되는 검증 기능이다. 실무에서는 대부분 하이버네이트 validator를 사용해 자유롭게 사용해도 된다.

## Bean Validation - 스프링 O

![](https://blog.kakaocdn.net/dn/lSkVS/btswcGbhmZF/YLeyhf6zyOKKW9X2KrHqb0/img.png)

Item 클래스에 애노테이션 선언 후 컨트롤러에서 위와 같이 사용하면 끝이다.

> 스프링이 어떻게 사용하나?

스프링 부트가 'spring-boot-starter-validation' 라이브러리를 넣으면 자동으로 Bean Validatior를 인지하고 스프링에 통합한다.

그리고 LocalValidatorFactoryBean을 글로벌 Validator로 등록한다.

이 Validator는 @NotNull 같은 애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에 @Valid, @Validated만 적용하면 된다.

검증 오류가 발생하면 FieldError, ObjectError를 생성해 BindingResult에 담아준다.

(참고로 직접 글로벌 Validator를 등록하면 직접 등록한 것이 우선이라 작동 안된다.)

> 검증 순서

- @ModelAttribute 각각의 필드에 타입 변환 시도
	- 성공하면 다음으로
	- 실패하면 typeMismatch로 FieldError 추가.
- Validator 적용

Bean Validator 는 바인딩에 실패한 필드는 Bean Validation을 적용하지 않는다.
사실 받는 값이 비정상이면 검증을 할 필요가 없기 때문에 당연하다고 볼 수도 있다.

## 에러 코드는 어떻게 수정하나

Bean Validation이 기본으로 제공하는 오류 메시지를 입맛에 맞게 변경하고 싶을 수 있다.

우선 bindingResult에 등록된 검증 오류 코드를 봐보면 오류 코드가 애노테이션 이름으로 등록된다. typeMismatch와 유사하다. 예시를 보자.

- @NotBlank
	- NotBlank.item.itemName
	- NotBlank.itemName
	- NotBlank.java.lang.String
	- NotBlank
- @Range
	- Range.item.price
	- Range.price
	- Range.java.lang.Integer
	- Range

얘들을 바탕으로 메시지를 errors.properties에서 관리해본다.

```
NotBlank={0} 공백X
Range={0}, {2] ~ {1} 허용
Max={0}, 최대 {1}
``` 

0은 필드명이고, 1,2 등등은 각 애노테이션마다 다르다.

좀 더 구체적으로 나누고 싶다면 NotBlank.item.itemName 과 같이 따로 구체적으로 지정해주면 된다.

![](https://blog.kakaocdn.net/dn/cgmH1m/btsv7W7qdXe/aycqGfQGkq6ux0YmasHlK1/img.png)

물론 HTML 뷰 부분은 이전과 같다.

```html
<form th:object="${item}">
	<div>
		<label for="itemName" th:text="#{label.item.itemName}">
		<input tpye="text" th:field="*{itemName}"
			th:errorclass="field-error"
			class="form-control">
		<div class="field-error" th:errors="*{itemName}"></div>
	</div>
//...	
```

## 오브젝트 오류 (글로벌 오류)

지금까지는 FieldError에 대한 부분만 다뤄졌다. 이전에 지정했던 price * quantity 제약 조건이 처리되지 않는다.

- @ScriptAssert()를 사용한다.

```java
@Data
@ScriptAssert(lang="javascript", script = "_this.price * _this.quantity >= 10000")
public Class Item {
}
```

jdk8 ~ jdk14의 JVM 상에서 사용되는 Nashorn 엔진은 javascript를 지원하지만 jdk14 이후 버전부터는 javascript가 지원되지 않는 GraalVM을 사용한다.

스프링부트 3.0 이후부터는 java17 이상을 사용해야 하기 때문에 위의 @ScriptAssert를 이용한 자바스크립트 표현식은 사용이 불가하다.

그리고 사용이 가능하더라도 실제 검증 기능이 해당 객체의 범위를 넘어서는 경우도 종종 있어 저렇게 사용하면 대응이 어렵다.

따라서 오브젝트 오류 (글로벌 오류)의 경우 오브젝트 오류 관련 부분만 직접 자바 코드로 작성하는 것이 권장된다.

- 직접 코드 작성

```java
@PostMapping("/add")
public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
	
	if (item.getPrice() != null && item.getQuantity() != null) {
		int resultPrice = item.getPrice() * item.getQuantity();
		if (resultPrice < 10000) {
			bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
		}
	}
//...
```

상기하자면 에러코드는 totalPriceMin 으로 관리하면 되고, BindingResult.reject를 통해 ObjectError를 생성한 것이다.

그리고 뷰에서는 

```html
<div th:if="${#fields.hasGlobalErrors()}">
	<p class="field-error" th:each="err : ${fields.globalErrors()}"
		th:text="${err}"></p>
</div>
```

와 같이 접근했었다.

## Bean Validation의 한계 및 극복

데이터를 등록할 때와 수정할 때는 요구사항이 다를 수 있다.

- quantity는 등록시 9999가 max였지만 수정 시 무제한.
- id는 등록 시 값이 없어도 되지만, 수정 시 id값이 필수.

이 요구사항을 지키겠다고 id에 @NotNull, quantity의 @Max를 지우면?

등록 폼에서 제대로 등록이 안된다. (id는 입력하는 칸이 없어 무조건 NotNull 오류가 발생하고, 수량 제한도 없이 등록된다.)

> 참고

현재 구조에서 수정 시 item의 id값은 항상 들어있도록 로직이 구성되어 있다. 그래서 검증하지 않아도 된다고 생각할 수 있다.

하지만 HTTP 요청은 언제든 악의적으로 변경해 요청할 수 있다. 따라서 서버에서 항상 검증해야 한다.

### BeanValidation - groups (극복)

위와 같은 문제를 해결하기 위해 Bean Validation은 groups라는 기능을 제공한다.

예를 들면 등록 시 검증할 기능과 수정 시 검증할 기능을 각각 그룹을 나누어 적용할 수 있다.

```java
//저장용 groups 생성
package hello.itemservice.domain.item;

public interface SaveCheck {
}
```

```java
//수정용 groups 생성
package hello.itemservice.domain.item;

public interface UpdateCheck {
}
```

```java
//groups 적용
package hello.itemservice.domain.item;

@Data
public class Item {
	@NotNull(groups = UpdateCheck.class)
	private Long id;

	@NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
	private String itemName;

	//...

	@NotNull(groups = {SaveCheck.class, UpdateCheck.class})
	@Max(value = 9999, groups = SaveCheck.class)
	private Integer quantity;
```

이런식으로 class를 이용해 groups를 나누어 준 후,

```java
@PostMapping("/add")
public String addForm(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
//...
}
```

와 같이 컨트롤러의 @Validated에서 나누어주면 된다.

> 참고

@Valid 에는 groups를 사용할 수 있는 기능이 없다. groups를 사용하기 위해서는 @Validated를 사용해주어야 한다.

> 정리

groups 기능을 사용하면 등록, 수정 시 각각 다르게 검증을 할 수 있었다. 그런데 groups 기능을 사용하는데 인터페이스를 생성하고, Item에도 코드를 추가하고 하는 등 전반적으로 복잡도가 올라갔다.

이 groups 기능은 실제로는 잘 사용되지 않는다. 그 이유는 실무에서 주로 등록용 폼 객체와 수정용 폼 객체를 분리해 사용하기 때문이다.

### Form 전송 객체 분리 (극복 2, 실무 사용)

groups를 잘 사용하지 않는 이유는 지금과 같은 학습용 프로젝트에서는 Item 도메인 객체와 폼에서 전달하는 데이터가 딱 맞지만 실무에서는 맞지 않을 수 있다.

예를 들면 회원 등록시 회원과 관련된 데이터만 전달받는 것이 아닌, 약관 정보도 추가로 받는 등의 Member와 관계없는 수 많은 부가 데이터가 넘어온다.

그래서 보통 Member을 직접 전달받는 것이 아닌, 복잡한 폼 데이터를 컨트롤러까지 전달할 별도의 객체를 만들어 전달한다.

예를 들면 MemberSaveForm 이라는 폼을 전달받는 전용 객체를 만들어서 @ModelAttribute로 사용하고 이것을 통해 컨트롤러에서 폼 데이터를 전달받고 이후 컨트롤러에서 필요한 데이터를 사용해 Member를 생성한다.

- 기존
	-	HTML Form -> Item -> Controller -> Item -> Repository
		-	장점 : 중간에 Item을 따로 만드는 과정이 필요없어 간단하다.
		-	단점 : 간단한 경우만 가능하다. groups를 사용해야 한다.
- 별도 객체 사용
	- HTML Form -> ItemSaveForm -> Controller -> Item 생성 -> Repository
		- 장점 : 전송하는 폼 데이터가 복잡해도 거기에 맞춘 별도의 폼 객체를 사용해 데이터를 전달받을 수 있다. 검증 중복도 될 일 없다.
		- 단점 : 폼 데이터를 기반으로 컨트롤러에서 Item 객체를 생성하는 변환과정이 추가된다.

#### 코드

우선 Item에서는 검증이 필요 없기 때문에 애노테이션을 지운다.

그리고 ItemUpdateForm, ItemSaveForm을 만드는데 이 친구들은 컨트롤러 단에서 사용되는 것이기 때문에 domain 패키지에 넣지 않고 컨트롤러 쪽에 따로 패키지를 만들어 저장한다.

```java 
package hello.itemservice.web.form;

@Data  
public class ItemSaveForm {  
  
	@NotBlank  
	private String itemName;  
  
	@NotNull  
	@Range(min = 1000, max = 1000000)  
	private Integer price;  
  
	@NotNull  
	@Max(value = 9999)  
	private Integer quantity;  
}
```

```java
package hello.itemservice.web.form;
@Data  
public class ItemUpdateForm {  
  
	@NotNull  
	private Long id;  
  
	@NotBlank  
	private String itemName;  
  
	@NotNull  
	@Range(min = 1000, max = 1000000)  
	private Integer price;  
  
	@NotNull  
	private Integer quantity;  
}
```

그리고 컨트롤러에서의 적용을 보자.

```java
@PostMapping("/add")  
public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
	if (form.getPrice() != null && form.getQuantity() != null) {  
		int resultPrice = form.getPrice() * form.getQuantity();  
		if (resultPrice < 10000) {  
			bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);  
		}  
	}  
  
	if (bindingResult.hasErrors()) {  
		log.info("errors={}", bindingResult);  
		return "validation/v4/addForm";  
	}  
  
	//성공 로직  
  
	Item item = new Item();  
	item.setItemName(form.getItemName());  
	item.setPrice(form.getPrice());  
	item.setQuantity(form.getQuantity());  
  
	Item savedItem = itemRepository.save(item);  
	redirectAttributes.addAttribute("itemId", savedItem.getId());  
	redirectAttributes.addAttribute("status", true);  
	return "redirect:/validation/v4/items/{itemId}";  
}
```

- @ModelAttribute를 form으로 바꾸어주었다.
	- 옆에 ("item")을 붙이지 않으면 model에 "itemSaveForm" 이라는 이름으로 저장되기 때문에 붙여준다.
	- 지금은 "item"으로 되어있는 뷰 템플릿을 수정하지 않을 용도로 이렇게 했지만 실제로는 뷰 템플릿을 고쳐주는게 옳다. (th:object)
- form의 값을 Item객체로 set해주어 객체를 만들고 리포지토리로 저장했다.
	- 실제로는 생성자로 객체를 만드는 방식이 권장된다. 


## Bean Validation - HTTP 메시지 컨버터

@Valid와 @Validated는 HttpMessageConverter (@RequestBody) 에도 적용할 수 있다.

- @ModelAttribute : HTTP 요청 파라미터 다룰 때.
- @RequestBody : HTTP Body의 데이터를 객체로 변환할 때.
	- 주로 API JSON 요청을 다룰 때

![](https://blog.kakaocdn.net/dn/uyq9h/btsv8OVwhWb/N0k5aYLEYiuuaoKgm8h3K1/img.png)

- JSON 데이터를 정상적으로 보내면 에러가 발생하지 않는다.
- JSON 객체를 정상적으로 생성하고 quantity 99999 같이 검증 로직을 위반하면 원하던대로 검증 오류가 발생한다.
- 만약 (quantity : ) 이런식으로 공백으로 보내면 JSON 객체를 생성하지 못하고, 이러면 컨트롤러 자체에 접근하지도 못한다.

> @ModelAttribute vs @RequestBody

@ModelAttribute는 각각의 필드 단위로 세밀하게 적용된다. 그래서 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리 된 것이다.

HttpMessageConverter는 각각의 필드 단위로 적용되는 것이 아니라, 전체 객체 단위로 적용된다. 따라서 메시지 컨버터의 작동이 성공해서 ItemSaveForm 객체가 만들어져야 @Valid, @Validatd가 적용된다.

그래서 차이를 보인 것이다.

(HttpMessageConverter 단계에서 실패하면 예외가 발생하는데 예외 발생 시 원하는 모양으로 예외를 처리하는 방법은 추후 포스트에서 다룬다.)