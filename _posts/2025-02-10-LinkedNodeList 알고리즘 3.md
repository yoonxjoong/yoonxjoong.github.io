---
title: LinkedNodeList 알고리즘 (3)
description: 단방향 LinkedList 의 끝에서 k번쨰 노드를 찾는 알고리즘을 구현하시오
author: yoonxjoong
date: 2025-02-10 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
- 단방향 LinkedList 의 끝에서 k번쨰 노드를 찾는 알고리즘을 구현하시오  
	2 -> 3 -> 1 -> 4   일 경우 k번째 노드를 찾는 알고리즘



```java
class Node {  
  
    int data;  
  
    Node next;  
}  
  
class LinkedList {  
  
    Node head;  
  
  
    LinkedList() {  
        head = new Node();  
    }  
  
  
    void append(int d) {  
        Node newNode = new Node();  
        newNode.data = d;  
  
        Node nowNode = head;  
  
        while (nowNode.next != null) {  
            nowNode = nowNode.next;  
        }  
  
        nowNode.next = newNode;  
    }  
  
    void delete(int d) {  
        Node nowNode = head;  
  
        while (nowNode.next != null) {  
  
            if (nowNode.next.data == d) {  
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
}  
  
class Reference {  
    int count = 0;  
  
    public Reference() {  
        count = 0;  
    }  
}  
  

public class test {  
  
    public static void main(String[] args) {  
        LinkedList l1 = new LinkedList();  
        l1.append(9);  
        l1.append(1);  
        l1.append(4);  
        l1.append(6);  
        l1.append(3);  
          
        Reference r = new Reference();  
  
        Node result = find(l1.head.next, 3, r);  
        System.out.println(result.data);  
    }  
  
    public static Node find(Node node, int k, Reference r) {  
        if (node == null) {  
            return null;  
        }  
  
        Node value = find(node.next, k, r);  
        r.count ++;  
        if (r.count == k) {  
            return node;  
        }  
        return value;  
    }  
}
```
