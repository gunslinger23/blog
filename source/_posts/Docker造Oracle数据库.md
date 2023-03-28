---
title: Docker造Oracle数据库
date: 2023-03-28 14:00:00
tags:
  - Docker
  - 数据库
  - Oracle
categories:
  - 运维
---

## 参考地址

1. https://github.com/oracle/docker-images/tree/main/OracleDatabase/SingleInstance
2. https://github.com/steveswinsburg/oracle21c-docker

## 本文使用软件

- Docker
- Oracle Database 21c

## 正文

### 准备

1. 克隆官方仓库 `https://github.com/oracle/docker-images` 或者直接下载压缩包 其中我们只需要 `docker-images/OracleDatabase/SingleInstance` 这个文件
2. 接着我们下载 `Oracle Database 21c` 的安装包 `LINUX.X64_213000_db_home.zip` https://www.oracle.com/database/technologies/oracle21c-linux-downloads.html
3. 将压缩包扔到目录 `OracleDatabase/SingleInstance/dockerfiles/21.3.0` 不要解压

### 构建

```bash
cd OracleDatabase/SingleInstance/dockerfiles
./buildContainerImage.sh -v 21.3.0 -e
```

> 确保构建机子环境拥有足够的内存和硬盘空间，4G内存，21G硬盘以上

### 运行

```bash
docker run \
--name oracle21c \
-p 1521:1521 \
-p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /opt/oracle/oradata \
-d \
oracle/database:21.3.0-ee
```

或者使用挂载的方式

```bash
docker run \
--name oracle21c \
-p 1521:1521 -p 5500:5500 \
-e ORACLE_PDB=orcl \
-e ORACLE_PWD=password \
-e INIT_SGA_SIZE=3000 \
-e INIT_PGA_SIZE=1000 \
-v /path/to/store/db/files/:/opt/oracle/oradata \
-d \
oracle/database:21.3.0-ee
```

> 请注意挂载路径权限
> oracle 初始化密码有要求，需要大于8位 + 至少一位大写字母

等待十几分钟后，容器日志出现以下内容，说明数据库已经启动成功

```
#########################
DATABASE IS READY TO USE!
#########################
```

### 杂项

遇到的小问题需要解决

#### 修改数据集

还原数据库字符集需要与备份的一致，所以需要转换

```bash
sqlplus / as sysdba;
```

```sql
conn sys/root as sysdba
shutdown immediate;
startup mount
ALTER SYSTEM ENABLE RESTRICTED SESSION;
ALTER SYSTEM SET JOB_QUEUE_PROCESSES=0;
ALTER SYSTEM SET AQ_TM_PROCESSES=0;
alter database open;
ALTER DATABASE character set INTERNAL_USE ZHS16GBK;
shutdown immediate;
```

#### 创建用户

```sql
create user USER identified by USER;

grant create session to USER;
grant connect, resource to USER;
grant unlimited tablespace to USER;
grant create any table to USER;
grant create any materialized view to USER;
grant create any index to USER;
grant create any view to USER;
grant create any procedure to USER;
grant create any trigger to USER;
grant create any sequence to USER;
grant create any synonym  to USER;
grant select any dictionary to USER;
grant debug any procedure to USER;
```

#### 恢复数据

如果要还原指定用户数据，需要登录该用户

```bash
imp user/user@servicename file=backup.dmp fromuser=user touser=user ignore=y
```
