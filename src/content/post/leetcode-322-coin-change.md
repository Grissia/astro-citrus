---
title: "Leetcode 322.Coin-Change 題解"
description: "Leetcode 322.Coin-Change Writeup"
publishDate: "18 Oct 2024"
updatedDate: "25 Aug 2025"
tags: ["cpp", "solutions", "dynamic programming"]
---

想不到吧 我在寫 leetcode 找錢問題

# 題目

給一個 vector，裡面是你手邊有的零錢幣值  
只要符合幣值規定，你可以無限的取用硬幣  
想辦法用最少的硬幣湊出題目要求的金額  
[Leetcode 題目連結](https://leetcode.com/problems/coin-change/)

# 測資

```
Example 1:

Input: coins = [1,2,5], amount = 11
Output: 3
Explanation: 11 = 5 + 5 + 1

Example 2:

Input: coins = [2], amount = 3
Output: -1

Example 3:

Input: coins = [1], amount = 0
Output: 0
```

# 解題關鍵

暴力枚舉是肯定吃 TLE 吃好吃滿  
貪心又可能會有例外  
所以我們就乖乖寫動態規劃吧 XD

## 解法原理

如果幣值有 m1, m2, m3 三種  
總共找 x 元，我先找 m1 元，就只剩下 x-m1 元要找了!  
發現這件事之後，我們就分別計算

- 找 m1 元後剩多少錢要找
- 找 m2 元後剩多少錢要找
- 找 m3 元後剩多少錢要找

以找 m1 元為例，還剩 x-m1 元要找  
接下來又可以分為:

- 再找 m1 元後，剩 (x-m1) - m1
- 再找 m2 元後，剩 (x-m1) - m2
- 再找 m3 元後，剩 (x-m1) - m3

以此類堆，直到餘額歸零為止

## 實際程式細節

### 找錢失敗問題

這樣不斷遞迴下去，就有可能出現餘額是負數的狀況  
這就代表這個組合沒辦法剛好找完錢  
就 return -1 表示這東西沒用

### TLE 問題

什麼??  
動態規劃還會 TLE 我寫他幹嘛???

冷靜，剛剛還只有遞迴，這樣程式會不斷計算相同的值  
所以我們就簡單用一個 map 去存計算過的值才算是真正的動態規劃

### 腦子燒壞問題

其實我常常遞迴寫一寫就不知道自己在寫什麼  
所以第一次寫這類題目的朋友  
在寫程式的過程中要不斷提醒自己  
**這個函式所代表的意義**

以這題為例，coinChange(11) 就表示
現有的幣值在找 11元時的最少用量

# 程式碼

```cpp
class Solution {
public:
    map<int,int> mp; // 用來存答案，才不會重複計算
    int coinChange(vector<int>& coins, int amount) {
        if(amount == 0) return 0; // 遞迴到底就定義為零
        if(mp[amount]) return mp[amount]; // 如果算過就不再算一次
        int ans = INT_MAX; // 存這回的答案
        for(auto i : coins){ // 用 range-based for 去讀每一種幣值，假設先找這個幣值
            if(amount < i) continue; // 如果幣值大於總共要找的金額就跳過
            int tmp = coinChange(coins, amount-i); // 如果用這個幣值，下一次最少找多少個硬幣
            if(tmp == -1) continue; // 如果下次不可能找到，那這次也不可能是這個幣值
            ans = min(ans, tmp+1); // 把答案更新為最少找錢數(tmp+1是因為每往下遞迴一階就會多找一個硬幣，所以要加回來)
        }
        if(ans == INT_MAX) mp[amount] = -1; // 如果 ans 都沒變表示完全無解
        else mp[amount] = ans;
        return mp[amount];
    }
};
```
