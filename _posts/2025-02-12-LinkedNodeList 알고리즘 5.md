---
title: LinkedNodeList 알고리즘 (5)
description: 주어진 값을 기준으로 작은 수는 왼쪽, 크거나 같은 수는 오른쪽으로 정렬한 노드를 반환하시오.
author: yoonxjoong
date: 2025-02-12 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
리스트 8 -> 5 -> 2 -> 7 -> 3  
주어진 값 : 5
- 기준 값보다 작은 노드의 시작 포인터, 종료 포인터, 기준 값보다 큰 노드의 시작 포인터, 종료 포인터를 설정하여 값을 비교해  
  종료 포인터 노드에 연결해줍니다.  
- 노드가 두개로 분리가 되면 마지막에 기준 값 보다 작은 종료 노드의 next 값을 기준 값보다 큰 노드의 시작 포인터로 연결하여 비교합니다

```java
  
public static Node partition(Node n, int v) {  
    Node s1 = null;  
    Node e1 = null;  
    Node s2 = null;  
    Node e2 = null;  
  
    while (n != null) {  
        Node tmp = n.next;  
        n.next = null;  
  
        if (n.data < v) {  
            if (s1 == null) {  
                s1 = n;  
                e1 = s1;  
            } else {  
                e1.next = n;  
                e1 = n;  
            }  
        } else {  
            if (s2 == null) {  
                s2 = n;  
                e2 = s2;  
            } else {  
                e2.next = n;  
                e2 = n;  
            }  
        }  
  
        n = tmp;  
    }  
  
    if (s1 == null) {  
        return s2;  
    }  
  
    e1.next = s2;  
  
    return s1;  
}  
  
public static void main(String[] args) {  
    System.out.println("hello world");  
  
    LinkedNodeList ll1 = new LinkedNodeList();  
  
    ll1.append(8);  
    ll1.append(5);  
    ll1.append(2);  
    ll1.append(7);  
    ll1.append(3);  
  
    Node result = partition(ll1.head.next, 5);  
  
    System.out.println("exit");  
}
```
