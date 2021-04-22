---
layout: post 
title: Modern Java Stream 정리
category: [Java]
tags: [Java]
redirect_from:

- /2021/04/22/

---

# Modern Java Stream 정리

## Stream  
자바의 컬렉션 클래스에는 같은 기능의 메서드들이 중복해서 정의되어 있다. List를 정렬할 때는 Collection.sort()를 사용하고, 배열을 정렬할 때는 Arrays.sort()를 사용한다.    
이렇게 Collection 마다 다른 방식으로 다루어야하는 문제점을 해결해주는 것이 Stream 이다.  
  
1. Stream 만들기
```java
public static void streamTest1(){
        // Arrays.asList(String[] => List<String>)
        String[] strArr = {"bbb", "ddd", "aaa", "ccc"};
        List<String> strList = Arrays.asList(strArr);

        Stream<String> strStream1 = strList.stream();       // List => Stream
        Stream<String> strStream2 = Arrays.stream(strArr);  // Arrays => Stream

        strStream1.sorted().forEach(System.out::println);
        strStream2.sorted(Comparator.reverseOrder()).forEach(System.out::println);
    }
```

2. 특정 범위의 정수 Stream
요소의 타입이 T인 스트림은 오토박싱/언박싱의 비효율을 줄이기 위해 데이터 소스의 요소를 기본형으로 다루는 InsStream, LongStream, DoubleStream이 제공된다.
```java
public static void streamTest2(){
    System.out.println("###range");
    IntStream.range(0, 10).forEach(num -> System.out.print(" " + num));
    System.out.println("");
    System.out.println("###rangeClosed");
    IntStream.rangeClosed(0, 10).forEach(num -> System.out.print(" " + num));
}
```

3. iterate(), generate()
Stream 클래스의 iterate()와 generate()는 람다식을 매개변수로 받아서, 이 람다식에 의해 계산되는 값들을 요소로 하는 무한 스트림을 생성한다. 아래 예제는 무한스트림이 되지 않도록 limit를 지정했다.  
```java
public static void streamTest3(){
    Stream<Integer> evenStream = Stream.iterate(0, n->n+2);
    evenStream.limit(5).forEach(System.out::println);
}

public static void streamTest4(){
    Stream<Double> randomStream = Stream.generate(Math::random);
    randomStream.limit(5).forEach(System.out::println);
}
```

4. Stream 연결 및 중복제거
Stream의 static 메소드인 concat()을 사용해서 두 스트림을 하나로 연결하고 distinct()를 사용해서 중복을 제거한다.
```java
public static void streamTest5(){
    String[] str1 = {"1", "2", "3", "1", "3"};
    String[] str2 = {"A", "B", "C", "B", "Z", "D"};

    Stream<String> strs1 = Stream.of(str1);
    Stream<String> strs2 = Stream.of(str2);
    Stream<String> strs3 = Stream.concat(strs1, strs2);   

    System.out.println("Stream 연결 중복제거");
    strs3.distinct().forEach(System.out::println);

}
```


## Github
<https://github.com/sisipapa/ModernJava.git>