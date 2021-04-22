---
layout: post 
title: Modern Java Stream 정리2
category: [Java]
tags: [Java]
redirect_from:

- /2021/04/22/

---

# Modern Java Stream 정리2

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