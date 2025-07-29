---
title: "\b릿코드 알고리즘 공부 - Climbing Stairs"
description: 
author: yoonxjoong
date: 2025-07-29 09:00:00 +0900
categories:
  - Algorithm
tags:
---

You are climbing a staircase. It takes n steps to reach the top.   
  
 Each time you can either climb 1 or 2 steps. In how many distinct ways can   
you climb to the top?   
  

 Example 1:   
    
Input: n = 2  
Output: 2  
Explanation: There are two ways to climb to the top.  
1. 1 step + 1 step  
2. 2 steps  
  
  
 Example 2:   

Input: n = 3  
Output: 3  
Explanation: There are three ways to climb to the top.  
3. 1 step + 1 step + 1 step  
4. 1 step + 2 steps  
5. 2 steps + 1 step  
  
 Constraints:   1 <= n <= 45


피보나치 수열의 원리
- 피보나치 수열은 수학에서 가장 잘 알려진 수열 중 하나입니다.
- 첫번째, 두번째 항이 미리 정의 되어 있고그 이후의 모든 항은 바로 앞의 두항을 더한 값으로 결정되는 수열
```
기본 형태 :
	가장 일반적인 피보나치 수열의 시작은 다음과 같음
	F0 = 0
	F1 = 1

	그리고 n >= 2인 모든 정수 n에 대해 다음 점화식을 따릅니다.
	Fn = Fn-1 + Fn-2

예시
	F0 = 0
	F1 = 1
	F2 = F1 + F2 = 1
	F3 = F2 + F1 = 2
	F4 = F3 + F2 = 3
	F5 = F4 + F3 = 5
	F6 = F5 + F4 = 8
	F7 = F6 + F5 = 13
	F8 = F7 + F6 = 21
```

가장 기본적인 피보나치 수열은 다음과 같습니다.
- 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55...

여기서 심화를 하면 F0=1, F1=1 주어집니다.

그럼 피보나치 수열은 다음과 같죠

1, 1, 2, 3, 5, 8, 13 ... 

문제로 돌아가면,


계단이 1개일 경우에는 1가지의 케이스만 있으므로 F1 = 1
계단이 2개일 경우에는 2가지의 케이스만 있으므로 F2 = 2

F1 = 1
F2 = 2
F3 = F2 + F1 = 3
F4 = F3 + F2 = 5
F5 = F4 + F3 = 8
F6 = F5 + F4 = 13
.
.
.


그럼 피보나치 수열을 자바 코드로 작성해 봅시다.

n = 1
n + 1 = 2

``` java

int recursive(int n) {
	if (n == 1) {
		return 1;	
	}

	if (n == 2) {
		return 2;
	}

	int val = recursive(n - 1) + recursive(n - 2);

	return val;
}

```


이렇게 테스트 코드를 작성하고 돌리면 성공은 합니다만, 입력된 데이터가 커지면 Time Limit Exceeded가 발생하여 통과에 실패하죠.

재귀 호출의 비효율성을 해결하기 위해 동적 계획법(DP)를 활용해 구현할 수 있습니다.

``` java

public int climbStairs(int n) {  
  
    int[] dp = new int[n + 2];  
  
    dp[1] = 1;  
    dp[2] = 2;  
  
    for (int i = 3; i <= n; i++) {  
        dp[i] = dp[i - 2] + dp[i - 1];  
    }  
  
    return dp[n];  
}
```



초기에는 재귀 호출 방식을 통해 피보나치 수열 알고리즘을 풀어보려고 했지만, n 값이 커질 수록 비효율적이였습니다.

recursive(n-1), recursive(n-2)를 호출할때 동일한 하위 로직들을 반복적으로 호출한다는 것이죠.

이를 해결 하기 위해 동적 계획법을 사용했습니다. 

DP는 크게 2가지 해결법이 있는데 저는 Bottom-up DP를 사용하여 풀었죠. 

- 가장 효율적인 방법으로, 재귀 호출 없이 **반복문**을 사용합니다.
    
- 작은 문제(F(1),F(2))의 답부터 시작하여 배열(테이블)에 저장합니다.
    
- 반복적으로 F(i)=F(i−1)+F(i−2) 공식을 사용하여 i=3부터 최종 n까지 배열을 채워나갑니다.
    
- 모든 계산이 완료되면, `dp[n]`에 최종 답이 저장됩니다.

이러한 문제 해결 과정은 알고리즘 최적화의 중요한 원리를 보여줍니다. 재귀의 직관성은 좋지만, 효율성 측면에서는 동적 계획법과 같은 기법이 필수적일 때가 많습니다.