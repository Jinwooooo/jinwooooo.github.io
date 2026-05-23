---
title: 컴퓨터처럼 생각하기
date: 2024-08-21 00:00:00 +0900
categories: [Problem Solving]
tags: [algorithm, dfs]
math: true
description: 컴퓨터처럼 생각하기 (기본 → 심화) 정리글
image:
  path: thumbnail.png
  alt: Thumbnail illustration by Sandbox Studio, Chicago with Steve Shanabruch
media_subpath: /assets/img/posts/2024-08-21-thinking-like-a-computer/
---

## Summary

- 기본 → 글로벌 IT 기업 인턴 면접때 나온 질문을 통해 과정 설명
- 심화 → Find All Unique Subset 문제

## 기본

글로벌 IT 기업 인턴 면접때 나온 질문 활용

**영어로 받은 질문** → Estimate the value of Pi (π) by using a random function that uniformly returns a random value.

**한국어로 번역** → 파이 (π) 근사값을 균등한 랜덤 값을 주는 함수로 구해라.

이게 처음에 들었을때 "여기있는 수박으로 태양의 부피를 구해라" 같은 말도 안되는 문제라고 생각했다. 글로벌 IT 기업은 문제를 풀기위해 면접자에게 무슨 질문을 하면서 생각을 어떻게 풀어나가는 성향을 알기 위한 질문이다. 컴퓨터처럼 생각 + 사람과 소통 둘다 같이 확인하는 단계라고 생각하면 된다. 이 문제는 다음과 같이 단계적으로 풀이를 설명하는게 도움이 된다고 생각한다.

### 1. 반복 작업

"이 랜덤값이 Pi (π) 근사값이 나올때까지 반복적으로 실행한다" → 사실 어떻게 보면 정답이긴 하지만 이렇게 장난스럽게 얘기하면 면접자가 Pi (π) 값을 따로 지정을 못하고 순수히 random function 활용하여 실행시 근사값을 줘야한다고 제약조건을 주게된다.

그래도 "랜덤 함수 여러번 호출" 한다가 이 문제를 풀기 위한 첫번째 걸음이다. 균등하게 랜덤값으로 나오는 다수 값들을 어떻게 활용할까가 다음 단계다.

### 2. 차원 개념

컴퓨터는 1차원 기준이다. 사람은 1~3차원까지 크게 설명이나 공식없이 직관적으로 이해가 가능하다는 파악이 중요하다. 기본적으로 원은 1차원에 존재하지 않고 원이라는 추상적인 개념은 2차원 이상부터 가능하다.

일단 그럼 x,y 축이 있는 그래프를 만들고 random function에게 랜덤값은 0~1 사이 제약조건을 부여 한다고 가정하자 (0 ≤ random value ≤ 1).

![pi graph](pi_graph.png)
_x,y 축 그래프_

반지름이 1인 원의 중앙을 Origin (0,0)에 넣으면 다음과 같은 그림이 나오게 된다.

![pi circle](pi_circle.png)
_반지름 1인 원_

이제 random function 값을 가져와서 여기에 있는 그래프에 점을 찍는다고 생각하면 된다. 예시로 (0.5, 0.5) 값을 받으면 다음과 같은 그림이 된다.

![pi point](pi_point.png)
_(0.5, 0.5) 점_

### 3. 수학 공식

마지막으로 수학적 공식을 활용하여 random point가 원안에 있는지 없는지 판단하기가 가능하다.

$$
\sqrt{x^2 + y^2} = distance
$$

- 원 내부 → distance ≤ 1
- 원 외부 → distance > 1

다음 단계는 원의 넓이 : 네모의 넓이 = 원 내부 : 원 외부 비율 공식으로 활용한다는 것이다.

$$
\pi r^2 : (2r)^2
$$

r = 1 으로 제약조건을 넣었음으로 마지막 공식은 다음과 같이 된다

$$
\pi = \frac{4 * (distance \le 1)}{(distance > 1)}
$$

### 4. 풀이

```python
import sys
import random
input = sys.stdin.readline

no_iter = int(input().strip())
in_circle = 0

for _ in range(no_iter):
    x = random.uniform(0,1)
    y = random.uniform(0,1)

    dist = x**2 + y**2

    if dist <= 1:
        in_circle += 1

est_pi = (4 * in_circle) / no_iter

print(est_pi)
```

## 심화

인터넷에서 [다른 사람의 코딩테스트 관련된 얘기 vlog](https://youtu.be/cUT3S4N8fS8?si=Emi4yv-moYWq7XHU)을 접하면서 알게된 [Find All Unique Subset](https://leetcode.com/problems/subsets/description/). 문제는 다음과 같다.

> Given an integer array `nums` of **unique** elements, return *all possible subsets.* The solution set **must not** contain duplicate subsets. Return the solution in **any order**.

**Example 1:**

```
Input: nums = [1,2,3]
Output: [[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]
```

**Example 2:**

```
Input: nums = [0]
Output: [[],[0]]
```

Vlog에서 수학적으로 총 몇가지가 가능한지 수학적으로 계산 (Combinatorics 공식 활용)해서 푸는 얘기를 나누었지만 개인적으로 내가 얘기한 "기본 1. 반복 작업" 개념을 활용 부족해서 이런 방식으로 생각하게 된 것 같다.

기본적으로 이 문제는 새로운 element을 찾아서 삽입 할때마다 선택이 가능하며, 이 선택을 할때마다 파생되는 result를 Tree 자료 구조로 표현 할 것이다.

### 1. Naive Approach

Naive하게 생각해서 같은 값만 제외하는 제약조건만 넣고 삽입하는 방식을 선택하면 아래 그림과 같은 Tree 자료가 생긴다. 여기서 그림만 봐도 same set, but different order 유형이 바로 보인다 ([1,2],[2,1]) 이런 경우는 이제 추가적으로 삭제를 바로바로 해줘야 폭발적으로 늘어나는 자료 회피가 가능하다.

![tree naive](tree_naive.png)
_Naive approach tree_

### 2. Naive Approach + Memoization

Naive 방식에 memoization 추가적인 활용을 한다면, 매번 새로운 선택지가 생성시 memoization 여부에 따라 새로운 값을 만들고 삽입을 하던가 벌써 계산된 값임으로 넘어가는 선택을 하는 것이다.

![tree memoization](tree_memo.png)
_Naive + memoization tree_

이렇게 문제를 풀게 되면 어떻게보면 매우 비효율적이다 (memo값에서 같은 값이 있는지 확인시 set length * memo length 의 시간 복잡도가 생긴다). 하지만 컴퓨터는 반복작업에 적합해서 작은 문제 크기는 충분히 이렇게 구현을 해도 판별이 가능하며, 모든 unique subset을 구하기 가능하다.

### 3. DFS-like Approach

이 문제의 정공법은 정렬된 integer array의 pattern을 활용하는 것이다. DFS의 처음 접하고 연습할때 많이 보게되는 유형이다. integer array 정렬 확정이면 사실 다음 값은 그냥 현재 있는 값보다 다음 크기에 있는 값을 가져오고 다음 단계로 넘어가면 된다.

![tree dfs](tree_dfs.png)
_DFS-like approach tree_

pattern 활용시 불필요한 subset 생성이 불가능해서 매우 효율적으로 문제풀이가 가능하다. 더 이상 subset생성이 불가능하다 판별시 backtracking 하기 때문이다.

회사 서류 통과 후 코딩 테스트 같은 경우는 사실 Naive Approach + Memoization까지 문제풀이가 가능하다면 사실 왠만한 문제가 주어지면 일단 구현은 가능하다는 단계까지 판별이 가능하다고 본다. 선택을 할때마다 파생되는 결과물이 전 결과물과 무슨 관계를 이해하고 컴퓨터가 문제없이 돌아가는 단계까지 구현하는게 기본이라 생각한다.
