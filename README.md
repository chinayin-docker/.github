# .github

组织级仓库，承载目录、组织首页与可复用的 skill：

| 路径 | 作用 |
| --- | --- |
| [`profile/README.md`](profile/README.md) | **组织首页**，渲染在 <https://github.com/chinayin-docker> |
| [`CATALOG.md`](CATALOG.md) | 全部镜像的完整目录（版本、底包、上游、tag 规则、构建方式、状态） |
| [`.github/FUNDING.yml`](.github/FUNDING.yml) | 赞助配置（组织级生效） |
| [`.github/workflows/dockerhub.yml`](.github/workflows/dockerhub.yml) | 把 `CATALOG.md` 同步成 Docker Hub 上 `chinayin/images` 落地页的描述 |
| [`skills/image-upgrade/`](skills/image-upgrade/SKILL.md) | **升级镜像的 skill**：查上游版本 → 校验 tag/底包 → 改 pin → 校验 → 提交 → 看 CI → 核对发布 tag → 更新目录，附踩坑手册 |

以前目录放在独立的 `images` 仓库里，已合并到这里：目录只有一个事实源，首页是它的精简版。
`archive/images` 分支保留了 `images` 仓库合并前的完整提交历史。

- 组织首页：<https://github.com/chinayin-docker>
- Docker Hub 落地页：<https://hub.docker.com/r/chinayin/images>
- Docker Hub 镜像：<https://hub.docker.com/u/chinayin>
