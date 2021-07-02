---
title: 基于Neo4j的医学论文推荐系统
date: 2021/3/10 14:31:14
categories:
- Tech
tags:
- Neo4j, NLP, 推荐系统, 论文, covid-19
---

## 前言
经过前面几篇官方文档的学习，Neo4j中的实体和关系之间的查询和计算，以及APOC的使用都过了一遍，接下来，我准备自己动手，贴合日常工作的业务，利用Neo4j构建一个基于内容和关系的论文推荐系统。

在现实世界中，医药工作者常常需要阅读大量的学术论文来提高自己的知识储备，了解前沿学术发展成果。

## 准备
生物科学相关的论文网站PUBMED上有成千上万的论文集，学术工作者通常利用关键字来搜索自己想获取的内容，这种方式虽然高效，但很容易忽略掉一些关联性高且重要的论文。

为了实验使用，我从pubmed上找了几篇和covid-19主题相关的论文，从以下两点用知识图谱构建推荐策略：
  
1. 利用Keywords，假设用户阅读了和covid-19和vaccine等关键词相关的内容，我们根据关键词和文章的关系来推荐其他相关的论文
2. 利用作者之间的关系，假设用户阅读了Steve的论文，那潜在的和Steve有合著论文关系作者的论文也有可能是用户所感兴趣的。
   
为了实验使用，我从Pubmed上简单复制下来几篇论文的Title，Authors，Abstract和Keywords作为实验数据，数据在我本地，如果需要，可以邮箱联系我。

下面我贴出代码，用于将数据加载进入Neo4j和创建实体之间的关系

```

# coding: utf-8
# author: xcfans
# This program is used for convert xlxs to neo4j format

import pandas as pd
from py2neo import Graph

graph = Graph(
    "bolt://127.0.0.1:7687",
    username="pubmed",
    password="123456"
)

def add_paper_entity():
    df = pd.read_excel('../data/covid-19.xlsx')
    for row in df.iterrows():
        t = row[1]['title']
        a = row[1]['abstract']
        #doi = row[1]['doi']
        #print("MERGE(p: Paper{title: '%s', abstract: '%s'})" % (t, a))

        graph.run('MERGE(p: Paper{title: "%s", abstract: "%s"})' % (t.replace("'","").strip(), a.replace("'","").strip()))


def add_keyword_entity():
    df = pd.read_excel('../data/covid-19.xlsx')
    for row in df.iterrows():

        keywords = str(row[1]['keywords']).split(';')
        for keyword in keywords:
            graph.run('MERGE(p: Keyword{name: "%s"})' % (keyword.replace("'","").strip()))

def add_author_entity():
    df = pd.read_excel('../data/covid-19.xlsx')
    for row in df.iterrows():

        authors = str(row[1]['authors']).split(',')
        for author in authors:
         
            graph.run('MERGE(p: Author{name: "%s"})' % (author.replace("'","").strip()))


def add_keyword_relation():
    df = pd.read_excel('../data/covid-19.xlsx')
    for row in df.iterrows():
        t = row[1]['title']
        a = row[1]['abstract']
        keywords = str(row[1]['keywords']).split(';')
        for keyword in keywords:
            if keyword == 'nan':
                continue
            print(keyword)
            a = graph.run(
                "MATCH(p: Paper), (k: Keyword) \
                WHERE p.title='%s' AND k.name='%s'\
                CREATE(p)-[rel:%s]->(kk)\
                RETURN rel" % (t.replace("'","").strip(), keyword.replace("'","").strip(), "keyword")
            )
            print(a)


def add_author_relation():
    df = pd.read_excel('../data/covid-19.xlsx')
    for row in df.iterrows():
        t = row[1]['title']
        authors = str(row[1]['authors']).split(',')
        for author in authors:
            if author == 'nan':
                continue
            print(author)
            a = graph.run(
                "MATCH(p: Paper), (a: Author) \
                WHERE p.title='%s' AND a.name='%s'\
                CREATE(p)-[rel:%s]->(a)\
                RETURN rel" % (t.replace("'","").strip(), author.replace("'","").strip(), "written_by")
            )
            print(a)

add_author_entity()
add_keyword_entity()
add_paper_entity()
add_keyword_relation()
add_author_relation()

```
我们看看结果：

`Match (p:Paper)-[r:written_by]-(a:Author) Return p,r,a`

![](https://ftp.bmp.ovh/imgs/2021/03/ba7767b49c82bf7b.png)


没问题，所有数据和关系已经倒入成功，那么利用上面的假设
> 1. 利用Keywords，假设用户阅读了和covid-19和vaccine等关键词相关的内容，我们根据关键词和文章的关系来推荐其他相关的论文
> 2. 利用作者之间的关系，假设用户阅读了Steve的论文，那潜在的和Steve有合著论文关系作者的论文也有可能是用户所感兴趣的。 


### 针对第一种假设，我们构造如下的Cypher:

```
Match (a:Paper{title:"Treatment for COVID-19: An overview"})-[:keyword]->(c:Keyword)<-[:keyword]-(b:Paper) Return b
```

我们尝试从图数据库中找到，与`Treatment for COVID-19: An overview` 拥有共同关键字的论文。

Result：

![](https://ftp.bmp.ovh/imgs/2021/03/9a0f8cc6e44a5543.png)

我们可以从上图中清楚的看到，结果中的两篇论文都被`Covid-19`的关键字为关系相连，达到了利用关键字作为实体间关系的推荐效果。

### 针对第二种假设，我们构造如下的Cypher：
为了实验效果，我们先找到哪两篇论文的作者是有合著关系的：

![](https://ftp.bmp.ovh/imgs/2021/03/2a1b9e0a887dabb8.png)

上图中，我们可以清楚的看到每篇论文的作者情况，也可以发现有部分作者不止贡献了一篇论文，因此我们可以挖掘出三种论文推荐给用户：

1. 合著作者的论文
2. 与合著作者有合著关系的作者的论文（2度关系），同样的道理，我们可以利用同理挖掘出潜在客户，以及客户的潜在关系。

![](https://ftp.bmp.ovh/imgs/2021/03/a7ec5e34d123a139.png)

Ok, 论文推荐的实验到此结束，拜拜啦！