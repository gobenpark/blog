---
title: "Greedy"
date: 2024-07-19T17:30:48+09:00
image: ''
draft: true
---

## Greedy 알고리즘 

탐욕알고리즘, 

탐욕 알고리즘은 최적해를 구하는 데 사용되는 근시안적인 방법이다.

당장 최적인 답을 선택을해서 적합한 결과를 뽑아내야하는데 순간선택에 대해서는 최적이지만 종합적으로 봤을경우

최적이라는 보장이 절대 없다는것.

leetcode의 캔디 문제를 보면 다음과같다

1. 각 child는 적어도 1개의 캔디를 가지고 있어야한다.
2. 높은 점수를 가진 child는 옆의 child보다 많은 캔디를 가져야한다.

input = [1,0,2]
Output = 5

해당 부분에서 최적의 조건은 2,1,2이다 
총 5개의 캔디가 필요한것으로 코드를 한번 작성해 보면 다음과같다.

```go
func candy(rating []int) int {
	candies = make([]int, len(rating))
	
	// 1로 캔디를 초기화해준다.
	for i := 0; i < len(rating); i++ {
	    candies[i] = 1
    }
	
	// 왼쪽부터 탐색하여 캔디를 증가시켜준다.
	for i := 1; i < len(rating); i ++ {
	    if rating[i] > rating[i - 1] {
			candies[i] = candies[i - 1] + 1
        }	
    }
	
	// 오른쪽부터 탐색하여 체크
	for i := len(rating) - 2; i >= 0; i-- {
	    if rating[i] > rating[i + 1] && candies[i] <= candies[i + 1]{
			candies[i] = candies[i+1] + 1
        }
    }
	c := 0
	for i := 0; i < len(candies); i++ {
	    c += candies[i]	
    }
	
	return c
}
```

다음 과정으로 양방향 탐색을 통해 greedy 알고리즘을 작성해봤다.
