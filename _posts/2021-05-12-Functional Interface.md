---
layout: post 
title: Functional Interface
category: [java]
tags: [java]
redirect_from:

- /2021/05/12/

---

## Functional Interface란 무엇인가? 
Functional Interface는 함수를 객체처럼 다룰 수 있게 해주는 어노테이션으로, 인터페이스에 선언하여 단 하나의 추상 메소드만을 갖도록 제한하는 역할을 한다. 함수형 인터페이스를 사용하는 이유는 Java의 람다식이 함수형 인터페이스를 반환하기 때문이다.  
  
아래와 같은 인터페이스를 함수형 인터페이스라고 한다.
```java
public interface MyFunctionalInterface{
    public abstract void plusOne(int i);
}  
```  

Function Interface를 사용하기 이전에는 아래와 같이 익명 함수를 재정의해서 사용을 했다.
```java
public static void notLambda(){
    System.out.println(new MyFuntionalInterface() {
        @Override
        public int plusOne(int i) {
            return i + 1;
        }
    }.plusOne(3));
}

public static void main(String[] args) {
    notLambda();
}
```
  
하지만 함수형 인터페이스를 사용해서 함수를 변수처럼 선언할 수 있게 되었고, 코드 역시 간결해졌다.  
```java
public static void lambda(){
    MyFuntionalInterface myFunction = (int i) -> i+1;
    System.out.println(myFunction.plusOne(3));
}

public static void main(String[] args) {
    lambda();
}
```  

## Java에서 기본적으로 제공하는 함수형 인터페이스  
- Runnable  
- Supplier  
- Consumer  
- Function<T,R>  
- Predicate  

### Runnable  
Runnable은 파라미터도 없고 리턴결과도 없는 인터페이스이다.
```java
public interface Runnable {
  public abstract void run();
}
```  
Runnable run() 사용예시  
```java
Runnable runnable = () -> System.out.println("This is run method");
runnable.run();
```  

### Supplier  
Supplier<T>는 인자를 받지 않고 T 타입 객체를 리턴한다.
```java
public interface Supplier<T> {
    T get();
}
```  
Supplier get() 사용예시
```java
Supplier<Integer> getFirstIndex = () -> 0;
Integer firstIndex = getFirstIndex.get();
System.out.println(firstIndex);
```  

### Consumer
Consumer<T>는 T타입 객체를 파라미터로 받고 리턴결과는 없다.  
```java
public interface Consumer<T> {
    void accept(T t);

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```  
Consumer<T> accept 사용예시  
```java
Consumer<String> getGender = paramGender -> System.out.println("Gender : " + paramGender);
getGender.accept("man");
```  
Conumer<T> andTHen()을 사용해서 두 개 이상의 Consumer 연속 실행예시
```java
Consumer<String> str1 = msg -> System.out.println(msg + " 첫번째 실행");
Consumer<String> str2 = msg -> System.out.println(msg + " 두번째 실행");
str1.andThen(str2).accept("sisipapa");
```  

### Function  
Function<T, R>은 T타입의 파라미터를 받고, R타입의 객체 결과를 리턴한다.  
```java
public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```  

Function<T, R> 사용예시  
```java
Function<Integer, Integer> plusOne = (value) -> value + 1;
Integer result = plusOne.apply(5);
System.out.println(result);
```  

Function<T, R> compose()는 두개의 Function을 조합하여 새로운 Function 객체를 만들어주는 메소드이다. compose()에 인자로 전달되는 Function이 먼저 수행되고 호출하는 객체의 Function이 나중에 호출된다.
```java
Function<Integer, Integer> minusOne = (value) -> value - 1;
Function<Integer, Integer> plusOne = (value) -> value + 1;

Function<Integer, Integer> composeMethod = minusOne.compose(plusOne);
Integer result = composeMethod.apply(5);
System.out.println(result);
```  

### Predicate  
Predicate<T>는 T타입 인자를 받고 결과로 boolean을 리턴한다.
```java
public interface Predicate<T> {
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```    
Predicate<T> test() 사용예시
```java
Predicate<Integer> isOddNumber = value -> value % 2 != 0;
System.out.println("Is the entered number odd? -> " + isOddNumber.test(3));
```


## Github
<https://github.com/sisipapa/ModernJava.git>

## 참고  
[Java8 - 함수형 인터페이스(Functional Interface) 이해하기](https://codechacha.com/ko/java8-functional-interface/)  
[MangKyu's Diary](https://mangkyu.tistory.com/113)

