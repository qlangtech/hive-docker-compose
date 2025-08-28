为了让你的Hive环境支持Paimon表查询，需要将Paimon Hive Connector的JAR包注入到Hive服务容器中，并确保Hive能识别Paimon的存储格式。以下是具体的`docker-compose.yml`修改方案：

---

### **修改后的`docker-compose.yml`**
```yaml
version: "3"

services:
  # ... 保持原有 namenode/datanode 配置不变 ...

  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      # 添加Hive Auxiliary JARS路径配置
      HIVE_AUX_JARS_PATH: "/opt/hive/lib/paimon-hive-connector.jar"  
      SERVICE_PRECONDITION: "hive-metastore:9083"
    volumes:
      # 挂载Paimon Connector JAR到Hive的lib目录
      - ./paimon-hive-connector.jar:/opt/hive/lib/paimon-hive-connector.jar
    ports:
      - "10000:10000"

  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    command: /opt/hive/bin/hive --service metastore
    environment:
      # 同样为Metastore配置Auxiliary JARS
      HIVE_AUX_JARS_PATH: "/opt/hive/lib/paimon-hive-connector.jar"  
      SERVICE_PRECONDITION: "namenode:50070 datanode:50075 hive-metastore-postgresql:5432"
    volumes:
      # 挂载Paimon Connector JAR到Metastore容器
      - ./paimon-hive-connector.jar:/opt/hive/lib/paimon-hive-connector.jar
    ports:
      - "9083:9083"

  # ... 其他服务配置保持不变 ...
```

---

### **关键修改说明**

#### 1. **挂载Paimon Connector JAR**
- 将本地目录中的`paimon-hive-connector.jar`挂载到Hive容器内的`/opt/hive/lib`目录（Hive默认加载该路径的JAR）。
- **操作步骤**：
  1. 从[Paimon官方仓库](https://repo.maven.apache.org/maven2/org/apache/paimon/paimon-hive/)下载与Paimon版本对应的JAR（例如：`paimon-hive-2.3-0.7.0.jar`）。
  2. 将JAR重命名为`paimon-hive-connector.jar`并放置在与`docker-compose.yml`同级的目录中。

#### 2. **配置Hive Auxiliary JARS路径**
- 通过环境变量`HIVE_AUX_JARS_PATH`显式指定Paimon Connector的路径，确保Hive服务启动时加载该JAR。

#### 3. **同时修改`hive-server`和`hive-metastore`**
- 两个服务都需要加载Paimon Connector，否则元数据操作和查询会因类缺失而失败。

---

### **验证步骤**
1. **启动服务**：
   ```bash
   docker-compose down -v && docker-compose up -d
   ```

2. **检查JAR是否加载**：
   ```bash
   # 进入hive-server容器查看JAR是否存在
   docker exec -it hive-server ls /opt/hive/lib | grep paimon
   ```

3. **创建Paimon表测试**：
   使用Hive CLI或JDBC客户端执行以下SQL：
   ```sql
   CREATE TABLE paimon_test (id INT) 
   STORED BY 'org.apache.paimon.hive.PaimonStorageHandler';
   ```

---

### **注意事项**
1. **版本兼容性**：
   - 确保Paimon Connector版本与Hive 2.3.2和Hadoop 2.7.4兼容。若使用最新版Paimon，可能需要调整Hive/Hadoop版本。
   - 若版本不匹配，可尝试构建[Paimon源码](https://github.com/apache/paimon)生成适配的JAR。

2. **日志排查**：
   - 检查HiveServer2和Metastore容器的启动日志，确认无`ClassNotFoundException`：
     ```bash
     docker logs hive-server
     docker logs hive-metastore
     ```

3. **动态加载依赖**：
   - 如果不想挂载JAR，可构建自定义Docker镜像（在Dockerfile中直接添加JAR），但需重新编译镜像。
