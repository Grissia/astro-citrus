---
title: "APCS Camp 2024 蘋果問題題解"
description: "Solving Problems with Bitmask Enumeration"
publishDate: "19 Aug 2024"
updatedDate: "25 Aug 2025"
tags: ["apcs", "cpp", "solutions"]
---

# 題目

第一行有一個整數 *n* 表示蘋果的數量
第二行有 *n* 個整數表示每個蘋果的重量

請輸出將蘋果分兩籃後 兩籃最小的重量差

```
測資
5
3 2 7 4 1
輸出
1
```

# 解題思路

假設五顆蘋果分別重 *2,4,5,3,1*
跑一個迴圈從二進位的 *00000* 到 *10000* (因為 *00001* 跟 *11110* 一樣，所以沒必要檢查到 *11111* )
也可以寫作 0 到小於 (1 << 5-1) (減會優先運算所以就是 1 << 4)
然後用這個二進位的位元來當作每顆蘋果進哪個籃子的依據
~~總之就是所有情況模擬一次拉 哈哈哈哈哈哈哈~~

# 程式碼

```cpp
#include <bits/stdc++.h>
using namespace std;

int main(){
    cin.tie(0); ios_base::sync_with_stdio(false); // cin加速器

    int n;
    cin >> n;
    vector<long long> arr; 
    // 存每個蘋果重
    
    for(int a,i = 0; i<n; i++){
        cin >> a;
        arr.push_back(a);
    }
    // 輸入蘋果
    
    long long ans = 1e18; // 先給一個很大的數
    for(int i = 0; i< (1 << n-1); i++){ // 預定一個範圍
        long long sum[2] = {}; // 籃子
        for(int j = 0; j<n; j++){ 
            sum[i >> j & 1] += arr[j];
            /*
            喔喔喔 這東西看起來超可怕
            就像前面解題思路寫的 i 可能是 01100 之類的
            那 j 就是從 0 到 5-1
            所以這行的意思就是把 i 從右移 j 之後檢查最後一位
            如果是 1 就丟 1 號籃
            如果是 0 就丟 0 號籃
            */ 
        }
        ans = min(ans,llabs(sum[0] - sum[1]));
        // 計算兩籃差 是不是小於 ans
    }
    
    cout << ans << endl;
    return 0;
}
```
