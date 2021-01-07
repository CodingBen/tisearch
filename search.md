[toc]

## TiDB 全文索引查询组件 Ti Search

**Author(s)**: 王思晋，朱琪，莫正华

### 全文索引的使用 需要做的的支持和使用


TiDB目标是和Mysql完全兼容，所以我们先看Mysql的标准:Mysql的全文检索有两种

###### 1.自然语言搜索模式,这是Mysql默认的搜索模式

这种情况下是默认的自然语言搜索模式
```sql
select * from fulltext_test where match(content,tag) against('xxx xxx');
```


```sql
select id,title FROM post WHERE MATCH(content) AGAINST ('search keyword' IN NATURAL LANGUAGE MODE)
```

和自然语言搜索模式相关的参数:


需要管理的参数 Token Size：

standard_token_size

max_token_size

min_token_size

```go
ft_min_word_len = 4;
ft_max_word_len = 84;
```


###### 2.布尔匹配

 
```txt
   ● IN BOOLEAN MODE的特色： 
      ·不剔除50%以上符合的row。 
      ·不自动以相关性反向排序。 
      ·可以对没有FULLTEXT index的字段进行搜寻，但会非常慢。 
      ·限制最长与最短的字符串。 
      ·套用Stopwords。

   ● 搜索语法规则：
     +   一定要有(不含有该关键词的数据条均被忽略)。 
     -   不可以有(排除指定关键词，含有该关键词的均被忽略)。 
     >   提高该条匹配数据的权重值。 
     <   降低该条匹配数据的权重值。
     ~   将其相关性由正转负，表示拥有该字会降低相关性(但不像-将之排除)，只是排在较后面权重值降低。 
     *   万用字，不像其他语法放在前面，这个要接在字符串后面。 
     " " 用双引号将一段句子包起来表示要完全相符，不可拆字。
```


##### 3. 查询扩展搜索 query expansion search

首先，MySQL全文搜索引擎查找与搜索词匹配的所有行。
第二步，检查搜索结果中的所有行并找到相关单词。
第三步，再次执行搜索，但是它是基于相关词语而不是用户提供的原始关键词。


###### 4.模糊查询 正则匹配 这个本身没有去调用全文索引，但是我们计划将其同样做到支持里。



```sql
# 即匹配姓名为“孙行者”，“行者孙，“行者孙”等包含“孙”类型的数据
SELECT * FROM character WHERE name LIKE ‘%孙%'；

SELECT * FROM character WHERE name LIke '%孙%' and name like '%行%';

#即匹配姓名为“孙行者”，“行者孙，“行者孙”等包含“孙”和“行”的数据


SELECT * FROM character WHERE name LIKE ‘_三'；
#即匹配姓名类似“...三”类型的数据，前面有且只有一个字符
```

1、LIKE'Mc%' 将搜索以字母 Mc 开头的所有字符串（如 McBadden）。
2、LIKE'%inger' 将搜索以字母 inger 结尾的所有字符串（如 Ringer、Stringer）。
3、LIKE'%en%' 将搜索在任何位置包含字母 en 的所有字符串（如 Bennet、Green、McBadden）。
4、LIKE'_heryl' 将搜索以字母 heryl 结尾的所有六个字母的名称（如 Cheryl、Sheryl）。
5、LIKE'[CK]ars[eo]n' 将搜索下列字符串：Carsen、Karsen、Carson 和 Karson（如 Carson）。
6、LIKE'[M-Z]inger' 将搜索以字符串 inger 结尾、以从 M 到 Z 的任何单个字母开头的所有名称（如 Ringer）。
7、LIKE'M[^c]%' 将搜索以字母 M 开头，并且第二个字母不是 c 的所有名称（如MacFeather）

```go
	ScalarFuncSig_LikeSig       ScalarFuncSig = 4310
	ScalarFuncSig_RegexpSig     ScalarFuncSig = 4311
	ScalarFuncSig_RegexpUTF8Sig ScalarFuncSig = 4312
```

需要修改的入口

### Ti-Search需要的概念

创建全文索引
```sql
create fulltext index ft_idx on article(title,body)；
```

创建全文索引并且指定分词器
```sql
create fulltext index ft_idx on article(title,body) with PARSER xxx;
```



由上的信息可得到Mysql本身有两个主要部分,Index和Parser
Index的本身名字，相关table的名字，column属性。
Parser，分词器类型，tokenSize，stopWords等。
status，该Index状态是否已经OK。



#### Index 定义

Index定义分为两个部分:

Index本身名字，相关的表名字

获取Index信息
方法:GET
URI:/tisearch/index/{indexName}

-创建Index信息
方法:POST
URI:/tisearch/index/{indexName}

更新Index信息
方法：PUT
URI:/tisearch/index/{indexName}

删除Index
方法:delete
URI:/tisearch/index/{indexName}

关于body的定义都是统一的:

```json
{
   "indexName":"AAA",
   "Parser": {
      "ParserType":"like",
      "tokenSize": 2,
      "stopWords": "",
      ...
   },
   "status": "indexing",
   "table":"xxx",
   "colunm":"xxx"
}
```

##### Ti-Search之下对于Solar和Es的封装


### 讨论问题

##### 1.TiDB是否需要自己定义分词器
我们针对性总结TiDB需要的分词器的的基本属性，然后对应去映射实现到Solar/Es的分词插件。

这个地方需要的属性需要和@莫正华一起讨论。

**讨论结果：** 这个地方我们先自定义分词名字为like，特性和IK差不多


##### 2.此次全文索引查询支持做到哪几种方案？

因为如上面提高的，全文索引会被用到的四种场景:
1.自然语言搜索 match agsinst

2.自然语言搜索扩展 

3.布尔匹配

4.模糊正则查询

我们此次在hackthon需要支持的目标是全部做完，还是选择其中几种。

**讨论结果：** 我们先完成第一种自然语言搜索的match against。

##### 3.TiDB向Ti Search传递的查询格式：

**讨论结果：**

查询格式如下
```json
{
    "index_name":"",
    "search_fileds":[
        {
            "field":"filed1",
            "word":"word1"
        },
        {
            "filed":"filed2",
            "word":"wordword"
        },
        {
            "filed":"filed3",
            "word":"w"
        }
    ],
    "mode":"NATURAL_LANGUAGE",
    "store_fileds":[
        "filed1",
        "filed2"
    ],
    "limit":10
}
```


##### 4.整体结构设计

![查询结构](https://ben-space-1252588607.cos.ap-shanghai.myqcloud.com/img/20210107133322.png)
