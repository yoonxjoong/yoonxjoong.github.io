---
title: LinkedNodeList 알고리즘 (1)
description: 정렬이 되어 있지 않은 링크드 리스트에 중복값을 제거 (버퍼를 사용하지 않고)
author: yoonxjoong
date: 2025-02-06 15:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
정렬되어 있지 않은 링크드리스트의 중복값을 제거하는 알고리즘을 구현해보았습니다.

가장 간단한 방법은 HashSet을 사용하여 구현할수 있지만 추가로 버퍼를 사용하지 않는 조건이 있어 

다음과 같은 방법으로 구현하였습니다.

```java
class LinkedNodeList {  
  
    Node head;  
  
    LinkedNodeList() {  
        head = new Node();  
    }  
  
    static class Node {  
  
        int data;  
  
        Node next = null;  
    }  
  
    void append(int n) {  
        Node newNode = new Node();  
        newNode.data = n;  
  
        Node nowNode = head;  
  
        while (nowNode.next != null) {  
            nowNode = nowNode.next;  
        }  
        nowNode.next = newNode;  
    }  
  
    void delete(int n) {  
        Node nowNode = head;  
  
        while (nowNode.next != null) {  
            if (nowNode.next.data == n) {  
                nowNode.next = nowNode.next.next;  
            } else {  
                nowNode = nowNode.next;  
            }  
        }  
    }  
  
    void retrieve() {  
        Node nowNode = head.next;  
  
        while (nowNode.next != null) {  
            System.out.print(nowNode.data + " -> ");  
            nowNode = nowNode.next;  
        }  
  
        System.out.println(nowNode.data);  
    }  
  
    void removeDup2() {  
        Node s = head.next;  
		while (s != null && s.next != null) {
            Node r = s;  
            while (r.next != null) {  
                if (s.data == r.next.data) {  
                    r.next = r.next.next;  
                } else {  
                    r = r.next;  
                }  
            }  
            s = s.next;  
        }  
    }  
  
    void removeDup() {  
        Node s = head.next;  
        Node e = s;  
  
        if (s.next != null) {  
            e = s.next;  
        }  
  
        while (s.next != null) {  
            while (e.next != null) {  
                if (s.data == e.data) {  
                    s.next = e.next;  
                }  
                e = e.next;  
            }  
            s = s.next;  
        }  
    }  
}  
  
public class test {  
  
    // 정렬이 되어있지 않은 링크드 리스트의 중복값을 제거 > 버퍼를 사용하지 않고  
    public static void main(String[] args) {  
        LinkedNodeList ll = new LinkedNodeList();  
  
        ll.append(2);  
        ll.append(2);  
        ll.append(2);  
        ll.append(4);  
        ll.append(3);  
        ll.append(5);  
  
        ll.retrieve();  
  
        ll.removeDup2();  
  
        ll.retrieve();;  
  
    }  
}
```
시간 복잡도 : O(n^2)
공간 복잡도 : O(1)

