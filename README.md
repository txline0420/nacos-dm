# nacos适配达梦数据库

## 1. 编绎原码中加入对达梦数据库的支持

### 1.1. 下载nacos2.2.1版本原码
```http request
https://github.com/alibaba/nacos/tree/2.2.1
```

### 1.2.新增达梦驱动依赖
#### 1.2.1.父工程pom.xml
```xml
<!-- 禁用达梦官方依赖 -->
<dependency>
  <groupId>com.dameng</groupId>
  <artifactId>DmJdbcDriver18</artifactId>
  <version>8.1.1.193</version>
</dependency>
```

#### 1.2.2. config工程pom.xml
```xml
<dependency>
  <groupId>com.dameng</groupId>
  <artifactId>DmJdbcDriver18</artifactId>
  <version>8.1.1.193</version>
</dependency>
```
### 1.3.修改nacos-console模块配置文件application.properties
```properties
nacos.core.auth.plugin.nacos.token.secret.key=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJuYWNvcyIsIm5hbWUiOiLpu4Tlrp3lurcifQ.WMuRlZfZd4VCbtmxnR_M1CgsM2SyOCSdiLXMG7FN-GE

### If use MySQL as datasource:
spring.datasource.platform=dm
### Count of DB:
db.num=1
### Connect URL of DB:
db.jdbcDriverName=dm.jdbc.driver.DmDriver
db.url.0=jdbc:dm://192.168.8.152:30236?schema=NACOS&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=SYSDBA
db.password.0=SYSDBA001
```

### 1.4. nacos-config模块修改java类,修改ExternalDataSourceProperties.java
```shell
    
    private String jdbcDriverName;

    public String getJdbcDriverName() {
        return jdbcDriverName;
    }

    public void setJdbcDriverName(String jdbcDriverName) {
        this.jdbcDriverName = jdbcDriverName;
    }

	List<HikariDataSource> build(Environment environment, Callback<HikariDataSource> callback) {
        List<HikariDataSource> dataSources = new ArrayList<>();
        Binder.get(environment).bind("db", Bindable.ofInstance(this));
        Preconditions.checkArgument(Objects.nonNull(num), "db.num is null");
        Preconditions.checkArgument(CollectionUtils.isNotEmpty(user), "db.user or db.user.[index] is null");
        Preconditions.checkArgument(CollectionUtils.isNotEmpty(password), "db.password or db.password.[index] is null");
        for (int index = 0; index < num; index++) {
            int currentSize = index + 1;
            Preconditions.checkArgument(url.size() >= currentSize, "db.url.%s is null", index);
            DataSourcePoolProperties poolProperties = DataSourcePoolProperties.build(environment);
            if (StringUtils.isEmpty(poolProperties.getDataSource().getDriverClassName())) {
                System.out.println("===================================");
                System.out.println("jdbcDriverName=" + jdbcDriverName);
                if (StringUtils.isNotEmpty(jdbcDriverName)) {
                    poolProperties.setDriverClassName(jdbcDriverName);
                } else {
                    poolProperties.setDriverClassName(JDBC_DRIVER_NAME);
                }
                System.out.println("===================================");
                System.out.println("dataSources=" + dataSources);
            }
            poolProperties.setJdbcUrl(url.get(index).trim());
            poolProperties.setUsername(getOrDefault(user, index, user.get(0)).trim());
            poolProperties.setPassword(getOrDefault(password, index, password.get(0)).trim());
            HikariDataSource ds = poolProperties.getDataSource();
            if (StringUtils.isEmpty(ds.getConnectionTestQuery())) {
                ds.setConnectionTestQuery(TEST_QUERY);
            }
            dataSources.add(ds);
            callback.accept(ds);
        }
        Preconditions.checkArgument(CollectionUtils.isNotEmpty(dataSources), "no datasource available");
        return dataSources;
    }

```

### 1.5. plugin模块下datasource目录中修改DataSourceConstant.java
```shell
 public static final String DM = "dm";
```

### 1.6 修改ExternalConfigInfoPersistServiceImpl.java
> 176行、194行异常DuplicateKeyException改成DataIntegrityViolationException（不然编辑yml文件时会报）
```shell
try {
    addConfigInfo(srcIp, srcUser, configInfo, time, configAdvanceInfo, notify);
} catch (DataIntegrityViolationException ive) { // Unique constraint conflict
    updateConfigInfo(configInfo, srcIp, srcUser, time, configAdvanceInfo, notify);
}

Map<String, Object> configAdvanceInfo, boolean notify) {
try {
    addConfigInfo(srcIp, srcUser, configInfo, time, configAdvanceInfo, notify);
    return true;
} catch (DataIntegrityViolationException ignore) { // Unique constraint conflict
    return updateConfigInfoCas(configInfo, srcIp, srcUser, time, configAdvanceInfo, notify);
}
```

### 1.7.nacos-datasource-plugin模块添加dm实现类DataSourceConstant类添加常量
> 内容拷贝了mysql包下类，改了类名，每个类里面 DataSourceConstant.DM
```shell
ConfigInfoAggrMapperByDm
ConfigInfoBetaMapperByDm
ConfigInfoMapperByDm
ConfigInfoTagMapperByDm
ConfigTagsRelationMapperByDm
HistoryConfigInfoMapperByDm
TenantInfoMapperByDm
TenantCapacityMapperByDm
GroupCapacityMapperByDm
```

### 1.8 com.alibaba.nacos.plugin.datasource.mapper.Mapper文件添加上图所加的达梦实现类路径
```shell
com.alibaba.nacos.plugin.datasource.impl.dm.ConfigInfoAggrMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.ConfigInfoBetaMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.ConfigInfoMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.ConfigInfoTagMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.ConfigTagsRelationMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.HistoryConfigInfoMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.TenantInfoMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.TenantCapacityMapperByDm
com.alibaba.nacos.plugin.datasource.impl.dm.GroupCapacityMapperByDm
```

### 1.9 打包
```shell
mvn -Prelease-nacos '-Dmaven.test.skip=true' '-Dcheckstyle.skip=true' clean install -U
```

### 1.10 打包完毕在nacos-distribution/target/目录下有nacos-server-2.2.1.tar.gz文件

## 2. 达梦数据库中注入相应的表
### 2.1 注入nacos的相应表,注意insert时需要提交
```sql
CREATE TABLE "NACOS"."CONFIG_INFO"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(255 CHAR),
    "CONTENT" TEXT NOT NULL,
    "MD5" NVARCHAR2(32 CHAR),
    "GMT_CREATE" DATE DEFAULT SYSDATE() NOT NULL,
    "GMT_MODIFIED" DATE DEFAULT SYSDATE() NOT NULL,
    "SRC_USER" TEXT,
    "SRC_IP" NVARCHAR2(50 CHAR),
    "APP_NAME" NVARCHAR2(128 CHAR),
    "TENANT_ID" NVARCHAR2(128 CHAR) DEFAULT '',
    "C_DESC" NVARCHAR2(256 CHAR),
    "C_USE" NVARCHAR2(64 CHAR),
    "EFFECT" NVARCHAR2(64 CHAR),
    "TYPE" NVARCHAR2(64 CHAR),
    "C_SCHEMA" TEXT,
    "ENCRYPTED_DATA_KEY" TEXT NOT NULL,
    NOT CLUSTER PRIMARY KEY("ID"),
    UNIQUE("DATA_ID", "GROUP_ID", "TENANT_ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."CONFIG_INFO_AGGR"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "DATUM_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "CONTENT" TEXT NOT NULL,
    "GMT_MODIFIED" DATE NOT NULL,
    "APP_NAME" NVARCHAR2(128 CHAR),
    "TENANT_ID" NVARCHAR2(128 CHAR),
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."CONFIG_INFO_BETA"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "APP_NAME" NVARCHAR2(128 CHAR),
    "CONTENT" TEXT NOT NULL,
    "BETA_IPS" TEXT,
    "MD5" NVARCHAR2(32 CHAR),
    "GMT_CREATE" DATE NOT NULL,
    "GMT_MODIFIED" DATE NOT NULL,
    "SRC_USER" TEXT,
    "SRC_IP" NVARCHAR2(50 CHAR),
    "TENANT_ID" NVARCHAR2(128 CHAR),
    "ENCRYPTED_DATA_KEY" TEXT NOT NULL,
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."CONFIG_INFO_TAG"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "TENANT_ID" NVARCHAR2(128 CHAR),
    "TAG_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "APP_NAME" NVARCHAR2(128 CHAR),
    "CONTENT" TEXT NOT NULL,
    "MD5" NVARCHAR2(32 CHAR),
    "GMT_CREATE" DATE NOT NULL,
    "GMT_MODIFIED" DATE NOT NULL,
    "SRC_USER" TEXT,
    "SRC_IP" NVARCHAR2(50 CHAR),
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."CONFIG_TAGS_RELATION"
(
    "ID" NUMBER(20,0) NOT NULL,
    "TAG_NAME" NVARCHAR2(128 CHAR) NOT NULL,
    "TAG_TYPE" NVARCHAR2(64 CHAR),
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "TENANT_ID" NVARCHAR2(128 CHAR),
    "NID" INT IDENTITY(1, 1) NOT NULL,
    NOT CLUSTER PRIMARY KEY("NID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."GROUP_CAPACITY"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "GROUP_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "QUOTA" NUMBER(11,0) NOT NULL,
    "USAGE" NUMBER(11,0) NOT NULL,
    "MAX_SIZE" NUMBER(11,0) NOT NULL,
    "MAX_AGGR_COUNT" NUMBER(11,0) NOT NULL,
    "MAX_AGGR_SIZE" NUMBER(11,0) NOT NULL,
    "MAX_HISTORY_COUNT" NUMBER(11,0) NOT NULL,
    "GMT_CREATE" DATE NOT NULL,
    "GMT_MODIFIED" DATE NOT NULL,
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

COMMENT ON TABLE "NACOS"."GROUP_CAPACITY" IS '集群、各Group容量信息表';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."ID" IS '主键ID';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."GROUP_ID" IS 'Group ID，空字符表示整个集群';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."QUOTA" IS '配额，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."USAGE" IS '使用量';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."MAX_SIZE" IS '单个配置大小上限，单位为字节，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."MAX_AGGR_COUNT" IS '聚合子配置最大个数，，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."MAX_AGGR_SIZE" IS '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."MAX_HISTORY_COUNT" IS '最大变更历史数量';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."GMT_CREATE" IS '创建时间';
COMMENT ON COLUMN "NACOS"."GROUP_CAPACITY"."GMT_MODIFIED" IS '修改时间';


CREATE TABLE "NACOS"."HIS_CONFIG_INFO"
(
    "ID" INT NOT NULL,
    "NID" INT IDENTITY(1, 1) NOT NULL,
    "DATA_ID" NVARCHAR2(255 CHAR) NOT NULL,
    "GROUP_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "APP_NAME" NVARCHAR2(128 CHAR),
    "CONTENT" TEXT NOT NULL,
    "MD5" NVARCHAR2(32 CHAR),
    "GMT_CREATE" DATE DEFAULT SYSDATE() NOT NULL,
    "GMT_MODIFIED" DATE DEFAULT SYSDATE() NOT NULL,
    "SRC_USER" TEXT,
    "SRC_IP" NVARCHAR2(50 CHAR),
    "OP_TYPE" NCHAR(10),
    "TENANT_ID" NVARCHAR2(128 CHAR),
    "ENCRYPTED_DATA_KEY" TEXT NOT NULL,
    NOT CLUSTER PRIMARY KEY("NID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

COMMENT ON TABLE "NACOS"."HIS_CONFIG_INFO" IS '多租户改造';
COMMENT ON COLUMN "NACOS"."HIS_CONFIG_INFO"."APP_NAME" IS 'app_name';
COMMENT ON COLUMN "NACOS"."HIS_CONFIG_INFO"."TENANT_ID" IS '租户字段';
COMMENT ON COLUMN "NACOS"."HIS_CONFIG_INFO"."ENCRYPTED_DATA_KEY" IS '秘钥';


CREATE TABLE "NACOS"."PERMISSIONS"
(
    "ROLE" NVARCHAR2(50 CHAR) NOT NULL,
    "RESOURCE" NVARCHAR2(255 CHAR) NOT NULL,
    "ACTION" NVARCHAR2(8 CHAR) NOT NULL) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."ROLES"
(
    "USERNAME" NVARCHAR2(50 CHAR) NOT NULL,
    "ROLE" NVARCHAR2(50 CHAR) NOT NULL) STORAGE(ON "MAIN", CLUSTERBTR) ;

CREATE TABLE "NACOS"."TENANT_CAPACITY"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "TENANT_ID" NVARCHAR2(128 CHAR) NOT NULL,
    "QUOTA" NUMBER(11,0) NOT NULL,
    "USAGE" NUMBER(11,0) NOT NULL,
    "MAX_SIZE" NUMBER(11,0) NOT NULL,
    "MAX_AGGR_COUNT" NUMBER(11,0) NOT NULL,
    "MAX_AGGR_SIZE" NUMBER(11,0) NOT NULL,
    "MAX_HISTORY_COUNT" NUMBER(11,0) NOT NULL,
    "GMT_CREATE" DATE NOT NULL,
    "GMT_MODIFIED" DATE NOT NULL,
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

COMMENT ON TABLE "NACOS"."TENANT_CAPACITY" IS '租户容量信息表';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."ID" IS '主键ID';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."TENANT_ID" IS 'Tenant ID';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."QUOTA" IS '配额，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."USAGE" IS '使用量';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."MAX_SIZE" IS '单个配置大小上限，单位为字节，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."MAX_AGGR_COUNT" IS '聚合子配置最大个数';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."MAX_AGGR_SIZE" IS '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."MAX_HISTORY_COUNT" IS '最大变更历史数量';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."GMT_CREATE" IS '创建时间';
COMMENT ON COLUMN "NACOS"."TENANT_CAPACITY"."GMT_MODIFIED" IS '修改时间';


CREATE TABLE "NACOS"."TENANT_INFO"
(
    "ID" INT IDENTITY(1, 1) NOT NULL,
    "KP" NVARCHAR2(128 CHAR) NOT NULL,
    "TENANT_ID" NVARCHAR2(128 CHAR),
    "TENANT_NAME" NVARCHAR2(128 CHAR),
    "TENANT_DESC" NVARCHAR2(256 CHAR),
    "CREATE_SOURCE" NVARCHAR2(32 CHAR),
    "GMT_CREATE" NUMBER(20,0) NOT NULL,
    "GMT_MODIFIED" NUMBER(20,0) NOT NULL,
    NOT CLUSTER PRIMARY KEY("ID")) STORAGE(ON "MAIN", CLUSTERBTR) ;

COMMENT ON TABLE "NACOS"."TENANT_INFO" IS 'tenant_info';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."ID" IS 'id';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."KP" IS 'kp';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."TENANT_ID" IS 'tenant_id';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."TENANT_NAME" IS 'tenant_name';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."TENANT_DESC" IS 'tenant_desc';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."CREATE_SOURCE" IS 'create_source';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."GMT_CREATE" IS '创建时间';
COMMENT ON COLUMN "NACOS"."TENANT_INFO"."GMT_MODIFIED" IS '修改时间';


CREATE TABLE "NACOS"."USERS"
(
    "USERNAME" NVARCHAR2(50 CHAR) NOT NULL,
    "PASSWORD" NVARCHAR2(500 CHAR) NOT NULL,
    "ENABLED" NUMBER(4,0) NOT NULL) STORAGE(ON "MAIN", CLUSTERBTR) ;

INSERT INTO "NACOS".USERS(USERNAME, PASSWORD, ENABLED) VALUES ( 'nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', 1);

INSERT INTO "NACOS".ROLES(USERNAME,ROLE) VALUES ( 'nacos', 'ROLE_ADMIN');
```

### 2.2 测试nacos，
#### 2.2.1 修改nacos-server-2.2.1.zip文件中的
> nacos-server-2.2.1\nacos\conf\application.properties
```properties
nacos.core.auth.plugin.nacos.token.secret.key=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJuYWNvcyIsIm5hbWUiOiLpu4Tlrp3lurcifQ.WMuRlZfZd4VCbtmxnR_M1CgsM2SyOCSdiLXMG7FN-GE

### If use MySQL as datasource:
spring.datasource.platform=dm
### Count of DB:
db.num=1
### Connect URL of DB:
db.jdbcDriverName=dm.jdbc.driver.DmDriver
db.url.0=jdbc:dm://192.168.8.152:30236?schema=NACOS&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=SYSDBA
db.password.0=SYSDBA001
```

#### 2.3 Windows环境中测试单体启动命令
```shell
startup.cmd -m standalone
```

## 3. 容器化
### 3.1 拉取nacos构建容器原码,并抽走build目录下的文件
```http request
https://github.com/nacos-group/nacos-docker
```

### 3.2 修改Dockerfile
```dockerfile
FROM centos:7.9.2009
LABEL maintainer="txline0420@163.com"

# set environment
ENV MODE="cluster" \
    PREFER_HOST_MODE="ip"\
    BASE_DIR="/home/nacos" \
    CLASSPATH=".:/home/nacos/conf:$CLASSPATH" \
    CLUSTER_CONF="/home/nacos/conf/cluster.conf" \
    FUNCTION_MODE="all" \
    JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk" \
    NACOS_USER="nacos" \
    JAVA="/usr/lib/jvm/java-1.8.0-openjdk/bin/java" \
    JVM_XMS="1g" \
    JVM_XMX="1g" \
    JVM_XMN="512m" \
    JVM_MS="128m" \
    JVM_MMS="320m" \
    NACOS_DEBUG="n" \
    TOMCAT_ACCESSLOG_ENABLED="false" \
    TIME_ZONE="Asia/Shanghai"

ARG NACOS_VERSION=2.2.1
ARG HOT_FIX_FLAG=""

WORKDIR $BASE_DIR


COPY nacos-server-2.2.1.tar.gz /home

RUN set -x \
    && yum update -y \
    && yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel iputils nc vim libcurl \
    && yum clean all
RUN tar -xzvf /home/nacos-server-${NACOS_VERSION}.tar.gz -C /home \
    && rm -rf /home/nacos-server.tar.gz /home/nacos/bin/* /home/nacos/conf/*.properties /home/nacos/conf/*.example /home/nacos/conf/nacos-mysql.sql \
    && ln -snf /usr/share/zoneinfo/$TIME_ZONE /etc/localtime && echo $TIME_ZONE > /etc/timezone

ADD bin/docker-startup.sh bin/docker-startup.sh
ADD conf/application.properties conf/application.properties


# set startup log dir
RUN mkdir -p logs \
	&& touch logs/start.out \
	&& ln -sf /dev/stdout start.out \
	&& ln -sf /dev/stderr start.out
RUN chmod +x bin/docker-startup.sh

EXPOSE 8848
ENTRYPOINT ["bin/docker-startup.sh"]
```

### 3.3 修改conf下的application.properties
```properties
# spring
server.servlet.contextPath=${SERVER_SERVLET_CONTEXTPATH:/nacos}
server.contextPath=/nacos
server.port=${NACOS_APPLICATION_PORT:8848}
server.tomcat.accesslog.max-days=30
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i
spring.datasource.platform=${SPRING_DATASOURCE_PLATFORM:""}
nacos.cmdb.dumpTaskInterval=3600
nacos.cmdb.eventTaskInterval=10
nacos.cmdb.labelTaskInterval=300
nacos.cmdb.loadDataAtStart=false
db.num=${DM_DATABASE_NUM:1}
db.url.0=jdbc:dm://${DM_SERVICE_HOST:192.168.20.4}:${DM_SERVICE_PORT:5236}?schema=${DM_SCHEMA:NACOS}&characterEncoding=UTF-8&useUnicode=true&serverTimezone=Asia/Shanghai
db.user.0=${DM_SERVICE_USER:SYSDBA}
db.password.0=${DM_SERVICE_PASSWORD:SYSDBA}
### The auth system to use, currently only 'nacos' and 'ldap' is supported:
nacos.core.auth.system.type=${NACOS_AUTH_SYSTEM_TYPE:nacos}
### worked when nacos.core.auth.system.type=nacos
### The token expiration in seconds:
nacos.core.auth.plugin.nacos.token.expire.seconds=${NACOS_AUTH_TOKEN_EXPIRE_SECONDS:18000}
### The default token:
nacos.core.auth.plugin.nacos.token.secret.key=${NACOS_AUTH_TOKEN:SecretKey012345678901234567890123456789012345678901234567890123456789}
### Turn on/off caching of auth information. By turning on this switch, the update of auth information would have a 15 seconds delay.
nacos.core.auth.caching.enabled=${NACOS_AUTH_CACHE_ENABLE:false}
nacos.core.auth.enable.userAgentAuthWhite=${NACOS_AUTH_USER_AGENT_AUTH_WHITE_ENABLE:false}
nacos.core.auth.server.identity.key=${NACOS_AUTH_IDENTITY_KEY:serverIdentity}
nacos.core.auth.server.identity.value=${NACOS_AUTH_IDENTITY_VALUE:security}
server.tomcat.accesslog.enabled=${TOMCAT_ACCESSLOG_ENABLED:false}
# default current work dir
server.tomcat.basedir=file:.
## spring security config
### turn off security
nacos.security.ignore.urls=${NACOS_SECURITY_IGNORE_URLS:/,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**}
# metrics for elastic search
management.metrics.export.elastic.enabled=false
management.metrics.export.influx.enabled=false
nacos.naming.distro.taskDispatchThreadCount=10
nacos.naming.distro.taskDispatchPeriod=200
nacos.naming.distro.batchSyncKeyCount=1000
nacos.naming.distro.initDataRatio=0.9
nacos.naming.distro.syncRetryDelay=5000
nacos.naming.data.warmup=true
```

### 3.4 构建镜像 build.sh
```shell
#!/bin/bash
sudo docker build -t nacos/nacos-server-dm:2.2.1 .
```

## 4. Docker容器运行
### 4.1 创建数据目录
```shell
sudo mkdir /data/nacos-dm-data/{logs,conf} && sudo chmod 777 /data/nacos-dm-data/logs
```

### 4.2 增加conf目下增加application.properties配置文件
```properties
#
# Copyright 1999-2021 Alibaba Group Holding Ltd.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

#*************** Spring Boot Related Configurations ***************#
### Default web context path:
server.servlet.contextPath=/nacos
### Include message field
server.error.include-message=ALWAYS
### Default web server port:
server.port=8848

#*************** Network Related Configurations ***************#
### If prefer hostname over ip for Nacos server addresses in cluster.conf:
# nacos.inetutils.prefer-hostname-over-ip=false

### Specify local server's IP:
# nacos.inetutils.ip-address=


#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=dm

### Count of DB:
db.num=1
db.jdbcDriverName=dm.jdbc.driver.DmDriver
### Connect URL of DB:
db.url.0=jdbc:dm://192.168.8.152:30236?schema=NACOS&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=SYSDBA
db.password.0=SYSDBA001

#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
### Deprecated configuration property, it is recommended to use `spring.sql.init.platform` replaced.
# spring.datasource.platform=mysql
# spring.sql.init.platform=mysql

### Count of DB:
# db.num=1

### Connect URL of DB:
# db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
# db.user.0=nacos
# db.password.0=nacos

### Connection pool configuration: hikariCP
db.pool.config.connectionTimeout=30000
db.pool.config.validationTimeout=10000
db.pool.config.maximumPoolSize=20
db.pool.config.minimumIdle=2

#*************** Naming Module Related Configurations ***************#

### If enable data warmup. If set to false, the server would accept request without local data preparation:
# nacos.naming.data.warmup=true

### If enable the instance auto expiration, kind like of health check of instance:
# nacos.naming.expireInstance=true

### Add in 2.0.0
### The interval to clean empty service, unit: milliseconds.
# nacos.naming.clean.empty-service.interval=60000

### The expired time to clean empty service, unit: milliseconds.
# nacos.naming.clean.empty-service.expired-time=60000

### The interval to clean expired metadata, unit: milliseconds.
# nacos.naming.clean.expired-metadata.interval=5000

### The expired time to clean metadata, unit: milliseconds.
# nacos.naming.clean.expired-metadata.expired-time=60000

### The delay time before push task to execute from service changed, unit: milliseconds.
# nacos.naming.push.pushTaskDelay=500

### The timeout for push task execute, unit: milliseconds.
# nacos.naming.push.pushTaskTimeout=5000

### The delay time for retrying failed push task, unit: milliseconds.
# nacos.naming.push.pushTaskRetryDelay=1000

### Since 2.0.3
### The expired time for inactive client, unit: milliseconds.
# nacos.naming.client.expired.time=180000

#*************** CMDB Module Related Configurations ***************#
### The interval to dump external CMDB in seconds:
# nacos.cmdb.dumpTaskInterval=3600

### The interval of polling data change event in seconds:
# nacos.cmdb.eventTaskInterval=10

### The interval of loading labels in seconds:
# nacos.cmdb.labelTaskInterval=300

### If turn on data loading task:
# nacos.cmdb.loadDataAtStart=false


#*************** Metrics Related Configurations ***************#
### Metrics for prometheus
#management.endpoints.web.exposure.include=*

### Metrics for elastic search
management.metrics.export.elastic.enabled=false
#management.metrics.export.elastic.host=http://localhost:9200

### Metrics for influx
management.metrics.export.influx.enabled=false
#management.metrics.export.influx.db=springboot
#management.metrics.export.influx.uri=http://localhost:8086
#management.metrics.export.influx.auto-create-db=true
#management.metrics.export.influx.consistency=one
#management.metrics.export.influx.compressed=true

#*************** Access Log Related Configurations ***************#
### If turn on the access log:
server.tomcat.accesslog.enabled=true

### The access log pattern:
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i

### The directory of access log:
server.tomcat.basedir=file:.

#*************** Access Control Related Configurations ***************#
### If enable spring security, this option is deprecated in 1.2.0:
#spring.security.enabled=false

### The ignore urls of auth
nacos.security.ignore.urls=/,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-ui/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**

### The auth system to use, currently only 'nacos' and 'ldap' is supported:
nacos.core.auth.system.type=nacos

### If turn on auth system:
nacos.core.auth.enabled=false

### Turn on/off caching of auth information. By turning on this switch, the update of auth information would have a 15 seconds delay.
nacos.core.auth.caching.enabled=true

### Since 1.4.1, Turn on/off white auth for user-agent: nacos-server, only for upgrade from old version.
nacos.core.auth.enable.userAgentAuthWhite=false

### Since 1.4.1, worked when nacos.core.auth.enabled=true and nacos.core.auth.enable.userAgentAuthWhite=false.
### The two properties is the white list for auth and used by identity the request from other server.
nacos.core.auth.server.identity.key=
nacos.core.auth.server.identity.value=

### worked when nacos.core.auth.system.type=nacos
### The token expiration in seconds:
nacos.core.auth.plugin.nacos.token.cache.enable=false
nacos.core.auth.plugin.nacos.token.expire.seconds=18000
### The default token (Base64 String):
nacos.core.auth.plugin.nacos.token.secret.key=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJuYWNvcyIsIm5hbWUiOiLpu4Tlrp3lurcifQ.WMuRlZfZd4VCbtmxnR_M1CgsM2SyOCSdiLXMG7FN-GE

### worked when nacos.core.auth.system.type=ldapï¼{0} is Placeholder,replace login username
#nacos.core.auth.ldap.url=ldap://localhost:389
#nacos.core.auth.ldap.basedc=dc=example,dc=org
#nacos.core.auth.ldap.userDn=cn=admin,${nacos.core.auth.ldap.basedc}
#nacos.core.auth.ldap.password=admin
#nacos.core.auth.ldap.userdn=cn={0},dc=example,dc=org
#nacos.core.auth.ldap.filter.prefix=uid
#nacos.core.auth.ldap.case.sensitive=true


#*************** Istio Related Configurations ***************#
### If turn on the MCP server:
nacos.istio.mcp.server.enabled=false

#*************** Core Related Configurations ***************#

### set the WorkerID manually
# nacos.core.snowflake.worker-id=

### Member-MetaData
# nacos.core.member.meta.site=
# nacos.core.member.meta.adweight=
# nacos.core.member.meta.weight=

### MemberLookup
### Addressing pattern category, If set, the priority is highest
# nacos.core.member.lookup.type=[file,address-server]
## Set the cluster list with a configuration file or command-line argument
# nacos.member.list=192.168.16.101:8847?raft_port=8807,192.168.16.101?raft_port=8808,192.168.16.101:8849?raft_port=8809
## for AddressServerMemberLookup
# Maximum number of retries to query the address server upon initialization
# nacos.core.address-server.retry=5
## Server domain name address of [address-server] mode
# address.server.domain=jmenv.tbsite.net
## Server port of [address-server] mode
# address.server.port=8080
## Request address of [address-server] mode
# address.server.url=/nacos/serverlist

#*************** JRaft Related Configurations ***************#

### Sets the Raft cluster election timeout, default value is 5 second
# nacos.core.protocol.raft.data.election_timeout_ms=5000
### Sets the amount of time the Raft snapshot will execute periodically, default is 30 minute
# nacos.core.protocol.raft.data.snapshot_interval_secs=30
### raft internal worker threads
# nacos.core.protocol.raft.data.core_thread_num=8
### Number of threads required for raft business request processing
# nacos.core.protocol.raft.data.cli_service_thread_num=4
### raft linear read strategy. Safe linear reads are used by default, that is, the Leader tenure is confirmed by heartbeat
# nacos.core.protocol.raft.data.read_index_type=ReadOnlySafe
### rpc request timeout, default 5 seconds
# nacos.core.protocol.raft.data.rpc_request_timeout_ms=5000

#*************** Distro Related Configurations ***************#

### Distro data sync delay time, when sync task delayed, task will be merged for same data key. Default 1 second.
# nacos.core.protocol.distro.data.sync.delayMs=1000

### Distro data sync timeout for one sync data, default 3 seconds.
# nacos.core.protocol.distro.data.sync.timeoutMs=3000

### Distro data sync retry delay time when sync data failed or timeout, same behavior with delayMs, default 3 seconds.
# nacos.core.protocol.distro.data.sync.retryDelayMs=3000

### Distro data verify interval time, verify synced data whether expired for a interval. Default 5 seconds.
# nacos.core.protocol.distro.data.verify.intervalMs=5000

### Distro data verify timeout for one verify, default 3 seconds.
# nacos.core.protocol.distro.data.verify.timeoutMs=3000

### Distro data load retry delay when load snapshot data failed, default 30 seconds.
# nacos.core.protocol.distro.data.load.retryDelayMs=30000

### enable to support prometheus service discovery
#nacos.prometheus.metrics.enabled=true

### Since 2.3
#*************** Grpc Configurations ***************#

## sdk grpc(between nacos server and client) configuration
## Sets the maximum message size allowed to be received on the server.
#nacos.remote.server.grpc.sdk.max-inbound-message-size=10485760

## Sets the time(milliseconds) without read activity before sending a keepalive ping. The typical default is two hours.
#nacos.remote.server.grpc.sdk.keep-alive-time=7200000

## Sets a time(milliseconds) waiting for read activity after sending a keepalive ping. Defaults to 20 seconds.
#nacos.remote.server.grpc.sdk.keep-alive-timeout=20000


## Sets a time(milliseconds) that specify the most aggressive keep-alive time clients are permitted to configure. The typical default is 5 minutes
#nacos.remote.server.grpc.sdk.permit-keep-alive-time=300000

## cluster grpc(inside the nacos server) configuration
#nacos.remote.server.grpc.cluster.max-inbound-message-size=10485760

## Sets the time(milliseconds) without read activity before sending a keepalive ping. The typical default is two hours.
#nacos.remote.server.grpc.cluster.keep-alive-time=7200000

## Sets a time(milliseconds) waiting for read activity after sending a keepalive ping. Defaults to 20 seconds.
#nacos.remote.server.grpc.cluster.keep-alive-timeout=20000

## Sets a time(milliseconds) that specify the most aggressive keep-alive time clients are permitted to configure. The typical default is 5 minutes
#nacos.remote.server.grpc.cluster.permit-keep-alive-time=300000
```

### 4.3 编写运行脚本start.sh
```shell
#!/bin/bash
sudo docker run -d --name nacos-dm \
--restart=always \
-p 8848:8848 \
-p 9848:9848 \
-p 9849:9849 \
-e TZ='Asia/Shanghai' \
-e MODE='standalone' \
-v /data/nacos-dm-data/conf/application.properties:/home/nacos/conf/application.properties \
-v /data/nacos-dm-data/logs:/home/nacos/logs \
-v /usr/share/zoneinfo/Asia/Shanghai:/usr/share/zoneinfo/Asia/Shanghai \
nacos/nacos-server-dm:2.2.1
```

### 4.4 编写日志脚本logs.sh
```shell
#!/bin/bash
sudo docker logs -f -t --tail 1000 nacos-dm
```

### 4.5 编写删除脚本remove.sh
```shell
#!/bin/bash
sudo docker stop nacos-dm && sudo docker rm nacos-dm
```

