# Docker Images · Catalog

<p align="left">
  <a href="https://hub.docker.com/u/chinayin"><img alt="Docker Hub" src="https://img.shields.io/badge/docker.io-chinayin-2496ED?logo=docker&logoColor=white"></a>
  <a href="https://github.com/chinayin-docker"><img alt="GitHub" src="https://img.shields.io/badge/github-chinayin--docker-181717?logo=github&logoColor=white"></a>
  <img alt="Rebuild" src="https://img.shields.io/badge/rebuild-daily%20schedule-brightgreen">
  <img alt="Images" src="https://img.shields.io/badge/images-23-informational">
</p>

A curated set of self-maintained container images. Every image lives in its own repository under
[`chinayin-docker`](https://github.com/chinayin-docker), is published to Docker Hub as `chinayin/<name>`,
and is rebuilt on a daily schedule so it keeps carrying upstream security fixes.

This file is the catalog: what exists, which versions are currently pinned, and where it comes from.
It lives in the [`chinayin-docker/.github`](https://github.com/chinayin-docker/.github) repository:

* a condensed version is rendered on the organization landing page —
  [github.com/chinayin-docker](https://github.com/chinayin-docker), from [`profile/README.md`](./profile/README.md)
* it is also published as the Docker Hub landing page for
  [`chinayin/images`](https://hub.docker.com/r/chinayin/images) by [`.github/workflows/dockerhub.yml`](./.github/workflows/dockerhub.yml)

## How it works

| | |
| --- | --- |
| **Source of truth** | one repository per image — `chinayin-docker/<name>` |
| **Version pinning** | `env.hcl` (`version` / `minor`) and the leading `ARG` / `ENV` of the `Dockerfile` |
| **Registry** | `docker.io/chinayin/<name>` |
| **Platforms** | `linux/amd64` + `linux/arm64` (multi-arch manifest; `openresty` is amd64 only) |
| **Build** | `docker buildx bake` driven by `.github/workflows/ci.yml` |
| **Rebuild trigger** | push to `master` **and** a daily `schedule` cron |
| **Base images** | `chinayin/debian` where upstream packages are apt-based, otherwise the vendor image |

## Tag conventions

* `<version>` — full version, e.g. `1.31.7`, `8.19.23`
* `<minor>` — rolling tag pinned to the current minor, e.g. `1.31`, `8`
* `-<variant>` — upstream variant or our build flavor, e.g. `-alpine`, `-trixie`, `-ci`, `-slim`

Because images are rebuilt daily, a rolling tag (`1.31`) picks up upstream patch + security updates;
a full version tag (`1.31.7`) is reproducible until the next explicit version bump.

---

## Catalog

### 1 · Base

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`debian`](https://github.com/chinayin-docker/debian) | `trixie` · `trixie-slim` · `bookworm` · `bookworm-slim` · `bullseye-slim` · `buster-slim` | `debian` (official) | [debian.org](https://www.debian.org/) |

Minimal Debian with the small conveniences we rely on downstream (`install_packages`, tzdata, locales).
Every apt-based image in this catalog starts here.

### 2 · Language runtimes & build toolchains

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`golang`](https://github.com/chinayin-docker/golang) | `1.27.3` (`1.27`) · `1.26.10` (`1.26`) — each as `-trixie`, `-alpine` and `-ci` | `golang:*-trixie` / `-alpine` | [go.dev](https://go.dev/dl/) |
| [`node`](https://github.com/chinayin-docker/node) | `24.16.0` (`24`) · `22.22.3` (`22`) · `20.19.6` (`20`) | `node` (official) | [nodejs.org](https://nodejs.org/) |
| [`python`](https://github.com/chinayin-docker/python) | `3.14.4` (`3.14`) · `3.12.13` (`3.12`) · `3.11.15` (`3.11`) | `python` (official) | [python.org](https://www.python.org/) |
| [`php`](https://github.com/chinayin-docker/php) | `8.4.8` (`8.4`) · `8.2.28` (`8.2`) · `7.4.33` (`7.4`) | `chinayin/debian` | [php.net](https://www.php.net/) |
| [`php-deploy`](https://github.com/chinayin-docker/php-deploy) | `8.3.6` · `8.2.16` (`-cli`, `-cli-imagick`) · `7.4.33` (`-srv`, `-sso`) | `chinayin/debian` | [php.net](https://www.php.net/) |
| [`openjdk`](https://github.com/chinayin-docker/openjdk) | `8` · `11.0.31` · `17.0.19` · `21.0.11` | `eclipse-temurin` on `trixie-slim` | [adoptium.net](https://adoptium.net/) |
| [`maven`](https://github.com/chinayin-docker/maven) | `3.9.6-jdk-{8,11,17,21}` | `chinayin/openjdk` | [maven.apache.org](https://maven.apache.org/) |

`golang:<version>-ci` additionally ships **goreleaser `v2.18.3`** and **golangci-lint `v2.15.0`**
(`golangci-lint` is a static binary, so it works on both `amd64` and `arm64`).

### 3 · Web, search & middleware

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`nginx`](https://github.com/chinayin-docker/nginx) | `1.31.7` (`1.31`, mainline) · `1.30.6` (`1.30`, stable) — njs `1.0.2` | `chinayin/debian:trixie-slim` | [nginx.org](https://nginx.org/) |
| [`openresty`](https://github.com/chinayin-docker/openresty) | `1.31.1.1` (`1.31`) — `-bookworm-slim`, amd64 | `chinayin/debian:bookworm-slim` | [openresty.org](https://openresty.org/) |
| [`elasticsearch`](https://github.com/chinayin-docker/elasticsearch) | `9.5.5` (`9`) · `8.19.23` (`8`) · `7.17.28` (`7`) | `elasticsearch` (official) | [elastic.co](https://www.elastic.co/) |
| [`kibana`](https://github.com/chinayin-docker/kibana) | `9.5.5` (`9`) · `8.19.23` (`8`) · `7.17.28` (`7`) | `kibana` (official) | [elastic.co](https://www.elastic.co/) |
| [`canal`](https://github.com/chinayin-docker/canal) | `1.1.8` (`1.1.8`) | `canal` (official) | [github/canal](https://github.com/alibaba/canal) |

`elasticsearch` and `kibana` keep the same version tracks, so `chinayin/elasticsearch:<x>` pairs with
`chinayin/kibana:<x>`. The `kibana` `7` track is held at `7.17.28` to match `elasticsearch`, although upstream
already publishes a `kibana:7.17.29` image.

`elasticsearch` bundles the Chinese analyzers **ik**, **pinyin** and **stconvert**, installed from
`https://get.infini.cloud/elasticsearch/<plugin>/<version>` (the old `medcl/*` GitHub release assets no longer exist).

> `openresty` is pinned to `bookworm`: the official OpenResty apt repository has no `trixie` distribution yet.

### 4 · Security & networking

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`hfish`](https://github.com/chinayin-docker/hfish) | `3.3.4` (`3`) · `0.6.5` — server & client | `chinayin/debian` / `alpine` | [hfish.io](https://hfish.net/) |
| [`v2ray`](https://github.com/chinayin-docker/v2ray) | `5.x` — `bookworm` / `bullseye` flavor tags | `chinayin/debian` | [v2fly.org](https://www.v2fly.org/) |

> `hfish` scheduled builds are currently **paused** (GitHub disabled the workflow after a long idle period,
> and its pinned actions are still on the pre-Node-24 generation). Re-enable together with an action bump.

### 5 · Dev & ops tooling

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`kaniko-project`](https://github.com/chinayin-docker/kaniko-project) | `v1.23.2` (`-debug`, `-slim`) | `kaniko` (official) | [github/kaniko](https://github.com/GoogleContainerTools/kaniko) |
| [`kiro-cli`](https://github.com/chinayin-docker/kiro-cli) | `latest` (tracking upstream) | `chinayin/debian` / `alpine` | [kiro.dev](https://kiro.dev/) |
| [`playwright`](https://github.com/chinayin-docker/playwright) | `1.58.0` on `python-3.11` / `python-3.12` | `chinayin/python` | [playwright.dev](https://playwright.dev/) |
| [`apidoc`](https://github.com/chinayin-docker/apidoc) | `1.2.0` | `chinayin/node` | [apidocjs.com](https://apidocjs.com/) |

### 6 · Applications

| Image | Versions | Base | Upstream |
| --- | --- | --- | --- |
| [`xxl-job-admin`](https://github.com/chinayin-docker/xxl-job-admin) | `2.4.0` (`2.4`, `2`) | `chinayin/openjdk` | [xuxueli/xxl-job](https://github.com/xuxueli/xxl-job) |
| [`xxl-job-executor-springboot`](https://github.com/chinayin-docker/xxl-job-executor-springboot) | `2.4.0` (`2.4`, `2`) | `chinayin/openjdk` | [xuxueli/xxl-job](https://github.com/xuxueli/xxl-job) |
| [`BOCEPayJavaBridge`](https://github.com/chinayin-docker/BOCEPayJavaBridge) | built from source (`master`, tags `v*`) | `chinayin/openjdk` | internal |

---

## Building locally

Each image repository is self-contained. With [buildx](https://docs.docker.com/build/) installed:

```bash
git clone git@github.com:chinayin-docker/golang.git && cd golang

# pick a pinned version file, then bake
docker buildx bake -f docker-bake.hcl -f 1.27/env.hcl --print     # inspect targets/tags
docker buildx bake -f docker-bake.hcl -f 1.27/env.hcl             # build locally
docker buildx bake -f docker-bake.hcl -f 1.27/env.hcl --push      # push to your registry
```

Repositories that keep one directory per version instead of an `env.hcl` (`nginx`, `elasticsearch`, `openresty`)
are built the same way, passing the directory's own bake file or the `major` matrix value used by CI.

## Bumping a version

1. Update the pinned version (`env.hcl`, `major:` matrix in `ci.yml`, or the `ARG …VERSION` in the `Dockerfile`).
2. Add the new version directory if the image keeps one directory per release.
3. Run `actionlint .github/workflows/*.yml`, commit, push — CI builds and publishes immediately,
   then keeps rebuilding daily.
4. Update the row in the catalog above.

## Status snapshot

| State | Images |
| --- | --- |
| ✅ building & publishing | `debian`, `golang`, `node`, `python`, `openjdk`, `maven`, `nginx`, `openresty`, `elasticsearch`, `kibana`, `canal`, `v2ray`, `kaniko-project`, `kiro-cli`, `playwright`, `xxl-job-admin`, `xxl-job-executor-springboot`, `BOCEPayJavaBridge` |
| ⚠️ known upstream breakage | `php`, `php-deploy` (Debian 11/bullseye apt sources were archived → 404), `apidoc` (`deasync` needs a native build that fails on current Node) |
| ⏸ paused | `hfish` (scheduled workflow disabled by GitHub after inactivity) |

## Notes

* `kibana` and `elasticsearch` tracks are bumped together; keep them in lockstep.
* `golang` no longer publishes `1.25`: the minor reached end-of-life, its last release was `1.25.14`.
  Older `php` (`7.2`, `7.4`) and `kibana` `6.8` tags remain pullable but track EOL upstreams.
* Debian 11 (bullseye) packages have been archived upstream; anything still based on `bullseye` needs to move to
  `bookworm` (Debian 12) or `trixie` (Debian 13) before its next rebuild can succeed.
