---
layout: post
title:  "자바에서의 Lazy evaluation을 고려하자"
date:   2018-10-09 10:11
author: Larry Jung
categories: Java
---

Java8서부터 함수형 패러다임이 도입된 후 스트림이나 Functional interface를 도입하여 개발하는 모습이 자주 보입니다. 오늘 이야기할 내용은 근래 고민하고 있었던 자바 스트림 사용 시 고려해야할 Lazy evaluation. 먼저 간단히 자바에서 Lazy evaluation을 함수형 인터페이스로 구현하는 것으로 시작하겠습니다.

참고:
[Java 8 - Lazy argument evaluation](https://blog.rapid7.com/2017/01/13/java-8-lazy-argument-evaluation/)  
[Are Java 8 streams truly lazy? Not completely!](https://jaxenter.com/java-8-streams-lazy-136183.html)  

Lazy evaluation은 해당 표현식(Expression)의 평가(Evaluation)이 필요한 시점에 평가를 하고, 그 이전까지는 표현식을 평가하지 않는 것이다. (JPA의 Lazy loading과는 다른 개념입니다)  

표현식(Expression) 이라는 것은 함수형 프로그래밍 개념에 익숙하지 않으면 다소 생소할 수도 있는데요, 코드를 보면서 이해하시면 될 것 같습니다.  

```java
public class Main {

    static boolean compute(String str) {
        System.out.println("executing...");
        // expensive computation here
        return str.contains("a");
    }

    static String eagerMatch(boolean b1, boolean b2) {
        return b1 && b2 ? "match" : "incompatible!";
    }

    public static void main(String [] args) {
        System.out.print(eagerMatch(compute("bb"), compute("aa")));
    }

}
```  

위 코드 처럼 `"a"`를 포함 여부에 따라서 true or false를 주는 `compute`메소드가 있고, `eagerMatch` 메소드를 만들어 실행을 해보겠습니다.  

```java
executing...
executing...
incompatible!
```

당연한 결과입니다. `compute("bb"), compute("aa")` 두 가지를 모두 평가한 후 결과를 내어야 하기 때문에 `executing...` 출력이 __두 번__ 일어난 것을 알 수 있습니다.  

## 여기서 Lazy Evaluation이 되는 상황이라면?  

`compute("bb") => false`   
`compute("aa") => true`  
인 상황에서 `&&`오퍼레이터의 경우 먼저 false가 나왔기 때문에, __뒤의 것은 true이건 false이건 계산을 하지 않아도 정답은 false라는 것을 알 수가 있습니다.__  

이런 상황을 ___short circuit___ 이라고 하는데요, 공학에서의 쇼트를 지칭하는 말이죠.  

Lazy evaluation을 하게 되면 결론적으로 `executing...` 프린팅이 단 한번만 실행됩니다. 코드로 확인해보져.  

```java
static String lazyMatch(BooleanSupplier a, BooleanSupplier b) {
        return a.getAsBoolean() && b.getAsBoolean() ? "match" : "incompatible!";
    }
```

자바8에서 제공하는 함수형 인터페이스인 Supplier가 나왔습니다. 서플라이어는 말에서 의미하듯이 공급하는 메소드를 제공하구요, 인자를 받지 않으면서 리턴값이 있는 메소드가 그것입니다. (.getXXX()와 같은)  
여기서는 서플라이어 인터페이스 중 BooleanSupplier을 사용했습니다.  

실행하기 위해서 Main 메소드가 살짝 달라집니다.  

```java
public static void main(String [] args) {
    System.out.print(eagerMatch(compute("bb"), compute("aa")));
    System.out.print(lazyMatch(() -> compute("bb"), () -> compute("aa")));
}
```

함수형 인터페이스의 구현체를 일일이 만들어 대입하지 않고, 익명함수이기 때문에 람다식을 사용하였습니다.  

```
executing...
incompatible!
```  
결과는 예상한대로 executing이 한 번 만 실행됐습니다! 즉 compute("aa")에 대한 평가가 결과를 주는 대 필요하지 않았다는 걸 알 수 있습니다.  

위와 같이 자바에서도 Lazy evaluation적용할 수 있게 되었습니다.  

향후 퍼포먼스를 향상시키는 것에도 고려해볼 수 있겠구요, 제가 Lazy evaluation을 포스팅한 주된 이유는 Stream 사용에 대한 고민때문이었습니다.  

```java
for (int i = 0; i < 20; i++) {
    int n = i * 3; System.out.println(n);
    if (n == 30) return;
}
```  

예를 들어 이와 같은 포문을 굉장히 많이 쓰고 있죠, 이걸 스트림으로 바꾸면 아래와 같을 겁니다.  

```java
IntStream.range(0, 20)
  .map(n -> n * 3)
  .peek(n2 -> System.out.println(n2))
  .filter(nn -> nn == 30)
  .findFirst().getAsInt();
```  

직관적으로 생각해 봤을 때는 map을 하기 때문에 0 -> 57까지 프린팅이 다 된 후에 filter를 적용할 것으로 생각되지만 사실 그렇지 않습니다.(굉장히 놀랍습니다)   

```java
0
3
6
9
12
15
18
21
24
27
30
```  

스트림이라고 하면 그냥 함수형 처럼 사용할 수 있도록 만들어 준 API에 그치는 줄 알았는데, 저러한 부분까지 고려가 되어있었습니다. 실은 함수형 패러다임에서 Lazy evaluation은 굉장히 중요한 요소인데, 그 것을 구현한 것이라고 생각하면 되겠습니다. 스칼라에서도 같은 목적으로 스트림이 있습니다.  

추가적으로 filter와 map 순서를 잘 정하자는 의미로 참고한 두번 째 포스팅을 참고해 주시면 감사하겠습니다. 순서에 따라 의도한 대로 결과가 다르게 나올 수 있습니다.  
