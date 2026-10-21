# Image Upgrade — Failure Playbook

Symptom → cause → fix, from failures we actually hit on these images. Read the symptom table first; then the
section it points at.

## Quick index

| Symptom in the CI log | Section |
| --- | --- |
| `404 Not Found [IP: …]` while installing apt packages, `E: Failed to fetch http://deb.debian.org/debian-security/…` | [1. Archived distro](#1-archived-distro-apt-404) |
| `failed to solve … elasticsearch-plugin install` / plugin URL 404 | [2. Dead plugin / artifact URLs](#2-dead-plugin--artifact-urls) |
| `No supported distribution was found for input adopt` | [3. Action major bumps that are not drop-in](#3-action-major-bumps-that-are-not-drop-in) |
| `Unable to process file command 'output' successfully` / `Invalid format` | [4. Workflow steps that write multi-line outputs](#4-workflow-steps-that-write-multi-line-outputs) |
| `/etc/security/limits.conf: No such file or directory`, `… /etc/sysctl.conf` | [5. Slimmer upstream images](#5-slimmer-upstream-images-missing-files) |
| `No such file or directory` starting `/usr/bin/bash` with a `working-directory` | [6. Matrix that no longer matches the tree](#6-matrix-that-no-longer-matches-the-tree) |
| `npm error gyp ERR! … not ok` during image build | [7. Native modules in language images](#7-native-modules-in-language-images) |
| Job never runs; workflow shows `disabled_inactivity` | [8. Auto-disabled workflows](#8-auto-disabled-workflows) |
| Workflow is fine but nothing was published | [9. Registry verification](#9-registry-verification) |
| `git push` rejected / tree looks older than remote | [10. Stale remote-tracking refs](#10-stale-remote-tracking-refs) |

## 1. Archived distro (apt 404)

**Symptom.** Any `apt-get install` in the Dockerfile fails with several `404 Not Found` for
`deb.debian.org/debian-security/pool/updates/...` packages, or `E: Failed to fetch …/ca-certificates_…deb11u1`.

**Cause.** Debian point-release security pools get rotated and eventually archived to `archive.debian.org`.
Images still building on `bullseye` (Debian 11) started failing this way long after the version pin was fine;
the version bump is usually innocent, the base image is the problem.

**Fix.** Move the base forward in the same commit:

| Base | Debian | State |
| --- | --- | --- |
| `bullseye` / `-slim` | 11 | archived — new builds fail |
| `bookworm` / `-slim` | 12 | works, older packages |
| `trixie` / `-slim` | 13 | current, preferred when the vendor supports it |

Then check the vendor side: an image whose upstream apt repo has no `trixie` suite **must stay on `bookworm`**
(this is why the OpenResty image is bookworm-only while nginx sits on trixie).

## 2. Dead plugin / artifact URLs

**Symptom.** A build that used to work now dies downloading a plugin, and the URL is a GitHub release asset:
`…/releases/download/v<version>/<artifact>-<version>.zip` → 404, even though the release page looks alive.

**Cause.** Upstream repos get renamed/absorbed, GitHub redirects them, and the new home publishes a single
`Latest` release with no per-version assets — every historical download URL dies at once.

**Fix.** Find the vendor's current distribution endpoint (the plugin's own README usually shows a
`bin/<tool>-plugin install <url>` example) and install from there. Example of the shape:

```dockerfile
ARG PLUGIN_VERSION=<release>
RUN ./elasticsearch-plugin install -b "https://<vendor-host>/elasticsearch/<plugin>/${PLUGIN_VERSION}"
```

Verify the endpoint with `curl -sIL …` and check `Content-Type: application/zip` before trusting it — a `200`
HTML error page installs nothing. Keep one `ARG` for the version so all plugins move together.

## 3. Action major bumps that are not drop-in

Bumping `uses:` to the current major is usually safe: since the 2024–2026 wave the changes are mostly the
Node 20 → Node 24 runtime and an ESM rewrite, and GitHub-hosted runners need no extra setup. Two exceptions
worth reading release notes for:

- Removed **inputs**: `docker/setup-buildx-action` v4 dropped deprecated inputs/outputs; `docker/bake-action`
  v7 merged `workdir` into `source` and dropped the `list-targets` subaction. Grep the workflows for those
  keys before bumping (`grep -rn "workdir:" .github/workflows`).
- Removed **input values**: `actions/setup-java` v5/v6 no longer accepts the `adopt` distribution alias →
  `##[error]No supported distribution was found for input adopt`. Use `temurin` (the AdoptOpenJDK successor).

A workflow pinned several majors behind may still be on Node 12-era actions that GitHub no longer runs; those
jobs cannot go green until the actions move, so an old pin is a reason to bump, not a reason to leave it.

## 4. Workflow steps that write multi-line outputs

**Symptom.** `Set variables` fails with `##[error]Unable to process file command 'output' successfully` /
`Invalid format '<some Dockerfile line>'`.

**Cause.** The step writes `echo "version=$(grep … Dockerfile | sed …)" >> $GITHUB_OUTPUT`, the `grep` matched
more than one line, so the value spans lines — `$GITHUB_OUTPUT` is line-based and rejects that.

**Fix.** Anchor the pattern to the single intended line and keep the capture to one field:

```bash
grep -E '^ARG RESTY_DEB_VERSION=' Dockerfile | sed -E 's/.*="=([0-9.]+)-.*/\1/'
```

Run the same command locally against the Dockerfile before pushing; a one-line result is the whole test.
The related trap: a `grep 'RESTY_VERSION '` that matches nothing still "works" and silently publishes an
empty version in the tag — verify the tag list after the first build of any repo whose version is grepped.

## 5. Slimmer upstream images (missing files)

**Symptom.** `printf … >> /etc/security/limits.conf` → `/bin/sh: line 1: /etc/security/limits.conf: No such
file or directory`; same for `/etc/sysctl.conf`.

**Cause.** Newer official images (`elasticsearch`, `kibana`, …) no longer ship those files. Older tags had
them, so the Dockerfile was written with a bare `>>`.

**Fix.** Create the path first, in the same `RUN`:

```dockerfile
&& mkdir -p /etc/security \
&& printf "* soft nofile 65536\n…" >> /etc/security/limits.conf \
&& touch /etc/sysctl.conf \
&& printf "vm.max_map_count=262144\n…" >> /etc/sysctl.conf \
```

Do it for **every** track in the repo at once — the same Dockerfile body is copied per major, and fixing only
the newest leaves the others to fail on their next rebuild.

## 6. Matrix that no longer matches the tree

**Symptom.** A job dies in seconds: `An error occurred trying to start process '/usr/bin/bash' with working
directory '…/<version>/<flavor>'. No such file or directory`.

**Cause.** `ci.yml` builds `matrix.major` × `matrix.os` as a path, and the directory was renamed or moved
(the tree says `1.25/`, the matrix still says `1.21/`).

**Fix.** Read the actual directories (`ls`) and align the matrix in the same commit as the version bump —
a `working-directory` that does not exist fails before Docker is even involved.

## 7. Native modules in language images

**Symptom.** `npm error gyp ERR! … node-gyp -v … not ok`, `gyp ERR! cwd …/node_modules/deasync`.

**Cause.** A globally installed npm package contains a native addon that no longer compiles against the Node
version in the base image. Nothing to do with the image version pin.

**Fix.** Treat it as a separate task: pin/fork the offending package, drop it, or move the base Node. Report it
as pre-existing breakage instead of folding it into the version bump — the two failure modes look alike in CI
and conflating them makes the next upgrade harder to read.

## 8. Auto-disabled workflows

Scheduled workflows in public repos get disabled after ~60 days without repository activity, and a push does
**not** bring them back. Check and restore:

```bash
gh api repos/<owner>/<repo>/actions/workflows -q '.workflows[] | "\(.id) \(.name) \(.state)"'
gh api -X PUT repos/<owner>/<repo>/actions/workflows/<id>/enable
```

Enabling a workflow whose actions are pinned to an unsupported runtime only produces a daily red run — bring
the actions up to date first, or leave it disabled and say why.

## 9. Registry verification

`gh run view` green means the build and push completed. Confirm the artifacts:

```bash
# exact tag exists?
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://hub.docker.com/v2/repositories/<namespace>/<image>/tags/<tag>"

# tag list, newest first
curl -s "https://hub.docker.com/v2/repositories/<namespace>/<image>/tags?page_size=100&ordering=last_updated" \
  | python3 -c "import json,sys; print([t['name'] for t in json.load(sys.stdin)['results']][:12])"

# multi-arch?
curl -s "https://hub.docker.com/v2/repositories/library/<image>/tags/<tag>" \
  | python3 -c "import json,sys; print(sorted({i.get('architecture') for i in json.load(sys.stdin)['images']}))"
```

If the registry API is unreachable from your machine, that is a network issue, not a build issue — say so and
either route through a proxy or verify from the CI log, which prints every pushed tag and manifest digest.

## 10. Stale remote-tracking refs

**Symptom.** `git status -sb` claims you are in sync, but `git fetch` then reports `[behind 2]`, or a push is
rejected.

**Fix.** `git fetch` first, always. When behind: stash or re-apply the version edits on top of
`git merge --ff-only origin/master` rather than merging local work — a botched rebase on a repo whose upstream
also touched the workflows will lose the other side's changes.

## Paired versions

Some images must move together or the pair is unusable:

| Pair | Rule |
| --- | --- |
| `elasticsearch` ↔ `kibana` | same version tracks; Kibana must match the Elasticsearch minor a user runs |
| `maven` ↔ `openjdk` | maven images copy the distribution from a vendor maven image built on a specific JDK |
| `node` / `python` / `php` → `playwright`, `apidoc` | the tool image is built on the runtime image, so a runtime bump can invalidate it |
| our images → `debian` base | a base bump reaches every apt-based image on its next rebuild |

When one side cannot move (a registry tag missing for the newest patch), pin both to the highest version that
exists on **both** sides and record the reason in the commit message and in `CATALOG.md`.
