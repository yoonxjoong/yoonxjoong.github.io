---
title: 2개의 스택을 이용해서 큐를 구현하시오.
description: 2개의 스택을 이용해서 큐를 구현하시오.
author: yoonxjoong
date: 2025-02-14 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
2개의 스택을 이용해서 큐를 구현하시오.  
- 새로운 스택에 데이터를 추가합니다.  
- 큐를 구현하기 위해서 새로운 스택의 데이터를 백업 스택으로 뒤집어 쌓고 그 스택에 데이터를 peek 하거나 remove를 할 수 있습니다.  
- peek를 하거나 remove를 하기전에 새로운 스택이 비워져 있다면 다시 쌓는 로직이 추가되어야 합니다.

``` java
class MyQueue<T> {  
  
    Stack<T> stackNewest;  
    Stack<T> stackOldest;  
  
    public MyQueue(){  
        // 새로운 스택 (노드를 저장)  
        stackNewest = new Stack<>();  
  
        // 백업 스택 (새로운 스택에 쌓인 노드를 반대로 쌓게함)  
        stackOldest = new Stack<>();  
    }  
  
    public void add(T data) {  
        // 새로운 스택에 데이터 push        stackNewest.push(data);  
    }  
  
    public void swapStack() {  
        // 백업 스택이 비워져 있다면  
        if(stackOldest.isEmpty()){  
            // 새로운 스택에 있는 노드를 반대로 쌓음  
            while(!stackNewest.isEmpty()) {  
                T data = stackNewest.pop();  
                stackOldest.push(data);  
            }  
        }  
    }  
  
    public T remove() {  
        // 백업 노드에 데이터 쌓음  
        this.swapStack();  
  
        // 백업 노드 데이터 POP        T data = stackOldest.pop();  
        return data;  
    }  
  
    public T peek(){  
        // 백업 노드에 데이터 쌓음  
        this.swapStack();  
  
        // 백업 노드 데이터 POP        return stackOldest.peek();  
    }  
}  
  
public class test {  
  
    public static void main(String[] args) {  
        System.out.println("hello world");  
  
        MyQueue<Integer> myQueue = new MyQueue<>();  
  
        myQueue.add(1);  
        myQueue.add(2);  
        myQueue.add(3);  
        System.out.println(myQueue.peek());  
  
        myQueue.remove();  
  
        System.out.println(myQueue.peek());  
  
        myQueue.add(4);  
  
        myQueue.remove();  
  
        System.out.println(myQueue.peek());  
  
        myQueue.remove();  
  
        System.out.println(myQueue.peek());  
  
        myQueue.remove();  
  
        System.out.println(myQueue.peek());  
  
        System.out.println("Exit");  
    }  
}
```
