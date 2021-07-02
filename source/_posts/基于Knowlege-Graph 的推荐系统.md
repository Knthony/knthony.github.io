---
title: 基于Knowlege-Graph的内容推荐系统
date: 2021/3/8 14:12:14
categories:
- Tech
tags:
- Neo4j, NLP, 推荐系统
---


根据上一期的[Neo4j NLP 自然语言处理](http://www.xcfans.top/2021/03/03/Neo4j%20NLP%20%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86/)实验，我们利用其数据完成  **根据用户的阅读记录进行推荐**


## 回顾
在上一篇文章中，我们利用APOC中的NLP precerdures 将所有blog的实体和其在文章中的TF-IDF 权重计算出来并将其存储为关系ENTITY的属性  `entity.score`

## 原理

假设我们的程序获取到用户点击了一篇他感兴趣的文章《This Week in Neo4j &#8211; Neo4j vs NetworkX, Accessing Neo4j with Spring Boot 2.4, Image Annotation on GCP》，判断其在上停留的时间证明其对本文章有一定的兴趣。
1. 计算出次文章中权重最高的三个代表实体
   ```
   MATCH (a:Article{title:"This Week in Neo4j &#8211; Covid Graph GraphQL API, Spring Data Neo4j 6 Experiments, CompoundDB4j"})-   [rel:ENTITY]->(e)
   WITH a, e, rel
   ORDER BY rel.score DESC
   RETURN collect(e.text + ":" + rel.score)[..3] AS entities
   ```
   result:
![](https://ftp.bmp.ovh/imgs/2021/03/214501e331d47774.png)

2. 我们使用剩下两个`QuickGraph`作为推荐的特征
  
  ```
  MATCH (a:Article)-[rel:ENTITY]->(e:Entity{text:"QuickGraph"})
  WITH a, e, rel
  ORDER BY a.date DESC, rel.score DESC
  RETURN (a.title),  collect(e.text)[..3] AS entities
  LIMIT 10;
  ```

  Result:
  ![](https://ftp.bmp.ovh/imgs/2021/03/44f6cc7d0bd6fe03.png)

得出的结果就是和`QuickGraph`相关的文章。

次过程仅限于实验使用，实际用途中，整个推荐筛选还需要更细致的设计和实现。