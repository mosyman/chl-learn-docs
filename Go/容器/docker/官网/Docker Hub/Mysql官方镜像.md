
我给你用**最清晰、最实用、能直接干活**的方式，逐段把这份 **MySQL 官方 Docker 镜像文档** 彻底讲透。
不讲废话，只讲你真正能用到的。

---

# 一、这是什么？
这是 **Docker 官方维护的 MySQL 镜像文档**，告诉你：
- 能用哪些版本
- 怎么启动
- 怎么配置
- 怎么持久化数据
- 环境变量含义
- 常见坑

你现在看的就是 **Docker Hub 上 mysql 镜像的完整说明**。

---

# 二、支持哪些版本（最重要的标签）
```
9.6.0, 9.6, 9, innovation, latest
8.4.8, 8.4, 8, lts
8.0.45, 8.0
```

- `latest` = 最新版（现在是 9.6）
- `lts` = 长期支持版（8.4）
- 最常用生产版：**8.0、8.4**

系统版本：
- `oraclelinux9`（默认）
- `debian`（bookworm）

---

# 三、最核心：怎么启动 MySQL 容器
## 最简启动命令
```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -p 3306:3306 \
  mysql:8.0
```

含义：
- `-d` 后台运行
- `--name mysql` 容器名字
- `MYSQL_ROOT_PASSWORD=123456` **必须给 root 设置密码**
- `-p 3306:3306` 端口映射
- `mysql:8.0` 使用 8.0 版本

---

# 四、环境变量（超级重要）
## 1. **MYSQL_ROOT_PASSWORD**
**必须设置**，root 密码。

## 2. **MYSQL_DATABASE**
启动时自动创建的数据库名。

## 3. **MYSQL_USER + MYSQL_PASSWORD**
自动创建一个新用户，并赋予上面数据库的全部权限。

示例：
```bash
-e MYSQL_USER=appuser
-e MYSQL_PASSWORD=123456
-e MYSQL_DATABASE=appdb
```

## 4. **MYSQL_ALLOW_EMPTY_PASSWORD=yes**
允许 root 空密码（**极不安全，不要用**）

## 5. **MYSQL_RANDOM_ROOT_PASSWORD=yes**
自动生成随机 root 密码，输出在日志里。

## 6. **MYSQL_ONETIME_PASSWORD=yes**
首次登录必须改密码（MySQL 5.6+ 支持）

---

# 五、使用 Docker Compose（最推荐）
```yaml
services:
  mysql:
    image: mysql:8.0.45-debian
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
```

启动：
```bash
docker compose up -d
```

---

# 六、数据持久化（必须懂）
MySQL 容器默认数据在容器内，删容器就丢数据。

**正确做法：挂载宿主机目录**
```bash
-v /host/path:/var/lib/mysql
```

一旦挂载，**容器内的 MySQL 就会使用宿主机上的数据**。

---

# 七、自定义 MySQL 配置（my.cnf）
MySQL 容器会自动加载：
```
/etc/mysql/conf.d/*.cnf
```

你只需要挂载目录：
```bash
-v /my/config:/etc/mysql/conf.d
```

里面放 `my.cnf` 即可。

---

# 八、启动时直接传 mysqld 参数
不需要配置文件，直接加在命令最后：
```bash
docker run ... mysql:8.0 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
```

---

# 九、进入容器内部
```bash
docker exec -it mysql bash
```

查看日志：
```bash
docker logs mysql
```

---

# 十、备份与恢复（生产常用）
## 备份
```bash
docker exec mysql sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' > backup.sql
```

## 恢复
```bash
docker exec -i mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < backup.sql
```

---

# 十一、关键注意事项（避坑）
1. **第一次启动会初始化数据库，此时无法连接**，需要等几十秒。
2. **一旦挂载了已有数据目录，环境变量会失效**（密码不会重新设置）。
3. 生产必须挂载数据目录，否则删容器=删库。
4. 不要在生产用 `latest`，固定版本号（8.0 / 8.4）。

---

# 十二、总结（你真正需要记住的）
- 启动必须给 `MYSQL_ROOT_PASSWORD`
- 数据必须挂载 `-v /host:/var/lib/mysql`
- 配置文件挂载到 `/etc/mysql/conf.d`
- 推荐用 **Docker Compose** 管理
- 备份用 `mysqldump` + `docker exec`

---

如果你愿意，我可以**直接给你一份生产可用的 mysql + docker compose 最优配置**，包含：
字符集、时区、慢查询、binlog、innodb 缓冲池、端口、数据持久化、安全配置。

