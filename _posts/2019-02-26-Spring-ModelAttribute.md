---
layout: single
title: "Spring ModelAttribute"
date: 2019-02-25 20:00:00 +0900
categories:
  - 기술공부
tags:
  - basecamp
comments: true
---

# @ModelAttribute란
@ModelAttribute란 데이터를 바인딩해서 VIEW로 넘겨주는 역할을 한다.

@ModelAttribute가 쓰이는 곳은 2가지이다.
1. Method Level
2. Method Argument Level

2가지 모두 데이터를 바인딩해서 VIEW 층으로 넘겨준다는 목적은 같다.
단지
1. Method Level 에서는 함수에서 return 되는 값을 VIEW층으로 넘겨주고
2. Method Argument Level 에서는 함수로 들어오는 값을 VIEW 층으로 넘겨준다.

```java
@Controller
public class ShoppingController {
  
  @ModelAttribute("ad")
  public List<String> addModelAttributeByMethodReturn() {
      List<String> advertise= new ArrayList<>();
      advertise.add("sale coffee");
      advertise.add("sale books");
      return advertise;
  }

  @GetMapping("/login")
  public String login(@ModelAttribute("vipUser") User user){
      return "login";
  }
}
```

```html
<!DOCTYPE html>
<html>
<body>
    <h1>ad : ${ad}</h1>
    <h2>id : ${vipUser.id}</h2>
    <h2>name : ${vipUser.name}</h2>
</body>
</html>
```

![Spring ModelAttribute1](/assets/images/model1.PNG)

위와 같이 사용이 가능하다.
1. @ModelAttribute("ad")가 붙은 메소드는 VIEW에서 "ad"를 key로 데이터를 부를 수 있다.
2. @ModelAttribute("vipUser")가 붙은 메소드 파라미터는 "vipUser"를 key로 데이터를 부를 수 있다.

두 가지의 사용용도가 약간 다른데
1. Method Level 에서 @ModelAttribute는 특정 handler에 관계없이 VIEW에서 공통으로 사용할 데이터를 바인딩 하기 위해 사용하고
2. Method Argument Level 에서는 handler로 들어온 request값 중에 Controller 내부로직 뿐만 아니라 VIEW에서도 사용해야할 데이터를 바인딩하기 위한 목적으로 사용한다.

예를 들어 
1. 국가별로 다르게 보이는 웹 페이지를 만든다고 가정해보자.
    - VIEW를 그릴 때 꼭 필요한 정보는 고객의 위치 정보이다.
    - 이때 위치 정보는 모든 페이지 요청에서 공통적으로 필요한 정보이다.
    - 이때는 Method Level의 @ModelAttribute 에서 처리해 주는게 좋다.
    - request header 의 ip 또는 Accept-Language 를 가져와 위치 정보를 찾아내고 해당 국가 데이터를 뷰로 넘긴다. 

2. 고객이 선택한 정보를 이후 로딩되는 화면에서 표기해야 하는 경우를 생각해보자.
    - 고객이 게시물 보기 화면에서 필터링을 했다. (ex) 2019.02.25 날짜 데이터만 보기
    - VIEW를 로딩할 때 고객이 선택했던 데이터를 명시를 해줌으로 사용자 친화적인 페이지를 만들 수 있을 것이다.
    - 이때는 Method Argument Level의 @ModelAttribute 에서 처리해 주는게 좋겠다.
    - 필터링 날짜 데이터는 DB select 로직에서 사용될 뿐만 아니라, 이후 VIEW에서도 사용한다.

3. 이처럼 위 2가지 경우를 따로 사용 할 수도 있지만 함께 사용하는 경우도 있다.
    - 고객의 레벨에 따라 처리를 달리 해줘야 하는 경우를 생각해보자.
    - 로그인된 고객의 세션을 값을 쿠키에서 받아와서 고객의 권한을 세팅하는 것은 공통적인 로직이다. (Method level)
    - 고객의 권한에 따라 요청시 데이터를 다르게 가져오는 것은 내부 service 로직을 타야한다. (Method Argument Level)
    - 이때 Method level 에서 데이터를 만들고 Argument Level 에서 데이터를 사용한다.

어떻게 3번째 경우가 가능할까?
둘다 데이터를 넘겨주기 위해서 사용하는 객체는 Model이다.
같은 객체를 사용해서 데이터를 넘겨주기 때문에 응용이 가능한 것이다.

1. Method Level의 @ModelAttribute는 항상 handler보다 먼저 시작하는데
2. 때문에 handler의 Method Argument Level에서 위 Method Level에서 바인딩한 데이터를 받아서 쓰는 것이 가능하다.

```java
@Controller
public class ShoppingController {
  
  @ModelAttribute("ad")
  public List<String> addModelAttributeByMethodReturn() {
      List<String> advertise= new ArrayList<>();
      advertise.add("sale coffee");
      advertise.add("sale books");
      return advertise;
  }

  @GetMapping("/login")
  public String login(@ModelAttribute("vipUser") User user,
                      @ModelAttribute("ad") List<String> advertise) {
      advertise.add("sale pencil");
      return "login";
  }
}
```

![Spring ModelAttribute2](/assets/images/model2.PNG)

또한 2가지 경우 모두 Model 객체를 공유하기 때문에 model을 바로 받아서 쓰는 것도 가능하다.

```java
@Controller
public class ShoppingController {

  @ModelAttribute
  public void addModelAttributeByModelObject(Model model) {
      model.addAttribute("msg", "big sales comming");
  }

  @GetMapping("/login")
  public String login(@ModelAttribute("vipUser") User user, Model model){
      model.addAttribute("date", "until 2020.01.01");
      return "login";
  }
}
```

```html
<!DOCTYPE html>
<html>

<body>
    <h2>id : ${vipUser.id}</h2>
    <h2>name : ${vipUser.name}</h2>

    ${msg}<br>
    ${date}

</body>
</html>
```
![Spring ModelAttribute3](/assets/images/model3.PNG)

위를 살펴보면,
1. Method Level의 @ModelAttribute에서 return 값이 없고, 직접  model을 가져와 쓰고있다.
2. Method Argument Level 에서도 직접  model을 가져와 쓰는 것이 가능하다.

위 2가지 레벨의 호출 순서에 따른 또다른 특징은, 같은 key 값의 객체를 2가지에서 동시에 쓸 경우
1. Method Level의 @ModelAttribute에서 데이터 값을 먼저 설정하고
2. 나머지 값을 Method Argument Level에서 request로 넘어온 값으로 바인딩하여 채울 수 있다.

```java
@Controller
public class ShoppingController {

    @ModelAttribute("vip")
    public User addModelAttributeByMethodReturn() {
        User user= new User();
        user.setAd(Arrays.asList("111", "222", "333"));
        return user;
    }

    @GetMapping("/login")
    public String login(@ModelAttribute("vip") User user){
        return "login";
    }
}
```
```java
@Data
public class User {
    private String id;
    private String name;
    private List<String> ad;
}
```


```html
<!DOCTYPE html>
<html>

<body>
    <h2>id : ${vip.id}</h2>
    <h2>name : ${vip.name}</h2>
    <h2>ad : ${vip.ad}</h2>
</body>
</html>
```
![Spring ModelAttribute4](/assets/images/model4.PNG)


마지막으로 Method Argument Level의 @ModelAttribute는 생략이 가능하다.
@ModelAttribute를 생략 했을 경우 바인딩한 객체의 CLASS명이 key값이 된다.

```java
@Controller
public class ShoppingController {

    @GetMapping("/login")
    public String login(User vip){
        return "login";
    }
}
```

```html
<!DOCTYPE html>
<html>

<body>
    <h2>id : ${user.id}</h2>
    <h2>name : ${user.name}</h2>
</body>
</html>
```

![Spring ModelAttribute5](/assets/images/model5.PNG)