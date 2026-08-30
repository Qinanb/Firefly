---

title: RAGFlow Elasticsearch + Kibana 图形化
published: 2026-08-30
description: RagFlow中向量存储图形化方案
image: ./gz.png
tags: [manual]
category: 工作台
draft: false
pinned: false

---

# RAGFlow Elasticsearch + Kibana 图形化

本文整理 RAGFlow v0.26.4 Docker 部署中，为 Elasticsearch 配置 Kibana，并在 Kibana 中查看 RAGFlow chunk 和向量字段的过程。

## 1. 架构和端口

~~~text
RAGFlow → Elasticsearch 8.11.3 → Kibana 8.11.3
~~~

Elasticsearch 保存 RAGFlow 的文本 chunk、元数据和 embedding；Kibana 提供查询和图形化界面。

本次端口映射：

| 服务 | 宿主机端口 | 容器端口 |
|---|---:|---:|
| Elasticsearch | 1202 | 9200 |
| Kibana | 6601 | 5601 |
| MinIO API | 9002 | 9000 |
| MinIO 控制台 | 9003 | 9001 |
| Redis | 6380 | 6379 |

宿主机端口可以修改，但容器内部通信端口不要改。例如 Kibana 连接 ES 使用 http://es01:9200，浏览器访问 ES 使用 http://服务器IP:1202。

## 2. 修改 .env

~~~bash
cd /data/ragflow/docker
nano .env
~~~

确认以下配置：

~~~env
DOC_ENGINE=elasticsearch
STACK_VERSION=8.11.3
ES_HOST=es01
ES_PORT=1202
KIBANA_PORT=6601
COMPOSE_PROFILES=elasticsearch,cpu,kibana
~~~

默认密码：

~~~env
ELASTIC_PASSWORD=infini_rag_flow
MYSQL_PASSWORD=infini_rag_flow
MINIO_PASSWORD=infini_rag_flow
REDIS_PASSWORD=infini_rag_flow
~~~

## 3. Elasticsearch 配置

编辑：

~~~bash
nano docker-compose-base.yml
~~~

es01 的安全相关配置保持为：

~~~yaml
- xpack.security.enabled=true
- xpack.security.enrollment.enabled=false
- xpack.security.http.ssl.enabled=false
- xpack.security.transport.ssl.enabled=false
~~~

说明：当前使用 HTTP + 用户认证，不使用 HTTPS，因此不能使用 Kibana enrollment token。不要为了生成 token 随意关闭安全认证。

启动并检查 ES：

~~~bash
docker compose up -d --force-recreate es01
docker compose ps
~~~

~~~bash
curl -u 'elastic:<ELASTIC_PASSWORD>' \
  'http://127.0.0.1:1202/_cluster/health?pretty'
~~~

正常应看到：

~~~json
"status" : "green"
~~~

## 4. 配置并启动 Kibana

### 4.1 生成 Kibana 系统用户密码

~~~bash
docker compose exec es01 \
  /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b
~~~

记录命令输出的密码。

### 4.2 把连接配置放在 kibana 服务下

在 docker-compose-base.yml 中找到 kibana:，将以下配置放入 Kibana 的 environment: 中：

~~~yaml
environment:
  - ELASTICSEARCH_HOSTS=http://es01:9200
  - ELASTICSEARCH_USERNAME=kibana_system
  - ELASTICSEARCH_PASSWORD=1Hs*Ly*7LF7-5cBu2a-6
~~~

完整结构类似：

~~~yaml
kibana:
  profiles:
    - kibana
  image: kibana:<STACK_VERSION>
  ports:
    - <KIBANA_PORT>:5601
  env_file: .env
  environment:
    - ELASTICSEARCH_HOSTS=http://es01:9200
    - ELASTICSEARCH_USERNAME=kibana_system
    - ELASTICSEARCH_PASSWORD=1Hs*Ly*7LF7-5cBu2a-6
~~~

重要：上述三行必须放在 kibana 下，不能放到 es01 的 environment 中。

启动 Kibana：

~~~bash
docker compose up -d --force-recreate kibana
docker compose logs --tail=100 kibana
~~~

看到以下日志说明 Kibana 已启动：

~~~text
http server running at http://0.0.0.0:5601
~~~

浏览器访问：

~~~text
http://服务器IP:6601
~~~

登录账号：

~~~text
用户名：elastic
密码：infini_rag_flow
~~~

不需要继续使用 enrollment token。

## 5. 创建 Data View

进入 Kibana：

~~~text
Discover → Create data view
~~~

推荐先创建主要 chunk 索引的 Data View：

~~~text
Name: RAGFlow Chunks
Index pattern: ragflow_5f07d1e892e511f1ac0ccb8de5a2f7dd
~~~

也可以使用：

~~~text
ragflow_*
~~~

但它可能同时匹配 ragflow_doc_meta_* 元数据索引。时间字段可以不选择：

~~~text
I don't want to use the time filter
~~~

保存后进入 Discover，即可查看 chunk。

## 6. 在 Discover 中确认向量

常见字段及作用：

| 字段 | 作用 |
|---|---|
| content_ltks | chunk 文本 |
| content_sm_ltks | 分词后的文本 |
| doc_id | 文档 ID |
| kb_id | 知识库 ID |
| chunk_order_int | chunk 顺序 |
| page_num_int | 页码 |
| q_1024_vec | 1024 维 embedding 向量 |

在左侧字段列表看到 q_1024_vec，即可确认该索引中有向量字段。Discover 适合查看 chunk 和字段；不适合直接阅读 1024 维向量或绘制向量聚类图。

在 Kibana 的 Dev Tools → Console 中可查看字段映射：

~~~http
GET ragflow_5f07d1e892e511f1ac0ccb8de5a2f7dd/_mapping
~~~

搜索：

~~~text
dense_vector
~~~

查看一条样例：

~~~http
GET ragflow_5f07d1e892e511f1ac0ccb8de5a2f7dd/_search
{
  "size": 1
}
~~~

## 7. 常见故障

### Connection reset by peer

通常是 Elasticsearch 正在重启或启动失败。检查：

~~~bash
docker compose ps
docker compose logs --tail=100 es01
~~~

### enrollment token 无法生成

如果报错提示 HTTP SSL 没有 keystore，这是因为：

~~~yaml
xpack.security.http.ssl.enabled=false
~~~

当前方案使用 kibana_system 用户手动配置 Kibana，不使用 enrollment token。

### .kibana 不存在

Kibana 首次启动时会自动创建 .kibana 及相关索引。只要最终日志显示 HTTP server running，通常不需要处理。

## 8. 快速检查清单

~~~bash
cd /data/ragflow/docker
docker compose ps
docker compose logs --tail=50 es01
docker compose logs --tail=50 kibana
curl -u 'elastic:infini_rag_flow' \
  'http://127.0.0.1:1202/_cluster/health?pretty'
~~~

浏览器地址：

~~~text
Kibana: http://211.90.219.102/:6601
ES:     http://211.90.219.102/:1202
~~~

