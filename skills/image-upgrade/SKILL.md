---
name: image-upgrade
description: Bump one of our published container images to a newer upstream release — check what upstream actually ships, verify the base distro and the registry tag both exist, edit the pinned version where that repo keeps it, validate, commit, then watch CI and verify the published tags, and finally update the catalog. Use this whenever someone says an image is behind or asks to升级镜像 / 升级版本 / bump <image> to the latest / "官方已经到 x.y.z 了" / upgrade nginx|openresty|elasticsearch|kibana|golang|node|python|php|openjdk|maven|debian or any other image in this organization, and also when a scheduled image build has been failing and the version turns out to be the reason. Do not use it for application repos that merely consume these images.
---

# Image Upgrade

Every image in this organization is one repository with a Dockerfile, a pinned upstream version, and a CI
workflow that rebuilds on push plus a daily schedule. "Upgrading an image" is therefore a **version-pin change
plus a verification loop**, not a rewrite. The whole risk lives in the facts you check before editing:
whether upstream really ships that version, whether the registry really has that tag, and whether the base
distro can still install its packages. Do those checks first and the edit itself is trivial.

## Ground rules

- **Never move a version based on memory or a blog post.** Query the upstream release list and the registry
  tag list. Model knowledge is stale by months and a "released" version frequently has no image yet.
- **Verify the artifact the Dockerfile actually pulls**, not just the product version: the registry tag for
  `FROM`, the apt package version string, the module/plugin URL, the tarball checksum.
- **One image per commit.** Each image is its own repository, so a failure is isolated and revertable.
- **Only stage what you changed.** `git add -- .github/workflows` / scoped paths, and
  `git commit -m "…" -- <paths>` (note: `-m` goes *before* `--`, everything after `--` is a pathspec).
  Several of these repos carry unrelated uncommitted work; a bare `git commit` would sweep it in.
- **`git fetch` before you trusting `origin/master`.** Local remote-tracking refs are often months stale;
  you will happily "upgrade" a tree that is behind and then push a conflict, or worse, silently drop
  someone else's commits. If fetch says you are behind, `git merge --ff-only origin/master`, re-apply the
  version edits on the new tree, and only then commit.

## The two repository shapes

Know which shape you are in before editing — it decides where the version lives.

| Shape | Where the version is pinned | Where the build list is | Examples |
| --- | --- | --- | --- |
| **variant dirs + bake** | `<minor>/env.hcl` (`version` / `minor`) and `docker-bake.hcl` | `env.hcl` `group` targets | golang, maven, openjdk, node, python, php |
| **one dir per major** | `<major>/Dockerfile` (`ARG …VERSION`) | `ci.yml` `strategy.matrix.major` | nginx, openresty, elasticsearch, kibana, canal, xxl-job-* |

Adding a new major usually means creating a directory (borrow the closest existing one, then change only the
version) **and** extending the matrix. Removing one means deleting the directory and shrinking the matrix —
and, for `env.hcl` repos, deleting the `env.hcl` plus its bake targets.

## Workflow

### 1. Find what upstream actually publishes

Check two things that are easy to confuse: the **project release**, and the **container tag**.

```bash
gh api "repos/<upstream>/releases?per_page=10" -q '.[] | "\(.tag_name) \(.published_at[:10])"'
gh api "repos/<upstream>/tags?per_page=15" -q '.[].name'
```

Then confirm the exact image tag you intend to put in `FROM` exists — a release without an image means you
have to pin the previous patch (this has happened: a 7.17.29 release existed while the Docker Hub tag stopped
at 7.17.28).

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://hub.docker.com/v2/repositories/library/<image>/tags/<tag>"
```

For apt-based images, check the **distribution** too: whether the vendor repo even has the distro you plan to
build on. Our own Debian base publishes `trixie`, `bookworm`, `bullseye`, `buster` (each `-slim` as well).

```bash
curl -s "https://<vendor>/dists/" | grep -oE 'href="[^"]+"'            # which suites exist
curl -s "…/dists/<suite>/…/binary-amd64/Packages" | grep -A2 '^Package: <name>$'
```

### 2. Check the base image can still install itself

Debian point releases get archived. A version bump that keeps `bullseye` will fail in `apt-get` with
`404 Not Found` from `deb.debian.org/debian-security`, and the failure has nothing to do with the version you
are bumping. When you touch such a repo, move the base forward (`bookworm` → Debian 12, `trixie` → Debian 13)
in the same commit, or state clearly that the repo stays broken.

### 3. Edit the pin

- `env.hcl` repos: `version`, and `minor` when the minor changes; `docker-bake.hcl` carries the same values
  for the shared targets.
- matrix repos: the `ARG …VERSION` / `ENV …_VERSION` line, the directory name, and the `matrix.major` list.
- Keep `COPY --from=<vendor image>:<version>…` lines in step with the pin — they are easy to miss because they
  are not the main `FROM`.
- Grep for the old version afterwards; leftover copies are the most common self-inflicted bug:
  `grep -rn "<old-version>" . --exclude-dir=.git`

### 4. Validate before pushing

```bash
actionlint .github/workflows/*.yml          # YAML / expression errors
docker buildx bake -f docker-bake.hcl -f <minor>/env.hcl --print   # resolves tags, context, args
```

`actionlint` output that is only `shellcheck reported issue … SC2086` inside existing `Set variables` steps is
pre-existing noise; anything else is yours. Building locally is optional — it needs registry access and the
CI runner has it, so a small image is often faster to validate by pushing.

### 5. Commit, push, watch

Commit message states **what moved and why** (`feat: nginx 1.30.6 (stable) + 1.31.7 (mainline) on trixie-slim`),
then push and follow the run rather than walking away:

```bash
gh run list --repo <owner>/<repo> --workflow=ci.yml --limit 1 \
  --json databaseId,status,conclusion,headSha
gh run view <run-id> --repo <owner>/<repo> --json jobs -q '.jobs[] | "\(.name) -> \(.conclusion)"'
gh run view <run-id> --repo <owner>/<repo> --log-failed | tail -40
```

A multi-matrix workflow fails fast: one red job cancels the rest, so read `--json jobs` and fix everything you
can see at once instead of one matrix cell per push.

### 6. Verify what got published

CI green only means the build ran; confirm the tags landed in the registry. Full-version, rolling-minor and
`-variant` tags should all be there, and the digest of a fresh tag should not equal the previous release's.

```bash
curl -s "https://hub.docker.com/v2/repositories/<namespace>/<image>/tags?page_size=100&ordering=last_updated" \
  | python3 -c "import json,sys; print([t['name'] for t in json.load(sys.stdin)['results']][:12])"
```

### 7. Update the catalog

The organization keeps one catalog — `CATALOG.md` in the `.github` repository — which is rendered three ways:
the GitHub org landing page (`profile/README.md`), the file itself, and the Docker Hub landing page (synced by
`.github/workflows/dockerhub.yml`). After any version change, update the version cell in **both**
`CATALOG.md` and `profile/README.md`, plus the status snapshot if an image moved between
working / broken / paused. Pushing `.github` re-syncs the Hub page automatically; verify by re-reading the
Hub description rather than trusting the green check.

## Failure playbook

Read [references/playbook.md](references/playbook.md) when a build goes red, when a repo has no working CI at
all, or when you need the exact commands for registry checks, workflow re-enabling, or catalogs of the
recurring upstream breakages (apt archives, dead plugin URLs, Node-runtime action generations, paired-version
images). It is a symptom → cause → fix catalog assembled from real failures; skim it before guessing.

## Reporting

Close the loop with facts, not reassurance: the commit sha per repo, the CI job conclusions, the tags that
actually appeared in the registry, and anything you deliberately left behind (deferred repos, gates waiting on
a decision). If a failure was pre-existing rather than caused by the bump, say so and show the evidence —
otherwise the next person will assume the upgrade broke it.
