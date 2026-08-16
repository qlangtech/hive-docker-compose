[![Gitter chat](https://badges.gitter.im/gitterHQ/gitter.png)](https://gitter.im/big-data-europe/Lobby)

# support paimon 
in order to support paimin in Hive engine
1. download hive supoort jar: `wget https://repo.maven.apache.org/maven2/org/apache/paimon/paimon-hive-connector-2.3/1.1.1/paimon-hive-connector-2.3-1.1.1.jar`
2. download hadoop-aws jar: `wget http://mirrors.163.com/maven/repository/maven-public/org/apache/hadoop/hadoop-aws/2.7.4/hadoop-aws-2.7.4.jar`
3. download aws-java-sdk jar: `wget http://mirrors.163.com/maven/repository/maven-public/com/amazonaws/aws-java-sdk/1.7.4/aws-java-sdk-1.7.4.jar`
4. using config file `docker-compose-paimon.yml` to launch docker-compose environment

# docker-hive

This is a docker container for Apache Hive 2.3.2. It is based on https://github.com/big-data-europe/docker-hadoop so check there for Hadoop configurations.
This deploys Hive and starts a hiveserver2 on port 10000.
Metastore is running with a connection to postgresql database.
The hive configuration is performed with HIVE_SITE_CONF_ variables (see hadoop-hive.env for an example).

To run Hive with postgresql metastore:
```
    docker-compose up -d
```

To deploy in Docker Swarm:
```
    docker stack deploy -c docker-compose.yml hive
```

To run a PrestoDB 0.181 with Hive connector:

```
  docker-compose up -d presto-coordinator
```

This deploys a Presto server listens on port `8080`

## Testing
Load data into Hive:
```
  $ docker-compose exec hive-server bash
  # /opt/hive/bin/beeline -u jdbc:hive2://localhost:10000
  > CREATE TABLE pokes (foo INT, bar STRING);
  > LOAD DATA LOCAL INPATH '/opt/hive/examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;
```

Then query it from PrestoDB. You can get [presto.jar](https://prestosql.io/docs/current/installation/cli.html) from PrestoDB website:
```
  $ wget https://repo1.maven.org/maven2/io/prestosql/presto-cli/308/presto-cli-308-executable.jar
  $ mv presto-cli-308-executable.jar presto.jar
  $ chmod +x presto.jar
  $ ./presto.jar --server localhost:8080 --catalog hive --schema default
  presto> select * from pokes;
```
测试PARQUET格式：
```
CREATE EXTERNAL TABLE parquet_table (
    foo INT,
    bar STRING
)
STORED AS PARQUET;
insert into parquet_table (foo,bar) values(1,'xxxx' );
```
PARQUET格式，带分区
```
CREATE TABLE IF NOT EXISTS sales_data (
    sale_id STRING,
    product_name STRING,
    quantity_sold INT,
    price DOUBLE
)
PARTITIONED BY (year INT, month INT)
ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS PARQUET;

INSERT INTO TABLE sales_data PARTITION (year=2023, month=1)
VALUES 
('s001', 'ProductA', 15, 19.99),
('s002', 'ProductB', 20, 29.99),
('s003', 'ProductC', 25, 9.99);
```

## 使用 MinIO 创建 Database

环境已通过 `hadoop-aws` 和 `aws-java-sdk` 支持 S3A 文件系统，可以创建 LOCATION 指向 `s3a://` 的 Hive Database。`docker-compose-paimon.yml` 中已经内置了 MinIO 服务，方便直接测试。

### 前置条件
- 默认 MinIO 服务已经随 compose 一起启动，访问地址为 `http://localhost:9000`，Console 地址为 `http://localhost:9001`。
- 默认凭据为 `baisui / 12345678`，如需修改请同时调整 `docker-compose-paimon.yml` 中的 `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` 和 `hadoop-hive.env` 中的 `CORE_CONF_fs_s3a_access_key` / `CORE_CONF_fs_s3a_secret_key`。
- 如果你使用**外部 MinIO**，请修改 `hadoop-hive.env` 中的 endpoint、access_key、secret_key 为你实际的地址和凭据：
  ```properties
  CORE_CONF_fs_s3a_endpoint=http://your-minio-host:9000
  CORE_CONF_fs_s3a_access_key=your-access-key
  CORE_CONF_fs_s3a_secret_key=your-secret-key
  ```

### 启动环境
```bash
docker-compose -f docker-compose-paimon.yml up -d
```

### 创建 Bucket
在 Hive 中使用 `s3a://my-bucket/...` 之前，需要先在 MinIO 中创建 `my-bucket`。可以通过 MinIO Console 登录后创建，也可以使用 `mc` 命令行：

```bash
mc alias set local http://localhost:9000 baisui 12345678
mc mb local/my-bucket
```

### 进入 Hive 并创建 MinIO Database
```bash
docker-compose -f docker-compose-paimon.yml exec hive-server bash
/opt/hive/bin/beeline -u jdbc:hive2://localhost:10000 -n "" -p ""
```

在 beeline 中执行：
```sql
CREATE DATABASE IF NOT EXISTS my_minio_db
COMMENT 'Database on MinIO'
LOCATION 's3a://my-bucket/path/to/database';
```

### 验证
```sql
SHOW DATABASES;
DESCRIBE DATABASE EXTENDED my_minio_db;
```

## Contributors
* Ivan Ermilov [@earthquakesan](https://github.com/earthquakesan) (maintainer)
* Yiannis Mouchakis [@gmouchakis](https://github.com/gmouchakis)
* Ke Zhu [@shawnzhu](https://github.com/shawnzhu)
