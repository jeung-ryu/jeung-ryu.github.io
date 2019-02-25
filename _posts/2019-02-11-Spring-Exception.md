---
layout: single
title: "Spring spring.profiles.include"
date: 2019-01-31 20:00:00 +0900
categories:
  - 기술공부
tags:
  - basecamp
comments: true
---


# Spring 예외 처리
완벽한 예외처리 방법이라고는 볼 수 없으니 각자 프로젝트에 알맞은 예외처리를 하실 것을 권장합니다.

## 순서

1. Exception 이란
2. Exception 처리 시점
3. 스프링 MVC 예외처리 방법

스프링 MVC 예외처리 방법을 이해하려면 Exception에 대한 이해와 예외처리 시점에 대한 이해가 필요합니다.


## Exception 이란
![exceptions-throwable.gif](/assets/images/exceptions.gif)
출처: [https://docs.oracle.com/javase/tutorial/essential/exceptions/throwing.html](https://docs.oracle.com/javase/tutorial/essential/exceptions/throwing.html)

자바에서 Throwable을 상속한 객체는 크게 두가지로 나누어 집니다.

* Error
* Exception

### Error 란
시스템 레벨의 심각한 상황이 생겼을 때 던져지고, Error를 코드상에서 Catch 한다고 하더라도 딱히 처리할 수 있는 방법이 없습니다.
ex)  java.lang.OutOfMemoryError 메모리가 풀인 상황에서 던져짐
이는 try catch로 해결 할 문제가 아니고 근본적인 코드를 최적화 해야지 해결 가능합니다.

### Exception 이란
개발자가 코드로 처리할 수 있는 오류를 말합니다. 자바는 Exception을 두가지로 분류했습니다.
1. Checked Exception
2. Unchecked Exception (RuntimeException)

Checked Exception은 개발자가 처리를 반드시 해야하는 것을 강제하는 예외입니다.
Unchecked Exception은 RuntimeException을 상속 했으며 개발자의 처리를 강제하지 않습니다.

대표적인 Checked Exception은 아래 두가지 입니다.

* IOException
* SQLException


위 예외가 발생 했을 때는 자바는 예외처리를 강제 하고 있습니다.
따라서 개발자는 

* Checked Exception이 발생하는 메소드에 throws 를 선언하여 메소드 밖으로 던질 것인 지
* try catch 로 메소드 안에서 예외를 처리할 것인 지를 정해야 합니다.

대표적인 Unchecked Exception은 아래 두가지 입니다.

* NullPointerException
* IndexOutOfBoundException

위 예외가 발생 했을 때는 자바는 예외처리를 강제 하지 않습니다.
따라서 개발자는

* 아무것도 하지 않거나.
* try catch 로 메소드 안에서 예외를 처리할 것인 지 선택 할 수 있습니다.

## Exception 처리 시점
그렇다면 과연 개발자는언제 

* Exception이 발생한 메소드에서 처리할 지
* Exception이 발생한 메소드 상위의 메소드에서 처리할 지
* Exception 발생해도 아무것도 안 할 지

를 구분할까요?

예외책임을 어떤 오브젝트가 져야 할지는 개발자의 고민이 필요한 부분입니다.

파일을 저장하거나 삭제하는 로직이 있다고 생각해봅시다.
메일 한개를 지우며 연관된 파일List를 For문을 돌면서 삭제하는 로직입니다.
IOException이 발생하면 두가지 방법을 고민 할 수 있습니다.

1. IOException을 현재 메소드에서 처리하고 나머지 for문을 계속하는 방법
2. IOException을 상위의 Transation이 적용된 서비스계층으로 던지고 이미 처리된 것을 롤백하는 방법

### 1번 방법을 선택 했을 경우

* 이메일은 Email Table 로우 삭제
* 삭제된 파일은 File Table 로우 삭제
* 삭제가 실패한 파일에 대해서는 File Table 컬럼에 tobeDeleted 컬럼에 true를 추가
* 예외를 로깅하고 해당 예외를 메신저로 받아서 개발자가 직접 삭제를 수행 가능하도록 함
* 나머지 파일을 삭제 진행
* tobeDeleted가 true인 컬럼은 나중에 배치를 돌면서 지움

순서가 될 것 입니다. 해당 메소드를 호출한 서비스 로직은 파일이 완전히 삭제 되었다는 가정으로 동작합니다.

### 2번 방법을 선택 했을 경우

* 해당 메소드에서 예외를 던짐
* 상위의 Service 계층에서 예외를 처리함 (Transaction Roleback)

현재 저희 하늘TF에서는 1번방식으로 예외를 처리하고 있습니다.
Transaction Roleback 처리가 까다로와서 선택한 방법입니다.



## 스프링 MVC 예외처리 방법
스프링MVC 어플리케이션에서 예외가 발생 할 경우

* 어떤 예외를 던질 것인가?
* 어디에서 예외를 처리할 것인가?

위 두가지에 대한 고민에 일반적으로 스프링에서 처리하는 방식을 알려드리겠습니다.

스프링은 거의 모든 Checked Exception을 Unchecked Exception으로 변환해서 던집니다.
메소드 마다 무의미한 throws선언을 피하기 위함이며,
Exception을 Controller 계층에서 관리하기 편하게 하기 위함입니다.

모든예외를 Controller 계층에서 처리해야 하는 것은 아닙니다.
하지만 스프링MVC 특성상 request에 응답을 해야하는 상황에서,
예외의 종류에 따라 다른 응답을 보내주어야 하는 상황이라면 Controller 계층에서 처리하는 편이 좋습니다.


메일을 읽을 때 권한이 없는 데이터에 접근한 경우 예를 들어 보겠습니다.
REST Controller이고 메소드는 한개 입니다.

```java
@RestController
public class MailReadingRestController {
	
	@Autowired
	private MailReadingService mailReadingService;
	
	@GetMapping("/mail/{emailId}")
	public Mail selectEmailbyId(@PathVariable("emailId") Long emailId) {
		return mailReadingService.getMailbyId(emailId);
	}
	
	//... 예외처리는 밑에 설명 하겠음
	
}
```

하나의 서비스 클래스를 가지고 있습니다.

```java
@Service
public class MailReadingServiceImpl implements MailReadingService {
	
	@Autowired
	private MailMapper mailMapper;
	@Autowired
	private EmailAtchFileMapper fileMapper;
	@Autowired
	AuthenticationUtils authUtils;

	@Override
	public Mail getMailbyId(Long emailId) {
		Mail mail = mailMapper.findByEmailId(emailId);
		// authentication
		authUtils.checkAuthentication(mail);
		
        // ...
		return mail;
	}
	
}
```

서비스 클래스에서 권한을 체크하는 공통 모듈 authUtils 을 탑니다.
여기에서 가져온 메일이 세션의 사용자와 일치하지 않을 때 UnAuthorizedException을 던집니다.

```java
@Component
public class AuthenticationUtils {
	
	public void checkAuthentication(Mail mail) {
		String user = (String) RequestContextHolder.getRequestAttributes().getAttribute("user", RequestAttributes.SCOPE_SESSION);
		if(!(user.equals(mail.getSender().split("@")[0]) || user.equals(mail.getRecipient().split("@")[0]))) {
			throw new UnAuthorizedException("unauthorized for accessing mail "+ mail.getEmailId());
		}
	}
	
    //...
}
```


UnAuthorizedException은 직접 만든 Custom RuntimeException입니다.
스프링에서는 이렇게 RuntimeException을 상속받은 사용자 예외를 던지는 것을 권장합니다.

```java
public class UnAuthorizedException extends RuntimeException {
	public UnAuthorizedException(String message) {
		super(message);
	}
	
	public UnAuthorizedException(String message, Throwable cause) {
		super(message, cause);
	}
}
```

현 상황에서 연결된 계층 구조를 본다면

Controller -> Service  -> CommonUtils

CommonUtils 에서  Unchecked Exception 을 던졌기 때문에
Controller에서 예외에 맞는 적합한 조치를 취하기 까지 Service, CommonUtils 메소드에 throws 향연을 피할 수 있습니다.

Controller에서 어떻게 예외를 잡는 방법은 아래와 같습니다.

```java
@RestController
public class MailReadingRestController {
	
	@Autowired
	private MailReadingService mailReadingService;
	
	@GetMapping("/mail/{emailId}")
	public Mail selectEmailbyId(@PathVariable("emailId") Long emailId) {
		return mailReadingService.getMailbyId(emailId);
	}
	
	@ResponseBody
	@ResponseStatus(HttpStatus.UNAUTHORIZED)
	@ExceptionHandler(UnAuthorizedException.class)
	public String unAuthorizedException() {
		return "unAuthorized";
	}
	
}
```

@ExceptionHandler를 통해서 자신이 만들었던 UnAuthorizedException클래스를 속성으로 주면
해당 예외가 발생 했을 경우 아래의 메소드 로직에서 반환값이 정해집니다.

unAuthorized를 문자로 리턴하도록 되어있습니다.
@ResponseStatus 를 통해서 HTTP상태 코드를 실어 보낼 수 있습니다.

현재 예외처리를 각각의 컨트롤러 마다 하도록 설정했는데
동일 예외가 여러 컨트롤러에서 발생한다면 한곳에 모아서 관리하는 방식도 있습니다.
위 방식은 @ControllerAdvice를 검색하거나 아래 URL을 참고하시길 바랍니다.
[https://www.baeldung.com/exception-handling-for-rest-with-spring](https://www.baeldung.com/exception-handling-for-rest-with-spring)

이상입니다. 화이팅!!