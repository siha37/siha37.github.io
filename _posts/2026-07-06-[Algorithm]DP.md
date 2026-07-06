---
title: "[Algorithm] 동적 계획법 (Dynamic programming, DP)"
author: siha
date: 2026-07-06 01:10:00 +0800
categories: [ALGORITHM]
tags: [Algorithm]
render_with_liquid: false
---

해당 글은 [DP자료](https://wikidocs.net/369502)을 참고했습니다.

## 동적 계획법
큰 문제를 작은 부분 문제로 나누어 풀고 문제의 답을 저장해 다시 쓰는 기법


## 동적 계획법 두 조건
### 중복 구분 문제
- 같은 부분 문제가 여러번 반복해서 나타난다. 중복이 있어야 "저장해서 사용한다"는 특징이 메리트가 된다.
### 최적 부분 구조
- 큰 문제의 최적해가 작은 부분 문제들의 최적해로 이루어진다. 부분의 답을 모으면 전체의 답이 된다.

## 예) 피보나치

``` c++
int fib(int n)
{
  if(n <= 1)
  {
    return n;
  }
  return fib(n-1)+fib(n-2);
}
```
해당 코드는 재귀함수가 호출되며 동일한 n이 여러번 호출된다.
n이 5로 시작하면 fib(3)가 2번,  fib(2)가 3번 계산 된다.

이 문제를 최적화하기 위해 두가지 방법을 사용할 수 있다.
- 메모이제이션
- 타뷸레이션


## 메모이제이션

재귀 구조를 그대로 사용하고 답을 메모해두고 다음에 같은 문제가 생기면 메모의 값을 가져다 쓴다.

큰 문제에서 시작해 필요한 부분 문제로 내려가므로 하향식(top-down)이라고 부른다.

``` c++
long long fibMemo(int n,std::vector<long long>& memo)
{
  // 1에 도달
  if(n <= 1)
  {
    return n;
  }

  // 이미 계산 전적 있음
  if(memo[n] != -1)
  {
    return memo[n];
  }

  // 계산한 값을 memo n번째에 기록
  memo[n] = fibMemo(n-1,memo) + fibMemo(n-2,memo);

  return memo[n];
}

long long fib(int n)
{
  std::vector<long long> memo(n+1,-1);
  return fibMemo(n,memo);
}
```

## 타뷸레이션

재귀 대신 표를 만들어 가장 작은 부분 문제부터 차례로 채워 올라가는 방식이다.
작은 것에서 큰 것으로 쌓아 가서 상향식(bottom-up)이라 부른다.

``` c++

long long fibTab(int n)
{
  if(n <= 1){
    return n;
  }

  std::vector<long long> dp(n+1,0);
  dp[1] = 1;
  for(int i = 2;i <= n; ++i )
  {
    dp[i] = dp[i-1] + dp[i-2];
  }
  return dp[n];
}

```

dp에 미리 n = 0 , n = 1 일때의 값을 지정해두고 for문을 통해 계산해 채워간다.

## 예제 문제

### 0/1 배낭 문제

무게와 가치가 정해진 물건들이 있고, 담을 수 있는 무게 한도 `W` 인 배낭이 있다. 
가치의 합이 최대가 되도록 물건을 고르되, 각 물건은 담거나 담지 않거나 둘 중 하나다(쪼갤 수 없다). 이를 `0/1 배낭 문제(0/1 knapsack)`라 한다.

- 점화식 : 물건i 까지 고려하고 무게 한도가 c일 때의 최대가치를 dp[i][c]로 정의

``` c++
#include <vector>
#include <algorithm>

int knapsack(const std::vector<int>& weights, const std::vector<int>& values,int W)
{
  // 물건 갯수
  int n = weights.size();

  std::vector<std::vector<int>> dp(n+1,std::vector<int>(W+1,0));

  for(int i=1;i<=n;++i)
  {
    // 무게 , 가치
    int w = weights[i -1], v = values[i-1];

    // 
    for(int c = 0; c <= W; ++c)
    {
      // 해당 무게 한도의 이전 물건의 가치 전달
      dp[i][c] = dp[i-1][c];

      // 현재 물건이 무게 허용 시
      if(c >= w)
      {
        // 이전 삽입된 가치 vs 현물건 무게를 뺀 전물건의 물건 가치 + 현가치
        dp[i][c] = std::max(dp[i][c],dp[i - 1][c - w] + v);
      }
    }
  }

  return dp[n][W];
}
```


### 최장 증가 부분 수열

수열에서 일부 원소를 순서대로 골라, 앞에서 뒤로 갈수록 값이 커지는 가장 긴 부분 수열의 길이를 구하는 문제다. 

이를 최장 증가 부분 수열(longest increasing subsequence, LIS)이라 한다. 
예를 들어 `{10, 9, 2, 5, 3, 7, 101, 18}`에서는 `{2, 5, 7, 101}` 같은 길이 4가 가장 길다.

부분 문제를 "인덱스`i`에서 끝나는 증가 부분 수열의 최대 길이"인 `dp[i]`로 정의한다. 
`i`앞의 모든 `j`를 보아, `a[j] < a[i]`라면 `j`에서 끝나는 수열 뒤에 `a[i]`를 이어 붙일 수 있다.

``` c++
#include <vector>
#include <algorithm>

int lis(const std::vector<int>& a) 
{
    int n = a.size();

    if (n == 0) 
    {
      return 0;
    }

    std::vector<int> dp(n, 1);

    for (int i = 0; i < n; ++i) 
    {
      for (int j = 0; j < i; ++j) 
      {
        // i가 j 보다 크면
        if (a[j] < a[i])
        {
          // i의 수열 길이 vs j의 수열 길이 + 1 (길이 추가)
          dp[i] = std::max(dp[i], dp[j] + 1); 
        }
      }
    }

    return *std::max_element(dp.begin(), dp.end());   // 가장 긴 것
}
// lis({10,9,2,5,3,7,101,18}) == 4

```


### 격자 경로의 최소 비용
각 칸에 비용이 적힌 격자에서, 왼쪽 위에서 오른쪽 아래까지 오른쪽 또는 아래로만 움직여 지나간 칸의 비용 합을 최소로 하는 문제다.


``` c++
#include <vector>
#include <algorithm>
#include <climits>

int minPath(const std::vector<std::vector<int>>& grid)
{
  // R = y , C = x
  int R = grid.size(), C = grid[0].size();

  std::vector<std::vector<int>> dp(R,std::vector<int>(C,0));

  // 초기 비용
  dp[0][0] = grid[0][0];

  for(int i=0;i<R;++i)
  {
    for(int j = 0; j < C; ++j)
    {
      if(i == 0 && j == 0)
      {
        continue;
      }

      // 현 위치에서 위쪽 or 왼쪽의 비용 구함
      int up = (i > 0)?dp[i-1][j] : INT_MAX;
      int left = (j > 0)?dp[i][j-1] : INT_MAX;

      // 현 위치의 비용과 최소비용을 합하여 최종 비용을 기록
      dp[i][j] = grid[i][j] + std::min(up,left);
    }
  }
  // 우측 아래 위치의 최종 비용 반환
  return dp[R - 1][C - 1];
}
```
