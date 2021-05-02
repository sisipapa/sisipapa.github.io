---
layout: post 
title: Modern Java Stream 정리3
category: [Java]
tags: [Java]
redirect_from:

- /2021/05/02/

---

## Optional 생성 및 내부객체 접근
Optional은 제네릭 클래스로 객체를 감싸는 래퍼 클래스이다.  
Optional 타입의 객체에는 모든 타입의 참조변수를 담을 수있다.   
Optional 클래스는 null 값을 처리할 때 많이 활용된다.  

Optional.of를 사용해서 String의 래퍼클래스로 생성하고 Optional.get을 사용해서 Optional객체 내부의 String객체에 접근한다.  
```java  
public static void basic1(){
    String value = "This is Optional";
    Optional<String> optionalStr = Optional.of(value);
    System.out.println("basic1 result : " + optionalStr.get());
}
```  
  
## null을 포함하는 Optional 객체 생성  
Optional.of()는 null이 아닌 객체만 사용하다.  
null값을 포함하는 객체를 Optional 객체를 래퍼클래스로 생성하기 위해서는 아래 두가지 방법이 있다.     
```java  
public static void basic2(){
    // 방법1 - Optional.ofNullable
    String nullValue = null;
    Optional<String> nullOptionalStr = Optional.ofNullable(nullValue);
    try{
        System.out.println(nullOptionalStr.get());
    }catch(Exception e){
        e.printStackTrace();
    }
}
```  
```java  
public static void basic3(){
    // 방법2 - Option.empty
    Optional<String> nullOptionalStr = Optional.empty();
    try{
        System.out.println(nullOptionalStr.get());
    }catch(Exception e){
        e.printStackTrace();
    }
}
```    
  
## isPresent(),ifPresent() 내부 객체 존재 여부확인  
```java  
public static void basic4(){
    Optional<String> optionalStr = Optional.of("This is not null String");
    Optional<String> nullOptionalStr = Optional.ofNullable(null);

    if (optionalStr.isPresent()) {
        System.out.println("optionalStr: " + optionalStr.get());
    }
    optionalStr.ifPresent(str -> System.out.println("optionalStr[ifPresent] : " + optionalStr.get()));

    if (nullOptionalStr.isPresent()) {
        System.out.println("nullOptionalStr[isPresent]: " + nullOptionalStr.get());
    }
    nullOptionalStr.ifPresent(str -> System.out.println("nullOptionalStr[ifPresent] : " + nullOptionalStr.get()));
}
```

## orElse - Optional이 null인 경우 orElse()의 param을 리턴  
```java  
public static void basic5(){
    Optional<String> optionalStr = Optional.of("This is not null String");
    Optional<String> nullOptionalStr = Optional.ofNullable(null);

    String optionalStr2 = optionalStr.orElse("optionalStr 대체 String 입니다.");
    String nullOptionalStr2 = nullOptionalStr.orElse("nullOptionalStr 대체 String 입니다.");

    System.out.println("optionalStr2 : " + optionalStr2);           // This is not null String
    System.out.println("nullOptionalStr2 : " + nullOptionalStr2);   // nullOptionalStr 대체 String 입니다.
}
```

## orElseGet - Optional이 null인 경우 특정 함수를 실행하고 그 결과를 리턴
```java  
public static void basic6(){
    Optional<String> optionalStr = Optional.of("This is not null String");
    Optional<String> nullOptionalStr = Optional.ofNullable(null);

    String optionalStr2 = optionalStr.orElseGet(() -> "[orElseGet] This is not null String");
    String nullOptionalStr2 = nullOptionalStr.orElseGet(() -> "[orElseGet] This is null String");

    System.out.println("optionalStr2 : " + optionalStr2);// This is not null String
    System.out.println("nullOptionalStr2 : " + nullOptionalStr2); // [orElseGet] This is null String
}
```  

## orElseThrow - Optional이 null인 경우 예외를 던진다.
```java  
public static void basic7(){
    Optional<String> optionalStr = Optional.of("This is not null String");
    Optional<String> nullOptionalStr = Optional.ofNullable(null);

    try{
        String optionalStr2 = optionalStr.orElseThrow(() -> {throw new NullPointerException();});
        System.out.println("optionalStr2 : " + optionalStr2);
    }catch(NullPointerException e){
        System.err.println("NullPointerException");
    }

    try{
        String nullOptionalStr2 = nullOptionalStr.orElseThrow(() -> {throw new NullPointerException();});
        System.out.println("nullOptionalStr2 : " + nullOptionalStr2);
    }catch(NullPointerException e){
        System.err.println("NullPointerException");
    }
}
```

## Github
<https://github.com/sisipapa/ModernJava.git>  

## 참고  
[Java8의 Optional 사용 방법 및 예제](https://codechacha.com/ko/java8-stream-optional/)  
[자바의 정석 - 스트림(Stream)](https://ryan-han.com/post/java/java-stream/)  
