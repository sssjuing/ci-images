# OCR CI 镜像

[OpenCodeReview](https://github.com/alibaba/open-code-review) (`ocr`) CI 工具镜像, 用于流水线和本地容器化执行 AI 代码审查.

## 文件说明

| 文件                 | 用途                                                                                   |
| -------------------- | -------------------------------------------------------------------------------------- |
| `Dockerfile`         | 镜像定义, 基于 `node:24.22-alpine3.23` (ocr 要求 git >= 2.41, alpine 自带版本满足要求) |
| `docker-compose.yml` | 指定镜像名, `docker compose build` 即可构建                                            |
| `.env.example`       | 复制为 `.env` 后维护 `OCR_VERSION`, 升级时只改这一行                                   |

## 构建与推送

```bash
docker compose build
docker compose push
```

## 使用方式

镜像 `ENTRYPOINT` 为 sh wrapper (先 `mkdir -p /src/.astro-cd` 再 `exec ocr`, 行为等同直接运行 `ocr`),
`WORKDIR` 为 `/src` (代码挂载点, 子命令默认对仓库根操作), 默认命令为 `--help`.
LLM 配置通过环境变量运行时注入, **禁止打入镜像**.

### 环境变量

| 环境变量          | 描述                                                 |
| ----------------- | ---------------------------------------------------- |
| OCR_LLM_URL       | LLM API 接口地址                                     |
| OCR_LLM_TOKEN     | LLM API 密钥                                         |
| OCR_LLM_MODEL     | LLM 模型名称                                         |
| OCR_USE_ANTHROPIC | `false`: OpenAI 兼容协议; `true`: Anthropic 兼容协议 |

前三者须同时设置 (OpenAI 兼容协议); astro-cd 平台侧由顶层 `ocr.llm_url/llm_token/llm_model/use_anthropic`
配置段注入, 本地手动运行时按下例经 `-e` 传入.

### 本地运行

```bash
# 审查当前分支相对 main 的改动, 结果写回当前目录
docker run --rm \
  -v "$(pwd):/src" \
  -e OCR_LLM_URL="https://api.deepseek.com/v1/chat/completions" \
  -e OCR_LLM_TOKEN="sk-..." \
  -e OCR_LLM_MODEL="deepseek-chat" \
  -e OCR_USE_ANTHROPIC="false" \
  registry.internal/your-team/ocr-ci:1.12.13 \
  review --from main --to HEAD --format sarif --output review.sarif

# 全量扫描指定目录 (审计陌生代码库, 无需 diff)
docker run --rm \
  -v "$(pwd):/src" \
  -e OCR_LLM_URL="https://api.deepseek.com/v1/chat/completions" \
  -e OCR_LLM_TOKEN="sk-..." \
  -e OCR_LLM_MODEL="deepseek-chat" \
  -e OCR_USE_ANTHROPIC="false" \
  registry.internal/your-team/ocr-ci:1.12.13 \
  scan --path internal --format sarif --output scan.sarif
```

### 连通性测试

首次使用或排查 401/403 错误时, 先验证配置:

```bash
docker run --rm \
  -v "$(pwd):/src" \
  -e OCR_LLM_URL="https://api.deepseek.com/v1/chat/completions" \
  -e OCR_LLM_TOKEN="sk-..." \
  -e OCR_LLM_MODEL="deepseek-chat" \
  -e OCR_USE_ANTHROPIC="false" \
  registry.internal/your-team/ocr-ci:1.12.13 \
  llm test
```

或把 `OCR_LLM_*` 填进 `.env` (从 `.env.example` 复制) 后用 compose, 无需重复传 `-e`:

```bash
docker compose up    # 跑完 docker compose down 清理
```
