---
title: 好用的 nodejs 多版本管理工具 nvm
date: 2024-09-28 18:08:23
categories:
  - 工具
tags:
  - tools
  - 工具
---

[nvm](https://github.com/nvm-sh/nvm) 是一款能够在一台主机上通过命令切换 node 版本，适合于需要支持多个项目且项目之间需要不同版本的 nodejs。安装 nvm 后可以指定下载不同 node 版本，切换 node 版本也是一条命令的事情，简单好操作。

### 安装命令

使用 curl 命令 或 wget 命令安装，需要执行 `install.sh` 。

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

```shell
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

脚本运行的目的是将 nvm 资源路径设置为 `~/.nvm` 并且安装路径设置为环境变量。

### 常用命令

#### 安装 nodejs

安装指定版本的 nodejs，如果只是输入大版本号则安装大版本号下的最新小版本的 nodejs.

```shell
# 直接安装指定版本号的 nodejs
nvm install 12.22.6
# 直接安装指定大版本号下最新版本的 nodejs
nvm install 12
# 直接安装 nodejs 发布的最新版本包
nvm install node
```

相应的卸载就是 install 换成 uninstall.

#### 切换使用 nodejs

切换使用指定版本的 nodejs

```shell
# 代表使用本地安装的12大版本下最大版本的 nodejs
nvm use 12
```

#### 其他常用命令

```shell
# 列出本地可用的 nodejs 版本号
nvm ls
# 查看 nodejs 版本的安装路径
nvm which v20.13.1
```

剩余其他包含指定镜像仓库等直接在 `github` 上找。ÍÍ

