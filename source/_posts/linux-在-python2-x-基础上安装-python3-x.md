---
title: linux 在 python2.x 基础上安装 python3.x
date: 2024-11-23 23:19:39
tags:
  - python
  - linux
  - 
categories:
  - python
---

这种文其实可以不写，浏览器中一大把，但也确实很多都不太对。这里说的安装 python3.x 使用 pyenv 进行管理，这样可以很好的管理所需的 python 3.x 各种版本。

我经历了比较多的问题，我会每步都记录，如果你这种问题就跳过。

### yum 修改阿里仓库源

```shell
# 备份当前的 yum 配置
sudo cp -ar /etc/yum.repos.d /etc/yum.repos.d.bak
# 删除原有的 repos 文件
sudo rm -f /etc/yum.repos.d/*.repo
# 下载阿里云的 CentOS YUM 源配置
# 根据你的 CentOS 版本选择下面的命令执行
# CentOS 7
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# CentOS 8
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo
# 清除缓存并生成新的缓存
sudo yum clean all
sudo yum makecache
```

### 安装 pyenv

pyenv 需要基于 github 上的仓库，所以前期需要安装 git 并设置 token 访问。安装 git 直接 yum 安装就行。这里会单独说明设置 token 访问。

```shell
git config --global credential.helper store
git clone https://${token}@${repo_url}
```

就是保存 token 验证信息后，再安装 pyenv。通过命令 `curl https://pyenv.run | bash`。并在 `~/.bashrc` 增加配置

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
export PATH="$PYENV_ROOT/shims:$PATH"
eval "$(pyenv init -)"
```

### 安装 python3.x

使用命令 `pyenv install 3.11.10` 结果卡住在包下载上，只能手动下载 python 安装包，放到 `~/.pip/cache` 之后再执行 `pyenv install 3.11.10` 命令。

安装后会出现 `ERROR: The Python ssl extension was not compiled. Missing the OpenSSL lib` 的问题，这里耗费的时间最长，百度的都不对。**最终还是得看官网说明的解决方案。**

#### openssl 安装

确定在 1.0.2 版本以上，我这里选的是 1.1.1 版本。先下载 openssl 的源码 [1.1.1 | Library](https://openssl-library.org/source/old/1.1.1/index.html)，完成手动安装

```shell
./config --prefix=/usr/local/openssl_1_1 no-zlib
make & make install
ln -s /usr/local/openssl_1_1/include/openssl /usr/include/openssl
ln -s /usr/local/openssl_1_1/lib/libssl.so.1.1 /usr/local/lib64/libssl.so
ln -s /usr/local/openssl_1_1/bin/openssl /usr/bin/openssl
echo "/usr/local/openssl_1_1/lib" >> /etc/ld.so.conf
ldconfig -v
```

使用 `openssl version -d` 命令确定安装路径，使用一下命令安装 安装 python3.x，命令是结合 linux 版本和要安装后的 python 版本以及尝试后的结果。

```shell
LDFLAGS="-L/usr/lib64/openssl" \
CPPFLAGS="-I/usr/include/openssl" \
CONFIGURE_OPTS="--with-openssl=<openssl install prefix> --with-openssl-rpath=auto" \
pyenv install -v 3.11.10
```

### 参考附件

- [error-the-python-ssl-extension-was-not-compiled-missing-the-openssl-lib](https://github.com/pyenv/pyenv/wiki/Common-build-problems#error-the-python-ssl-extension-was-not-compiled-missing-the-openssl-lib)

- [【linux】centos编译安装openssl1.1.1_centos安装openssl1.1.1-CSDN博客](https://blog.csdn.net/2301_76933862/article/details/143360338)
