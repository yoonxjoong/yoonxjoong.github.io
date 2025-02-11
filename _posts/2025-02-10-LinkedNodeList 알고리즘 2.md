---
title: LinkedNodeList 알고리즘 (2)
description: 2개의 링크드 리스트의 제일 마지막 노드부터 합을 구하고 결과를 출력하는 알고리즘을 작성하시오.
author: yoonxjoong
date: 2025-02-10 09:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
2개의 링크드 리스트의 제일 마지막 노드부터 합을 구하고 결과를 출력하는 알고리즘을 작성하시오.

링크드리스트 1 : 2 -> 4 -> 5
링크드리스트 2: 3 -> 5 -> 2

결과 :  5 ->  9  ->  7

이 알고리즘은 링크드리스트의 마지막 노드부터 첫번쨰 노드까지 반복문을 돌면서 두 리스트간의 합을 더하여 새로운 링크드 리스트를 반환하는 알고리즘 입니다.

합의 경우 10 이상이 되면 carry 값 1이 생기고 이 carry 값과 그 다음 노드들과 더해야 합니다.

우선, 마지막 노드까지 재귀 함수 호출을 통해 이동합니다. 

마지막 노드에 도달하면 합을 계산하고 그 계산된 노드를 반환하며 재귀 함수를 빠져나옵니다. 

이때 합이란 두 리스트의 합 + carry 값입니다. 

```java
public class test {  
  
    public static void main(String[] args) {  
        LinkedList l1 = new LinkedList();  
        l1.append(9);  
        l1.append(1);  
        l1.append(4);  
  
        LinkedList l2 = new LinkedList();  
        l2.append(6);  
        l2.append(4);  
        l2.append(3);  
  
        Node result = sumReverseNode(l1.head.next, l2.head.next, 0);  
        while (result.next != null) {  
            System.out.print(result.data + " -> ");  
            result = result.next;  
        }  
  
        System.out.println(result.data);  
    }  
  
    public static Node sumReverseNode(Node n1, Node n2, int carry) {  
        if (n1 == null && n2 == null && carry == 0) {  
            return null;  
        }  

        Node result = new Node();  
        int value = carry;  
  
        if (n1 != null) {  
            value += n1.data;  
        }  
  
        if (n2 != null) {  
            value += n2.data;  
        }  
  
        //  
        result.data = value % 10; // 5,  
  
        if (n1 != null || n2 != null) {  
            Node next = sumReverseNode(  
                    n1 == null ? null : n1.next,  
                    n2 == null ? null : n2.next,  
                    value >= 10 ? 1 : 0  
            );  
  
            result.next = next;  
        }  
  
        return result;  
    }  
  
}

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
```
