---
title: Stack, Queue 알고리즘 2
description: 정렬되지 않는 스택이 있습니다. 또다른 스택만을 이용해서 정렬을 하시오.
author: yoonxjoong
date: 2025-02-17 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
정렬되지 않는 스택이 있습니다. 또다른 스택만을 이용해서 정렬을 하시오.


``` java
public class test {  
  
    public static void sort(Stack<Integer> s1) {  
        // 다른 스택 생성  
        Stack<Integer> s2 = new Stack<>();  
  
        // 1번째 스택에 있을 경우  
        while(!s1.isEmpty()) {  
            // top 데이터 임시 저장  
            Integer tmp = s1.pop();  
  
            // 2번째 스택이 없고 1번째 top 보다 2번째 top이 큰 경우  
            while(!s2.isEmpty() && s2.peek() > tmp){  
                // 1번째 스택에 저장  
                s1.push(s2.pop());  
            }  
  
            // 그 다음 1번째 top데이터를 2번째 스택에 저장  
            s2.push(tmp);  
        }  
  
        // 2번째 스택에 저장된 값을 1번쨰 스택으로 옮김 (출력을 위해)  
        while(!s2.isEmpty()) {  
            s1.push(s2.pop());  
        }  
    }  
  
    public static void main(String[] args) {  
        System.out.println("hello world");  
  
        Stack<Integer> s1 = new Stack<>();  
  
        s1.push(1);  
        s1.push(5);  
        s1.push(6);  
        s1.push(3);  
  
        sort(s1);  
  
        System.out.println(s1.pop());  
        System.out.println(s1.pop());  
        System.out.println(s1.pop());  
        System.out.println(s1.pop());  
    }  
}
```
