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

Function Interface를 사용하기 이전에는 익명 함수를 재정의해서 사용
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
  
함수형 인터페이스를 사용해서 함수를 변수처럼 선언할 수 있게 되었고, 코드 역시 간결해졌다.  
```java
public static void lambda(){
    MyFuntionalInterface myFunction = (int i) -> i+1;
    System.out.println(myFunction.plusOne(3));
}

public static void main(String[] args) {
    lambda();
}
```


## 참고  
[Java8 - 함수형 인터페이스(Functional Interface) 이해하기](https://codechacha.com/ko/java8-functional-interface/)  
[MangKyu's Diary](https://mangkyu.tistory.com/113)

