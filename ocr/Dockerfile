# OpenCodeReview (ocr) CI 工具镜像
# 用于 CI 流水线中执行 AI 代码审查, LLM 密钥运行时通过环境变量注入, 禁止打入镜像
# ocr 要求 git 版本 >= 2.41, alpine 的 git 包版本 >= 2.45, apk add 就可满足, 且镜像体积小
FROM node:24.22-alpine3.23

# ca-certificates: 调用 LLM API 需要 TLS
# safe.directory '*': 挂载的仓库属主与容器内用户不同, 跳过 git >= 2.35.2 的属主检查
RUN apk add --no-cache git ca-certificates \
  && git --version \
  && git config --system --add safe.directory '*'

# 固定版本, 避免上游升级引入不可控变更
ARG OCR_VERSION=1.12.13
RUN npm install -g @alibaba-group/open-code-review@${OCR_VERSION} --registry=https://registry.npmmirror.com/ \
  && npm cache clean --force \
  && ocr --version

# # 非 root 用户运行
# RUN addgroup -S ocr && adduser -S ocr -G ocr
# USER ocr

# 工作目录即代码挂载点, 平台与本地统一约定: 代码在 /src, ocr 默认对 cwd(仓库根)操作
WORKDIR /src

# sh wrapper: 先尽量确保 /src/.astro-cd 存在再 exec ocr.
# ocr 的 resolveOutputWriter 只 Stat 不 MkdirAll, 父目录缺失直接报
# "--output directory does not exist"; 而平台扫描 workspace 全新时没有
# .astro-cd 目录. 路径须与 internal/pkg/ci/codescan/ocr.go 的 --output 保持同步.
# mkdir 用 ";" 而非 "&&" 且忽略失败: /src 只读挂载等场景下不挡
# --help/llm test 等与输出目录无关的命令, 真写不出时由 ocr 自身报错.
# 末位 "ocr" 是 $0, CMD/Args 经 "$@" 透传, exec 后 ocr 即 PID 1, 信号与退出码不变.
ENTRYPOINT ["/bin/sh", "-c", "mkdir -p /src/.astro-cd 2>/dev/null; exec ocr \"$@\"", "ocr"]
CMD ["--help"]
