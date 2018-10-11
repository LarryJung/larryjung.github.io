---
layout: post
title:  "[번역] Monads for Java developers: Part 1 — The Optional monad"
date: 2018-10-10 22:42:50
author: Larry Jung
category: Java
---

이 글은 번역글입니다. (번역 수준이 다소 낮다고 생각합니다.. )    
원문 : [Monads for Java developers: Part 1 — The Optional monad](https://medium.com/@afcastano/monads-for-java-developers-part-1-the-optional-monad-aa6e797b8a6e)  

__abstract__ : 모나드는 간단하지만 강력한, 코드의 구조를 개선시킬 수 있으면서 때로는 원치 않는 부수 효과(side effect)를 다루는 데 도움을 줄 수 있는 타입이다. 이번 시리즈에서는 자바 예제를 이용하여 모나드를 설명하고 자바 exception의 대안으로써 Result Monad의 간략히 살펴볼 것이다.   

### What is a monad?  
기술적으로, 모나드는 parameterised type 이다. 자바의 옵셔널과 스트림 같은 것인데 이는:   
- flatMap(a.k.a bind) 와 unit(a.k.a identity, return, Optional.of(), etc...). 메소드가 구현되어 있다.  
- 다음의 세 가지 법칙을 만족한다: 좌항등원, 우항등원이 존재하고, 결합법칙을 만족한다(본 포스트의 수준을 벗어나지만)  

---
용어 정리를 위해서, Optional<String> 과 같은 parameterised type 은 타입 파라미터를 갖는다고 하자: String과 Wrapper type: 이 경우에는 " ***String 타입이 Optional context에 의해 감싸져(wrapped)있다.*** "  

---

### How are monads useful?  
모나드는 모든 parameterised type들의 조합(구성, composition)에 대한 것이다. Optional모나드를 예로 들자. 우리는 파라미터를 적절히 다룰 수 있으려면 Optional 이 empty인지 아닌지에 대한 확인이 필요하단 것을 알고 있다. 또한 Empty일 특정 조치를 취하는 것도 좋은 행위이다. 이제 전반적인 프로세스에 대해 "Optional을 사용하는 환경으로부터 파라미터를 unwrapping하는 것"이라고 부를 것이다.  

코드를 적어보자. 첫 째, two Optional numbers를 더한다. 만약 한 쪽이 empty라면 결과적으로 empty를 반환하고 반대의 경우에는 두 값을 더한 Optional을 반환한다.  

```java
public Optional<Integer> optionalAdd(Optional<Integer> val1, Optional<Integer> val2) {
    if(val1.isPresent() && val2.isPresent()) {
        return Optional.of(val1.get() + val2.get());
    }

    return Optional.empty();
}
```

여기서 우리가 Integer를 Optional에서 꺼내오기 위해 숫자의 존재 여부를 확인하고 empty case에서 수동적으로 처리해 줘야 했다는 것을 확인해라.  

이 부분이 정확히 모나드가 우리에게 도움을 줄 수 있는 부분이다:
__To avoid dealing with the context when composing parameterised types__ (parameterised type(모나드)을 조합하는 경우 context(ex. optional에서 present여부를 확인하는 등의 고려해야할 상황 및 환경 맥락 등으로 이해하고 있습니다.)를 고려하는 것을 회피하는 것).  
다음 예제에서는 Optional addition을 조합하는 경우에 대해서 다시 살펴본다. 즉, Optional monad는 adding Optional types할 때 context를 고려하지 않아도 되게끔 도와준다는 것이다.  

flatMap과 Optional.of이 그 것을 도와준다. flatMap 메소드는 모나드의 파라미터를 취하면서 같은 타입의 또 다른 모나드를 생성하는 연산을 수행한다. 이러한 특성 때문에 파리미터들을 조합하기 위해서 각기 다른 모나드에 nest flatMap을 여러번 수행할 수 있게 된다. 또한 Optional.of은 굉장히 사용하기 쉬운 메소드이고 그 것은 어떤 값이든 취해서 다른 Optional을 생성해 준다. 코드를 보자.  

```java
public Optional<Integer> optionalAdd(Optional<Integer> val1, Optional<Integer> val2) {
    return
           val1.flatMap( first ->
           val2.flatMap( second ->
           Optional.of(first + second)
    ));
}
```

---
1 — Bear with me with the weird indentation, it’s for didactical purposes. I will present an alternative later in this post.  
2 — The flatMap method receives a Function. We’ll call this function the “__mapper__” function.  
---


어떻게 value의 present, empty 여부를 체크하지 않아도 됐는지 알아야 한다. Optional모나드는 flatMap 메소드를 통해서 적절하게 감싸진 내부 파라미터를 꺼내준다. 첫 째의 Optional이 empty일 경우에는 두 번 째 Optional의 flatMap을 실행하지 않는다. 사실, Optional 조합 연산(composition) 안에 하나의 empty라도 있으면 결과는 empty Optional이다.  

또한 어떤 방식으로 "val.flatMap"이 "val.flatMap" 의 mapper function속으로 들어가게 되었는지 확인해보자. 이는 메인 연산이었던 "first + second" 가 두 개의 다른 Optional속의 두 파라미터 값을 필요로 했기 때문이다. nested Optional이었던 val2의 mapper function은 스코프 안에서 두 파라미터값들을 가지게 될 것이다.  

비록 Optional 모나드의 경우 매우 간단한 케이스였지만, 이 것은 단순히 syntax sugar에 그치지 않는다. 이러한 방법은 실제로 모나드의 타입 파라미터의 존재 여부를 확인해야 하는 필요 없이도 연산을 다룰 수 있게 해준다. 우리는 empty case를 가지고 뭘 해야할 지 생각하지 않았다. 그리고 어떻게 모나드 내부의 값을 가져와야 하는지에 대한 생각도, empty Optional 내부의 값을 빼올 때 무슨 일이 일어날 지에 대한 생각도 하지 않았다. This is knowledge that optionalAdd method doesn’t need to have to be able to compose the monads. 이러한 속성은 그리 간단하지 않은 모나드 파라미터를 unwrapping할 때 더 중요해진다. 다음 포스트의 Result monad를 다룰 때 다시 강조하게 될 것이다.  

한편, 또 다른 Optional example 를 보자. 우리는 nested Optional을 다루는 것이 다소 까다로울 수 있다는 것을 알고 있다. 다음을 보자.  

```java
private abstract class Counter {
    abstract Optional<Integer> colourCount();
}
private abstract Optional<Counter> fetchThisMonthCounter();
private abstract Optional<Counter> fetchPreviousMonthCounter();
// ************ blah blah blah
// ...
public Optional<Integer> totalColourCount() {

    Optional<Counter> thisMonth = fetchThisMonthCounter();
    Optional<Counter> previousMonth = fetchPreviousMonthCounter();

    if(thisMonth.isPresent() && thisMonth.get().colourCount().isPresent()
        && previousMonth.isPresent() && previousMonth.get().colourCount().isPresent()) {

        Integer colour1 = thisMonth.get().colourCount().get();
        Integer colour2 = previousMonth.get().colourCount().get();
        return Optional.of(colour1 + colour2);
    }

    return Optional.empty();
}
```  

모나드 Optional<Counter>을 리턴하는 두 가지 메소드가 있고, 목표는 total 두 Counter의 값을 사용하여 총합 colourCount을 구하는 것이다.  

Optional<Counter>의 속성에 접근하기 위해 unwrap과정이 필요하다는 것을 확인할 수 있다.  

flatMap 메소드와 java8의 문법을 사용하게 되면 Optional context를 다루는 부분을 피할 수가 있다.  

```java
private abstract Optional<Counter> fetchThisMonthCounts();
private abstract Optional<Counter> fetchPreviousMonthCounts();

public Optional<Integer> totalColour() {

    return fetchThisMonthCounts().flatMap(Counter::colourCount).flatMap(colour1 ->
           fetchPreviousMonthCounts().flatMap(Counter::colourCount).flatMap(colour2 ->
           Optional.of(colour1 + colour2)
    ));
}
```

다시 한번 Optional 모나드는 context를 다루는 부분에 있어서 도움을 준다. Counter의 colourCount 모나드를 꺼내오기 위해서 어떻게 flatMap을 사용했는지 확인하고(fetchThisMonthCounts().flatMap(Counter::colourCount)), 모나드를 조합하기 위해서 nested Optional을 어떻게 사용했는지 확인해 보라.  

게다가, `fetchPreviousMonthCounts()`메소드는 `fetchThisMonthCounts()`가 non empty일 때에만 호출된다. 우리는 flatMap을 사용한 조합 연산을 통한 방법으로부터 이러한 좋은 특징을 얻게 되었다.  

### Wrapping it up  
모나드란 어떤 조합에 관한 일반적인 방법을 제공하며 그 안에서 context의 조작에 관한 부분들을 모나드 스스로 처리하여 준다. 이 것은 앞서 모나드 법칙을 만족하는 모든 parametrised type에 대해서는 같은 방식으로 일관된다.  

자바에서는 Optional이 모든 모나드 법칙들을 만족하고 있으며 flatMap의 체이닝 연산을 통해서 아주 훌륭하게 연산을 조합할 수가 있다. Optional과 같이, 자바는 다른 monadic type들을 제공하고 있는데 stream이나 CompletableFuture 등이 그런 것이다. 이들 또한 Optional와 같이 모나드 법칙들을 만족하며 조합에 있어서도 동일하게 취급한다.  

다음 글에서는 monad context를 고려하지 않는 좀 더 구체적인 사례 Result monad에 대해서 살펴볼 것이다. 또한 그 것이 자바의 checked exception의 대안으로써 어떻게 사용될 수 있을 지도 생각해보고, side-effects-free한 logging에 관해서도 flatMap 메소드를 수정함으로써 보여줄 것이다.  

### An alternative to nested flatMaps for composing Optionals
If you prefer to avoid nesting flatMap calls when composing monads, there are utility functions that can do it for you. You can find plenty of libraries for that[4], however, implementing these functions is not complicated at all:  

```java
public static <T, U, R> Optional<R> flatMap2(Optional<T> opt1,
                                             Optional<U> opt2,
                                             BiFunction<T, U, Optional<R>> mapper) {
    return
            opt1.flatMap(a ->
            opt2.flatMap(b ->
            mapper.apply(a, b)
    ));
}
```  

The flatMap2 function takes two Optionals and a BiFunction which in turn, receives the two Optional parameters and returns another Optional. Here, we just need to compose the Optionals as we have been doing it so far. With this function, the caller won’t need to nest the flatMaps when operating with the Optionals:  

```java
public Optional<Integer> addOptionals(Optional<Integer> opt1, Optional<Integer> opt2) {
    return flatMap2(opt1, opt2, (param1, param2) -> Optional.of(param1 + param2));
}
```  

Using a similar implementation, we can have flatMap3, flatMap4 and so on. Note that this approach will preserve all of the benefits of Monadic composition that we have discussed in this post.
The full implementation can be found here and tests showing how to use it are in here.  
