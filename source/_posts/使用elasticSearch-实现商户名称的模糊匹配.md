---
title: 使用elasticSearch 实现商户名称的模糊匹配
date: 2018-09-07 15:33:04
tags: [elasticSearch,es,term,ik]
---

## 需求背景
> 总公司那边不允许输出流水中的刷卡地址,所以应对的方案是允许客户输入商户的关键字然后模糊匹配刷卡地址中是否存在

<!--more-->


## 技术选型

> 先从数据量着手选择技术,mysql 通过like来匹配肯定不能满足如此大的数据量,存量数据量来看已经有六千万之多,所以mysql技术不行,而且mysql仅仅是解决了包含的匹配关系,对于关键词和查询词组的相关度支持有限,另一方面考虑到数据的复用行,另一项目中数据已经存于es中.

## 技术实现

>开始之初,选择的是已经做好IK 分词的刷卡地址,但是无论使用wildcardQuery还是fuzzyQuery亦或prefixQuery热词还好,但是出现了冷门词语(是否包含与既定的分词包中),就会出现某个汉字多排名就考前的弊病,而且默认返回十条,这样就限制了包含模糊查询的二次匹配,次路不通.
>后来鉴于上文提及的已做好分词命中不固定的原因,选择增加一个字段数据导入的时候不做分词(keyword)

+ 知识点
>5.x以上已经没有string类型。如果需要分词的话使用text，不需要分词使用keyword。但是生成索引的时候你会看到后面还有一个关键词:index,默认为true,如果设置为false,该字段将不能被索引,keyword是整个文档被索引,text根据指定的分词器被索引(不指定的话默认的分词器是standard),
>keyword,index:true=>doc被每个term切词并与doc建立索引，可以用于模糊匹配， keyword,index:fals则doc不被切词，所以你要做的就是要么使用keyword,index:true的字段模糊匹配或者使用text类型的字段的全文检索匹配过滤，然后这些不被切词（查询项（二次查找（schame）））和被切词的字段作为查询项返回.
>使用了keyword和index:false,该字段只能是作为查询项来返回,用法如下:

``` java
		MatchQueryBuilder termQueryBuilder = QueryBuilders.
				matchQuery("content", "中国渔船").analyzer("ik_max_word");
		String indexes = "CP5178,CP5177,CP5176";
		String[] includesIndexes = indexes.split(",");
		String[] excludesIndexes = new String[]{};
		SearchResponse searchResponse = client.prepareSearch(INDEX).setTypes(TYPE).setQuery(termQueryBuilder)
				.setFetchSource(includesIndexes, excludesIndexes).execute().actionGet();
```
>综上所述我们需要得到的最终技术实现

 1. 使用keyword,index:true配合wildcardQuery实现模糊匹配的查找(like "%key%")
 2. 使用text字段配合分词器实现长关键词的二次匹配,搜索的关键词越长,关联度越高.
 
 
+ 上代码

``` java
       String mid = "A4816456E30D07B17C23B8D16FC26CCEC0E7EBE10A7554732A25753EBE533569";
        String searchKey = "南通新时代电器科技有限公司";

        MatchQueryBuilder merId = QueryBuilders.matchQuery("mid", mid);


        BoolQueryBuilder midMust = QueryBuilders.boolQuery().must(merId);


        SearchResponse searchResponse = client.prepareSearch(INDEX).setTypes(TYPE).setQuery(midMust).execute().actionGet();

        SearchHit[] hits = searchResponse.getHits().getHits();
        String msg = "未找到";

        if (0 < hits.length) {
            WildcardQueryBuilder mchntName = QueryBuilders.wildcardQuery("name", "*" + searchKey + "*");
            QueryBuilders.prefixQuery()
            BoolQueryBuilder mchntNameMust = QueryBuilders.boolQuery().must(merId).must(mchntName);
            searchResponse = client.prepareSearch(INDEX).setTypes(TYPE).setQuery(mchntNameMust).execute().actionGet();
            hits = searchResponse.getHits().getHits();
            if (0 < hits.length) {
                for (SearchHit searchHit : hits) {
                    System.out.println(searchHit.getSourceAsMap().get("name"));
                }
                msg = "匹配成功";
            } else if (4 < searchKey.length()) {
                MatchQueryBuilder nameQuery = QueryBuilders.
                        matchQuery("name_term", searchKey).analyzer("ik_max_word");
                BoolQueryBuilder nameMust = QueryBuilders.boolQuery().must(merId).must(nameQuery);
                searchResponse = client.prepareSearch(INDEX).setTypes(TYPE).setQuery(nameMust).execute().actionGet();
                hits = searchResponse.getHits().getHits();
                if (0 < hits.length) {
                    for (SearchHit searchHit : hits) {
                        System.out.println(searchHit.getSourceAsMap().get("name_term"));
                    }
                    msg = "匹配成功";
                }

            }
        } else {
            System.out.println("未匹配到商户名");
        }

        System.out.println(msg);


```
