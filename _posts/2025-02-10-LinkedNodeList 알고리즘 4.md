---
title: LinkedNodeList 알고리즘 (4)
description: 주어진 두개의 단방향 Linked List에서 교차점 노드를 찾으시오
author: yoonxjoong
date: 2025-02-11 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
주어진 두개의 단방향 Linked List에서 교차점 노드를 찾으시오
(단 값이 아닌 주소로 찾아야함)  
- list 1 : 5 -> 7 -> 9 -> 10 -> 7 -> 6  
- list 2 : 6 -> 8 -> 10 -> 7 -> 6

풀이방법 1.
- 교차점 노드를 찾으려면 마지막 노드를 맞춰 앞쪽으로 차례로 이동하면서 서로 일치한지 확인하고 달라지면 다를 경우 교차점 노드를 확인할수 있다

- 길이가 다른 노드를 서로 맞추기는 알고리즘으로 구현이 어려웠습니다. 마지막 노드의 자리수를 맞추려면 길이가 짧은 리스트의 마지막 노드를 계속 연결하여 비교를 했지만 일치 여부의 비교문이 복잡해서 다른 방법을 생각해봤습니다. 

풀이방법 2.
- 두개의 리스트의 길이가 다르므로 서로의 리스트 길이를 맞추고 비교를 해보는 방향으로 생각했습니다. 
- 길이가 짧은 노드의 길이부터 반복문을 돌면 교차점을 쉽게 찾을수 있었습니다.
- 짧은 리스트의 시작지점을 기준으로 나머지 다른 노드의 시작 노드를 찾았습니다. 
- 시작 노드는 (길이가 긴 노드의 길이 - 길이가 짧은 노드의 길이)의 인덱스를 얻을수 있었고
- 각 리스트의 시작 인덱스를 설정하여 마지막 노드까지 반복문을 수행하면서 값을 비교해 동일하면 해당 노드(두개의 리스트중 아무거나) 반환합니다.
- 반환된 노드를 출력하면, 교차점 노드를 찾을 수 있습니다.

```java
  
class LinkedNodeList {  
  
    Node head;  
  
    LinkedNodeList() {  
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
    // head -> 4 -> 5-> 7->6  
    void delete (int d) {  
        Node nowNode = head;  
  
        while (nowNode.next != null) {  
            if (nowNode.next.data == d) {  
                nowNode.next = nowNode.next.next;  
            } else{  
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
	// 노드 길의 구하는 함수
    int getLength() {  
        Node nowNode = head;  
        int count = 0;  
  
        while (nowNode.next != null) {  
            count++;  
            nowNode = nowNode.next;  
        }  
  
        return count;  
    }  

	// 인덱스에 해당되는 노드 조회
    Node getNode(int index) {  
	    // 헤더 노드의 다음 노드부터 시작
        Node nowNode = head.next;  
        int count = 0; // 리스트 반복문을 수행하면서 카운트를 증가
  
        while (nowNode.next != null) {  
            if (index == count) {  
                return nowNode;  
            }  
            count++;
            // 다음 노드로 이동
            nowNode = nowNode.next;  
        }  
  
        return nowNode;  
    }  
}  
  
class Node {  
    int data;  
    Node next;  
  
    Node() {  
        data = 0;  
        next = null;  
    }  
}  
  
public class test {  
    public static Node listSeparate(LinkedNodeList ll1, LinkedNodeList ll2) {  
        int length1 = ll1.getLength();  
  
        int length2 = ll2.getLength();  

		// 리스트 1의 시작 인덱스
        int tmp1 = 0;  

		// 리스트 2의 시작 인덱스
        int tmp2 = 0;  

		// 리스트 1의 길이가 리스트 2 보다 클 경우
        if (length1 > length2) {  
            tmp1 = length1 - length2;  
        }else{  
	        // 리스트 1의 길이가 리스트 2보다 같거나 작을 경우
            tmp2 = length2 - length1;  
        }  

		// 리스트 1의 시작 노드
        Node n1 = ll1.getNode(tmp1);  

		// 리스트 2의 시작 노드
        Node n2 = ll2.getNode(tmp2);  
  
        while (n1.next != null) {  
            if(n1.data == n2.data) {  
                return n1;  
            }  

			// 다음 노드로 이동
            n1 = n1.next;  
            n2 = n2.next;  
        }  

		// 마지막 노드가 같을 경우
        if(n1.data == n2.data) {  
            return n1;  
        }  
  
        return null;  
    }  

    public static void main(String[] args) {  
        LinkedNodeList ll1 = new LinkedNodeList();  
        LinkedNodeList ll2 = new LinkedNodeList();  
  
        ll1.append(5);  
        ll1.append(7);  
        ll1.append(9);  
        ll1.append(10);  
        ll1.append(7);  
        ll1.append(6);  
  
        ll2.append(6);  
        ll2.append(8);  
        ll2.append(10);  
        ll2.append(7);  
        ll2.append(6);  
  
        Node node = listSeparate(ll1, ll2);  
  
        if( node != null) {  
            System.out.println(node.data);  
        }  
  
    }  
}
```
