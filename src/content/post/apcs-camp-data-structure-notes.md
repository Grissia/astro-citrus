---
title: "APCS Camp 2024 資料結構筆記"
description: "APCS Camp Data Structure"
publishDate: "21 Aug 2024"
updatedDate: "25 Aug 2025"
tags: ["apcs", "cpp", "data structure"]
---

資料結構就是 *儲存資料的工具&方式*

# STL(Standard Template Library) 標準模板庫

- cpp內建的標準通用模板，有一堆好用的資料結構與演算法
- 要用 迭代器 存取

## STL容器的迭代器類型

- 單向迭代器:只能往後，用遞增運算子(\++it或it++)移動
    如 *unordered_set, unordered_multiset, unordered_map, unordered_multimap*
- 雙向迭代器:可以往前往後，(用++跟\--)
    如 *list, set, multiset, map, multimap*
- 隨機存取迭代器:類似陣列 arr[i]，可以用(it+k或it-k)一次移動多格
    如 *vector, Deque, array, string*
- 沒有迭代器
    如 *stack, queue, priority_queue*

```cpp
// 迭代器的遍歷
for (vector<int>::iterator it = vt.begin(); it != vt.end(); it++) {
    cout << *it << " ";
}
for (vector<int>::reverse_iterator rit = vt.rbegin(); rit != vt.rend(); rit++) {
    cout << *rit << " ";
}
for (int &x : vt) {
    cout << x << " ";
}
```


## 常見資料結構

### Pair

- 需要 #include \<utility>
- 能綁定兩組數據

- 宣告
    - pair<T1,T2> p;
    - T1、T2是該位置的型態
- 初始化
    - pair<T1,T2> p(a,b);
    - pair<T1,T2> a = make_pair(a,b);
    - pair<T1,T2> p = {a,b};
- 取值
    - p.first
    - p.second

### Tuple

- 需要 #include \<tuple>
- 可以綁三組以上的資料

- 宣告
    - tuple<T1,T2,T3,T4,...> t;
    - T1、T2、T3...是該位置的型態
- 初始化
    - tuple<T1,T2,T3,...> t(a,b,c,...);
    - tuple<T1,T2> a = make_tuple(a,b);
    - tuple<T1,T2> t = {a,b};
- 取值
    - get<0> t
    - get<1> t

### Vector

- 需要 #include \<vector>

- 宣告:

    - vector\<T> v;
    - T是該vector的型態

- 初始化:

    - vector\<T> v(n,a);
    - n 是 vector 長度
    - a 是初始化的值

- 常用函式
    ```cpp
    .push_back(a)
    // 就 push
    .pop_back()
    // 就 pop
    .size()
    // 回傳容器長度
    .empty()
    // 回傳容器是否為空
    .front()
    // 回傳第一個值
    .back()
    // 回傳最後一個值
    .begin()
    // 回傳 iterator 並指向第一個
    .end()
    // 回傳 iterator 並指向最後一個的下一個
    .clear()
    // 清空容器
    ```

### Stack

- 需要先 #include \<stack>
- FILO

- 常用函式:
    ```cpp
    .push(a)
    .pop()
    .size()
    .empty()
    .top()
    // 回傳最前端的值
    ```
    
### Queue
    
- 需要先 #include \<queue>
- FIFO
- 常用函式:
    ```cpp
    .push(a)
    .pop()
    .back()
    .front()
    .size()
    .empty()
    ```

### Deque
    
- 需要先 #include \<deque>
- 可以存取前端跟末端

- 常用函式:
    ```cpp
    .push_back(a)
    .push_front(a)
    .pop_back()
    .pop_front()
    .back()
    .front()
    .size()
    .empty()
    .clear()
    .begin()
    .end()
    ```
    
### Linked List
    
- 需要先 #include \<list>
- 需要用迭代器存取

- 常用函式:
    ```cpp
    .push_back(a)
    .push_front(a)
    .pop_back()
    .pop_front()
    .back()
    .front()
    .size()
    .empty()
    .clear()
    .begin()
    .end()
    .advance(n)
    // 迭代器往後移動 n 格
    .insert(n,a)
    // n 是迭代器的位置 例如:(lt.begin()+2)
    // a 是要插入的值
    erase(n)
    // n 同上，刪除位置 n 的值
    ```

```cpp
// 範例
list<T1> lt = {2,4,5};
lt.insert()
```