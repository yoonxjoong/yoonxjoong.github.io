---
title: 단방향 LinkedList 구현
description: 단방향 LinkedList 구현
author: yoonxjoong
date: 2025-02-04 14:00:00 +0900
categories:
  - Algorithm
tags:
  - java
---
**노드(Node)**:  
각 노드는 두 개의 구성요소를 가집니다.

- **데이터(data)**: 노드가 저장하는 실제 값.
- 다음 노드에 대한 참조(next): 다음 노드를 가리키는 포인터. 마지막 노드의 경우, `null`로 설정됩니다.


```java
class Node {  
    int data;  
  
    Node next;  
  
    Node (int d) {  
        data = d;  
    }  
  
    void append (int d) {  
        // 추가할 노드를 생성합니다.  
        Node newNode = new Node(d);  
  
        // 현재 노드 변수를 선언합니다.  
        Node nowNode = this;  
  
        // 현재 노드의 다음 노드 포인터가 연결되어 있다면  
        while(nowNode.next != null) {  
            // 현재 노드는 다음 노드로 이동  
            nowNode = nowNode.next;  
        }  
  
        // 다음 노드가 없는 현재 노드에 새로운 노드 추가합니다.  
        nowNode.next = newNode;  
    }  
  
    void delete (int d) {  
        Node nowNode = this;  
  
        // 다음 노드가 있다면 반복문 수행  
        while (nowNode.next != null ) {  
            // 다음 노드의 data의 값과 삭제하려는 값이 같으면 다음 노드 포인터에 다다음 노드 포인터를 저장  
            if (nowNode.next.data == d) {  
                nowNode.next = nowNode.next.next;  
            }else {  
                // 삭제하려는 값이 없다면 nowNode 변수를 다음 노드로 이동  
                nowNode = nowNode.next;  
            }  
        }  
    }  
  
    void recursive () {  
        Node nowNode = this;  
  
  
  
        while (nowNode.next != null) {  
            System.out.print(nowNode.data + " -> ");  
            nowNode = nowNode.next;  
        }  
        System.out.println(nowNode.data);  
    }  
}  
  
public class SinglyLinkedList {  
    public static void main(String[] args) {  
  
        Node node = new Node(3);  
        node.append(5);  
        node.append(7);  
        node.append(8);  
  
        node.delete(5);  
  
        node.recursive();  
    }  
}
```