---
title: Neo4j Desktop for Mac 如何安装Plugin APOC
date: 2021/3/2 16:25:25
categories:
- Tech
tags:
- Neo4j
---

1. 安装Neo4j Desktop for mac后，创建新的Project，打开Manger

   ![](https://ftp.bmp.ovh/imgs/2021/03/feb5e8c3c3676c3c.png)

   ![](https://ftp.bmp.ovh/imgs/2021/03/35d343c1f5b2fa9f.png)

   

   插件下面既没有Install 按钮，还有一个 **Failed to find compatible version** 的错误

   

2. 先找到Plugin的安装目录

   打开Manager —> Logs，找到project的plugin地址。

   ![](https://ftp.bmp.ovh/imgs/2021/03/3c5457a2f36e9f87.png)

   

3. 先下载APOC，一定注意和Neo4j的版本一致，一定一定一定要一致。

   下载地址 `https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases?after=4.0.0.18`

   我安装的neo4j是3.5.14，因此我找到

   ![](https://ftp.bmp.ovh/imgs/2021/03/81374d11e0467753.png)

   下载完毕后，将apoc-3.5.0.14-all.jar复制到前面找到的文件夹内，重新启动neo4j服务

4. 安装成功

   ![](https://ftp.bmp.ovh/imgs/2021/03/f17015b9aee09af9.png)

   
