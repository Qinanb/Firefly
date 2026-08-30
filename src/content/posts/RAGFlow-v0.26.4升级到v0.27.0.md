---
title: RAGFlow v0.26.4 升级到 v0.27.0 完整操作记录
published: 2026-08-30
description: RAGFlow版本升级
image: ./gz.png
tags: [manual]
category: 工作台
draft: false
pinned: false

---



# RAGFlow v0.26.4 升级到 v0.27.0 完整操作记录

本文记录本次 Linux Docker Compose 环境中的升级过程，并整理后续服务器升级、备份、回滚和数据库兼容问题的处理方法。

官方文档：https://ragflow.com.cn/docs/upgrade_ragflow

## 1. 基本信息和升级原则

本次部署目录：

~~~bash
cd /data/ragflow/docker
~~~

服务：

~~~text
ragflow-cpu  RAGFlow 应用
mysql        MySQL 元数据数据库
minio        原始文件和对象存储
es01         Elasticsearch 文档和向量索引
redis        Redis/Valkey
~~~

升级前镜像：

~~~text
infiniflow/ragflow:v0.26.4
~~~

升级后镜像：

~~~text
infiniflow/ragflow:v0.27.0
~~~

RAGFlow 的核心数据不在应用容器本身，而在 MySQL、MinIO 和 Elasticsearch 数据卷中。因此升级时只更新 RAGFlow 应用镜像，保留原有数据服务和数据卷。

生产环境不要执行：

~~~bash
docker compose down -v
~~~

该命令可能删除 Docker Compose 管理的数据卷。

## 2. 升级前检查

~~~bash
cd /data/ragflow/docker
docker compose ps
docker compose images
~~~

保存配置：

~~~bash
docker compose config > compose-v0.26.4.backup.yml
cp .env .env.v0.26.4.backup
cp docker-compose.yml docker-compose.v0.26.4.backup.yml
cp init.sql init.v0.26.4.backup.sql
~~~

查看 Compose 声明的卷：

~~~bash
docker compose config --volumes
~~~

查看实际卷：

~~~bash
docker volume ls
~~~

本次使用的实际卷：

~~~text
docker_mysql_data
docker_minio_data
docker_esdata01
docker_redis_data
docker_kibana_data
~~~

确认挂载：

~~~bash
docker inspect docker-mysql-1 --format '{{range .Mounts}}{{.Type}} | {{.Name}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect docker-minio-1 --format '{{range .Mounts}}{{.Type}} | {{.Name}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect docker-es01-1 --format '{{range .Mounts}}{{.Type}} | {{.Name}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
~~~

本次确认：

~~~text
docker_mysql_data -> /var/lib/mysql
docker_minio_data -> /data
docker_esdata01   -> /usr/share/elasticsearch/data
~~~

检查空间：

~~~bash
df -h /data
du -sh /data/docker/volumes/docker_minio_data/_data
~~~

## 3. 升级前备份

创建目录：

~~~bash
BACKUP_DIR=/data/ragflow-backup/$(date +%F-%H%M%S)
mkdir -p "$BACKUP_DIR"
echo "$BACKUP_DIR"
~~~

备份配置：

~~~bash
cp .env "$BACKUP_DIR/"
cp docker-compose.yml "$BACKUP_DIR/"
cp init.sql "$BACKUP_DIR/"
docker compose config > "$BACKUP_DIR/compose-rendered.yml"
~~~

备份 MySQL：

~~~bash
docker compose exec -T mysql mysqldump \
  -uroot -p'你的MYSQL_ROOT_PASSWORD' \
  --single-transaction --routines --triggers \
  rag_flow > "$BACKUP_DIR/rag_flow.sql"
~~~

停止服务：

~~~bash
docker compose stop
~~~

备份卷：

~~~bash
docker run --rm -v docker_mysql_data:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/mysql-volume.tar.gz -C /data .

docker run --rm -v docker_minio_data:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/minio-volume.tar.gz -C /data .

docker run --rm -v docker_esdata01:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/es-volume.tar.gz -C /data .
~~~

tar 默认不显示进度。MinIO 数据量大时长时间没有输出不一定是卡住。本次 MinIO 压缩备份约 3.3G，最终成功。

验证：

~~~bash
ls -lha "$BACKUP_DIR"
tar -tzf "$BACKUP_DIR/mysql-volume.tar.gz" >/dev/null && echo "MySQL卷备份完整"
tar -tzf "$BACKUP_DIR/minio-volume.tar.gz" >/dev/null && echo "MinIO卷备份完整"
tar -tzf "$BACKUP_DIR/es-volume.tar.gz" >/dev/null && echo "ES卷备份完整"
~~~

至少应有：

~~~text
.env
docker-compose.yml
compose-rendered.yml
init.sql
rag_flow.sql
mysql-volume.tar.gz
minio-volume.tar.gz
es-volume.tar.gz
~~~

## 4. 更新镜像

查找生效配置：

~~~bash
grep -R "v0.26.4\|RAGFLOW_IMAGE" .
~~~

只修改 .env 中真正生效的配置：

~~~bash
sed -i 's#^RAGFLOW_IMAGE=infiniflow/ragflow:v0.26.4#RAGFLOW_IMAGE=infiniflow/ragflow:v0.27.0#' .env
grep '^RAGFLOW_IMAGE=' .env
docker compose config | grep -n 'infiniflow/ragflow'
~~~

拉取并启动：

~~~bash
docker compose pull ragflow-cpu
docker compose up -d
docker compose ps
docker inspect docker-ragflow-cpu-1 --format '{{.Config.Image}}'
~~~

预期：

~~~text
infiniflow/ragflow:v0.27.0
~~~

查看启动：

~~~bash
docker compose logs -f ragflow-cpu
~~~

## 5. 本次遇到的问题和修复

### 5.1 docker inspect 模板缺少结束标记

错误：

~~~text
template parsing error: template: :1: unexpected EOF
~~~

正确模板必须包含最后的结束标记：

~~~bash
docker inspect docker-mysql-1 --format '{{range .Mounts}}{{.Type}} | {{.Name}} | {{.Source}} -> {{.Destination}}{{println}}{{end}}'
~~~

### 5.2 缺少 tenant_ocr_id

错误：

~~~text
OperationalError: (1054, "Unknown column 't1.tenant_ocr_id' in 'field list'")
~~~

原因是 v0.27.0 的 tenant 表新增字段，旧数据库没有完整迁移。

修复：

~~~bash
docker compose stop ragflow-cpu
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "ALTER TABLE tenant ADD COLUMN tenant_ocr_id VARCHAR(32) NULL, ADD INDEX idx_tenant_ocr_id (tenant_ocr_id);"
docker compose up -d ragflow-cpu
~~~

### 5.3 schema 同步脚本缺少 peewee_migrate

错误：

~~~text
ModuleNotFoundError: No module named 'peewee_migrate'
~~~

原因是当前 v0.27.0 镜像缺少该 Python 依赖，官方 db_schema_sync.py 无法运行。

不要把容器中临时 pip install 当作生产修复，因为容器重建后会丢失。对于明确的字段，可以在已有备份基础上使用定向 SQL。

### 5.4 model_type 字符串和整数不兼容

错误：

~~~text
TypeError: unsupported operand type(s) for &: 'str' and 'int'
~~~

原因是旧表 tenant_model.model_type 为 varchar(32)，数据是字符串；v0.27.0 改为整数位掩码。

映射：

~~~text
chat         -> 1
embedding    -> 2
image2text   -> 8
rerank       -> 16
~~~

备份：

~~~bash
docker compose exec -T mysql mysqldump -uroot -p'密码' rag_flow tenant_model > /data/ragflow-backup/model-type-fix/tenant_model-before.sql
~~~

转换并修改类型：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "UPDATE tenant_model SET model_type = CASE model_type WHEN 'chat' THEN '1' WHEN 'embedding' THEN '2' WHEN 'image2text' THEN '8' WHEN 'rerank' THEN '16' ELSE model_type END;"
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "ALTER TABLE tenant_model MODIFY model_type INT NOT NULL;"
~~~

### 5.5 Embedding 使用旧数字 ID

错误：

~~~text
101 Field: embedding_model
Message: embedding model identifier must follow model_name@provider format
Value: 776
~~~

原因是 tenant_embd_id 仍是旧数字 ID，而 v0.27.0 需要 tenant_model.id。

本次正确模型 ID：

~~~text
776be3aa947011f1ac0ccb8de5a2f7dd
~~~

修复：

~~~bash
docker compose exec -T mysql mysqldump -uroot -p'密码' rag_flow tenant > /data/ragflow-backup/model-type-fix/tenant-before-embd-fix.sql
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "ALTER TABLE tenant MODIFY tenant_embd_id VARCHAR(32) NULL;"
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "UPDATE tenant SET tenant_embd_id='776be3aa947011f1ac0ccb8de5a2f7dd' WHERE id='5f07d1e892e511f1ac0ccb8de5a2f7dd' AND tenant_embd_id='776';"
~~~

### 5.6 LLM、VLM、Rerank 默认模型显示旧数字

界面显示：

~~~text
LLM       369
VLM       2147483647
Rerank    5
Embedding 776
~~~

原因是 tenant_llm_id、tenant_img2txt_id、tenant_rerank_id 也保留了旧引用。

本次映射：

| 类型 | 新 tenant_model.id |
|---|---|
| LLM qwen3:32b Ollama | 369f5bf49d1211f1869a59ebce91e2b5 |
| Embedding bge-m3 Ollama | 776be3aa947011f1ac0ccb8de5a2f7dd |
| VLM qwen2.5vl:7b Ollama | 776e98ca947011f1ac0ccb8de5a2f7dd |
| Rerank Qwen3-Reranker-4B Xinference | 5da014e6947011f1ac0ccb8de5a2f7dd |

检查字段：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "SHOW COLUMNS FROM tenant WHERE Field IN ('tenant_llm_id','tenant_img2txt_id','tenant_rerank_id','tenant_embd_id');"
~~~

本次发现：

~~~text
tenant_llm_id       int
tenant_img2txt_id   int
tenant_rerank_id    int
tenant_embd_id      varchar(32)
~~~

调整字段类型：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "ALTER TABLE tenant MODIFY tenant_llm_id VARCHAR(32) NULL, MODIFY tenant_img2txt_id VARCHAR(32) NULL, MODIFY tenant_rerank_id VARCHAR(32) NULL;"
~~~

更新引用：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "UPDATE tenant SET tenant_llm_id='369f5bf49d1211f1869a59ebce91e2b5', tenant_img2txt_id='776e98ca947011f1ac0ccb8de5a2f7dd', tenant_rerank_id='5da014e6947011f1ac0ccb8de5a2f7dd' WHERE id='5f07d1e892e511f1ac0ccb8de5a2f7dd';"
~~~

### 5.7 重启后短暂 502

错误：

~~~text
请求错误 502: undefined 网关错误
~~~

原因是容器启动后，Python API、Admin、数据同步和任务执行器仍在初始化。

检查：

~~~bash
docker compose ps
docker compose logs --since=5m ragflow-cpu
~~~

等待出现：

~~~text
RAGFlow admin is ready
RAGFlow server is ready
RAGFlow ingestion is ready
~~~

再刷新页面，必要时使用 Ctrl+F5。刚重启后的短暂 502 不应立即触发回滚。

## 6. 检查所有租户

检查旧数字引用：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "SELECT id,name,tenant_llm_id,tenant_embd_id,tenant_img2txt_id,tenant_rerank_id FROM tenant WHERE tenant_llm_id REGEXP '^[0-9]+$' OR tenant_embd_id REGEXP '^[0-9]+$' OR tenant_img2txt_id REGEXP '^[0-9]+$' OR tenant_rerank_id REGEXP '^[0-9]+$';"
~~~

检查失效引用：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "SELECT t.id,t.name,t.tenant_llm_id,t.tenant_embd_id,t.tenant_img2txt_id,t.tenant_rerank_id FROM tenant t LEFT JOIN tenant_model llm ON t.tenant_llm_id=llm.id LEFT JOIN tenant_model embd ON t.tenant_embd_id=embd.id LEFT JOIN tenant_model vlm ON t.tenant_img2txt_id=vlm.id LEFT JOIN tenant_model rerank ON t.tenant_rerank_id=rerank.id WHERE (t.tenant_llm_id IS NOT NULL AND llm.id IS NULL) OR (t.tenant_embd_id IS NOT NULL AND embd.id IS NULL) OR (t.tenant_img2txt_id IS NOT NULL AND vlm.id IS NULL) OR (t.tenant_rerank_id IS NOT NULL AND rerank.id IS NULL);"
~~~

本次两个查询均无返回数据，说明所有租户的默认模型引用关系有效。

检查模型类型：

~~~bash
docker compose exec -T mysql mysql -uroot -p'密码' rag_flow -e "SELECT model_type,COUNT(*) AS total FROM tenant_model GROUP BY model_type ORDER BY model_type;"
~~~

本次结果：

~~~text
1   15
2    3
8    2
16   3
~~~

SQL 检查只能确认数据库引用正确，不能确认 API Key、Base URL、模型服务和显存状态。仍需实际测试聊天、解析、向量检索和 Rerank。

## 7. 升级后验证

~~~bash
docker compose ps
docker compose logs --since=5m ragflow-cpu
~~~

应确认：

~~~text
MySQL: healthy
MinIO: healthy
Elasticsearch: healthy
Redis/Valkey: healthy
RAGFlow: Up
RAGFlow Admin: ready
RAGFlow Server: ready
RAGFlow Ingestion: ready
~~~

Web 端测试：

1. 登录。
2. 用户、租户和团队。
3. 知识库和文档数量。
4. 文档原文和切片。
5. 关键词检索。
6. 向量检索。
7. 聊天助手。
8. LLM、Embedding、VLM、Rerank。
9. Agent。
10. 新上传文件解析。

## 8. 后续升级命令模板

~~~bash
cd /data/ragflow/docker

docker compose ps
docker compose images

BACKUP_DIR=/data/ragflow-backup/$(date +%F-%H%M%S)
mkdir -p "$BACKUP_DIR"

cp .env "$BACKUP_DIR/"
cp docker-compose.yml "$BACKUP_DIR/"
docker compose config > "$BACKUP_DIR/compose-rendered.yml"

docker compose exec -T mysql mysqldump \
  -uroot -p'你的MYSQL_ROOT_PASSWORD' \
  --single-transaction --routines --triggers \
  rag_flow > "$BACKUP_DIR/rag_flow.sql"

docker compose stop

docker run --rm -v docker_mysql_data:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/mysql-volume.tar.gz -C /data .

docker run --rm -v docker_minio_data:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/minio-volume.tar.gz -C /data .

docker run --rm -v docker_esdata01:/data:ro -v "$BACKUP_DIR":/backup alpine \
  tar czf /backup/es-volume.tar.gz -C /data .

# 手动修改 .env 中的 RAGFLOW_IMAGE

docker compose config
docker compose pull ragflow-cpu
docker compose up -d
docker compose ps
docker compose logs --since=5m ragflow-cpu
~~~

## 9. 回滚原则

如果只是应用启动失败且数据库尚未迁移，可将 .env 改回 v0.26.4：

~~~env
RAGFLOW_IMAGE=infiniflow/ragflow:v0.26.4
~~~

然后：

~~~bash
docker compose up -d
~~~

如果 v0.27.0 已修改数据库结构，不建议直接降级。应停止服务，恢复升级前的 MySQL、MinIO、Elasticsearch 备份和旧配置，再启动 v0.26.4。

## 10. 最终结果

~~~text
RAGFlow: infiniflow/ragflow:v0.27.0
MySQL: healthy
MinIO: healthy
Elasticsearch: healthy
Redis/Valkey: healthy
RAGFlow Admin: ready
RAGFlow Server: ready
RAGFlow Ingestion: ready
~~~

原有 MySQL、MinIO 和 Elasticsearch 数据卷均保留，知识库和文档数据未删除。通过补充数据库字段、转换模型类型、扩展旧字段长度以及修复租户模型引用，解决了 v0.27.0 与旧数据库结构之间的兼容问题。

---
## 11. 各类问题的详细根因分析

### 11.1 总体原因：应用版本和数据库历史结构发生漂移

本次问题的共同根因不是 Docker 数据卷丢失，而是：

~~~text
RAGFlow 应用代码已经升级到 v0.27.0
但数据库中的部分字段、字段类型和历史模型数据仍保留旧版本格式
~~~

新版本代码按照新的字段和模型引用规则查询 MySQL。当新代码读取旧结构时，就会出现 Unknown column、Data truncated、TypeError 和模型显示不完整等问题。

RAGFlow 升级不只是替换应用镜像，还可能包含：

- 新增数据库字段；
- 修改字段类型；
- 修改模型类型的存储方式；
- 修改租户默认模型的引用方式；
- 修改模型 ID 的长度和语义。

### 11.2 tenant_ocr_id 为什么缺失

错误：

~~~text
OperationalError: (1054, "Unknown column 't1.tenant_ocr_id' in 'field list'")
~~~

v0.27.0 的 tenant 表模型新增了 tenant_ocr_id。旧数据库没有该字段，但新代码已经把它当作 tenant 表的一部分读取，于是 MySQL 返回 1054 Unknown column。

这类问题的本质是：

~~~text
应用代码需要的字段集合，大于数据库实际拥有的字段集合
~~~

补充允许为空的字段和索引是安全的，因为旧租户没有 OCR 模型时可以保存 NULL，不会修改已有文档、知识库或用户数据。

### 11.3 db_schema_sync.py 为什么不能运行

错误：

~~~text
ModuleNotFoundError: No module named 'peewee_migrate'
~~~

db_schema_sync.py 依赖 peewee_migrate 执行 Peewee 数据库迁移，但当前 v0.27.0 镜像中缺少这个 Python 依赖。因此脚本在连接数据库前就退出了。

这属于镜像依赖或脚本打包问题，不是 MySQL 数据损坏。生产环境不建议直接在容器内临时 pip install，因为：

1. 容器重建后，手动安装的依赖会消失。
2. 可能引入与官方镜像不一致的依赖版本。
3. 迁移脚本可能还依赖其他未安装组件。
4. 迁移执行一半时会增加回滚难度。

本次因此采用先备份、再对明确问题执行定向 SQL 的方式。

### 11.4 model_type 为什么会出现字符串和整数冲突

错误：

~~~text
TypeError: unsupported operand type(s) for &: 'str' and 'int'
~~~

v0.27.0 通过位掩码判断模型能力，代码会执行类似：

~~~python
model_type & mt.value
~~~

这要求 model_type 必须是整数。但旧数据库中的 tenant_model.model_type 是 varchar(32)，历史值为：

~~~text
chat
embedding
image2text
rerank
~~~

程序实际相当于执行字符串和整数的按位运算，因此报 TypeError。

v0.27.0 的整数映射为：

| 类型 | 数值 |
|---|---:|
| Chat | 1 |
| Embedding | 2 |
| Vision 或 image2text | 8 |
| Rerank | 16 |
| TTS | 32 |
| OCR | 64 |

修复时必须先更新数据，再修改字段类型。若先把 varchar 直接改成 INT，chat、embedding 等字符串可能被转换成 0，导致模型类型信息丢失。

### 11.5 模型 ID 为什么从数字变成 32 位字符串

旧版本租户默认模型字段保存的是旧模型体系中的数字 ID：

~~~text
tenant_embd_id = 776
tenant_llm_id = 369
tenant_rerank_id = 5
~~~

v0.27.0 通过 tenant_model 表管理模型，租户字段需要引用 tenant_model.id：

~~~text
776be3aa947011f1ac0ccb8de5a2f7dd
~~~

因此旧字段不仅数据值过时，字段类型也可能过时。实际修复必须分两步：

1. 将字段从 INT 或旧短字段改为 VARCHAR(32)。
2. 根据当前租户、Provider、实例和模型类型找到正确的 tenant_model.id。

不能把所有数字全局替换成同一个模型，因为不同租户、Provider 和模型实例可能不同。

### 11.6 为什么会出现 Data truncated

错误：

~~~text
ERROR 1265: Data truncated for column tenant_llm_id
~~~

当数据库字段是 INT，而更新值是 32 位模型 ID 时，MySQL 无法把完整字符串写入整数列，因此出现 Data truncated。

正确顺序：

~~~text
备份 tenant 表
  -> 检查字段类型
  -> 修改为 VARCHAR(32)
  -> 写入新的 tenant_model.id
  -> 查询验证
~~~

### 11.7 为什么 Embedding 修复后 LLM、VLM、Rerank 仍不正常

每种默认模型都有独立字段：

~~~text
tenant_llm_id
tenant_embd_id
tenant_img2txt_id
tenant_rerank_id
~~~

修复 Embedding 只会修改 tenant_embd_id，不会自动修改另外三个字段。本次界面中出现：

~~~text
LLM       369
VLM       2147483647
Rerank    5
Embedding 776
~~~

后来根据当前租户的 tenant_model 查询结果修复为：

| 类型 | 新模型 ID |
|---|---|
| LLM qwen3:32b Ollama | 369f5bf49d1211f1869a59ebce91e2b5 |
| Embedding bge-m3 Ollama | 776be3aa947011f1ac0ccb8de5a2f7dd |
| VLM qwen2.5vl:7b Ollama | 776e98ca947011f1ac0ccb8de5a2f7dd |
| Rerank Qwen3-Reranker-4B Xinference | 5da014e6947011f1ac0ccb8de5a2f7dd |

### 11.8 迁移日志为什么显示跳过

日志中出现：

~~~text
Database migration version is v0.26.1, target version is v0.26.1, skipping all stages
~~~

这表示迁移程序根据数据库中的版本标记判断目标迁移已经执行过，因此跳过了阶段。但实际数据库仍存在旧字段和旧模型引用，说明迁移标记与实际 schema 不一致，或者历史迁移没有覆盖当前这套数据。

因此升级时不能只看迁移程序是否打印 completed，还必须检查：

- SHOW COLUMNS 的字段是否存在；
- 字段类型是否正确；
- tenant_model.model_type 是否为整数；
- tenant 默认模型是否仍是数字；
- tenant 的引用是否能关联到 tenant_model.id；
- Web 页面和 API 实际行为。

### 11.9 502 为什么通常是暂时的

错误：

~~~text
请求错误 502: undefined 网关错误
~~~

RAGFlow 容器启动后，内部 Nginx、Admin、Python API、数据同步和任务执行器并不是同时就绪。如果浏览器在 API ready 前访问，Nginx 暂时找不到后端，就会返回 502。

启动顺序大致为：

~~~text
容器启动
  -> 初始化数据库表
  -> 启动 Nginx
  -> 启动 Admin 服务
  -> 启动数据同步
  -> 启动 RAGFlow API
  -> 启动任务执行器
  -> API ready
~~~

真正就绪的日志是：

~~~text
RAGFlow admin is ready
RAGFlow server is ready
RAGFlow ingestion is ready
~~~

刚重启后的短暂 502 不代表数据丢失，也不代表必须回滚，应先等待并查看近期日志。

### 11.10 根因总结

本次问题可以归纳为：

~~~text
新版本应用代码
    + 新数据库字段
    + 新字段类型
    + 新模型类型编码
    + 新模型 ID 引用方式
        > 旧数据库中部分历史结构和数据没有完全转换
~~~

所以后续升级不能只执行 docker compose pull 和 docker compose up -d，还必须：

1. 备份 MySQL 和数据卷。
2. 查看官方迁移日志。
3. 检查 schema 是否真的匹配新代码。
4. 检查模型字段类型和历史引用。
5. 检查所有租户，而不是只检查当前账号。
6. 通过 Web 实际测试模型调用。

