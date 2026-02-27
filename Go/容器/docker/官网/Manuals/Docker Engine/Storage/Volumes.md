

# Docker Volumes 超详细整理
我把你这段官方文档**逐段翻译 + 结构化 + 重点提炼**，做成一份能直接拿去学习、面试、写笔记的完整版 Volumes 教程。

---

# 1. 什么是 Volumes（数据卷）
**Volumes 是 Docker 管理的、用于容器的持久化数据存储。**

- 由 Docker 创建、管理、维护
- 存储在 Docker 主机的指定目录里
- 挂载到容器后，容器读写这个目录
- 与 bind mount 类似，但**完全由 Docker 托管**，与主机核心功能隔离

你可以：
- 手动创建：`docker volume create`
- 容器/服务启动时自动创建

---

# 2. 什么时候应该用 Volumes
**Volumes 是 Docker 官方推荐的容器数据持久化方案。**

## ✅ 推荐使用场景
- 需要**数据持久化**（容器删了数据还在）
- 需要**备份、迁移、共享数据**
- 需要跨 Linux/Windows 容器使用
- 需要**多个容器安全共享数据**
- 需要**新卷自动预填充容器原有文件**
- 需要**高性能 I/O**

## ❌ 不适合的场景
- 你需要**从宿主机直接读写卷内文件**
  → 这种情况用 **bind mount**

---

# 3. 为什么 Volumes 比直接写容器层更好
1. **不会让容器体积变大**
2. **性能更高**
    - 容器可写层需要 storage driver（联合文件系统）
    - Volumes 直接写主机文件系统，少一层抽象
3. **数据生命周期独立**，容器删了数据不删

---

# 4. 非持久化数据：用 tmpfs mount
如果数据不需要落地，只是临时状态：
→ 使用 `tmpfs mount`，只存在内存中，性能最高。

---

# 5. Volume 的生命周期（核心重点）
- Volume **独立于容器生命周期**
- 容器删除，卷**不会自动删除**
- 可以**同时挂载给多个容器**
- 无容器使用时依然保留
- 手动清理：`docker volume prune`

---

# 6. 挂载卷到已有数据目录会发生什么？
## ① 挂载 **非空卷** 到容器已有目录
- 容器**原有文件会被隐藏**
- 类似 Linux 把 U 盘挂载到 `/mnt`，原有内容看不见
- 无法简单恢复，只能**重建容器**

## ② 挂载 **空卷** 到容器已有目录
- **容器原有文件会自动复制到卷里**
- 这是“预填充数据”的常用方式

如果不想自动复制：
加参数 `volume-nocopy`

---

# 7. 命名卷 & 匿名卷
## 命名卷（Named Volume）
- 有自定义名字，如 `my-volume`
- 可复用、可共享、可管理

## 匿名卷（Anonymous Volume）
- 随机唯一名字
- 容器删除时**不会自动删**，除非启动时加 `--rm`

---

# 8. 挂载语法：--mount vs --volume(-v)
## 官方推荐：`--mount`
更清晰、功能更强、支持所有高级选项。

```bash
docker run --mount type=volume,src=卷名,dst=容器内路径
```

## 简洁写法：`-v/--volume`
```bash
docker run -v 卷名:容器内路径:选项
```

---

# 9. --mount 完整参数表
```
type=volume,src|source=卷名,dst|target=容器路径
[,readonly|ro]
[,volume-subpath=子目录]
[,volume-nocopy]
[,volume-opt=驱动参数]
```

| 参数 | 作用 |
|------|------|
| src/source | 卷名 |
| dst/target | 容器内挂载点 |
| readonly/ro | 只读 |
| volume-subpath | 挂载卷内子目录 |
| volume-nocopy | 禁止自动复制容器文件到空卷 |
| volume-opt | 传给卷驱动的参数 |

---

# 10. -v 参数格式
```
[卷名]:容器路径[:选项]
```

示例：
```bash
-v myvol:/data:ro
```

---

# 11. Volume 常用命令
## 创建卷
```bash
docker volume create my-vol
```

## 查看卷列表
```bash
docker volume ls
```

## 查看卷详情
```bash
docker volume inspect my-vol
```

## 删除卷
```bash
docker volume rm my-vol
```

## 清理所有未使用卷
```bash
docker volume prune
```

---

# 12. 启动容器并挂载卷
## --mount
```bash
docker run -d --name devtest \
  --mount source=myvol2,target=/app \
  nginx:latest
```

## -v
```bash
docker run -d --name devtest \
  -v myvol2:/app \
  nginx:latest
```

---

# 13. 只读卷
```bash
--mount ...,readonly

# 或
-v ...:ro
```

---

# 14. 挂载卷内子目录 volume-subpath
```bash
--mount src=logs,dst=/var/log/app1,volume-subpath=app1
```
用途：
- 一个卷分多个子目录给不同容器用
- 隔离日志/数据

---

# 15. 在 Docker Compose 中使用卷
## 自动创建卷
```yaml
services:
  frontend:
    image: node:lts
    volumes:
      - myapp:/home/node/app
volumes:
  myapp:
```

## 使用外部已创建的卷
```yaml
volumes:
  myapp:
    external: true
```

---

# 16. 在 Docker Swarm 服务中使用卷
```bash
docker service create \
  --replicas=4 \
  --mount source=myvol2,target=/app \
  nginx:latest
```

注意：
- `local` 驱动的卷**不能跨节点共享**
- 要共享需用 NFS、云存储、第三方驱动

---

# 17. 用容器初始化卷内容
如果你挂载空卷到容器里**已有文件的目录**：
→ Docker 会**自动把容器里的文件复制到卷中**。

示例（NGINX 默认页面）：
```bash
-v nginx-vol:/usr/share/nginx/html
```

---

# 18. 跨机器/共享存储：Volume Driver
Volumes 支持第三方驱动，如：
- NFS
- CIFS/Samba
- AWS S3
- rclone
- 块设备（硬盘/分区）

## NFS 示例
```bash
docker service create \
  --mount 'type=volume,src=nfsvol,dst=/app,volume-driver=local,volume-opt=type=nfs,volume-opt=device=:/data,"volume-opt=o=addr=10.0.0.10"' \
  nginx
```

## CIFS/Samba 示例
```bash
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt device=//server/share \
  --opt o=username=user,password=pass \
  cifsvol
```

---

# 19. 备份与恢复卷
## 备份
```bash
docker run --rm --volumes-from 容器名 -v $(pwd):/backup \
  ubuntu tar cvf /backup/backup.tar /容器内路径
```

## 恢复
```bash
docker run --rm --volumes-from 新容器 -v $(pwd):/backup \
  bash -c "cd /路径 && tar xvf /backup/backup.tar --strip 1"
```

---

# 20. 自动删除匿名卷
启动时加 `--rm`，容器删除时**匿名卷一起删**：
```bash
docker run --rm -v /foo busybox
```

---

# 21. Volumes vs Bind Mounts vs tmpfs 对比（极简总结）
| 方式 | 管理方 | 性能 | 跨平台 | 共享 | 适合场景 |
|------|--------|------|--------|------|----------|
| **Volumes** | Docker | 高 | ✅ | ✅ | 数据库、持久化、共享 |
| Bind Mount | 宿主机 | 中 | ❌ | ✅ | 开发挂载代码 |
| tmpfs | 内存 | 极高 | ❌ | ❌ | 临时缓存、密钥 |

---

如果你需要，我可以再给你做一份：
**Docker Volumes 高频面试题 + 答案**
或者
**生产环境最佳实践（数据库、配置、日志怎么挂卷）**











