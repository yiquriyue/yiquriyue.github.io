---
layout: post
title: 'elasticsearch使用'
date: '2019-01-16'
header-img: "img/home-bg.jpg"
tags:
     - python
     - note
author: 'yiquriyue'
---

# elasticsearch使用
## 查询
### 游标查询
```
import elasticsearch
es_servers = [{
            "host":es_host,
            "port":es_port
        }]
es_client = elasticsearch.Elasticsearch(hosts=es_servers)
my_body = {"sort": {"timestamp": "desc"}, "query": {"bool": {"filter": {"range": {"timestamp": {"gte": start_time, "lte": end_time}}}}},"size": 100}
# 这里的size里面指定了每次查询的个数
queryData = es_client.search(
            index='caoyue_test',
            doc_type='doc',
            body=my_body,
            scroll='10m',
            request_timeout=60
        )
# 这里需要指定scroll，否则会出错
mdata = queryData.get("hits").get("hits")
if not mdata:
    print 'the index in ',start_time,'to ',end_time,'is :empty!'
scroll_id = queryData["_scroll_id"]
total = queryData["hits"]["total"]
for i in range((total/100)+1):
    res = es_client.scroll(scroll_id=scroll_id,scroll='10m')
    # 这根据上面size的设定，每次取100条，因此可以确定需要取(total/100)+1次
    print res
    # res即为查询的100条数据，通过循环可以得到所有数据
    
```
### 聚类	
#### 过滤
```
{
	"sort": {
		"timestamp": "desc"
	},
	"query": {
		"bool": { # bool过滤是精确查询
			"filter": {
				"range": {
					"timestamp": { # 指定比较的取值域
						"gte": "2019-01-14T02:23:42", # 比较过滤，这里是上限
						"lte": "2019-01-14T02:23:44" # 这里是下限 
					}
				}
			}
		}
	},
	"size": 100,
	"aggs": { # 对query结果使用聚合，并放入新的dict中
		"all_ip": { # 这里是将查询结果放入的dict的key，由用户自定义
			"terms": {
				"field": "details.network.src.ip.keyword",# 以details.network.src.ip对应的值作为聚类对象，如果要以key作为聚类对象则将keyword去掉
				"size":100 # 设置页大小
			}
		}
	}
}
```

#### 后过滤器
**当搜索与聚合同时使用时，后过滤器用来，只过滤搜索结果，不过滤聚合结果**

```
GET /cars/transactions/_search
{
    "size" : 0,
    "query": {
        "match": {
            "make": "ford"
        }
    },
    "post_filter": {  # 聚合后对聚合结果进行过滤，为了提高首次过滤的效率
        "term" : {
            "color" : "green"
        }
    },
    "aggs" : {
        "all_colors": {
            "terms" : { "field" : "color" }
        }
    }
}
```