---
title: "How to: Build Your Blog with Hexo"
description: "Building a blog with Hexo"
publishDate: "21 Aug 2024"
updatedDate: "25 Aug 2025"
tags: ["website", "tutorial", "hexo"]
---

# 第一步:下載 node.js

可以到 [node.js](https://nodejs.org/) 下載最新的版本
下一步 下一步 下一步 安裝成功

# 第二步: github 設定

因為我是用 github pages
所以會需要用到 github 來公開網站
然後最好下載 git 這樣可以用 git bash 比較不會出錯

## 在 github 創建一個新的 repository

名字取作 你的帳號.github.io 例如:(grissia.github.io)
然後其他都留預設就可以了(尤其是 repo 要設公開)

# 第三步:安裝 hexo

可以到 [Hexo的官網](https://hexo.io/)
或用我底下附的指令
**這些指令在git bash執行才不會出錯**

```bash
// 安裝hexo
npm install hexo-cli -g

// 創建一個叫做blog的網頁專案
hexo init blog

// 進入剛創的資料夾
cd blog

// 加載 npm
npm install

// 在本地運行(目前不需要)
hexo server
```

# 第四步:挑選主題

到剛剛的 hexo 官網選擇喜歡的主題，然後根據該主題的說明一步步安裝即可
**記得改 _config.yml 裡面的 theme 選項**

# 第五步:把網頁交給 github

```bash
// 刪除舊版檔案
hexo clean

// 生成網頁檔案
hexo generate
```

輸入後就會看到多了一個 public 資料夾
把裡面的東西全部上傳到剛剛創的repository就大功告成
(不過當然如果你會用 git 就當我沒說)

# 延伸補充

## 常用指令

```bash
// 刪除上次 generate 的檔案
hexo clean

// 把檔案整理到 public 資料夾
hexo generate

// 一鍵同步，需要先設定，下方附上連結
hexo deploy

// 新增頁面之類的，layout主要有 
// post(一篇新貼文) page(頁面，例如"關於我"頁面) draft(草稿，會先存著等待發布)
// title 如果有包含空格要用雙引號包起來
hexo new [layout] <title>

// 在本地運行
hexo server
```
[Hexo 一鍵部屬](https://hexo.io/docs/one-command-deployment)
