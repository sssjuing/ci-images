# reviewdog CI 工具镜像
# 用于审查完成后把 ocr 产出的 SARIF findings 以行内评论推回 GitLab MR (gitlab-mr-discussion).
# GitLab 凭证运行时经环境变量注入 (REVIEWDOG_GITLAB_API_TOKEN 等), 禁止打入镜像.
#
# 二进制取法说明: 经 golang 构建阶段 go install 固定版本源码编译,
# 纯 Go 静态编译, 最终镜像为 alpine + git + ca-certificates + reviewdog 单二进制.
# 基础镜像版本须 >= reviewdog go.mod 的 toolchain 要求 (v0.21.3 为 go1.27.3):
# 用 GOTOOLCHAIN=local 禁止运行时下载更新 toolchain, 构建环境零外联依赖 dl.google.com.
FROM golang:1.27.3-alpine AS builder

# 固定版本, 避免上游升级引入不可控变更
ARG REVIEWDOG_VERSION=0.21.3
# 国内网络可取消注释改用国内源; 默认官方源
# ARG GOPROXY=https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOTOOLCHAIN=local GOPROXY=${GOPROXY} \
  go install github.com/reviewdog/reviewdog/cmd/reviewdog@v${REVIEWDOG_VERSION}

# 运行镜像: alpine 自带 git 版本 >= 2.45, 满足 reviewdog 的 git diff 需求.
# ca-certificates: 调 GitLab API 需要 TLS.
# safe.directory '*': 挂载的仓库属主与容器内用户不同, 跳过 git >= 2.35.2 的属主检查
# (与 ocr-ci 镜像同一处理, reviewdog 在 /src 跑 git diff 也受此检查影响).
FROM alpine:3.23.8
RUN apk add --no-cache git ca-certificates \
  && git --version \
  && git config --system --add safe.directory '*'
COPY --from=builder /go/bin/reviewdog /usr/local/bin/reviewdog
RUN reviewdog --version

# 工作目录即代码挂载点: 约定仓库挂载到 /src.
# reviewdog 以 cwd 为仓库根跑 git diff; SARIF findings 由调用方从 stdin 传入.
WORKDIR /src

ENTRYPOINT ["reviewdog"]
CMD ["--help"]
