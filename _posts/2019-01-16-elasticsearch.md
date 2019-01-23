---
layout: post
title: 'elasticsearch使用'
date: '2019-01-16'
header-img: "img/home-bg.jpg"
tags:
     - database
     - note
author: 'yiquriyue'
---

# elasticsearch使用
## 插入数据

```python
from elasticsearch.helpers import bulk
import elasticsearch
# 批量删除时，存在编码错误，尝试切换系统默认编码格式，得以解决
import sys
reload(sys)
sys.setdefaultencoding('utf8')
es_host = 'localhost'
es_port = 9200
class ElasticSearchClient(object):
    @staticmethod
    def get_es_servers():
        es_servers = [{
            "host":es_host,
            "port":es_port
        }]
        es_client = elasticsearch.Elasticsearch(hosts=es_servers)
        return es_client
class LoadElasticSearch(object):
    def __init__(self, index=None, doc_type=None):
        self.index = index
        self.doc_type = doc_type
        self.es_client = ElasticSearchClient.get_es_servers()
        
    def add_date(self, row_obj):
        """
        单条插入ES
        """
        if "_id" in row_obj:
            _id = row_obj.get("_id", 1)
            row_obj.pop("_id")
        else:
            _id = None
        self.es_client.index(index=self.index, doc_type=self.doc_type, body=row_obj, id=_id)
        
    def add_date_bulk(self,body,num):
        for i in range(num):
            action = {
                "_index":"caoyue_test",
                "_type":"doc",
                # 这里的id有限制，存在插入失败的可能，失败时，尝试将id删除
                "_id":i,
                "_source":body
            }
            actions.append(action)
            if(len(actions)==500):
                bulk(self.es_client, actions)
                del actions[0:len(actions)]
        if (len(actions) > 0):
            # 返回成功个数和失败个数
            return bulk(self.es_client, actions)

if (len(actions) > 0):
    print bulk(es.es_client, actions)
# 这是通过流量捕获的一条数据，可以参考格式
body = {
          "node_ip": "",
          "node_id": "",
          "source": "ids_flow",
          "actor_hash": "635cb3df206700321ea277cfbbcc1e10",
          "event_hash": "e7c65ffd0e3a1e8d61e8d62f3f490ff8",
          "timestamp": "2019-01-17T18:30:39",
          "collection_time": "2019-01-17T16:30:39.605325+0800",
          "details": {
            "network": {
              "src": {
                "ip": "172.31.50.224",
                "port": 49934,
                "location": {
                  "city": "",
                  "country": "局域网",
                  "iso_code": "",
                  "latitude": 0,
                  "longitude": 0,
                  "province": "",
                  "time_zone": ""
                }
              },
              "dst": {
                "ip": "118.190.126.180",
                "port": 80,
                "location": {
                  "city": "杭州",
                  "country": "中国",
                  "iso_code": "CN",
                  "latitude": 30.2936,
                  "longitude": 120.1614,
                  "province": "浙江省",
                  "time_zone": "Asia/Shanghai"
                }
              },
              "protocol": {
                "transport": "TCP",
                "application": {
                  "protocol": "http",
                  "http": {
                    "hostname": "s.ludashi.com",
                    "url": "/url2?pid=baidu&mid=6bf7836b9a817e98472c6deeabd45c40&appver=5.15.16.1305&modver=2.0.0.1080&type=HpSvc&action=HpSvc"
                  },
                  "dns": {
                    "rrname": ""
                  }
                }
              }
            },
            "behavior": {}
          },
          "threat": {
            "category": "拒绝服务攻击",
            "phase": "Invasion",
            "org": "",
            "type": [
              "DDoS"
            ],
            "c2": "118.190.126.180:80",
            "asset_status": "疑似失陷",
            "direction": {
              "source": "118.190.126.180",
              "target": "172.31.50.80"
            },
            "signature": {
              "id": "100000001",
              "family": "Unknown",
              "cve": "",
              "refer": "",
              "behavior": [
                "CompleteRequestC2Domain"
              ],
              "level": "HighRisk"
            }
          }
        }
es = LoadElasticSearch('caoyue_test','doc')
# 插入单条数据
es.add_date(body)
# 重复插入body1000次
es.add_date_bulk(body,1000)
```

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