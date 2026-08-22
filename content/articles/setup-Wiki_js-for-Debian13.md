+++
date = '2025-10-25T11:54:00+08:00'
draft = false
title = '在Debian 13上搭建Wiki.js'
+++

### 1. 安装必要依赖（nodejs setup_*.x版本18以上）
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt install -y nodejs postgresql curl
```

### 2. 创建专用系统用户

```bash
adduser --system --group --shell /bin/bash --home /opt/wiki wiki
```

- `--system`：创建系统用户（UID < 1000）
- `--group`：同时创建同名组
- `--home /opt/wiki`：指定主目录为 `/opt/wiki`
- 用户名为 `wiki`，后续 Wiki.js 将以此用户运行

### 3. 启动并启用 PostgreSQL
```bash
systemctl enable --now postgresql
```

### 3. 创建数据库和用户
```bash
sudo -u postgres psql <<EOF
CREATE DATABASE yourwiki;
CREATE USER wiki WITH ENCRYPTED PASSWORD 'your_secure_password_here';
GRANT ALL PRIVILEGES ON DATABASE yourwiki TO wiki;
GRANT CREATE ON SCHEMA public TO wiki;
\q
EOF
```

### 5. 切换到 wiki 用户并进入其主目录
```bash
su - wiki
cd /opt/wiki
```
### 6. 获取Wki.js
```
wget https://github.com/Requarks/wiki/releases/latest/download/wiki-js.tar.gz
tar -xzf wiki-js.tar.gz
rm wiki-js.tar.gz
```

### 7. 初始化
```bash
mv config.sample.yml config.yml
```

编辑你的`config.yml`，按照注释填入你设定的值。

### 8. 临时运行
```bash
node server/index.js
```

查看运行的效果。成功运行后通常可访问`http://主机ip:设定端口`查看wiki页面。

### 9. 创建 systemd 服务

```bash
exit
cat > /etc/systemd/system/wiki.service <<EOF
[Unit]
Description=Wiki.js
After=network.target

[Service]
Type=notify
User=wiki
Group=wiki
WorkingDirectory=/opt/wiki
ExecStart=/usr/bin/node server/index.js
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

- **`Description=`**  
  服务的简短描述，用于 `systemctl status` 等命令显示。

- **`After=network.target`**  
  表示：**必须在网络服务启动之后，才启动 Wiki.js**,避免因网络未就绪导致数据库连接失败。

- **`Type=notify`**
  表示 Wiki.js 会通过 `sd_notify` 协议主动通知 systemd “我已启动完成”。Wiki.js 内部支持此协议（基于 Node.js 的 `systemd` 模块），所以用 `notify` 最合适。

- **`User=wiki`和`Group=wiki`**
  **以 `wiki` 用户身份运行进程**（`Group=`即创建用户时设定的同名组），这是**安全最佳实践**：即使 Wiki.js 被攻破，攻击者也只能获得 `wiki` 用户权限，无法控制系统。

- `WorkingDirectory=/opt/wiki/server`
  进程启动时的**当前工作目录**。Wiki.js 会在此目录下寻找 `config.yml`，所以必须设为 `/opt/wiki/server`。

- `ExecStart=/usr/bin/node index.js`
  **启动命令**：运行 `/usr/bin/node` 并执行 `index.js`必须使用**绝对路径**（`/usr/bin/node` 而不是 `node`），因为 systemd 不加载用户 PATH。

- `Restart=always`
  **无论因何原因退出（崩溃、被杀、错误），都自动重启**。其他选项：`on-failure`（仅失败时重启）、`no`（默认，不重启）

### 10. 启用服务
```bash
systemctl daemon-reload
systemctl enable wiki
```

重启主机，无需登录即可进入网页。
