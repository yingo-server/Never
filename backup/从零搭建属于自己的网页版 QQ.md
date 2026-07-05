

# 从零搭建属于自己的网页版 QQ：NapCatQQ + Stapxs QQ Lite 完全部署指南

## 引言：为什么要自己搭建网页版 QQ？

在即时通讯工具层出不穷的今天，QQ 依然是我们日常生活中不可或缺的一部分。然而，官方 QQ 客户端功能臃肿、资源占用高，而 Web 版 QQ 早已停止维护。对于开发者、社群管理者或者单纯想要一个轻量级 QQ 客户端的用户来说，自己搭建一个网页版 QQ 成了一种极具吸引力的选择。

本文将带你从零开始，在 Linux 服务器上部署一套完整的网页版 QQ 系统。这套系统的核心由两个开源项目构成：

- **NapCatQQ**：基于 NTQQ 的 OneBot v11 协议实现，负责与 QQ 服务器通信，提供 HTTP 和 WebSocket 接口。简单来说，NapCatQQ 是这套系统的“消息引擎”。
- **Stapxs QQ Lite**：一个兼容 OneBot 协议的非官方 QQ Web 客户端，基于 Vue 构建。它是这套系统的“用户界面”，让你在浏览器中就能收发 QQ 消息。

通过本文，你将学会在 2 核 2GB 的 Linux 服务器上完成 NapCatQQ 的安装、配置和开机自启，部署 Stapxs QQ Lite 前端，并通过 WebSocket 将两者连接起来。更值得一提的是，我们还会深入探讨如何为这套系统配置 HTTPS/WSS 加密，确保通信安全——这部分内容将作为专门的扩展章节，满足你对安全性的更高追求。

---

## 第一章：准备工作

### 1.1 硬件与系统要求

根据 NapCatQQ 的官方文档，Linux 部署支持 NodeLoader、AppImage 和 Docker 三种主要方式。本文采用最推荐的 **NodeLoader 原生部署方案**，以下是最低系统要求：

| 项目 | 最低要求 | 推荐配置 |
|------|----------|----------|
| CPU | 1 核 | 2 核 |
| 内存 | 1 GB | 2 GB 或以上 |
| 硬盘 | 20 GB | 40 GB SSD |
| 系统 | Ubuntu 20.04+ / Debian 10+ / CentOS 9 | Ubuntu 22.04 LTS |
| 网络 | 公网 IP | 公网 IP + 域名（推荐） |

**特别说明**：NapCatQQ 设计为在无图形界面的服务器上运行，通过 headless NTQQ 架构绕过 GUI 需求。如果你的服务器是 2 核 2GB 配置，完全足够运行本文所述的全部服务。

### 1.2 软件依赖

在开始安装之前，需要确保系统已安装以下软件：

```bash
# Ubuntu/Debian 系统
sudo apt update
sudo apt install -y curl wget git xvfb screen

# 安装 Node.js 18.x（NapCatQQ 要求 Node.js 18+）
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

**各依赖的作用**：

- **xvfb**（X Virtual Framebuffer）：为无图形界面的服务器提供虚拟显示环境，让 QQ 客户端能够“无头”运行。
- **screen**：用于在后台保持 NapCatQQ 进程运行，即使关闭 SSH 连接也不会中断。
- **Node.js 18+**：NapCatQQ 运行所需的 JavaScript 运行时环境。

### 1.3 域名与端口规划（可选但推荐）

虽然可以使用 IP 地址直接访问，但使用域名能让你更方便地配置 HTTPS 证书，也更容易记忆。

| 服务 | 端口 | 说明 |
|------|------|------|
| NapCatQQ WebUI | 6099 | 管理面板，建议配置 HTTPS |
| NapCatQQ WebSocket | 3001 | 与前端通信的 WebSocket 服务 |
| Nginx 代理（可选） | 3003 | 用于 WSS 反向代理 |

如果你有域名，建议提前将其解析到服务器的公网 IP。如果没有域名，后续可以使用 IP 地址访问，但 HTTPS 配置会稍微复杂一些。

---

## 第二章：NapCatQQ 部署

NapCatQQ 提供了多种 Linux 部署方式。本文采用**一键脚本安装（Rootless 模式）**，这种方式安装快捷、配置清晰，且以普通用户身份运行，安全性更高。

### 2.1 创建普通用户并切换

**不要使用 root 用户运行 NapCatQQ**。建议创建一个专用用户（如 `admin`）：

```bash
# 如果还没有 admin 用户
sudo useradd -m -s /bin/bash admin
sudo passwd admin

# 切换到 admin 用户
su - admin
whoami  # 应输出 admin
```

### 2.2 执行一键安装脚本

**关键**：以普通用户身份执行脚本，**不要加 sudo**。脚本会在需要时自动请求管理员权限。

```bash
curl -o napcat.sh https://nclatest.znin.net/NapNeko/NapCat-Installer/main/script/install.sh && bash napcat.sh
```

安装过程中，脚本会提示你进行选择：
- 是否使用 Shell 安装？选择 `y`
- 是否配置管理工具？选择 `y`（推荐）

安装完成后，你会看到类似以下的输出：

```
[2026-07-05 16:00:31]: - Shell (Rootless) 安装完成 -
[2026-07-05 16:00:31]: 安装位置: /home/admin/Napcat
[2026-07-05 16:00:31]: 启动 Napcat (无需 sudo):
[2026-07-05 16:00:31]:   xvfb-run -a /home/admin/Napcat/opt/QQ/qq --no-sandbox
```

**重要信息**：
- **NapCatQQ 主目录**：`/home/admin/Napcat/opt/QQ/resources/app/app_launcher/napcat`
- **配置文件目录**：`/home/admin/Napcat/opt/QQ/resources/app/app_launcher/napcat/config/`
- **qq 命令路径**：`/home/admin/Napcat/opt/QQ/qq`

---

## 第三章：WebUI 配置（三种方法）

NapCatQQ 的 WebUI 是一个基于 Express 的 Web 管理界面，用于管理账号、配置网络服务和查看日志。以下是三种配置 WebUI 的方法，按推荐程度排序。

### 3.1 方法一：通过 napcat 命令配置（最推荐）

NapCatQQ 安装后自带 `napcat` 命令行工具，这是最直观、最不容易出错的配置方式。

**第一步：进入 napcat 管理界面**

```bash
napcat
```

你会看到一个交互式菜单，选择 **“配置”** 或 **“Config”** 选项。

**第二步：选择 WebUI 配置**

在配置菜单中，选择 **“WebUI”** 相关选项，进入 WebUI 配置界面。

**第三步：填写配置信息**

根据提示依次填写以下信息：

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| Host | `0.0.0.0` | 监听所有网络接口 |
| Port | `6099` | WebUI 访问端口 |
| Token | 自定义强密码 | **必须 13 位以上**，包含字母和数字 |
| Login Rate | `20` | 每小时最大登录尝试次数 |

> **安全提示**：Token 务必设置一个强密码，不要使用默认值。这是 WebUI 的唯一登录凭证。

**第四步：配置 WebSocket 服务端**

回到主菜单，选择 **“网络配置”** → **“新建”** → **“WebSocket 服务端”**，填写：

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| 名称 | `ws-server` | 自定义标识 |
| Host | `0.0.0.0` | 监听所有网络接口 |
| Port | `3001` | WebSocket 服务端口 |
| Token | 可与 WebUI Token 相同或不同 | 前端连接认证 |

**第五步：保存并退出**

按提示保存配置，然后选择 **“重启 NapCat”** 使配置生效。

**第六步：扫码登录**

重启后，终端会显示一个二维码。用手机 QQ 扫描二维码即可完成登录。

---

### 3.2 方法二：通过 napcat 命令快速配置（备选）

如果你熟悉命令行操作，可以使用 `napcat` 命令的子命令快速配置：

```bash
# 进入 napcat 管理界面
napcat

# 选择 "启动" → 新建用户 → 输入 QQ 号
# 选择 "配置" → 进入 WebUI 配置
```

操作流程简述：

1. 输入 `napcat` 进入管理界面
2. 选择 **“启动”**，然后 **“新建用户”**，输入你的 QQ 号
3. 选择 **“配置”**，进入 WebUI 配置页面
4. 依次设置 Host、Port、Token（13 位以上）、Login Rate
5. 返回主菜单，选择 **“网络配置”**，添加 WebSocket 服务端
6. 保存后 **“停止”** 再 **“启动”** NapCatQQ
7. 终端显示二维码，手机 QQ 扫码登录

这种方式的优点是所有配置都在一个界面中完成，不需要手动编辑文件。

---

### 3.3 方法三：手动编辑配置文件（高级）

如果你更喜欢直接编辑配置文件，或者需要通过脚本自动化部署，可以手动创建 `webui.json` 文件。

**第一步：进入配置目录**

```bash
cd /home/admin/Napcat/opt/QQ/resources/app/app_launcher/napcat/config/
```

**第二步：创建 webui.json**

```bash
nano webui.json
```

写入以下内容（将 `your_secure_token` 替换为你的强密码，**必须 13 位以上**）：

```json
{
  "host": "0.0.0.0",
  "port": 6099,
  "token": "your_secure_token",
  "loginRate": 10
}
```

> **注意**：确保使用英文字符和标点。保存时按 `Ctrl+X`，输入 `y`，然后回车。

**第三步：配置 WebSocket 服务端（可选）**

如果通过 WebUI 配置 WebSocket 服务端更方便，你也可以在 `config` 目录下找到账号相关的配置文件（如 `protocol_你的QQ号.json`）进行手动配置，但更推荐通过 WebUI 完成这一步。

**第四步：重启 NapCatQQ**

```bash
# 如果使用 systemd
sudo systemctl restart napcat.service

# 或使用 screen
screen -r napcat
# 按 Ctrl+C 停止，然后重新启动
xvfb-run -a /home/admin/Napcat/opt/QQ/qq --no-sandbox
```

**第五步：验证配置**

访问 `http://你的服务器IP:6099/webui`，输入你设置的 token，应该能成功登录。

---

### 3.4 配置项完整说明

| 字段 | 说明 | 推荐值 |
|------|------|--------|
| `host` | 监听地址，`0.0.0.0` 表示监听所有网络接口 | `"0.0.0.0"` |
| `port` | WebUI 端口 | `6099` |
| `token` | 登录密码，**至少 13 位且包含字母和数字** | 自定义强密码 |
| `loginRate` | 每小时最大登录尝试次数 | `10` |

**安全提示**：如果你的服务器暴露在公网，**务必修改默认 token**。在 v4.8.106 以下版本中，NapCat WebUI 默认使用弱密码（密钥为 `napcat`），存在严重安全风险。

---

## 第四章：后台运行与开机自启

### 4.1 使用 systemd（推荐）

创建服务文件：

```bash
sudo nano /etc/systemd/system/napcat.service
```

写入以下内容（注意 `User` 和 `Group` 应设置为你的普通用户名）：

```ini
[Unit]
Description=NapCatQQ Service
After=network.target

[Service]
Type=simple
User=admin
Group=admin
ExecStart=/usr/bin/xvfb-run -a /home/admin/Napcat/opt/QQ/qq --no-sandbox
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启用并启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable napcat.service
sudo systemctl start napcat.service
sudo systemctl status napcat.service
```

### 4.2 使用 screen（备选）

如果你更喜欢使用 `screen` 来管理进程：

```bash
# 后台启动
screen -dmS napcat bash -c "xvfb-run -a /home/admin/Napcat/opt/QQ/qq --no-sandbox"

# 附加到会话查看
screen -r napcat

# 分离会话（保持后台运行）
# 按 Ctrl+A 然后按 D

# 停止会话
screen -S napcat -X quit
```

### 4.3 验证 NapCatQQ 是否正常运行

```bash
# 检查服务状态（systemd 方式）
sudo systemctl status napcat.service

# 或检查进程（screen 方式）
ps aux | grep qq

# 检查端口监听
sudo netstat -tlnp | grep -E "6099|3001"
```

你应该看到 `6099`（WebUI）和 `3001`（WebSocket）端口正在监听。

---

## 第五章：Stapxs QQ Lite 部署

Stapxs QQ Lite 是一个基于 Vue 的单页应用，可以部署在任何 Web 服务器或静态托管服务上。

### 5.1 获取部署文件

**方式一：下载预构建包（推荐）**

访问 [Stapxs QQ Lite 2.0 Releases 页面](https://github.com/Stapxs/Stapxs-QQ-Lite-2.0/releases)，下载 `Stapxs.QQ.Lite-<版本>-web.zip` 文件。

**方式二：从源码构建**

```bash
git clone https://github.com/Stapxs/Stapxs-QQ-Lite-2.0.git --recursive
cd Stapxs-QQ-Lite-2.0
yarn install
yarn build
```

构建产物在 `dist` 目录中。

> **注意**：如果你的网站不在根域名下运行（例如 `https://example.com/qq/`），需要在构建前修改 `vue.config.js` 中的 `publicPath` 字段。

### 5.2 部署到 Netlify（推荐）

Netlify 是最简单的静态托管平台，支持拖拽部署和自动 HTTPS。

1. 登录 [Netlify](https://app.netlify.com)
2. 将下载的 `web.zip` 文件直接拖拽到部署区域
3. Netlify 会自动解压并生成一个公网访问地址（如 `https://你的项目名.netlify.app`）

**或者**，你也可以通过 GitHub 仓库连接实现自动部署。

### 5.3 部署到自己的 Web 服务器（Nginx）

如果你有自己的 Web 服务器，也可以将 Stapxs 部署到 Nginx 中：

```bash
# 解压部署文件到 Nginx 目录
sudo unzip Stapxs.QQ.Lite-*.web.zip -d /var/www/stapxs/

# 配置 Nginx 站点
sudo nano /etc/nginx/sites-available/stapxs
```

Nginx 配置示例：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    root /var/www/stapxs;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

启用站点并重启 Nginx：

```bash
sudo ln -s /etc/nginx/sites-available/stapxs /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 第六章：连接 NapCatQQ 与 Stapxs QQ Lite

NapCatQQ 与 Stapxs QQ Lite 之间通过 **WebSocket** 协议通信。NapCatQQ 作为 WebSocket 服务端（正向 WS），Stapxs 作为客户端主动连接。

### 6.1 确认 NapCatQQ WebSocket 服务端配置

1. 访问 NapCatQQ WebUI：`http://你的服务器IP:6099/webui`
2. 输入你在 `webui.json` 中设置的 token 登录
3. 进入左侧菜单的 **“网络配置”**
4. 确认存在一个 **“WebSocket 服务端”** 配置，且状态为 **“启用”**
5. 确认端口为 `3001`，Host 为 `0.0.0.0`

如果不存在，点击 **“新建”** 添加一个。

### 6.2 在 Stapxs 中配置连接

1. 访问你部署的 Stapxs QQ Lite 页面
2. 进入设置页面
3. 填写连接信息：

| 字段 | 值 |
|------|-----|
| **WebSocket 地址** | `ws://你的服务器IP:3001` |
| **Token** | 你在 webui.json 中设置的 token |

4. 保存配置，页面应自动连接并显示 QQ 好友和群聊列表。

### 6.3 协议匹配问题（重要）

如果 Stapxs 页面是通过 `https://` 访问的，那么 WebSocket 地址**必须**使用 `wss://`（加密 WebSocket），否则浏览器会因为混合内容安全策略而阻止连接。

如果你的 Stapxs 页面是 `http://`，则可以使用 `ws://`。

---

## 第七章：扩展篇 —— 为 NapCatQQ 配置 HTTPS/WSS

**本章为扩展内容**，适合希望进一步提升系统安全性的用户。如果你仅在内网使用或对安全性要求不高，可以跳过本章。

### 7.1 为什么需要 HTTPS/WSS？

- **数据加密**：防止通信内容被中间人窃听或篡改
- **浏览器安全策略**：HTTPS 页面必须使用 WSS 连接 WebSocket
- **提升信任度**：浏览器地址栏显示安全锁标志

### 7.2 NapCatQQ 原生 SSL 支持（v4.14.1+）

从 NapCatQQ v4.14.1 版本开始，WebUI 原生支持 SSL 证书管理。

**方法一：通过 WebUI 上传证书**

在 WebUI 的设置页面中，找到 **“SSL 证书管理”** 选项，分别上传 `cert.pem`（证书）和 `key.pem`（私钥）文件。

**方法二：直接放置证书文件**

将 `cert.pem` 和 `key.pem` 文件放入 NapCatQQ 的 `config` 目录：

```bash
cp /path/to/cert.pem /home/admin/Napcat/opt/QQ/resources/app/app_launcher/napcat/config/
cp /path/to/key.pem /home/admin/Napcat/opt/QQ/resources/app/app_launcher/napcat/config/
```

**修改 webui.json 启用 SSL**：

```json
{
  "host": "0.0.0.0",
  "port": 6099,
  "token": "your_token",
  "loginRate": 10,
  "ssl": true
}
```

重启 NapCatQQ 后，即可通过 `https://你的服务器IP:6099/webui` 访问 WebUI。

### 7.3 使用 Nginx 反向代理实现 WSS（推荐方案）

如果 NapCatQQ 原生 SSL 配置遇到困难，使用 Nginx 反向代理是更灵活、更稳定的选择。

**整体架构**：

```
浏览器 (wss://) → Nginx (端口 3003, SSL) → NapCatQQ (ws://127.0.0.1:3001)
```

#### 7.3.1 安装 Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

#### 7.3.2 准备 SSL 证书

你可以使用 Let's Encrypt 免费申请证书，或使用已有的证书文件。

```bash
# 将证书复制到 Nginx 目录
sudo mkdir -p /etc/nginx/ssl
sudo cp /path/to/cert.pem /etc/nginx/ssl/server.crt
sudo cp /path/to/key.pem /etc/nginx/ssl/server.key
sudo chmod 644 /etc/nginx/ssl/server.crt
sudo chmod 600 /etc/nginx/ssl/server.key
```

#### 7.3.3 配置 Nginx

创建 Nginx 配置文件：

```bash
sudo nano /etc/nginx/sites-available/napcat-wss
```

写入以下配置（将 `server.344977.xyz` 替换为你的实际域名或 IP）：

```nginx
server {
    listen 3003 ssl;
    server_name server.344977.xyz;

    ssl_certificate      /etc/nginx/ssl/server.crt;
    ssl_certificate_key  /etc/nginx/ssl/server.key;
    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;
    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location /ws/ {
        proxy_pass http://127.0.0.1:3001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400;

        # 跨域配置
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
        add_header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept";
    }
}
```

**关键配置说明**：

- `proxy_http_version 1.1;` 和 `Upgrade` / `Connection` 头是 WebSocket 代理的必备条件
- `proxy_read_timeout 86400;` 防止 WebSocket 长连接因超时被断开
- `Access-Control-Allow-Origin *` 解决浏览器的跨域问题

#### 7.3.4 启用配置并重启 Nginx

```bash
# 禁用可能冲突的默认站点
sudo rm /etc/nginx/sites-enabled/default

# 启用 napcat-wss 站点
sudo ln -s /etc/nginx/sites-available/napcat-wss /etc/nginx/sites-enabled/

# 测试配置并重启
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

#### 7.3.5 放行防火墙端口

**阿里云安全组**：添加入方向规则，放行 TCP 端口 `3003`。

**服务器内部防火墙**：

```bash
# Ubuntu (ufw)
sudo ufw allow 3003/tcp
sudo ufw reload
```

#### 7.3.6 测试 WSS 连接

```bash
curl -v -k -H "Upgrade: websocket" https://你的域名:3003/ws/
```

如果返回 `101 Switching Protocols`，说明 WSS 代理配置成功。

#### 7.3.7 更新 Stapxs 连接地址

在 Stapxs QQ Lite 设置中，将 WebSocket 地址改为：

```
wss://你的域名:3003/ws/
```

Token 保持不变。

### 7.4 常见 HTTPS/WSS 问题排查

| 问题现象 | 可能原因 | 解决方案 |
|----------|----------|----------|
| `ERR_SSL_PROTOCOL_ERROR` | 用 HTTPS 访问了只支持 HTTP 的服务 | 检查是否正确配置了 SSL 证书 |
| `wscat: unable to verify the first certificate` | 自签名证书或证书链不完整 | 使用 `NODE_TLS_REJECT_UNAUTHORIZED=0` 绕过验证（仅测试用） |
| `400 Bad Request` | WebSocket 握手缺少必要头或 Token 认证失败 | 检查 Nginx 是否正确传递了 `Upgrade` 和 `Authorization` 头 |
| 连接成功但无法收发消息 | WebSocket 连接建立但 NapCatQQ 未正确响应 | 检查 NapCatQQ 日志，确认 WebSocket 服务端已启用 |

---

## 第八章：常见问题与故障排除

### 8.1 NapCatQQ 启动失败

**问题**：执行 `xvfb-run -a ...` 后进程立即退出

**排查步骤**：
1. 检查是否安装了 `xvfb`：`which xvfb-run`
2. 查看错误日志：`journalctl -u napcat.service -e --no-pager`（systemd 方式）
3. 检查配置文件语法是否正确

### 8.2 WebUI 无法访问

**问题**：浏览器无法打开 `http://服务器IP:6099/webui`

**排查步骤**：
1. 检查 NapCatQQ 是否正在运行：`ps aux | grep qq`
2. 检查端口是否监听：`sudo netstat -tlnp | grep 6099`
3. 检查阿里云安全组是否放行了 `6099` 端口
4. 检查服务器防火墙：`sudo ufw status`

### 8.3 Stapxs 无法连接 NapCatQQ

**问题**：Stapxs 显示“连接失败”或“WebSocket 错误”

**排查步骤**：
1. 确认 NapCatQQ 的 WebSocket 服务端已启用（WebUI → 网络配置）
2. 确认 WebSocket 端口（默认 3001）已放行
3. 检查协议是否匹配：HTTPS 页面必须用 WSS，HTTP 页面可用 WS
4. 检查 Token 是否正确
5. 查看浏览器控制台（F12）的具体错误信息

### 8.4 端口被占用

**问题**：Nginx 启动失败，错误日志显示 `bind() to 0.0.0.0:80 failed (98: Address already in use)`

**解决方案**：
1. 检查占用端口的进程：`sudo netstat -tlnp | grep :80`
2. 停止冲突的服务，或修改 Nginx 的监听端口

---

## 结语

通过本文的完整指南，你已经学会了如何在 Linux 服务器上从零搭建一套完整的网页版 QQ 系统。从 NapCatQQ 的安装配置，到 Stapxs QQ Lite 的前端部署，再到 WebSocket 连接的建立，每一步都经过了详细的讲解。

更重要的是，我们在扩展章节中深入探讨了 HTTPS/WSS 的配置方案——无论是 NapCatQQ 原生的 SSL 支持，还是通过 Nginx 反向代理实现加密通信，都能让你的系统在安全性上更上一层楼。

这套系统的优势在于：

- **轻量级**：2 核 2GB 的服务器即可流畅运行
- **跨平台**：只要有浏览器，随时随地访问你的 QQ
- **可扩展**：NapCatQQ 提供了完整的 OneBot API，你可以在此基础上开发机器人、自动化工具等

当然，也需要提醒几点注意事项：

1. **账号安全**：建议使用小号测试，避免主号因异常登录被限制
2. **合规使用**：本文方案仅供学习交流，请勿用于商业或非法用途
3. **持续维护**：NapCatQQ 和 Stapxs 都在持续更新，建议定期关注项目动态

希望这篇指南能帮助你顺利搭建属于自己的网页版 QQ。如果在部署过程中遇到任何问题，欢迎查阅项目的官方文档或在社区中寻求帮助。

---

## 参考资源

- [NapCatQQ GitHub 仓库](https://github.com/NapNeko/NapCatQQ)
- [NapCatQQ 官方文档](https://deepwiki.com/NapNeko/NapCatQQ)
- [Stapxs QQ Lite 2.0 GitHub 仓库](https://github.com/Stapxs/Stapxs-QQ-Lite-2.0)
- [OneBot 协议规范](https://onebot.dev/)