---
layout: post
title:  "180818 JPA 준영속성"
date:   2018-08-18 21:36
author: Larry Jung
categories: JPA
---

# JPA 엔티티 생명주기 중 하나인 준영속(detached) 상태에 대해 이야기해보고자 한다.   

## 엔티티매니저가 관리하는 엔티티 상태는 아래와 같이 4가지 상태로 나뉜다.  
![1](/assets/img/180818_jpa_detached/1.png)  

## Detached(준영속) 상태란 한번 영속화 된 엔티티 즉, id값을 가지고 있지만 영속성 컨텍스트를 벗어난 상태를 말한다.  

영속성 컨텍스트 벗어난 엔티티를 가지고 로직을 짤 때, 영속성 컨텍스트에서 해 줄 수 있는 몇 가지 기능을 활용할 수 없다.  

1. Insert, Update불가!  
2. 지연 로딩 불가!  

앞으로 이야기할 JPA에 대한 내용은 Spring DATA JPA로 한정해서 설명하겠다.  
Spring DATA JPA는 hibernate를 조금 더 추상화한 접근으로 사용할 수 있도록 spring에서 제공하는 서브 프로젝트 중 하나이다. 그냥 hibernate만 사용할 때보다 비침투적인 설계를 하도록 도와준다.  

가장 큰 특징은 repository 인터페이스의 구현체가 Spring의 자체 @Transactional이 기본으로 설정되어 있는데, 이 뜻은 repository의 메서드를 실행할 때 **영속성 컨텍스트 생존 범위 = 트랜잭션 범위** 를 기본으로 한다는 뜻이다.    

Service계층에 @Transactional 을 붙인다면, 영속성 컨텍스트는 해당 계층까지 유지된다는 것이다. 반대로 Controller계층에서 Service계층을 통해 반환된 값은 준영속 상태라는 것을 알아야 한다.  

## 준영속 상태의 어떤 특징을 주의해야 할까?  

**1. Dirty checking을 할 수 없다.**  
- dirty checking이란 영속화된 엔티티에 대해서 스냅샷으로 변경 사항을 비교하는 것을 말하는 데, 영속 상태의 엔티티 값의 변동이 여러번 일어났지만 commit시점에 엔티티가 초기 상태와 똑같았다면 SQL을 날리지 않는다. 준영속 상태인 엔티티는 값의 변동이 일어났어도 다시 flush 하지 않는 이상 값의 변동이 일어나지 않는다.

아래 예제를 보면, Repository에서 찾은 Account는 더 이상 영속 상태가 아니다. 따라서 값의 변경이 생겨도 영속성 컨텍스트가 dirty checking을 하지 않기 때문에 꼭 repository.save()를 해 주어야 한다.

```java
public class AccountService {

...
  // @Transactional 을 붙이면 save()를 하지 않아도 된다.
  public void updateAccount(Long id, String password) {
    Account account = repository.findById(id);
    Account updateAccount = account.updatePassword(password);
    repository.save(updateAccount);
  }

...

}
```

**2. Lazy Loading을 할 수 없다.**  
- 지연로딩(lazy loading)은 JPA성능 관리에 있어서 큰 부분을 차지하는데, 아래 Account가 Item을 여러 개 가지고 있을 때 Account를 메모리에 로딩하는 순간 Item객체들을 프록시 형태로 로딩해 놓고, Item객체를 직접 사용하는 시점에 메모리에 로딩(초기화)하는 것을 말한다.

```java
public class Account {

  ...
  @OneToMany
  private List<Item> items;

  ...
}

```

따라서 지연 로딩은 OneToMany관계의 디폴트가 FetchType.LAZY로 설정되어 있을 만큼 꼭 알고 넘어가야 할 하이버네이트의 특징이다.
지연로딩은 영속성 컨텍스트의 기능이므로 준영속 상태의 Account가 `account.getItems()` 를 수행하는 순간
`org.hibernate.LazyInitializationException` 이 발생한다.  

***어디서 이런 문제들이 발생할 수 있을까?***  
컨트롤러나 뷰에서 문제가 생길 수 있고, 말고도 테스트 코드를 작성할 때 빈번히 발생할 수 있다.  
코드를 보고 생각해보자.  

***!!참고!!***  
스프링 부트를 사용하고 있다면 application.properties에 우선 설정을 하나 해주자.
`spring.jpa.open-in-view=false`  

[What is this spring.jpa.open-in-view=true property in Spring Boot?](https://stackoverflow.com/questions/30549489/what-is-this-spring-jpa-open-in-view-true-property-in-spring-boot)  
[Spring - Open Session In View](http://kingbbode.tistory.com/27)    

---

### 지연로딩이 발생하는 상황을 좀 더 살펴보자.  

1. 컨트롤러나 뷰 단에서 자식 객체들 접근  

```java
@RestController
public class AccountController {

...

  @GetMapping("/accounts/{id}")
  public List<Item> retrieveItemsByAccountId(@PathVariable Long id) {
    Account account = accountService.findById(id); // 준영속 상태로 바뀜
    return account.getItems(); // LazyInitializationException 발생!
  }

...

}

```  

2. 테스트 할 때  

```java

@Test
    public void create() {
        Account dbAccount = restTemplate.getForObject("/accounts/1", Account.class);
        System.out.println(dbAccount.toString());
    }
```

```
2018-08-19 19:46:02.816  WARN 5744 --- [o-auto-1-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver : Failed to write HTTP message: org.springframework.http.converter.HttpMessageNotWritableException: Could not write JSON: failed to lazily initialize a collection of role: com.larry.studyjpa.domain.entities.Account.items, could not initialize proxy - no Session; nested exception is com.fasterxml.jackson.databind.JsonMappingException: failed to lazily initialize a collection of role: com.larry.studyjpa.domain.entities.Account.items, could not initialize proxy - no Session (through reference chain: com.larry.studyjpa.domain.entities.Account["items"])   
Account{id=null, name='null', items=null}
```

준영속 상태의 객체의 필드 접근 시 예외가 발생하는 것을 알 수 있다.  

### 해결법  
여러가지 해결이 있지만, 아직 제대로 사용해 본 것들이 하나도 없는 관계로(ㅠㅠ) 일단 내가 생각했을 때 가장 우선순위로 두고 싶은 것은
1. FetchType.LAZY  
2. Open session in view 기능을 사용하여 트랜잭션을 service layer에서 종료시키고 뷰 단에서는 영속성 컨텍스트를 유지하는 방법이다.  
3. fetch join을 사용  


FetchType.EAGER를 설정하는 글로벌 페치 전략은 1+N 문제가 발생하기 때문에 매우 비 추천하는 방법이다. 일단 ManyToOne관계라도 Lazy로 설정한 다음에 문제가 될 경우에 다른 성능 최적화 전략을 우선 고려해 보는 것이 좋겠다.  
