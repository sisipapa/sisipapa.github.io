---
layout: post 
title: Modern Java Stream 정리2
category: [Java]
tags: [Java]
redirect_from:

- /2021/04/22/

---

## Stream 요소 걸러내기(filter)  
filter는 말 그대로 특정조건으로 스트림의 컨텐츠를 필터링하는 것입니다.  
같은 조건을 filter 하나로 사용, 두개로 사용했을 때의 예제입니다.  
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

## Stream item 변경(map)  
Map은 각각의 item을 변경하여 새로운 컨텐츠를 생성하는 기능입니다.  
```java  
public static void basic3(){
    List<String> list = List.of("a1", "a2", "b1", "b2", "c2", "c1", "c3");
    Stream<String> stream = list.stream();
    stream.map(String::toUpperCase).forEach(System.out::println);
}
```  

## Stream flatMap
FlatMap은 여러개의 스트림을 한개의 스트림으로 합쳐줍니다. 복잡한 스트림을 간단한 스트림으로 변경되는데 사용할 수 있습니다.  
```java  
public static void basic4(){
    String[][] arrays = new String[][]{ {"aa1", "aa2"}, {"bb1", "bb2"}, {"cc1", "cc2", "cc3"} };
    Stream<String[]> stream1 = Arrays.stream(arrays);
    Stream<String> stream2 = stream1.flatMap(s -> Arrays.stream(s));
    stream2.forEach(System.out::println);
}
```


## Stream 정렬(sort)  

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

## Stream filter,map의 활용
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