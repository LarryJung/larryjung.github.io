---

layout: post
title:  "이펙티브 자바 Item01 생성자 대신 정적 팩터리 메서드를 고려하라"
date:   2018-11-06 18:00
author: Larry Jung
categories: JAVA

---

# 2018-11-06  

### Effective Java  

**Item 1. 생성자 대신 정적 팩터리 메서드를 고려하라**  

정적 팩터리 메서드가 생성자보다 좋은 장점 *다섯* 가지  

1. 이름을 가질 수 있다.  

   `Optional.of(String string)`    

   `Optional.nullable(String string)`   

2. 호출될 때마다 인스턴스를 새로 생성하지는 않아도 된다.  

   싱글턴, Enum 등을 사용하는 이유와 연관되어 있다.   

3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.  

   이 뜻을 이해하기 위해서는 Collections class 내부의 static method들

   ```java
   // 정적 메서드를 사용함으로써 유틸리티 기능이 있는 구현체(예시로는 불편 컬렉션)를 정확히 알지 못해도(반환 타입이 인터페이스) 객체를 생성할 수 있고, 아래의 예시 말고도 약 50가지에 이르는 유틸성 컬렉션들을 Collections의 정적 메서드만으로 호출할 수 있다는 장점이 있다. 반대로 이러한 유연함을 public 생성자를 사용하여 구현한다고 했을 때는 api를 제공하는(만드는) 입장에서는 약 50여개의 클래스를 한 곳이 아닌 각각의 파일로 관리해야 한다는 단점이 있을 수 있겠다.
   public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c) {
           return new UnmodifiableCollection<>(c);
       }
   ```

   을 살펴볼 필요가 있고, 인터페이스 프로그래밍 관점에서 생각해야 한다.  

4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.   

   쉽게 생각하면, 결과는 객체를 생성하는 것이지만 어떤 객체를 생성할 것인지에 대한 로직이 담긴 메서드로 포장할 수 있다는 뜻이다.  

5. 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.   

   [service provider interface란?](https://blog.seulgi.kim/2014/08/service-provider-interface.html)   

   JDBC 드라이버의 `DriverManager.getConnection(String url)` 을 보면, static method로 되어 있다. DB vendor들의 경우는 자바가 제공하는 service provider framework를 통해 `(DriverManager.registerDriver())` 자사의 db를 등록할 수 있고 사용처는 `DriverManager.getConnection(String url)` 인 것인데, 이는 객체를 생성하는 명령이다. 즉, 이 것은 일반 public 생성자로 대체할 수도 있었겠지만 인터페이스를 반환하는 static factory method 를 사용함으로써 여러 타입의 객체 생성에 대해 유연한 구조를 갖춘 것이다.   



단점   

1. 상속을 하려면 public이나 protected 생성자가 필요하니 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.   
2. 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.   

