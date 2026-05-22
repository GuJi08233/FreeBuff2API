# GitHub Actions: Docker 镜像自动发布到 GHCR

> 工作流路径: `.github/workflows/docker-publish.yml`
> 引入版本: v0.11.x（2026-05-22）
> Registry: `ghcr.io/<owner>/<repo>`（自动来自 `${{ github.repository }}`，大小写由 metadata-action 转小写）
>
> **设计原则**：工作流不硬编码仓库名，所有 owner/repo 信息从 GitHub Actions 上下文动态取得 →
> fork、改名、转移所有权后无需改一行 yaml 即可继续构建到对应 GHCR namespace。
> 当前仓库 `GuJi08233/FreeBuff2API` 的实际镜像地址为 `ghcr.io/guji08233/freebuff2api`，
> 下文示例命令均按此展开，fork 者请自行替换为自己的 `owner/repo`（lowercase）。

---

## 1. 触发矩阵

| 事件 | 是否构建 | 是否推送 | 生成的 tag |
|---|---|---|---|
| `push` 到 `master` | ✅ | ✅ | `master`, `latest`, `sha-xxxxxxx` |
| `push` tag `v1.2.3` | ✅ | ✅ | `1.2.3`, `1.2`, `1`, `latest`, `sha-xxxxxxx` |
| `pull_request` → `master` | ✅ | ❌（仅验证 Dockerfile） | `pr-N`（不入 registry） |
| 手动 `workflow_dispatch` | ✅ | ✅ | 跟随当前分支/tag 规则 |

**latest 行为**：只在默认分支（`master`）或推 semver tag 时打出，避免被任意分支覆盖。

---

## 2. 多架构构建

```yaml
platforms: linux/amd64,linux/arm64
```

- 项目零 CGO，纯静态二进制 → arm64 0 改动即可构建
- 适配场景：树莓派、Apple Silicon Docker Desktop、AWS Graviton、Oracle Ampere
- 用 QEMU 跨架构编译，首次 ~3-5 min，命中 GHA cache 后 ~1 min

---

## 3. 所需权限（已在 workflow 中声明）

```yaml
permissions:
  contents: read           # checkout
  packages: write          # 推送到 ghcr
  id-token: write          # OIDC 用于 attestation
  attestations: write      # build provenance
```

`GITHUB_TOKEN` 由 GHA 自动注入，**无需**额外配置 secret。

---

## 4. 安全 / 供应链

- **SBOM**: `sbom: true` → 镜像随附 SPDX 软件清单
- **Provenance**: `provenance: mode=max` → SLSA Level 3 构建溯源
- **Attestation**: `actions/attest-build-provenance@v2` 把 attestation 推到 registry，可被 `cosign verify-attestation` 校验

校验示例（当前仓库）：
```bash
gh attestation verify oci://ghcr.io/guji08233/freebuff2api:latest -R GuJi08233/FreeBuff2API
```

通用形式：
```bash
gh attestation verify oci://ghcr.io/<owner>/<repo>:<tag> -R <owner>/<repo>
```

---

## 5. 缓存策略

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

走 GitHub Actions 自带缓存（10 GB / repo），无需额外服务。`mode=max` 缓存所有中间层（包括 builder stage 的 `go mod download`）。

---

## 6. 首次发布前需要做的（一次性手工操作）

### 6.1 启用 GHCR 写入
仓库默认就允许 `GITHUB_TOKEN` 写 packages，**通常无需改设置**。如果遇到 403：
- 进入 `Settings → Actions → General → Workflow permissions`
- 勾选 **Read and write permissions** + **Allow GitHub Actions to create and approve pull requests**

### 6.2 把 Package 设为公开（推送后才会出现）
首次推送后会在 GitHub 的 Packages 页出现 container，路径形如：
- 个人账户：`https://github.com/users/<owner>/packages/container/<repo-lowercase>`
- 组织账户：`https://github.com/orgs/<org>/packages/container/<repo-lowercase>`

当前仓库示例：`https://github.com/users/GuJi08233/packages/container/freebuff2api`

操作：
1. 进入 package 详情页 → 右下 `Package settings`
2. 滚到底部 `Danger Zone → Change visibility → Public`
3. 之后任何人都可 `docker pull ghcr.io/<owner>/<repo-lowercase>:latest`

> ⚠️ 默认 visibility 跟随仓库（public repo → public package），但 GHCR 偶有第一次推送被建为 private 的情况，需手工切换。

### 6.3 把 Package 关联到本仓库（可选但推荐）
package 设置页 → `Manage Actions access` → 把当前仓库加入 → 这样 Actions 后续推送不需要 PAT。

---

## 7. 发布流程

### 日常发布
```bash
# 直接推 master，自动构建 latest + sha-xxx
git push origin master
```

### 版本发布
```bash
git tag -a v0.12.0 -m "release: v0.12.0"
git push origin v0.12.0
# → 生成 0.12.0, 0.12, 0, latest, sha-xxx
```

### 拉镜像（以当前仓库为例，fork 者请替换 owner/repo）
```bash
docker pull ghcr.io/guji08233/freebuff2api:latest
docker pull ghcr.io/guji08233/freebuff2api:0.12.0
docker pull ghcr.io/guji08233/freebuff2api:sha-a1b2c3d
```

> 通用形式：`ghcr.io/<owner-lowercase>/<repo-lowercase>:<tag>`

### 在 docker-compose.yml 中切换为远程镜像（生产部署可选）
```yaml
services:
  freebuff2api:
    # 通用：ghcr.io/<owner>/<repo>:<tag>
    # 当前仓库：
    image: ghcr.io/guji08233/freebuff2api:latest
    # build: .   # 注释掉，改用远程镜像
    ...
```

---

## 8. 常见问题

### Q1: PR 工作流为什么不推送？
PR 来自 fork 时 `GITHUB_TOKEN` 没有写权限，强行 push 会失败；统一不推，仅验证 Dockerfile 可构建。

### Q2: 大写仓库名 `GuJi08233/FreeBuff2API` 会报错吗？
不会。`docker/metadata-action@v5` 会自动把 image name 转小写：
```
${{ github.repository }} = "GuJi08233/FreeBuff2API"
→ ghcr.io/guji08233/freebuff2api  ✅
```

### Q3: 我 fork 后镜像会推到哪？
推到**你自己的** GHCR namespace。`${{ github.repository }}` 是运行时变量，自动展开为运行
该 workflow 的仓库 `owner/repo`，不需要改 yaml。例：
- fork 到 `alice/FreeBuff2API` → `ghcr.io/alice/freebuff2api`
- 改名为 `bob/myproxy` → `ghcr.io/bob/myproxy`

### Q4: GHA 缓存满了怎么办？
GitHub 自动按 LRU 驱逐，10 GB 上限。需要手动清理：
- `gh cache list -R <owner>/<repo>`
- `gh cache delete <id> -R <owner>/<repo>`

### Q5: 想加 cosign keyless signing？
当前 `attest-build-provenance` 已包含 keyless OIDC 签名（Sigstore Fulcio），与 cosign 签名功能等价；如需 cosign 风格签名再单独加 step 即可。

---

## 9. 关键设计决策

- **不用 Docker Hub**：GHCR 与 repo 同源，权限/可见性继承 repo，零额外 secret 维护
- **不发 PR 镜像到 ghcr**：避免被恶意 fork 用 PR 推垃圾镜像消耗存储
- **SBOM + Provenance 默认开启**：边际成本几乎为 0，提供供应链可追溯性（符合 CLAUDE.md "可追溯" 使命）
- **multi-arch 默认 amd64 + arm64**：覆盖 99% 用户场景；如未来需要 armv7/ppc64le 再扩
