---
title: 开发者指引
description: ""
weight: 3
---

## 前提条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 测试用的工具 netstat、cat、grep、openssl、curl、shpec

## 快速开始

所有的 PipyJS 脚本都位于 `pjs` 目录中，FGW 启动后将会运行于 [Pipy Repo 模式](https://flomesh.io/pipy/docs/en/operating/repo/0-intro)，并将所有的 PipyJS 脚本发布到 Repo 中。

因此开始前，请确保已经安装了 Pipy，可以参考[安装文档](https://flomesh.io/pipy/docs/en/getting-started/build-install)。

## 构建

执行 `main build` 来编译 FGW，编译完成后可以在 `bin` 目录找到 fgw 二进制文件。

## 测试

在 tests/shpec 目录下，先运行 pre_test.sh 检查测试环境，再运行 shpec 测试所有用例。