---
title: java项目环境配置
publishDate: 2026-01-15
description: '介绍java项目中的各种环境配置步骤'
tags:
  - java
  - project
  - env
language: '中文'
---

## 基本环境

我用的是windows + WSL2(ubuntu22) + docker

windows负责写代码

WSL提供Linux环境（类似服务器）

Docker提供MYSQL，Redis镜像，将这些组件容器化，方便启动，停止或者删除

如果是云服务器，操作基本类似

### ssh客户端

ssh客户端直接用vscode，轻量级，自带文件传输（直接拖或者复制粘贴），自带图形界面好管理

## WSL2

jdk和maven都要直接安装到wsl2环境上

### JDK

java直接安装jdk17的包：

```bash
sudo apt-get install openjdk-17-jdk
```

如果想搜索有什么版本的jdk能安装，可以用：

```bash
sudo apt-cache search java sdk
```

### maven

ubuntu22默认安装maven3.6，一般需要maven3.8版本往上

在github仓库中有对应的环境安装包：[env](https://github.com/fuzhengwei/xfg-dev-tech-docker-install)

clone下来，在`environment/maven`目录下，有3.8版本安装包，目前里面的安装脚本有点问题（默认使用apt下载），不要使用

在`environment/maven`目录打开终端：

新建一个安装脚本，安装的maven会被放在`/opt`目录下：

```bash
#!/usr/bin/env bash
set -e

# ===== 配置区 =====
MAVEN_VERSION="3.8.8"
MAVEN_ARCHIVE="apache-maven-${MAVEN_VERSION}.zip"
INSTALL_BASE="/opt"
INSTALL_DIR="${INSTALL_BASE}/apache-maven-${MAVEN_VERSION}"
PROFILE_FILE="$HOME/.bashrc"

# ===== 工具函数 =====
info()  { echo -e "\033[0;34m[INFO]\033[0m $1"; }
ok()    { echo -e "\033[0;32m[OK]\033[0m   $1"; }
warn()  { echo -e "\033[0;33m[WARN]\033[0m $1"; }
err()   { echo -e "\033[0;31m[ERR]\033[0m  $1"; exit 1; }

# ===== 前置检查 =====
info "检查 Java 环境"
command -v java >/dev/null 2>&1 || err "未检测到 Java，请先安装 JDK"

info "检查 Maven 安装包"
[[ -f "$MAVEN_ARCHIVE" ]] || err "未找到 $MAVEN_ARCHIVE"

# ===== 解压并安装 =====
info "安装 Maven 到 $INSTALL_DIR"
sudo mkdir -p "$INSTALL_BASE"

if [[ -d "$INSTALL_DIR" ]]; then
    warn "已存在 $INSTALL_DIR，跳过解压"
else
    sudo unzip -q "$MAVEN_ARCHIVE" -d "$INSTALL_BASE"
    ok "解压完成"
fi

[[ -x "$INSTALL_DIR/bin/mvn" ]] || err "mvn 可执行文件不存在"

# ===== 配置环境变量（用户级）=====
info "配置环境变量到 $PROFILE_FILE"

# 删除旧配置（仅 Maven）
sed -i '/# >>> maven >>>/,/# <<< maven <<</d' "$PROFILE_FILE"

cat >> "$PROFILE_FILE" <<EOF

# >>> maven >>>
export MAVEN_HOME=$INSTALL_DIR
export PATH=\$MAVEN_HOME/bin:\$PATH
# <<< maven <<<
EOF

# 立即生效
export MAVEN_HOME="$INSTALL_DIR"
export PATH="$MAVEN_HOME/bin:$PATH"

# ===== 验证 =====
info "验证 Maven"
mvn -v | head -n 1

ok "Maven ${MAVEN_VERSION} 安装完成"
info "如新终端中 mvn 不生效，请执行: source ~/.bashrc"

```

如果想**卸载**：

```bash
#!/usr/bin/env bash
set -e

# ===== 配置区 =====
MAVEN_VERSION="3.8.8"
INSTALL_DIR="/opt/apache-maven-${MAVEN_VERSION}"
PROFILE_FILE="$HOME/.bashrc"

info() { echo -e "\033[0;34m[INFO]\033[0m $1"; }
ok()   { echo -e "\033[0;32m[OK]\033[0m   $1"; }
warn() { echo -e "\033[0;33m[WARN]\033[0m $1"; }

# ===== 删除 Maven =====
if [[ -d "$INSTALL_DIR" ]]; then
    info "删除 $INSTALL_DIR"
    sudo rm -rf "$INSTALL_DIR"
    ok "Maven 文件已删除"
else
    warn "$INSTALL_DIR 不存在，跳过"
fi

# ===== 清理环境变量 =====
info "清理 ~/.bashrc 中的 Maven 配置"
sed -i '/# >>> maven >>>/,/# <<< maven <<</d' "$PROFILE_FILE"

ok "环境变量已清理"
info "请执行: source ~/.bashrc 或重新打开终端"

```



## docker

docker上需要做的事：拉取mysql，redis，mq等镜像，然后直接启动，还有就是部署一整个项目

windows上安装docker直接参考[WSL 上的 Docker 容器入门 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/tutorials/wsl-containers)

按教程一路点下去（先配置WSL2再安装docker，这样docker会自动在wsl2配好对应docker环境）

之后要使用docker的时候先在windows上启动docker desktop，然后在wsl2中操作

### mysql

拉取mysql镜像

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/xfg-studio/mysql:8.0.32
```

在clone下来的[env](https://github.com/fuzhengwei/xfg-dev-tech-docker-install)环境中，找到`run_install_software.sh`

添加可执行权限后运行

选择安装mysql

默认用户：root

默认密码：123456

默认端口：wsl2的13306

### redis

拉取redis镜像

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/xfg-studio/redis:6.2
```

在clone下来的[env](https://github.com/fuzhengwei/xfg-dev-tech-docker-install)环境中，找到`run_install_software.sh`

添加可执行权限后运行

选择安装redis

无用户名无密码

默认端口：wsl2的16379

可以使用docker tag给镜像起其他名称

### 项目部署



## 内网穿透

下载地址：[NATAPP-内网穿透](https://natapp.cn/)

使用方式也很简单，下载exe客户端放到一个固定位置，配置`config.ini`：

```ini
# https://natapp.cn/ - 支持免费使用，但付费会更稳定一些
# 将本文件放置于natapp同级目录 程序将读取 [default] 段
# 在命令行参数模式如 natapp -authtoken=xxx 等相同参数将会覆盖掉此配置
# 命令行参数 -config= 可以指定任意config.ini文件
[default]
authtoken=      #对应一条隧道的authtoken，你需要更换为你的。否则不能正常启动。
clienttoken=                    #对应客户端的clienttoken,将会忽略authtoken,若无请留空,
log=none                        #log 日志文件,可指定本地文件, none=不做记录,stdout=直接屏幕输出 ,默认为none
loglevel=ERROR                  #日志等级 DEBUG, INFO, WARNING, ERROR 默认为 DEBUG
http_proxy=                     #代理设置 如 http://10.123.10.10:3128 非代理上网用户请务必留空
```

### 微信鉴权

**配置NATAPP**：在NATAPP中购买一个VIP隧道，顺道买一个**二级域名**，配置本地web端口为本地web服务的端口

**微信测试平台**：[测试平台](https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index)

注册之后：

- 记录右上角的微信号

- 接口配置信息的`URL`配置为本地web服务中处理微信鉴权的controller地址：组合以下两个

  二级域名：`http://baishui.nat100.top/`

  本地接口：`api/v1/weixin/portal/receive`

- 接口配置信息的`Token`随便设置就行

- JS接口安全域名就填上面二级域名

配置完之后，启动本地服务，此时提交接口配置信息会成功，失败的话查看本地服务的报错即可

扫描**测试号二维码**，关注，发送信息，此时信息会被微信服务器转发给本地服务处理
