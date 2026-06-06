# moredoc 在 a30 上的实际部署存档与验证手册

> 这是 2026-06-06 在 a30 算力机上完成的一次 **可工作的** moredoc 测试部署的完整存档。  
> 与 `06-moredoc-Docker部署手册.md` 的关系：06 是通用部署指引（任意服务器都能套用）；本文档是 **a30 这一次部署的实际拓扑、参数、操作记录与验证清单**，用于你手动复测和后续运维。

---

## 0. 一眼速览

| 项 | 值 |
|---|---|
| 公网访问地址（主站） | <http://183.56.181.9:34445/> |
| 公网访问地址（管理后台） | <http://183.56.181.9:34445/admin> |
| Swagger 文档 | <http://183.56.181.9:34445/swagger> |
| 默认管理员账号 | `admin` / `mnt.ltd` ⚠️ **首次登录立即改密码** |
| 服务器 SSH | `ssh -p 34451 a30@183.56.181.9`（免密码登录已配） |
| 工作目录 | `/home/a30/moredoc-test/` |
| 应用版本 | moredoc CE v3.4.0（开源版） |

---

## 1. 服务器环境（a30）

### 1.1 硬件 / 系统

| 项 | 值 |
|---|---|
| OS | Ubuntu 24.04 LTS (Noble) |
| 内核 | 6.8.0-90-generic |
| Docker | 29.1.5 |
| Compose | v5.0.1（独立 plugin） |
| 用户 | `a30`（uid=1000，免密 sudo，在 `docker` 组） |
| 磁盘可用 | / 上 749G 空闲 |

### 1.2 网络拓扑（关键）

a30 在云算力平台的 NAT 后面，**对外只有特定端口映射可用**：

```
    公网 IP 183.56.181.9
            │
            ▼
    ┌──────────────────────┐
    │  云平台前置网关/反代  │
    └──────────────────────┘
            │
            ▼ 端口映射规则：
            │   公网 34451     ──► 内网 22         (SSH)
            │   公网 34440-34450 ──► 内网 18000-18010 (10 个发布端口)
            ▼
    ┌──────────────────────┐
    │ a30 物理机            │
    │ 内网 IP 192.168.255.114│
    │ 出口 IP 113.87.81.204  │
    └──────────────────────┘
```

**moredoc 占用：**
- 宿主机端口 **18005** → 公网 **34445**

**18000-18010 中可用余量：** 18000-18004、18006-18010（9 个）可用于后续新增服务。

### 1.3 Docker daemon.json（仅供参考，不要随意改）

```json
{
  "default-runtime": "nvidia",
  "runtimes": { "nvidia": { "args": [], "path": "nvidia-container-runtime" } },
  "bip": "172.17.0.1/16",
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m", "max-file": "3" },
  "registry-mirrors": [
    "https://09ae59743a0010e70fedc00db70f1da0.mirror.swr.myhuaweicloud.com",
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com"
  ],
  "max-concurrent-downloads": 3,
  "max-download-attempts": 10,
  "max-concurrent-uploads": 5
}
```

`default-runtime: nvidia` 说明这台机器主用途是 GPU 容器；普通容器不会受影响。

---

## 2. moredoc 部署拓扑

### 2.1 容器与网络

```
┌──────────────────── 宿主机 a30 ────────────────────┐
│                                                     │
│   工作目录：/home/a30/moredoc-test/                 │
│                                                     │
│   ┌────────────────── docker bridge ──────────────┐ │
│   │  网络名 moredoc-test_moredoc-net (172.18.0.0) │ │
│   │                                                │ │
│   │   ┌──────────────────┐    ┌─────────────────┐ │ │
│   │   │ moredoc-mysql    │    │ moredoc-app     │ │ │
│   │   │ image mysql:8.0  │◄───┤ image           │ │ │
│   │   │ 3306（容器内）   │    │ moredoc-ce:     │ │ │
│   │   │ healthcheck ✓    │    │ v3.4.0-local    │ │ │
│   │   │                  │    │ 8880（容器内）  │ │ │
│   │   └────────┬─────────┘    └──────┬──────────┘ │ │
│   │            │                     │            │ │
│   │     mysql-data 卷            uploads/logs/    │ │
│   │                              cache 卷         │ │
│   └────────────────────────────────────┼──────────┘ │
│                                        │            │
│                                  ports 18005:8880   │
└────────────────────────────────────────┼────────────┘
                                         │
                              0.0.0.0:18005 (docker-proxy)
                                         │
                                公网 34445 (云平台映射)
```

### 2.2 文件清单（/home/a30/moredoc-test/）

| 文件 | 用途 | 大小（近似） |
|---|---|---|
| `Dockerfile` | 基于 `ubuntu:22.04`，COPY 二进制构建运行镜像 | <1 KB |
| `docker-compose.yml` | 编排 mysql + moredoc 两个容器、网络、卷 | ~1.3 KB |
| `app.toml` | moredoc 应用配置（DB DSN、JWT、日志级别等） | ~0.6 KB |
| `extracted/` | 解压后的 moredoc CE 二进制 + 前端资源 + 字典 | ~280 MB |
| `moredoc_ce_linux_amd64.tar.gz` | 原始下载包，可保留备份 | ~28 MB |
| `build.log` | 最近一次构建/启动的输出（如果有） | 视情况 |

### 2.3 命名卷（持久化数据）

| 卷名 | 挂载点（容器内） | 内容 |
|---|---|---|
| `moredoc-test_mysql-data` | `/var/lib/mysql` | MySQL 所有数据 |
| `moredoc-test_moredoc-uploads` | `/app/uploads` | 用户上传的文档 |
| `moredoc-test_moredoc-logs` | `/app/logs` | moredoc 应用日志 |
| `moredoc-test_moredoc-cache` | `/app/cache` | 文档转换缓存 |

`docker compose down` 不会删卷；`docker compose down -v` 才会。

### 2.4 关键配置参数

**docker-compose.yml 要点：**
- `mysql` 服务用 `healthcheck`（`mysqladmin ping`）；moredoc 用 `depends_on.condition: service_healthy` 等待 MySQL 真正可用
- MySQL 强制 `mysql_native_password`（兼容 moredoc 用的 Go MySQL driver）
- MySQL **不映射端口到宿主机**（仅容器网络内可达）
- moredoc 容器 CMD：`./moredoc syncdb && exec ./moredoc serve`（首次/每次启动都跑 syncdb，幂等）

**app.toml 要点：**
- `port = "8880"`
- `[database] dsn`：连 compose 服务名 `mysql:3306`，用户 `moredoc`，独立数据库 `moredoc`
- `[jwt] secret`：随机 32 字节 hex（已设置）
- `[jwt] expireDays = 365`
- `enableSwagger = true`（开了 Swagger 接口浏览）
- `level = "info"`（生产级日志）

> 实际密钥不在文档中明文写出——存在 a30 上的 `app.toml` 与 `docker-compose.yml`。如需查看：`ssh -p 34451 a30@183.56.181.9 cat /home/a30/moredoc-test/app.toml`

---

## 3. 验证清单（手动测试）

按顺序逐项验证。每项给出**预期结果**，便于你判断是否正常。

### 3.1 SSH 与容器层

```bash
# 1) 登录 a30
ssh -p 34451 a30@183.56.181.9

# 2) 容器状态——两个都该是 Up
cd ~/moredoc-test
docker compose ps
```
**预期：**
```
NAME            STATUS                        PORTS
moredoc-app     Up xxx                        0.0.0.0:18005->8880/tcp
moredoc-mysql   Up xxx (healthy)              3306/tcp, 33060/tcp
```

```bash
# 3) moredoc 应用日志关键行
docker compose logs --tail=80 moredoc | grep -E "syncdb success|server start|error"
```
**预期：** 看到 `syncdb success` 和 `server start {"port": 8880}`。开头会有一条 `Table 'moredoc.mnt_user' doesn't exist` 的 error，**这是首次启动正常现象**（syncdb 之前先查表，查不到才去建表），随后立刻 success。

```bash
# 4) MySQL 健康检查
docker compose exec mysql mysqladmin ping -uroot -p<root_pw>
```
**预期：** `mysqld is alive`

### 3.2 本机网络层

```bash
# 5) 宿主机自测 18005
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:18005/
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:18005/admin
```
**预期：** 主站 200；/admin 200（前端路由 SPA，admin/login 也是 200）。

```bash
# 6) docker-proxy 是否监听 18005
sudo ss -tlnp | grep 18005
```
**预期：** `LISTEN 0.0.0.0:18005  users:(("docker-proxy",...))`

### 3.3 公网访问层（最终用户体验）

在你本机（不在 a30 上）：

```bash
# 7) 公网 34445 直连——注意如果本机有 http_proxy 要绕过
curl -sS --noproxy '*' -o /dev/null -w "%{http_code}\n" http://183.56.181.9:34445/
curl -sS --noproxy '*' -L -o /dev/null -w "%{http_code}\n" http://183.56.181.9:34445/admin
```
**预期：** 都是 200。

```bash
# 8) 浏览器访问
# 直接访问：http://183.56.181.9:34445/
```
**预期：** 看到魔豆文库首页（深蓝色主题，有"文档" "文章"等导航）。

### 3.4 业务功能层（登录 + 上传）

1. 浏览器打开 <http://183.56.181.9:34445/admin>
2. 用 `admin` / `mnt.ltd` 登录
3. **立即去【用户管理】改密码**——这是公网可达的实例
4. 主站点击"上传"，传一个小 PDF（<5MB），检查能否预览
5. 后台【文档管理】里看到刚上传的文档

**预期：** 全部成功。如果文档转码失败（比如 office 文档），请见 §5 排错。

---

## 4. 常用运维操作

### 4.1 查看日志

```bash
cd ~/moredoc-test

# 应用日志（实时滚动）
docker compose logs -f moredoc

# 数据库日志
docker compose logs -f mysql

# 最近 100 行
docker compose logs --tail=100 moredoc
```

### 4.2 重启 / 停止

```bash
# 重启 moredoc 应用（不动数据库，最常用）
docker compose restart moredoc

# 全部重启
docker compose restart

# 停止（保留数据）
docker compose stop

# 启动
docker compose start

# 停止并删除容器（保留命名卷数据）
docker compose down
```

### 4.3 进入容器

```bash
# 进 moredoc 容器
docker compose exec moredoc bash

# 进 mysql shell
docker compose exec mysql mysql -uroot -p<root_pw> moredoc

# 看 admin 表结构
docker compose exec mysql mysql -uroot -p<root_pw> -D moredoc -e "DESC mnt_user;"
```

### 4.4 修改配置后生效

```bash
# 改了 app.toml 之后
docker compose restart moredoc

# 改了 docker-compose.yml 或 Dockerfile 之后
docker compose down
docker compose up -d --build
```

### 4.5 备份 / 恢复数据

```bash
# 备份数据库（在 a30 上）
docker compose exec -T mysql mysqldump -uroot -p<root_pw> moredoc | gzip > moredoc-$(date +%F).sql.gz

# 备份上传的文件
docker run --rm -v moredoc-test_moredoc-uploads:/src -v $PWD:/dst alpine \
  tar czf /dst/uploads-$(date +%F).tar.gz -C /src .

# 恢复（举例）
gunzip < moredoc-2026-06-06.sql.gz | docker compose exec -T mysql mysql -uroot -p<root_pw> moredoc
```

### 4.6 升级 moredoc 版本

未来出新版时：

```bash
cd ~/moredoc-test
docker compose down                    # 停服务（数据卷保留）
# 备份数据库（重要！）
docker compose exec mysql mysqldump ... > backup.sql   # 如果还在跑就先备份

# 下载新版二进制包
rm -rf extracted
curl -fSL -o moredoc_ce_new.tar.gz "https://gitee.com/mnt-ltd/moredoc/releases/download/<新版本>/moredoc_ce_<新版本>_linux_amd64.tar.gz"
mkdir extracted && tar -xzf moredoc_ce_new.tar.gz -C extracted

# 重新构建镜像（注意改 image tag）
# 编辑 docker-compose.yml，把 image: moredoc-ce:v3.4.0-local 改成新版本号
docker compose up -d --build
docker compose logs -f moredoc       # 看 syncdb 是否成功（新版表结构会自动迁移）
```

### 4.7 彻底清理（不留痕）

```bash
cd ~/moredoc-test
docker compose down -v                                # 容器 + 命名卷
docker image rm moredoc-ce:v3.4.0-local mysql:8.0     # 可选：删镜像
docker network prune -f                               # 可选：清空闲网络
cd ~ && rm -rf ~/moredoc-test                         # 删工作目录
```

执行完，宿主机除了 Docker 引擎本身，不留任何 moredoc 相关内容。

---

## 5. 故障排查

### 5.1 公网访问 502 Bad Gateway

**症状：** `curl http://183.56.181.9:34445/` 返回 502。

**可能原因 1（最常见）：** 你本机有 HTTP 代理（v2ray/clash/proxychains），代理拒绝 34445 这种非标端口。
**确认：** `echo $http_proxy; env | grep -i proxy`
**修复：** 用 `curl --noproxy '*'`，或浏览器临时关掉代理，或在代理软件里把 `183.56.181.9` 加直连。

**可能原因 2：** moredoc 服务真的挂了。
**确认：** `ssh -p 34451 a30@183.56.181.9 'cd ~/moredoc-test && docker compose ps && curl -sI http://127.0.0.1:18005/'`
**修复：** 看日志找原因，`docker compose restart moredoc`。

### 5.2 公网访问 Connection refused / Cannot connect

**症状：** `curl --noproxy '*' http://183.56.181.9:34445/` 直接连不上。

**原因：** 云平台端口映射没生效，或 docker 没把端口映射出去。
**确认：**
```bash
ssh -p 34451 a30@183.56.181.9 'sudo ss -tlnp | grep 18005'
# 应该看到 0.0.0.0:18005
```
**修复：** 如果 18005 没监听 → `docker compose up -d`；如果有监听但外网仍连不上 → 联系算力平台确认 34445↔18005 端口映射状态。

### 5.3 容器启动失败：iptables not found

**症状：** `docker compose up` 时报 `fork/exec /usr/sbin/iptables: no such file or directory`，或 docker daemon 自身起不来。

**原因：** a30 上 `/usr/sbin/iptables` 软链被破坏（这台机器已观察到过一次）。
**修复：**
```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-nft
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-nft
sudo systemctl reset-failed docker.service
sudo systemctl restart docker
iptables --version   # 应该输出 v1.8.10 (nf_tables)
docker run --rm hello-world   # 验证 docker 正常
cd ~/moredoc-test && docker compose up -d
```

### 5.4 文档上传后无法预览

**症状：** PDF 能传能看；Office（docx/pptx/xlsx）能传但预览空白或报错。

**原因：** moredoc 转码 Office 文档要调用 LibreOffice，但当前 Dockerfile 没装 LibreOffice（为了镜像小）。
**修复方案 A（最快）：** 只测 PDF/EPUB/TXT。
**修复方案 B（要支持 Office）：** 在 `Dockerfile` 的 `apt-get install` 行追加 `libreoffice-core libreoffice-writer libreoffice-calc libreoffice-impress`，然后 `docker compose up -d --build`。镜像会从 ~270MB 增加到 ~1.5GB。

### 5.5 syncdb 反复报错

**症状：** moredoc 容器反复重启，日志里 `syncdb` 后接 panic / connection refused。

**确认：**
```bash
docker compose logs mysql | tail -30      # MySQL 是否真的 ready
docker compose exec mysql mysql -uroot -p<root_pw> -e "SHOW DATABASES;"
```

**典型原因：** MySQL 首次初始化没完成 moredoc 就来了（healthcheck 应该已防住，但极端情况）。
**修复：** `docker compose restart moredoc`（不要重启 mysql，数据可能损坏）。

### 5.6 admin 登录提示密码错误

**原因 A：** 密码确实改过。去后台或 mysql 直接重置：
```bash
# 用 syncdb 不会覆盖已存在的 admin；要手动重置只能改 hash
# 紧急情况下可以删用户让 syncdb 重建：
docker compose exec mysql mysql -uroot -p<root_pw> -D moredoc \
  -e "DELETE FROM mnt_user WHERE id=1;"
docker compose exec moredoc /app/moredoc --config /app/app.toml syncdb
# 然后用 admin / mnt.ltd 再登
```

**原因 B：** 真的是默认密码不对。看本文 §0，是 `mnt.ltd`（带点）不是 `mntltd`。

---

## 6. 附录：当前实际使用的关键文件内容（结构供参考）

> 实际文件在 a30 的 `/home/a30/moredoc-test/`。密钥/密码隐去。

### 6.1 Dockerfile

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive \
    TZ=Asia/Shanghai

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates tzdata curl \
    && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY extracted/ /app/
RUN chmod +x /app/moredoc

EXPOSE 8880

# 启动时先 syncdb（幂等：同步表结构与初始化数据），再 serve
CMD ["bash", "-c", "set -e; cd /app && ./moredoc --config /app/app.toml syncdb && exec ./moredoc --config /app/app.toml serve"]
```

### 6.2 docker-compose.yml（结构）

```yaml
name: moredoc-test

services:
  mysql:
    image: mysql:8.0
    container_name: moredoc-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "<隐去>"
      MYSQL_DATABASE: "moredoc"
      MYSQL_USER: "moredoc"
      MYSQL_PASSWORD: "<隐去>"
      TZ: Asia/Shanghai
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [moredoc-net]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p<隐去>"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  moredoc:
    build:
      context: .
      dockerfile: Dockerfile
    image: moredoc-ce:v3.4.0-local
    container_name: moredoc-app
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "18005:8880"
    volumes:
      - ./app.toml:/app/app.toml:ro
      - moredoc-uploads:/app/uploads
      - moredoc-logs:/app/logs
      - moredoc-cache:/app/cache
    networks: [moredoc-net]

volumes:
  mysql-data:
  moredoc-uploads:
  moredoc-logs:
  moredoc-cache:

networks:
  moredoc-net:
    driver: bridge
```

### 6.3 app.toml（结构）

```toml
level = "info"
logEncoding = "console"
enableSwagger = true
port = "8880"

[database]
driver = "mysql"
dsn = "moredoc:<隐去>@tcp(mysql:3306)/moredoc?charset=utf8mb4&loc=Local&parseTime=true"
showSQL = false
maxOpen = 10
maxIdle = 10

[jwt]
secret = "<隐去>"
expireDays = 365

[logger]
filename = ""
maxSizeMB = 10
maxBackups = 10
maxDays = 30
compress = true
```

---

## 7. 部署过程的关键决策记录（FYI）

部署过程中遇到并解决的两个非典型问题，记下来便于以后参考：

1. **官方 mnt-ltd 阿里云镜像 namespace 是私有的**（`registry.cn-shenzhen.aliyuncs.com/mnt-ltd/moredoc` 报 `unauthorized: authentication required`）。**应对：** 从 Gitee release 拉 CE 版 Linux amd64 二进制（`moredoc_ce_v3.4.0_linux_amd64.tar.gz`），自己写最小 Dockerfile 包装，避开私有 registry。

2. **a30 上 `/usr/sbin/iptables` 软链曾被破坏，导致 docker daemon 无法启动且 buildkit 报 iptables not found**。**应对：** `sudo update-alternatives --set iptables /usr/sbin/iptables-nft` 重建软链，详见 §5.3。

3. **moredoc 默认账号密码不在二进制 strings 里**——是在源码里硬编码 `admin`/`mnt.ltd`，syncdb 时加盐 MD5 入库。**结论：** 公网可达的实例**必须首次登录立即改密码**。

---

## 参考

- 通用 Docker 部署手册：`docs/06-moredoc-Docker部署手册.md`
- moredoc Gitee 仓库：<https://gitee.com/mnt-ltd/moredoc>
- moredoc 使用文档：<https://www.bookstack.cn/read/moredoc>
