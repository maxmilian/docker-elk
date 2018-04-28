Nginx Log Analytics by ELK
===========

# ELK stack 简介

![ELK Stack](https://github.com/maxmilian/docker-elk/blob/x-pack/images/elk_stack.png)

ELK 就是 Elasticsearch + Logstash + Kibana 的缩写。

Elasticsearch 是开源、分散式、Restful、JSON-based 的搜寻引擎。

Logstash 是一个开源的 server-side 数据管理管道，可同时從多个来源中获取数据，并将其转换，然后转发至目的地。

Kiban 是一个开源的 Elasticsearch 数据可视化插件。

三个结合起来，并把数据源(例如说 Nginx log)接上，并可成为强大的数据分析工具。

# Docker

使用 [docker-elk](https://github.com/deviantony/docker-elk) 来作为初始环境，并加入 Nginx 的相关设定。

# x-pack
因为要导入监控功能，所以使用 [x-pack](https://github.com/maxmilian/docker-elk/tree/x-pack) 分支，可以直接使用 docker-compose 来建立环境。

> NOTE: x-pack 为订阅功能，可供30日之试用


```bash
git checkout x-pack
docker-compose build
docker-compose up
```

> NOTE: [docker compose 说明](https://docs.docker.com/compose/)


预设上，会开启以下 ports

| port | Description                 |
|:----:|:---------------------------:|
| 5000 | Logstash TCP input          |
| 9200 | Elasticsearch HTTP          |
| 9300 | Elasticsearch TCP transport |
| 5601 | Kibana                      |


# Nginx
直接使用一简单的 Nginx 设定 *nginx/config/nginx.conf* ，并且修改其中的 log_format 如下：

```bash
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```

其中 logstash 导入 nginx log 的方式，这里使用 file，参考 *logstash/config/logstash.yml* ，此 path 应该为 nginx log 的输出位置。

```bash
input {
  file {
    path => "/nginx_logs/*"
    start_position => "end"
  }
}
```

设定完四个档案后，便可再次执行 docker-compose build && docker-compose up 来启动
- nginx/Dockerfile
- nginx/config/nginx.conf
- logstash/config/logstash.yml
- docker-compose.yml

启动后，即可以开启浏览器 http://localhost:8080/ 可以看到 nginx 画面
![Nignx](https://github.com/maxmilian/docker-elk/blob/x-pack/images/nginx.png)

# Kibana

启动后，开启 http://localhost:5601/ 进入 kibana，并用预设帐号密码登入 (设定于 *kibana/config/kibana.conf* )。

## Index Patterns
登入 kibana 后，设定完 Index Patterns 后，便可以于 Discover 分页中查看 Nginx log

![Index Patterns](https://github.com/maxmilian/docker-elk/blob/x-pack/images/index_patterns.png)

点选其一可以查看细节

![LegalMiner Log](https://github.com/maxmilian/docker-elk/blob/x-pack/images/legalminor_log_detail.png)


# Watcher
设定 [Watcher](https://www.elastic.co/guide/en/watcher/current/introduction.html)。于 Kibana 中 Management > Elasticsearch > Watcher ，可以新建一 Watcher

填入范例json

```json
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ".marvel-es-1-*",
        "types": "cluster_state",
        "body": {
          "query": {
            "bool": {
              "filter": [
                {
                  "range": {
                    "timestamp": {
                      "gte": "now-2m",
                      "lte": "now"
                    }
                  }
                },
                {
                  "terms": {
                    "cluster_state.status": ["green", "yellow", "red"]
                  }
                }
              ]
            }
          },
          "_source": [
            "cluster_state.status"
          ],
          "sort": [
            {
              "timestamp": {
                "order": "desc"
              }
            }
          ],
          "size": 1,
          "aggs": {
            "minutes": {
              "date_histogram": {
                "field": "timestamp",
                "interval": "5s"
              },
              "aggs": {
                "status": {
                  "terms": {
                    "field": "cluster_state.status",
                    "size": 3
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "throttle_period": "30m",
  "condition": {
    "script": {
      "inline": "if (ctx.payload.hits.total < 1) return false; def rows = ctx.payload.hits.hits; if (rows[0]._source.cluster_state.status != 'red') return false; if (ctx.payload.aggregations.minutes.buckets.size() < 12) return false; def last60Seconds = ctx.payload.aggregations.minutes.buckets[-12..-1]; return last60Seconds.every { it.status.buckets.every { s -> s.key == 'red' }}"
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "maxmilian@gmail",
        "subject": "Watcher Notification - Cluster has been RED for the last 60 seconds",
        "body": "Your cluster has been red for the last 60 seconds."
      }
    }
  }
}
```

![Wather](https://github.com/maxmilian/docker-elk/blob/x-pack/images/watcher.png)

其他许多范例监测可供使用 [Example Watches](https://github.com/elastic/examples/tree/master/Alerting/Sample%20Watches)



# 参考资料
- [Open Source Search & Analytics · Elasticsearch | Elastic](https://www.elastic.co/)
- [ELK docker-compose 全家桶实现Nginx access log 可视化 | Gyorou的草稿本](https://www.bocchi.tokyo/2017/02/13/elk-container-for-nginx-log/)
- [ELK 集群 Kibana 使用 X-Pack 权限控制，监控集群状态，警报，监视,cpu，内存，磁盘空间，报告和的可视化图形 - 搜云库 - SegmentFault 思否](https://segmentfault.com/a/1190000010981283)
