---
layout: post
title:  "이진 탐색 (Binary Search)"
date:   2024-10-28 22:53:42 +0900
categories: algorithm, binary search, 99클럽, 코딩테스트준비, 개발자취업, 항해99, TIL
---
이진 탐색은 **정렬된 배열이나 리스트에서 특정 값을 빠르게 찾기 위해 사용하는 알고리즘**으로, **탐색 범위를 절반씩 줄여나가는 방식**이다.

이진 탐색의 시간 복잡도는 **O(log N)**이므로 탐색 횟수가 매우 클 때, 사용하면 효율적이다.

[이진 탐색(binary search) 정리글](https://178kg78cm.github.io/algorithm/2024/10/28/binary-serach.html)


## 이진 탐색 동작 과정

1. **중간값 계산**: 주어진 배열의 중간값을 찾는다.
2. **값 비교**:
   - 찾으려는 값이 중간값보다 크면, **탐색 범위를 오른쪽 절반**으로 줄인다.
   - 찾으려는 값이 중간값보다 작으면, **탐색 범위를 왼쪽 절반**으로 줄인다.
3. **재귀적 반복 또는 반복문**: 찾을 때까지 범위를 계속 반으로 줄이며 반복한다.
4. **탐색 종료**: 찾으려는 값이 없으면 탐색을 종료한다.

## 코드 예제 (C++)

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

bool binary_search(const vector<int>& arr, int target) {
    int left = 0;
    int right = arr.size() - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;

        if (arr[mid] == target) {
            return true;
        }
        if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return false;
}
```