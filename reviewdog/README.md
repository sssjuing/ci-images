# reviewdog CI 镜像

[reviewdog](https://github.com/reviewdog/reviewdog) CI 工具镜像, 用于审查完成后把 ocr 产出的 SARIF findings 以行内评论 (MR discussions) 推回 GitLab.

reviewdog 官方不发镜像 (其目标场景是 GitHub Actions/Jenkins runner 直接跑 CLI); 为便于容器化执行, 自建此镜像.

## 文件说明

| 文件                 | 用途                                                                                   |
| -------------------- | -------------------------------------------------------------------------------------- |
| `Dockerfile`         | 镜像定义, golang 多阶段构建 + 运行层 alpine + git (reviewdog 跑 git diff 锚定行号需要) |
| `docker-compose.yml` | 指定镜像名, `docker compose build` 即可构建                                            |
| `.env.example`       | 复制为 `.env` 后维护 `REVIEWDOG_VERSION` / `REGISTRY`, 升级只改版本行                  |

二进制取法: 经 `go install github.com/reviewdog/reviewdog/cmd/reviewdog@v<版本>` 源码编译 (纯 Go, 静态链接).

## 构建与推送

在有 docker 的机器上执行 (构建拉取 golang 基础镜像走 daemon 配置的 registry mirror):

```bash
docker compose build
docker compose push
```

## 使用方式

镜像 `ENTRYPOINT` 为 `sh -c` (reviewdog 只从 stdin 读输入, 文件位置参数被其静默忽略),
`WORKDIR` 为 `/src` (代码挂载点), 默认命令 `reviewdog --help`.
审查完成后, 在仓库根目录为 `/src` 的容器内执行:

```bash
reviewdog -f=sarif -name=OpenCodeReview -reporter=gitlab-mr-discussion -filter-mode=diff_context \
-fail-level=none < /src/ocr-review.sarif
```

注意 reviewdog 只从 stdin 读取 findings: SARIF 文件必须用 shell 重定向接入 stdin,
把路径作为命令行参数传入会被静默忽略 (reviewdog 不处理位置参数, 表现为空输入、
不发评论且退出码为 0). 若不经 shell 直接 exec 进程, 需自行把 SARIF 文件接到容器 stdin.

GitLab 凭证与 MR 定位通过环境变量运行时注入, **禁止打入镜像** (与 ocr 镜像的 LLM 密钥同一原则).

### 环境变量

reviewdog 不读 `CI_PROJECT_ID` (那是 GitLab CI 原生注入的数字 id, reviewdog 用不到),
项目定位用 owner/repo 路径形式, GitLab API 接受 URL-encoded 路径作为 `:id`.

| 环境变量                         | 说明                                                        |
| -------------------------------- | ----------------------------------------------------------- |
| `REVIEWDOG_GITLAB_API_TOKEN`     | GitLab access token (需 `api` scope)                        |
| `CI_API_V4_URL`                  | GitLab API 地址, 如 `https://git.example.com/api/v4`        |
| `CI_REPO_OWNER`                  | 项目所在 namespace (嵌套子组时为 `group/subgroup` 完整路径) |
| `CI_REPO_NAME`                   | 项目名                                                      |
| `CI_MERGE_REQUEST_IID`           | Merge Request IID                                           |
| `CI_COMMIT_SHA`                  | MR head commit SHA                                          |
| `CI_MERGE_REQUEST_DIFF_BASE_SHA` | MR diff base commit SHA                                     |

### 本地运行

```bash
# 在已 checkout 的 MR 仓库目录里把 SARIF findings 推回 MR (验证链路用)
# -i 必须保留: 不 attach stdin 时容器内 reviewdog 读到的是 EOF, 不会发任何评论
docker run --rm -i \
  -v "$(pwd):/src" \
  -e REVIEWDOG_GITLAB_API_TOKEN="glpat-..." \
  -e CI_API_V4_URL="https://git.example.com/api/v4" \
  -e CI_REPO_OWNER="group" \
  -e CI_REPO_NAME="repo" \
  -e CI_MERGE_REQUEST_IID="14" \
  -e CI_COMMIT_SHA="$(git rev-parse HEAD)" \
  -e CI_MERGE_REQUEST_DIFF_BASE_SHA="$(git merge-base origin/main HEAD)" \
  registry.example.com/your-team/reviewdog-ci:0.21.3 \
  -f=sarif -reporter=gitlab-mr-discussion -filter-mode=diff_context -fail-level=none \
  < "$(pwd)/ocr-review.sarif"
```

reviewdog 会在 `/src` 内执行 `git diff <base_sha> <head_sha>`, 所以仓库必须同时包含
`CI_COMMIT_SHA` 与 `CI_MERGE_REQUEST_DIFF_BASE_SHA` 两个 commit 对象; 浅克隆只取 head 时会报
`bad object`, 需先 `git fetch origin <base_sha>`.
