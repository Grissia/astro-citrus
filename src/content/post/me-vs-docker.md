---
title: "那些我與 Docker 的辛酸血淚史"
description: "How to deploy a Docker"
publishDate: "17 Dec 2024"
updatedDate: "25 Aug 2025"
tags: ["Docker", "Tutorial"]
---

# 什麼？我連錯誤訊息都看不到？

歡迎來到 `Docker 的世界`
在這裡 我們有數不完的 `docker-compose` 跟 `Dockerfile` 要寫 
> 開心吧 : )))))))))))))

# SQL 與我

> 關於我在 Docker 上面架 MySQL 那檔事

總之，先給你們看成功版的 Dockerfile

```
web/
├── docker-compose.yml
├── SoQuiteLa/
|   ├── src/
|   |    ├── welcome.php
|   |    ├── index.php
|   ├── config/
|   |    ├── init.sql
|   ├── Dockerfile
```

```Dockerfile
FROM php:8.1-apache

COPY ./src/ /var/www/html/

RUN docker-php-ext-install mysqli

EXPOSE 80
```

```yml
services:
  soquietla:
    build: ./SoQuietLa
    container_name: web_soquietla
    ports:
      - "3003:80"
    depends_on:
      - soquietla_db
    networks:
      - db_network

  soquietla_db:
    image: mysql:5.7
    container_name: web_soquietla_db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ctf
    volumes:
      - ./SoQuietLa/config/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - db_network

networks:
  db_network:
```
~~至於為什麼會叫這些奇怪名字，我在幫社團寫 CTF challenge，就先別管~~

---

## 第一個錯誤 - 檔案位置

我在本地運行的時候是把 `docker-compose.yml` 放在 `SoQuietLa` 裡面的
但傳到伺服器上去後忘記把

```yml
    volumes:
      - ./config/init.sql:/docker-entrypoint-initdb.d/init.sql
```

改成

```yml
    volumes:
      - ./SoQuietLa/config/init.sql:/docker-entrypoint-initdb.d/init.sql
```

而且這東西(Docker)還真有你的，完全沒有跳錯誤
 
```bash
docker compose build
docker compose up -d
```

跑完，完全沒錯誤

```bash
docker ps
```

docker ps 一看
那時我心裡就是 `不是哥們 我的 db 呢 怎麼沒有`

用了 `docker ps -a` 只看到 `web_soquietla_db exited 3 minutes ago`  
我一整個就是無法理解

## 第二個錯誤 - MySQL 版本

當時怎麼跑就是跑不出來  

```bash
docker logs web_soquietla
```

後跳出 `Fatal glibc error: CPU does not support x86-64-v2`

然後查老半天才知道我用的那台 server 的硬件架構跟 container 不兼容  
> 最後把 MySQL 版本從 8.0 改到 5.7 才解決 (累

---

## 第三個錯誤 - DB name

因為用 Docker 架的 MySQL 名稱會自動隨著 `docker-compose.yml` 改動  
所以我只要改 `docker-compose.yml` 上的名稱，我的 source code 就要改連接資訊  
~~可想而知，我忘記改了~~

# 心得

我的 Docker 要不是不得不學  
早就放棄了  
還好是有一些專案在逼我
我才能堅持下去學好 Docker  
~~再加上一些好玩的 projects 像是用 docker 架 minecraft server~~
