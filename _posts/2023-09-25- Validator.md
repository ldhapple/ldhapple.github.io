---
title: Validator 분리
author: leedohyun
date: 2023-09-25 21:13:00 -0500
categories: [Spring, 스프링 MVC 활용]
tags: [java, Spring, SpringBoot]
---

## Validator 분리

이전의 두 포스트에서 다루었던 방법은 하나의 컨트롤러에서 검증 로직을 다 담당하고 있었다.

이러한 경우 검증 로직을 별도의 클래스가 담당하도록 해 역할을 분리시키는 것이 좋다. 이렇게 분리하면 재사용도 쉬워진다.

ItemValidator를 만들어본다.

```java
@Component  
public class ItemValidator implements Validator {  
  
	@Override  
	public boolean supports(Class<?> clazz) {  
		return Item.class.isAssignableFrom(clazz);  
	}  
  
	@Override  
	public void validate(Object target, Errors errors) {  
		Item item = (Item) target;  
  
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "itemName",  "required");  
  
		if (item.getPrice() == null || 
			item.getPrice() < 1000 ||  
			item.getPrice() > 1000000) {  
				errors.rejectValue("price", "range", new Object[]{1000, 1000000},  null);  
		}  
		if (item.getQuantity() == null || item.getQuantity() > 10000) {  
			errors.rejectValue("quantity", "max", new Object[]{9999}, null);  
		}  
		//특정 필드 예외가 아닌 전체 예외  
		if (item.getPrice() != null && item.getQuantity() != null) {  
			int resultPrice = item.getPrice() * item.getQuantity();  
			if (resultPrice < 10000) {  
				errors.reject("totalPriceMin", new Object[]{10000,  resultPrice}, null);  
			}  
		}  
	}  
}
```

- supports() : 해당 검증기를 지원하는 지 여부 확인
- validate() : 검증 대상 객체와 BindingResult

컨트롤러에서의 사용을 보자.

```java
@PostMapping("/add")  
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {  
  
	itemValidator.validate(item, bindingResult);  
  
	//검증에 실패하면 다시 입력 폼으로  
	if (bindingResult.hasErrors()) {  
		log.info("errors={}", bindingResult);  
		return "validation/v2/addForm";  
	}  
  
	//성공 로직  
	Item savedItem = itemRepository.save(item);  
	redirectAttributes.addAttribute("itemId", savedItem.getId());  
	redirectAttributes.addAttribute("status", true);  
	return "redirect:/validation/v2/items/{itemId}";  
}
```

검증로직을 한 줄로 해결 가능하다.

### 애노테이션 사용 (v2)

스프링이 Validator 인터페이스를 별도로 제공하는 이유는 체계적인 검증 기능을 도입하기 위함이다.

위에서는 검증기를 직접 불러 사용한 것이고, 실제로 이렇게 사용해도 된다.

하지만 Validator 인터페이스를 사용해 검증기를 만들면 스프링의 추가적인 도움을 받을 수 있다.

컨트롤러에 아래의 코드를 추가한다.

```java
@InitBinder 
public void init(WebDataBinder dataBinder) {
 	log.info("init binder {}", dataBinder);
 	dataBinder.addValidators(itemValidator); 
}
```

이렇게 WebDataBinder에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다.

- @InitBinder: 해당 컨트롤러에만 영향을 준다. (글로벌 설정은 따로 해줘야한다.)


실제로 사용은 아래와 같이 할 수 있다.

```java
@PostMapping("/add") 
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) { 
	if (bindingResult.hasErrors()) { 
		log.info("errors={}", bindingResult); 
		return "validation/v2/addForm"; 
	} 

	//성공 로직 
	Item savedItem = itemRepository.save(item); 
	redirectAttributes.addAttribute("itemId", savedItem.getId()); 
	redirectAttributes.addAttribute("status", true); return 
	"redirect:/validation/v2/items/{itemId}"; 
}
```

validator를 직접 호출하는 부분이 사라지고, 대신 검증 대상 앞에 @Validated가 붙는다.

> 동작 방식

@Validated는 검증기를 실행하라는 애노테이션이다.

이 애노테이션이 붙으면 앞서 WebDataBinder에 등록한 검증기를 찾아 실행한다. 그런데 여러 검증기를 등록한다면 그 중 어떤 검증기가 실행되어야 할 지 구분이 필요하다.

이 때 supports() 가 사용된다.

```java
@Override  
public boolean supports(Class<?> clazz) {  
	return Item.class.isAssignableFrom(clazz);  
}
```

supports(Item.class)가 호출되고, 결과가 true이므로 ItemValidator의 validate()가 호출되는 것이다.

> 글로벌 설정

```java
@SpringbootApplication 부분에 추가

@Override
public Validator getValidator() {
	return new ItemValidator();
}
```

이런식으로 각 컨트롤러에 @InitBinder를 사용하지 않고 글로벌 설정을 추가할 수 있지만 실제로는 글로벌 설정을 직접 사용하는 경우는 드물다.
