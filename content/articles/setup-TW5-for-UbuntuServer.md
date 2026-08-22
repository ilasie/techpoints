+++
date = '2025-10-03T23:28:00+08:00'
draft = false
title = '在Ubuntu Server上安装TiddlyWiki5'
+++

### Step 1. install the system

获取[安装映像](https://mirrors.ustc.edu.cn)，在**获取发行版映像**中选择`Ubuntu (amd64, Server)`，下载。\
在另一台电脑上安装[Rufus](https://rufus.ie/zh)，取一块U盘刷入下载的安装映像，插入主机，从U盘启动。安装系统。

### Step 2. set up the environment

登录用户。安装`nodejs`环境。

```bash
sudo apt install -y curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

sudo apt install -y nodejs

# verify
node -v
npm -v
```

### Step 3. deploy TW5-Bob

```bash
sudo apt install git

git clone --depth=1 --branch v5.1.22  https://github.com/Jermolene/TiddlyWiki5.git
git clone --depth=1 https://github.com/OokTech/TW5-Bob.git TiddlyWiki5/plugins/OokTech/Bob
```

### Step 4. Create your wiki with TW5-Bob

```bash
mkdir TiddlyWiki5/Wikis
cp -r TiddlyWiki5/plugins/OokTech/Bob/MultiUserWiki TiddlyWiki5/Wikis/BobWiki/

# start up
cd TiddlyWiki5
node ./tiddlywiki.js Wikis/BobWiki --wsserver
```

默认端口为`127.0.0.1:8080`，可在`BobWiki/settings/settings.json`中更改。
