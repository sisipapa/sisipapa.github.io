---
layout: post 
title: Modern Java Stream 정리2
category: [Java]
tags: [Java]
redirect_from:

- /2021/04/22/

---

# Modern Java Stream 정리2

## 1. Stream 요소 걸러내기(filter)  
같은 조건을 filter 하나로 사용, 두개로 사용했을 때의 예제이다.  
```java
public static void basic1(){
    // 하나의 filter를 사용
    IntStream intStream1 = IntStream.rangeClosed(1, 10);
    intStream1.filter(i -> i%2!=0 && i%3!=0).forEach(System.out::println);

    // 두개의 filter를 사용
    IntStream intStream2 = IntStream.rangeClosed(1, 10);
    intStream2.filter(i -> i%2!=0).filter(i -> i%3!=0).forEach(System.out::println);
}
```

## 2. Stream 정렬(sort)  

```java  
  
public static void basic2(){
    System.out.println("===기본정렬");
    Stream<String> strStream1 = Stream.of("FFFFFFF", "ccc", "A", "BB", "eeeee", "dddd");
    strStream1.sorted().forEach(System.out::print);
    System.out.println("\n===============================================================================");

    System.out.println("###기본정렬 역순");
    Stream<String> strStream2 = Stream.of("FFFFFFF", "ccc", "A", "BB", "eeeee", "dddd");
    strStream2.sorted(Comparator.reverseOrder()).forEach(System.out::print);
    System.out.println("\n===============================================================================");

    System.out.println("###대소문자 구분없이");
    Stream<String> strStream3 = Stream.of("FFFFFFF", "ccc", "A", "BB", "eeeee", "dddd");
    strStream3.sorted(String.CASE_INSENSITIVE_ORDER).forEach(System.out::print);
    System.out.println("\n===============================================================================");

    System.out.println("###길이 정렬");
    Stream<String> strStream4 = Stream.of("FFFFFFF", "ccc", "A", "BB", "eeeee", "dddd");
    strStream4.sorted(Comparator.comparing(String::length)).forEach(System.out::print);
    System.out.println("\n===============================================================================");
}
```

## 2. Stream filter,map의 활용
반복문과 제어문으로 된 프로그램을 Stream,filter,map 을 활용해서 변경한 예제입니다.  
```java

/**
 * 반복문/제어문을 활용
 */
public static void method1(){
    Integer result = null;
    for (final Integer number : NUMBERS) {
        if (number > 3 && number < 9) {
            final Integer newNumber = number * 2;
            if (newNumber > 10) {
                result = newNumber;
                break;
            }
        }
    }
    System.out.println("\n==================================");
    System.out.println("반복문 제어문의 결과: " + result);
}

/**
 * Stream lamda1
 */
public static void method2(){
    System.out.println("\n==================================");
    System.out.println("Stream filter와 map을 활용한 결과 lamda1 : " +
            NUMBERS.stream()
                    .filter(number -> number > 3)
                    .filter(number -> number < 9)
                    .map(number -> number * 2)
                    .filter(number -> number > 10)
                    .findFirst()
    );
}

/**
 * Stream lamda2
 */
public static void method3(){
    System.out.println("\n==================================");
    System.out.println("Stream filter와 map을 활용한 결과 lamda2 : " +
            NUMBERS.stream()
                .filter(number -> {
                return number > 3;
                })
                .filter(number -> {
                return number < 9;
                })
                .map(number -> {
                return number * 2;
                })
                .filter(number -> {
                return number > 10;
                })
                .findFirst()
    );
    System.out.println("\n==================================");
}
```


## Github
<https://github.com/sisipapa/ModernJava.git>