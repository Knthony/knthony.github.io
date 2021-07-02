---
title: Neo4j APOC — 初体验
date: 2021/3/3 10:36:21
categories:
- Tech
tags:
- Neo4j
---


## APOC 到底是什么

APOC作为Neo4j实验室的项目之一，目的是为了扩展数据库和Cypher，以便在程序中使用查询、加载、以及导入导出等你能想象到的所有功能。

目前已经有超过450个标准的procedures。

官方文档入口 [官方文档入口](https://neo4j.com/labs/apoc/)。



## 如何使用Procedure

1. `CALL <package>.<subpackage>.<procedure>(<argument1>,<argument2>,...);`

2. `CALL apoc.help('keyword')`

如果我们想查看所有用来修改，转换，或者处理数据的关于`text` 的procedures，用以下命令查询：

`CALL apoc.help('text')`

![](https://ftp.bmp.ovh/imgs/2021/03/22d7a797a6425097.png)

命令执行的结果包含了示例，参数介绍以及数据类型。


## 常见用例

1. `apoc.date.format(dateForConversion, [timeUnit], [format])`  — 将标准时间转换为所需要的时间格式。

   > 需要注意的是在neo4j 3.4往后的released版本中，一些日期和时间的处理功能已经包含在产品内部了，不需要从APOC中调取使用。

2. `apoc.load.json(url) `— 从Url 或Json格式的文件中读取数据然后使用`Cypher`创建或更新到Neo4j数据库中。

3. `apoc.periodic.iterate(query1, query2, {param1: value1})`— 用于批量装载数据，可以从`query1`中提取数据列表，然后会对每一个`query1`的结果执行`query2`以更新或检索其他数据。

## 使用示例
以`apoc.load.json`为例；
1. 先用`apoc.help('load.json')` 查看此procedure
![](https://ftp.bmp.ovh/imgs/2021/03/5f6ef5f913aec14a.png)
2. 执行
   ```
   WITH 'https://raw.githubusercontent.com/neo4j-contrib/neo4j-apoc-procedures/4.1/core/src/test/resources/person.json' AS url
   CALL apoc.load.json(url) YIELD value as person
   MERGE (p:Person {name:person.name})
      ON CREATE SET p.age = person.age, p.children = size(person.children)
   ```
   1. 因为网络问题，无法直接访问到githubusercontent.
   ![](https://ftp.bmp.ovh/imgs/2021/03/abf8df4d19c2ad4b.png)
   2. 用浏览器访问将`person.json`，保存为本地文件
   3. 在Neo4j配置文件中添加 `dbms.security.apoc.import.file.enabled=true`
   4. 将`person.json`文件放在数据库的`import`文件夹之下
   5. 执行
   ```
      WITH 'person.json' AS url
   CALL apoc.load.json(url) YIELD value as person
   MERGE (p:Person {name:person.name})
      ON CREATE SET p.age = person.age, p.children = size(person.children)
   ```
   ![](https://ftp.bmp.ovh/imgs/2021/03/7b308c304e837026.png)
   OK！ 搞定！



