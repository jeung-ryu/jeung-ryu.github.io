---
layout: single
title: "Spring ArgumentResolver"
date: 2019-01-31 20:00:00 +0900
categories:
  - 기술공부
tags:
  - basecamp
comments: true
---
스프링의 HandlerMethodArgumentResolver에 대해서 알아보고자 한다.
HandlerMethodArgumentResolver를 포스팅하는 이유는 
이메일 서비스에서 권한문제를 처리하기 위함이다.

기존에 request 세션을 일일이 호출하여 유저 정보를 읽어오는 방식을 개선 할 수 있다.

먼저, HandlerMethodArgumentResolver를 살펴보기에 앞서 스프링의 전체적이 구조를 살펴보자.

![Spring Mvc 구조](/assets/images/argument1.PNG)

스프링에서는 DispatcherServlet이 중요한 역할을 하는데
1. request가 들어오면 요청에 적합하게 매핑되는 controller method를 선택한다.
2. 선택한 controller method 가 어떠한 구조라도 일관되게 실행해준다.
3. return 받은 값에 따라서 적합한 View를 선택하여 응답한다.

이때, DispatcherServlet이 사용하는 객체는
1. 적합한 controller method (= handler)를 선택하는데 HandlerMapping 객체를 사용한다.
2. 동일한 방식으로 controller method를 실행하기 위해 HandlerAdapter를 사용한다.
3. 적합한 View를 결정하기 위해 ViewResolver를 사용한다.



그렇다면, HandlerMethodArgumentResolver는 무엇이고 스프링의 구조상 어디에 관여하는가?

스프링에서 내린 정의는 다음과 같다.
> 'Strategy interface for resolving method parameters into argument values in the context of a given request.'

메소드의 파라미터를 request의 인자값으로 변형시키는 전략인터페이스라고 한다.

이것이 무슨말일까?
Controller method 파라미터에 @RequestParam, @PathVariable, HttpServletRequest, HttpSession을 넣어 사용해 보았을 것이다.
```java
@Controller
public class SendMailController {

	@PostMapping("/send")
	public ResponseEntity<HttpStatus> sendMail(
			HttpSession session,
			@RequestParam("recipient") String[] recipients, 
			@RequestParam("title") String title,
			@RequestParam("content") String content) {
      
      // ...
      }
}
```
이때 메소드의 파라미터의 순서를 지켜야 하는 것도 아니었고, 필수로 넣어야 하는 타입이 있는 것 도 아니었다.
하지만 스프링이 알아서 원하는 파라미터에 원하는 인자값을 넣어주었다.

이것은 스프링이 기본적으로 제공하는 DefaultArgumentResolver가 엄청 많기 때문이다.
이렇게 파라미터에 요청값을 적절하게 넣어줘서 사용할 수 있도록 하는 것이 HandlerMethodArgumentResolver 의 역할이다.

아래는 RequestMappingHandlerAdapter의 HandlerMethodArgumentResolver 리스트를 get하는 함수다.
스프링의 DefaultArgumentResolver가 많이 있다는 것을 알 수 있다.
```java
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```

위 함수를 보고 유추 할 수 있듯이 HandlerMethodArgumentResolver는 HandlerAdapter가 사용한다.

HandlerAdapter가 컨트롤러를 실행하기 전에 파라미터 값을 채우기 위해 HandlerMethodArgumentResolver의 도움을 받는다.
![Spring Mvc ArgumentResolver](/assets/images/argument2.PNG)

HandlerAdapter는 Handler(=controller method)를 선택한 이후 실행 가능 하도록 InvocableHandlerMethod로 감싼다.
이때 해당 메소드의 파라미터를 처리할 HandlerMethodArgumentResolvers를 세팅한다.
또한 리턴값을 처리하는 HandlerMethodReturnValueHandlers 등등을 세팅한다.

![Spring Mvc ArgumentResolver](/assets/images/argument3.PNG)

컨트롤러의 메소드를 호출하게 되면 해당 파라미터에 어떠한 HandlerMethodArgumentResolver를 적용할 수 있는지 검사하는 구문이 돌아간다.
적용 가능한 파라미터일 때 HandlerMethodArgumentResolver는 request, session 등에서 값을 읽어와 파라미터 타입에 적절하게 변경시켜준다.

따라서 HandlerMethodArgumentResolver는 다음과 같은 역할을 해야한다.
1. 적용가능한 파라미터인지 검사
2. 값을 읽어와 파라미터 타입으로 적절하게 변형시켜 return

```java
public interface HandlerMethodArgumentResolver {
	boolean supportsParameter(MethodParameter parameter);

	Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;

}
```

따라서 위와 같은 인터페이스를 하고 있다
1. supportsParameter()는 검사 로직이고
2. resolveArgument()는 값을 변형하는 로직이다.


Custom HandlerMethodArgumentResolver를 사용하고 싶다면 위 인터페이스를 구현하면 된다.

정확히는 다음과 같이 한다.
1. 파라미터에 사용할 사용자정의클래스를 만든다.
2. HandlerMethodArgumentResolver를 구현한 클래스를 만들고, resolveArgument() 메소드로 사용자정의클래스를 리턴한다.
3. CustomResolver를 스프링에서 사용할 수 있도록 ArgumentResolvers에 등록한다.







ex)

컨트롤러 메소드에 받을 커스텀 파라메터를 지정한다.
```java
@PostMapping
public String writeMail(Mail mail) throw Exception {
  // ...
}
```

커스텀 클레스 생성
title, content는 request에서 받아오고
userName은 session에서 받아온다.
```java
public class Mail {
	private String title;
	private String content;
    private String userName;

  // setter, getter
}

```

CustomArgumentResolver 구현
```java
public class MailHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    return Mail.class.isAssignableFrom(parameter.getParameterType());
  }
 
  @Override
  public Mail resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
    Mail mail = new Mail();
    mail.setTitle(webRequest.getParameter("title"));
    mail.setContent(webRequest.getParameter("content"));
    mail.setUserName(webRequest.getAttribute("userName", WebRequest.SCOPE_SESSION));
    return mail;
  }
}
```

스프링에서 사용가능 하도록 CustomArgumentResolver 등록
```java
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {
    
    @Bean
    public MailHandlerMethodArgumentResolver mailHandlerMethodArgumentResolver() {
    		MailHandlerMethodArgumentResolver argumentResolver = new MailHandlerMethodArgumentResolver();
    		return argumentResolver;
    }
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
    		argumentResolvers.add(mailHandlerMethodArgumentResolver());
    		super.addArgumentResolvers(argumentResolvers);
    }
}
```
