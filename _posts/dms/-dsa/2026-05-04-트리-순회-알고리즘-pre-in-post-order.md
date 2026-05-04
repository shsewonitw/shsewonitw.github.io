---
layout: post
title: "[Daily morning study] 트리 순회 알고리즘 (Pre/In/Post-order)"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 트리 순회 알고리즘

트리(Tree)를 탐색할 때 모든 노드를 정확히 한 번씩 방문하는 방법을 **트리 순회(Tree Traversal)**라고 한다. 이진 트리 기준으로 전위, 중위, 후위 세 가지 DFS 순회와 레벨 순서(BFS) 순회가 있다.

---

## 기본 용어

- **루트(Root)**: 트리의 최상단 노드
- **자식(Child)**: 특정 노드의 하위 노드
- **서브트리(Subtree)**: 특정 노드를 루트로 하는 부분 트리
- **리프(Leaf)**: 자식이 없는 노드

아래 트리를 예시로 사용한다.

```
        1
       / \
      2   3
     / \   \
    4   5   6
```

---

## 전위 순회 (Preorder Traversal)

### 방문 순서

**루트 → 왼쪽 서브트리 → 오른쪽 서브트리**

```
결과: 1 → 2 → 4 → 5 → 3 → 6
```

### 특징

- 루트를 제일 먼저 방문하기 때문에 트리 구조를 그대로 직렬화(Serialize)할 때 유용하다.
- 복사본을 만들거나 디렉터리 구조를 출력할 때 자주 쓰인다.

### 구현

```python
def preorder(node):
    if node is None:
        return
    print(node.val)       # 루트 처리
    preorder(node.left)   # 왼쪽
    preorder(node.right)  # 오른쪽
```

---

## 중위 순회 (Inorder Traversal)

### 방문 순서

**왼쪽 서브트리 → 루트 → 오른쪽 서브트리**

```
결과: 4 → 2 → 5 → 1 → 3 → 6
```

### 특징

- **이진 탐색 트리(BST)**에서 중위 순회를 하면 오름차순으로 정렬된 값을 얻는다.
- BST의 유효성 검사나 k번째 원소 탐색 문제에서 핵심적으로 활용된다.

### 구현

```python
def inorder(node):
    if node is None:
        return
    inorder(node.left)    # 왼쪽
    print(node.val)       # 루트 처리
    inorder(node.right)   # 오른쪽
```

### BST 예시

```
    4
   / \
  2   6
 / \ / \
1  3 5  7

중위 순회 결과: 1 2 3 4 5 6 7  ← 자동으로 정렬됨
```

---

## 후위 순회 (Postorder Traversal)

### 방문 순서

**왼쪽 서브트리 → 오른쪽 서브트리 → 루트**

```
결과: 4 → 5 → 2 → 6 → 3 → 1
```

### 특징

- 자식을 먼저 처리한 뒤 부모를 처리하는 구조여서, **하위 항목부터 정리해야 하는 경우**에 적합하다.
- 디렉터리 삭제(자식 파일 먼저 삭제 후 폴더 삭제), 수식 트리의 계산(Expression Tree)에 활용된다.

### 구현

```python
def postorder(node):
    if node is None:
        return
    postorder(node.left)  # 왼쪽
    postorder(node.right) # 오른쪽
    print(node.val)       # 루트 처리
```

---

## 레벨 순서 순회 (Level Order / BFS)

### 방문 순서

위에서 아래로, 같은 레벨은 왼쪽에서 오른쪽으로 방문한다.

```
결과: 1 → 2 → 3 → 4 → 5 → 6
```

### 특징

- 큐(Queue)를 사용해 구현한다.
- 트리의 최솟값 깊이 탐색, 지그재그 출력 등의 문제에 활용된다.

### 구현

```python
from collections import deque

def level_order(root):
    if root is None:
        return
    queue = deque([root])
    while queue:
        node = queue.popleft()
        print(node.val)
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)
```

---

## 세 가지 DFS 순회 비교

| 순회 방식 | 방문 순서 | 대표 활용 사례 |
|-----------|-----------|----------------|
| 전위(Preorder) | 루트 → 왼 → 오 | 트리 복사, 직렬화 |
| 중위(Inorder) | 왼 → 루트 → 오 | BST 정렬 출력, k번째 탐색 |
| 후위(Postorder) | 왼 → 오 → 루트 | 디렉터리 삭제, 수식 계산 |
| 레벨 순서(BFS) | 레벨별 좌→우 | 최단 깊이, 넓이 우선 처리 |

---

## 반복(Iterative) 방식으로 구현하기

재귀는 깊은 트리에서 스택 오버플로우 위험이 있다. 스택을 명시적으로 사용하면 반복문으로 대체할 수 있다.

### 전위 순회 반복 구현

```python
def preorder_iterative(root):
    if root is None:
        return
    stack = [root]
    while stack:
        node = stack.pop()
        print(node.val)
        # 오른쪽을 먼저 push해야 왼쪽이 먼저 pop됨
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)
```

### 중위 순회 반복 구현

```python
def inorder_iterative(root):
    stack = []
    curr = root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        print(curr.val)
        curr = curr.right
```

---

## 코딩 테스트 활용 패턴

### 트리의 최대 깊이

후위 순회 방식으로 자식 깊이를 먼저 구한 뒤 부모에서 합산한다.

```python
def max_depth(node):
    if node is None:
        return 0
    return 1 + max(max_depth(node.left), max_depth(node.right))
```

### BST에서 k번째 작은 값

중위 순회 결과가 정렬된 배열이므로 카운터로 k번째를 찾는다.

```python
def kth_smallest(root, k):
    result = []
    def inorder(node):
        if node and len(result) < k:
            inorder(node.left)
            result.append(node.val)
            inorder(node.right)
    inorder(root)
    return result[k - 1]
```

---

## 정리

- 전위(Preorder): 루트 우선 → 트리 구조 자체를 다룰 때
- 중위(Inorder): BST에서 정렬 순서 보장
- 후위(Postorder): 자식 처리 후 부모 처리 → 삭제·계산
- 레벨 순서(BFS): 너비 우선 → 레벨 단위 처리

세 DFS 순회는 모두 `O(n)` 시간, `O(h)` 공간(h = 트리 높이)이다. 균형 트리면 `O(log n)`, 편향 트리면 최악 `O(n)`.
