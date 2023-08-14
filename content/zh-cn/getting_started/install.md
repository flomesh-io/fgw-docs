---
title: "安装"
weight: 1
---

## 安装 Pipy

参考 [Pipy 安装文档](https://flomesh.io/pipy/docs/en/getting-started/build-install) 安装最新版本的 Pipy，可以从 [Pipy 的 Release 页面](https://github.com/flomesh-io/pipy/releases) 下载安装。

> Pipy 是一个单一的可执行文件，下载后可以将其复制到操作系统的 bin 目录即可。

## 运行

FGW 的运行有两种模式：从文件系统启动和从网络启动，后者就是 [Pipy Repo 模式](https://flomesh.io/pipy/docs/en/operating/repo/0-intro)。

### 从文件系统启动

可以从 GitHub 仓库克隆 FGW 的脚本。

```shell
git clone https://github.com/flomesh-io/fgw.git
```

克隆完仓库之后，进入到工程根目录中，执行下面的命令即可：

```shell
pipy ./pjs/main.js
```

### 从网络启动

运行 Repo 模式，我们可以使用 FGW 的 Docker 镜像。

```shell
docker run --rm -d --name fgw -p 6060:6060 flomesh/fgw:{{< param pipy_version >}}
```

执行下面的命令启动：

```shell
pipy http://localhost:6060/repo/base/fgw/
```