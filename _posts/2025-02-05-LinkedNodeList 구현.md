---
title: LinkedNodeList 구현
description: LinkedNodeList 구현
author: yoonxjoong
date: 2025-02-05 14:00:00 +0900
categories:
  - Algorithm
tags:
  - java
---
이전시간에 Singly LinkedList를 구현해보았는데요. 

SinglyLinkedList에서는 첫번째 노드가 Header 노드이면서 첫번째 노드로 코드를 작성했어요.

그렇게 되면 첫번째 노드를 삭제하려고 할때 에러가 발생할거에요.
왜냐하면 LinkedList의 시작이 첫번째 노드인데 이 노드를 지워버리면 2번째 노드를 불러올수 없거든요 

다음과 같이 헤더 노드를 추가하고 그 클래스 안에 노드 클래스와 변수를 선언 한뒤에,
헤더 노드로 시작한 리스트를 구현해볼거에요

여기서 헤더노드는 단순히 시작을 가르키는 노드라고 생각하면 쉬울거에요.

```java  
/**  
 * 링크드 노드 리스트 구현  
 */  
  
class LinkedNodeList {
	// 헤더 노드
    Node header;  

	// 링크드 노드 리스트 생성자
    LinkedNodeList() {  
        header = new Node();  
    }  

	// 노드 클래스
	static class Node {  
        int data;  
  
        Node next = null;  
		
        Node () {  
        }  
        
        Node (int d) {  
            data = d;  
            next = null;  
        }  
    }  

	// 노드 삽입
    void append (int d) {  
	    // 추가될 노드 생성
        Node newNode = new Node(d);  

		// 시작 노드 할당
        Node nowNode = header;  

		// 제일 마지막 노드까지 반복문
        while (nowNode.next != null) {  
            nowNode = nowNode.next;  
        }  

		// 마지막 노드에 새로운 노드 연결
        nowNode.next = newNode;  
    }  

	// 노드 삭제
    void delete (int d) {
        // 시작 노드 할당
        Node nowNode = header;  

		// 마지막 노드까지 반복문
        while (nowNode.next != null) {  
		    // 다음 노드의 data와 d가 일치하는지 확인
            if (nowNode.next.data == d) {  
	            // 일치한다면 그 다음 노드의 포인터를 저장
                nowNode.next = nowNode.next.next;  
            }else{  
	            // 일치하지 않는다면 다음 노드로 이동
                nowNode = nowNode.next;  
            }  
        }  
    }  
  
    void retrieve () {  
	    // 헤더는 시작을 알리는 노드로 다음 노드부터 반복문 실행
        Node nowNode = header.next;  
        while (nowNode.next != null ) {  
            System.out.print(nowNode.data + " -> ");  
            nowNode = nowNode.next;  
        }  
  
        System.out.println(nowNode.data);  
    }  
}  
  
  
  
public class test {  
    public static void main(String[] args) {  
        LinkedNodeList linkedNodeList = new LinkedNodeList();  
        linkedNodeList.append(1);  
        linkedNodeList.append(2);  
        linkedNodeList.append(3);  
        linkedNodeList.append(4);  
  
        linkedNodeList.retrieve();  
        // 1 -> 2 -> 3 -> 4
  
        linkedNodeList.delete(1);  
  
        linkedNodeList.retrieve();  
        // 2 -> 3 -> 4
    }  
}
```