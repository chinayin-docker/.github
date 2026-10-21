<h1 align="center">chinayin-docker</h1>

<p align="center">
  <b>自建自维护的容器镜像</b> · 一个镜像一个仓库 · 每日 schedule 自动重建<br/>
  Curated & self-maintained container images · one repo per image · rebuilt daily
</p>

<p align="center">
  <a href="https://hub.docker.com/u/chinayin"><img alt="Docker Hub" src="https://img.shields.io/badge/docker.io-chinayin-2496ED?style=flat-square&logo=docker&logoColor=white"></a>
  <img alt="Images" src="https://img.shields.io/badge/images-23-informational?style=flat-square">
  <img alt="Platforms" src="https://img.shields.io/badge/platforms-amd64%20%7C%20arm64-informational?style=flat-square">
  <img alt="Rebuild" src="https://img.shields.io/badge/rebuild-daily-brightgreen?style=flat-square">
</p>

---

### 📦 镜像目录 · Catalog

完整说明、tag 规则与构建方式见 [**CATALOG.md**](https://github.com/chinayin-docker/.github/blob/master/CATALOG.md)（唯一的目录源）。

#### 基础镜像 · Base

| 镜像 | 版本 | 说明 |
| --- | --- | --- |
| [`debian`](https://github.com/chinayin-docker/debian) | `trixie` · `bookworm` · `bullseye` · `buster`（含 `-slim`） | 所有 apt 系镜像的底座 |

#### 语言运行时与构建工具链 · Runtimes & toolchains

| 镜像 | 版本 |
| --- | --- |
| [`golang`](https://github.com/chinayin-docker/golang) | `1.27.3` · `1.26.10` — `-trixie` / `-alpine` / `-ci`（goreleaser + golangci-lint） |
| [`node`](https://github.com/chinayin-docker/node) | `24.16.0` · `22.22.3` · `20.19.6` |
| [`python`](https://github.com/chinayin-docker/python) | `3.14.4` · `3.12.13` · `3.11.15` |
| [`php`](https://github.com/chinayin-docker/php) | `8.4.8` · `8.2.28` · `7.4.33` |
| [`php-deploy`](https://github.com/chinayin-docker/php-deploy) | `8.3.6` · `8.2.16` · `7.4.33`（`-cli` / `-srv` / `-sso`） |
| [`openjdk`](https://github.com/chinayin-docker/openjdk) | `8` · `11.0.31` · `17.0.19` · `21.0.11`（Adoptium Temurin） |
| [`maven`](https://github.com/chinayin-docker/maven) | `3.9.16-jdk-{8,11,17,21}` |

#### Web · 搜索 · 中间件

| 镜像 | 版本 |
| --- | --- |
| [`nginx`](https://github.com/chinayin-docker/nginx) | `1.31.7`（mainline）· `1.30.6`（stable）· njs `1.0.2` · `trixie-slim` |
| [`openresty`](https://github.com/chinayin-docker/openresty) | `1.31.1.1` · `bookworm-slim` |
| [`elasticsearch`](https://github.com/chinayin-docker/elasticsearch) | `9.5.5` · `8.19.23` · `7.17.28`（内置 ik / pinyin / stconvert 分词） |
| [`kibana`](https://github.com/chinayin-docker/kibana) | `9.5.5` · `8.19.23` · `7.17.28`（与 elasticsearch 同轨） |
| [`canal`](https://github.com/chinayin-docker/canal) | `1.1.8` |

#### 安全与网络 · Security & networking

| 镜像 | 版本 |
| --- | --- |
| [`hfish`](https://github.com/chinayin-docker/hfish) | `3.3.4` · `0.6.5`（server / client） |
| [`v2ray`](https://github.com/chinayin-docker/v2ray) | `5.x`（`bookworm` / `bullseye`） |

#### 开发与运维工具 · Dev & ops tooling

| 镜像 | 版本 |
| --- | --- |
| [`kaniko-project`](https://github.com/chinayin-docker/kaniko-project) | `v1.23.2`（`-debug` / `-slim`） |
| [`kiro-cli`](https://github.com/chinayin-docker/kiro-cli) | `latest` |
| [`playwright`](https://github.com/chinayin-docker/playwright) | `1.58.0` on `python-3.11` / `python-3.12` |
| [`apidoc`](https://github.com/chinayin-docker/apidoc) | `1.2.0` |

#### 应用 · Applications

| 镜像 | 版本 |
| --- | --- |
| [`xxl-job-admin`](https://github.com/chinayin-docker/xxl-job-admin) | `2.4.0` |
| [`xxl-job-executor-springboot`](https://github.com/chinayin-docker/xxl-job-executor-springboot) | `2.4.0` |
| [`BOCEPayJavaBridge`](https://github.com/chinayin-docker/BOCEPayJavaBridge) | 源码构建（`master` / tag `v*`） |

---

### 🚀 使用

```bash
docker run --rm -it chinayin/golang:1.27
docker run --rm -it chinayin/node:24
docker run --rm -it chinayin/nginx:1.31.7
docker run --rm -it chinayin/elasticsearch:9.5.5
```

也可以直接当基础镜像用：

```dockerfile
FROM chinayin/golang:1.27-ci
FROM chinayin/debian:trixie-slim
```

### 🏷️ Tag 规则

| Tag | 含义 |
| --- | --- |
| `<version>` | 完整版本，如 `1.31.7`、`8.19.23` |
| `<minor>` / `<major>` | 滚动 tag，跟随该轨道的补丁与安全更新，如 `1.31`、`8` |
| `-<variant>` | 变体，如 `-alpine`、`-trixie`、`-ci`、`-slim` |

### 🔄 怎么维护

- 每个镜像一个仓库（`chinayin-docker/<name>`），版本在 `env.hcl` 或 `Dockerfile` 的 `ARG` 里 pin 住
- `docker buildx bake` + GitHub Actions：**push 即构建**，另外**每日 schedule 重建**，保证带上游安全更新
- 双架构 `linux/amd64` + `linux/arm64` 多架构清单（`openresty` 目前仅 amd64）

### 📊 状态

| 状态 | 镜像 |
| --- | --- |
| ✅ 正常构建 | 18 个（含 `debian`、`golang`、`nginx`、`elasticsearch`、`kibana`、`playwright` …） |
| ⚠️ 上游导致失败 | `php`、`php-deploy`（Debian 11 源已归档）、`apidoc`（`deasync` 原生编译） |
| ⏸ 暂停 | `hfish`（调度工作流被 GitHub 自动停用，待升级后重启） |

> 细节、变更记录与构建命令都在 [**CATALOG.md**](https://github.com/chinayin-docker/.github/blob/master/CATALOG.md) 里。
