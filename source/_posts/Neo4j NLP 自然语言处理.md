---
title: Neo4j NLP 自然语言处理
date: 2021/3/3 15:32:14
categories:
- Tech
tags:
- Neo4j, NLP
---


## Natural Language Processing (自然语言处理)

Neo4j提供了对结构化的数据强有力的查询能力，但是真实世界中存在大量的文本数据，NLP技术可以从这些文本中
提取出潜在的结构化数据，该结构可能与表示句子中标记的节点一样简单，也可能与表示使用命名实体识别算法提取的实体的节点一样复杂。

## 为什么用NLP

从文本文件中提取出结构化的数据，然后存储到图数据库中后在多个场景中使用，包括：
- 基于内容的推荐系统
- 自然语言搜索
- 文档相似性

## 构建一个迷你内容推荐系统
接下来，我们使用Procedure为This Week in Neo4j (TWIN4j)构建一个推荐系统，内容是每周Neo4j的实时资讯，咨询包含了很多主题，意味着如果你喜欢其中的一个咨询主题，并不意味着你会喜欢它后面的一个资讯。

NLP Procedures 提供了对每一个资讯建立实体图的能力，这让我们可以推荐可能会喜欢阅读的资讯。

## 导入TWiIN4j blog posts
在我们做NLP的工作之前，先把TWIN4j 的blog导入进来，我们使用APOC的Load Json procedure 将这些资讯内容通过API导入进来。

1. Neo4j的交互命令中输入如下：
   ```
   CALL apoc.load.json("https://neo4j.com/wp-json/wp/v2/posts?tags=3201")
   YIELD value
   RETURN keys(value)
   LIMIT 1
   ```
   Result:
   ```
   ["date", "template", "_links", "link", "type", "title", "content", "featured_media", "modified", "id", "categories", "date_gmt", "slug", "modified_gmt", "author", "yst_prominent_words", "format", "comment_status", "yoast_head", "tags", "ping_status", "meta", "sticky", "guid", "excerpt", "status"]
   ```
   以上结果是每一条资讯所包含的字段

2. 以上得到的结果中`title`,`date`和`link`是我们重点关注的字段，创建一个Article标签的节点。
    ```
    CALL apoc.load.json("https://neo4j.com/wp-json/wp/v2/posts?tags=3201")
    YIELD value
    MERGE (a:Article {id: value.id})
    SET a.title = value.title.rendered,
    a.date = datetime(value.date),
    a.link = value.link;
    ```
3. 默认的API每页会返回10条内容，接下来用query为TWINN4J的实体创建节点，
我们使用APOC's Priodic Iterate procedure循环API的内容。
    ```
    CALL apoc.periodic.iterate(
    "UNWIND range(1,16) AS page RETURN page",
    "CALL apoc.load.json('https://neo4j.com/wp-json/wp/v2/posts?tags=3201&page=' + page)
    YIELD value
    MERGE (a:Article {id: value.id})
    SET a.title = value.title.rendered,
        a.date = datetime(value.date),
        a.link = value.link;
    ",
    {}
    );
    ```
    循环取出16页的数据，注意，官网给的代码`a.date = datetime(value.date)`后面少一个逗号，会报错，一定要在句尾加上句号。

4. 接下来验证一下数据是否导入成功。
    ```
    MATCH (:Article)
    RETURN count(*);
    ```
    Result:
    ```
    count(*)
    159
    ```
5. 接下来，我们用APOC 将HTML中的content内容提取出来存储在节点上。
    ```
    CALL apoc.periodic.iterate(
    "MATCH (a:Article) WHERE not(exists(a.body)) RETURN a",
    "CALL apoc.load.html(a.link, {body: 'div.entry-content'})
    YIELD value
    SET a.body = value.body[0].text",
    {batchSize: 10 });
    ```

  ## 使用APOC NLP procedures
NLP procedures 在我们安装APOC后不会被默认激活，我们需要将NLP dependencies的jar包放在plugin文件夹下面,切记下载与你neo4j数据库版本对应的NLP denpendencies

验证是否安装成功

`CALL apoc.help("nlp.aws");`

![](https://ftp.bmp.ovh/imgs/2021/03/a2860430d5d5486a.png)

## 用NLP procedures 取出数据实体

1. 先从AWS中拿到AWS access key ID 和 secrect access key,然后在neo4j的browser中定义好apiKey和apiSecret。
```
:parma apiKey => ('apiKey')
:parma apiSecret =>('apiSecret')
```

接下来，我们将每一篇资讯中的实体提取出来
AWS'NLP API 限制每个文档的最大size为5000bytes，因此我们首先要将小于这个限制的文档筛选出来。
```
MATCH (n:Article)
WHERE size(n.body) <= 5000
RETURN n.link, size(n.body) AS sizeInBytes, n.date
ORDER BY n.date DESC
LIMIT 5;
```
![](https://ftp.bmp.ovh/imgs/2021/03/0c331fc49efe42eb.png)

2. 返回文章的实体
```
MATCH (n:Article)
WHERE size(n.body) <= 5000
WITH n
ORDER BY n.date DESC
LIMIT 1
CALL apoc.nlp.aws.entities.stream(n, {
  key: "",
  secret: "",
  nodeProperty: "body"
})
YIELD value
UNWIND value.entities AS entity
RETURN DISTINCT entity.text, entity.type
LIMIT 10;

```
![](https://ftp.bmp.ovh/imgs/2021/03/a34737e8e2bb89f8.png)

3. 上面的结果返回了十个结果，下面的query使用procedure可视化这些实体。

```
MATCH (n:Article)
WHERE size(n.body) <= 5000
WITH n
ORDER BY n.date DESC
LIMIT 1
CALL apoc.nlp.aws.entities.graph(n, {
  key: "",
  secret: "",
  nodeProperty: "body",
  write: false
})
YIELD graph AS g
RETURN g;
```

![](https://ftp.bmp.ovh/imgs/2021/03/d8fb03e76e94cd82.jpg)

## 构建整个TWIN4J的实体图谱

有必要先解释一下TWIN4J不是语言也不是新的数据库，是This Week in Neo4J,neo4j的新闻资讯博客

```
MATCH (n:Article)
WHERE size(n.body) <= 4500
WITH collect(n) AS articles
CALL apoc.nlp.aws.entities.graph(articles, {
  key: "",
  secret: "",
  nodeProperty: "body",
  writeRelationshipType: "ENTITY",
  write: true
})
YIELD graph AS g
RETURN g;
```
Result:
![](https://ftp.bmp.ovh/imgs/2021/03/63ac1267c39b5d35.png)

## 哪些是最常见的实体？
```
MATCH (e:Entity)<-[:ENTITY]-()
RETURN e.text, labels(e) AS labels, count(*) AS occurrences
ORDER BY occurrences DESC
LIMIT 10;

```
Result:
![](https://ftp.bmp.ovh/imgs/2021/03/605088a28b60ca60.png)

## 为什么人名被频繁的提起
```
MATCH (e:Entity:Person)<-[:ENTITY]-(article)
RETURN e.text, count(*) AS occurrences, date(max(article.date)) AS lastMention
ORDER BY occurrences DESC
LIMIT 10;
```
Result:
![](https://ftp.bmp.ovh/imgs/2021/03/88a1e64c2d626624.png)


## 寻找每篇文章里最有关联的词
> TF-IDF(Term frequency - inverse document frequency), 是一种反映单个词在整个文档中的重要性的统计方法，通常被用作信息检索和文本挖掘中的权重因子。TF-IDF的值随着单词在文档中的出现频率升高而升高。但存在于语料库中的词的频率会被抵消。

Adam Cowley 最近写了一片文章 [ explaining how to calculate tf-idf scores using Cypher](https://adamcowley.co.uk/neo4j/calculating-tf-idf-score-cypher/)，因此我们可以借用他写的query来计算图中实体的权重。

先用id为119258的文章来测试
```
// 所有文章的数量
MATCH (:Article) WITH count(*) AS totalArticles

// 找到文章对应的所有关联实体
MATCH (a:Article {id: 119258})-[entityRel:ENTITY]-(e:Entity)

// 统计文章和实体
WITH a, e,
    totalArticles,
    size((a)-[:ENTITY]->(e)) AS occurrencesInArticle,
    size((a)-[:ENTITY]->()) AS entitiesInArticle,
    size(()-[:ENTITY]->(e)) AS articlesWithEntity

// Calculate TF and IDF
WITH a, e,
    totalArticles,
    1.0 * occurrencesInArticle / entitiesInArticle AS tf,
    log10( totalArticles / articlesWithEntity ) AS idf,
    occurrencesInArticle,
    entitiesInArticle,
    articlesWithEntity

// Combine together to return a result
RETURN a.id, e.text, tf * idf as tfIdf
ORDER BY tfIdf DESC
LIMIT 10;
```
Result:
![](https://ftp.bmp.ovh/imgs/2021/03/af7a3456da7a7936.png)

接下来，将计算TF-IDF所得到的Entity在文章中的权重保存为ENTITYREL的score属性
```
CALL apoc.periodic.iterate(
  "MATCH (:Article)
   WITH count(*) AS totalArticles
   MATCH (a:Article)
   RETURN a, totalArticles",
  "MATCH (a)-[entityRel:ENTITY]-(e:Entity)
   WITH a, e, entityRel,
        totalArticles,
        size((a)-[:ENTITY]->(e)) AS occurrencesInArticle,
        size((a)-[:ENTITY]->()) AS entitiesInArticle,
        size(()-[:ENTITY]->(e)) AS articlesWithEntity

   WITH a, e, entityRel,
        totalArticles,
        1.0 * occurrencesInArticle / entitiesInArticle AS tf,
        log10( totalArticles / articlesWithEntity ) AS idf,
        occurrencesInArticle,
        entitiesInArticle,
        articlesWithEntity

   SET entityRel.score = tf * idf",
  {}
);
```
计算完成后，我们可以找到每一篇文章权重最高的实体。
```
MATCH (a:Article)-[rel:ENTITY]->(e)
WITH a, e, rel
ORDER BY a.date DESC, rel.score DESC
RETURN date(a.date),  collect(e.text)[..10] AS entities
ORDER BY date(a.date) DESC
LIMIT 10;
```
Result:
![](https://ftp.bmp.ovh/imgs/2021/03/df5ddddaff1bc89e.png)

