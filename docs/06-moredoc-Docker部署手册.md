# moredoc Docker 部署手册（测试环境）

> 目标：在服务器上用 Docker 干净地部署 moredoc，通过 **18005** 端口对外访问，用于测试。  
> 原则：所有组件跑在 Docker 里，使用独立网络 + 命名卷，不污染宿主机；一条命令即可彻底清除。

---

## 0. 重要安全提醒（务必先看）

你在对话里贴出了服务器地址、端口、用户名和密码（密码为 `1`）。请注意：

1. **请尽快修改这个密码。** `a30` / `1` 这种弱口令 + 对公网暴露端口，极易被自动化扫描爆破，服务器有被入侵风险。
2. 出于安全，我（助手）**不会也不应该代你登录服务器输入密码**。下面的命令请你自己在终端执行。
3. moredoc 默认会开放注册等面向 C 端的功能，测试完若要长期使用，记得参照《05-最终实施方案》做内网化裁剪。

---

## 1. 部署思路

moredoc 运行需要两个组件：

- **moredoc 应用本身**（Go 单服务，默认 HTTP 端口 `8880`）
- **MySQL 数据库**

我们用 Docker Compose 把这两者编排在一个独立的 bridge 网络里，数据用命名卷持久化。对外只把 moredoc 的 `8880` 映射到宿主机的 **`18005`**，MySQL 不对外暴露。这样宿主机除了 Docker 本身，不会被装任何东西，清理时一条命令连数据一起删干净。

---

## 2. 部署步骤

### 步骤 1：SSH 登录服务器

在你自己的终端执行（会提示输入密码）：

```bash
ssh -p 34451 a30@183.56.181.9
```

### 步骤 2：确认 Docker 环境

```bash
docker --version
docker compose version    # 或老版本：docker-compose --version
```

如果没有 Docker，先安装（需要 sudo 权限）：

```bash
curl -fsSL https://get.docker.com | sh
```

### 步骤 3：建立独立工作目录

```bash
mkdir -p ~/moredoc-test && cd ~/moredoc-test
```

### 步骤 4：获取官方部署包（推荐做法）

moredoc 官方的 release 包里自带正确的 `docker-compose.yml` 和 `app.example.toml`，能保证镜像名和配置 schema 都是最新且正确的。推荐用官方包，再改两个地方（端口、密码）即可：

```bash
# 从官方仓库的 release 页面下载最新版部署包（请到下方地址确认最新版本号）
# https://github.com/mnt-ltd/moredoc/releases
# 也可从国内 Gitee 镜像下载：https://gitee.com/mnt-ltd/moredoc/releases

# 解压后进入目录，里面通常包含 docker-compose.yml 与 app.example.toml
```

> 说明：moredoc 的镜像名/版本 tag 会随版本更新，以官方 release 包内的 `docker-compose.yml` 为准最稳妥。下面第 5 节给出一份等价的参考编排文件，你可以直接用，但镜像 tag 请对照官方最新值。

### 步骤 5：准备配置

把官方包里的 `app.example.toml` 复制为 `app.toml`：

```bash
cp app.example.toml app.toml
```

需要确认/修改的关键项（在 `app.toml` 中）：

- **数据库连接 `[database] dsn`**：用户名/密码/库名要和下面 compose 里 MySQL 的设置一致，host 用 compose 的服务名 `mysql`。
- **存储 `[storage]`**：测试阶段用默认的本地存储即可（`local`），后续再切 S3/MinIO。
- **JWT 密钥**：改成一个随机字符串。

### 步骤 6：启动

```bash
docker compose up -d
```

首次启动会拉取镜像、初始化数据库，稍等 1~2 分钟。查看日志：

```bash
docker compose logs -f moredoc
```

### 步骤 7：放行端口并访问

确保云服务器安全组 / 防火墙放行 **18005**：

```bash
# ufw 示例
sudo ufw allow 18005/tcp
# firewalld 示例
sudo firewall-cmd --add-port=18005/tcp --permanent && sudo firewall-cmd --reload
```

然后浏览器访问：

```
http://183.56.181.9:18005
```

管理后台默认地址一般是 `http://183.56.181.9:18005/admin`，初始管理员账号见官方文档（通常首次需在后台或通过初始化数据创建）。

---

## 3. 参考 docker-compose.yml

> 以下为一份结构等价的参考编排（端口已设为 18005）。**镜像 tag 请对照官方 release 内的值替换**，避免用到不存在的版本。

```yaml
name: moredoc-test

services:
  mysql:
    image: mysql:8.0
    container_name: moredoc-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "ChangeMe_Root_8x!"
      MYSQL_DATABASE: "moredoc"
      MYSQL_USER: "moredoc"
      MYSQL_PASSWORD: "ChangeMe_Moredoc_8x!"
      TZ: Asia/Shanghai
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - moredoc-net
    # 注意：不映射端口到宿主机，DB 仅容器内网可见，更安全

  moredoc:
    # ↓↓↓ 镜像请以官方 release / 仓库说明为准，替换成最新 tag ↓↓↓
    image: registry.cn-shenzhen.aliyuncs.com/mnt-ltd/moredoc:latest
    container_name: moredoc-app
    restart: unless-stopped
    depends_on:
      - mysql
    ports:
      - "18005:8880"          # 宿主机 18005 -> 容器内 moredoc 8880
    volumes:
      - ./app.toml:/app/app.toml        # 挂载配置
      - moredoc-uploads:/app/uploads    # 本地存储的上传文件持久化
      - moredoc-logs:/app/logs
    networks:
      - moredoc-net

volumes:
  mysql-data:
  moredoc-uploads:
  moredoc-logs:

networks:
  moredoc-net:
    driver: bridge
```

> 关于挂载路径：moredoc 容器内的工作目录、配置文件名、上传目录可能与上面略有出入（如 `/app` vs 其它），请以官方包里的 compose 为准对齐 `volumes` 路径。核心思路不变：**配置文件挂进去、上传目录用命名卷持久化、只暴露 18005**。

---

## 4. app.toml 关键片段（参考）

数据库部分要和 compose 的 MySQL 对应（host 写服务名 `mysql`）：

```toml
[database]
driver = "mysql"
dsn = "moredoc:ChangeMe_Moredoc_8x!@tcp(mysql:3306)/moredoc?charset=utf8mb4&parseTime=true&loc=Local"

[jwt]
secret = "请改成一段随机字符串"
expireDays = 365

# 存储：测试阶段用本地存储；将来切对象存储时改这里
# moredoc 支持 local / minio / aliyunoss / ... ，参见官方手册
```

> 具体字段名以官方 `app.example.toml` 为准（不同版本可能有差异），不要照搬到报错。

---

## 5. 验证清单

- [ ] `docker compose ps` 两个容器都是 `Up` 状态
- [ ] `docker compose logs moredoc` 无致命报错，提示监听 8880
- [ ] 安全组/防火墙已放行 18005
- [ ] 浏览器能打开 `http://183.56.181.9:18005`
- [ ] 能登录后台、上传一个测试文件、在线预览正常

---

## 6. 清理（测试完彻底删除，不留痕）

```bash
cd ~/moredoc-test
docker compose down -v        # 停止并删除容器 + 命名卷（数据一并清除）
docker image rm registry.cn-shenzhen.aliyuncs.com/mnt-ltd/moredoc:latest mysql:8.0   # 可选：删镜像
cd ~ && rm -rf ~/moredoc-test # 删除工作目录
```

执行完以上命令，宿主机除了 Docker 引擎本身，不会残留任何 moredoc 相关内容。

---

## 7. 常见问题

- **18005 打不开**：先排查云厂商安全组（最常见），再排查宿主机防火墙，最后 `docker compose ps` 看容器是否起来。
- **moredoc 连不上数据库**：检查 `app.toml` 的 dsn 里 host 是否为 `mysql`（compose 服务名），密码是否与 compose 一致；MySQL 首次初始化较慢，可重启 moredoc 容器重试。
- **镜像拉不动**：国内可换用 moredoc 官方提供的国内镜像源，或先 `docker pull` 单独拉取。
- **找不到镜像 tag**：到 https://github.com/mnt-ltd/moredoc 或其 Gitee 镜像确认最新版本号和镜像地址。

---

## 参考

- moredoc GitHub：https://github.com/mnt-ltd/moredoc
- moredoc 使用手册（含 Docker 部署、存储配置）：https://www.bookstack.cn/read/moredoc
