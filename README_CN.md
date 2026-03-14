# auth2api

[English](./README.md)

一个轻量级代理，可将 Claude OAuth token 转换为 OpenAI 兼容 API，让你可以在任意 OpenAI 兼容客户端中使用 Claude 模型，或者直接配合 Claude Code 使用。

## 功能特性

- **OpenAI 兼容 API**：可直接替换 OpenAI 客户端接口，支持 `/v1/chat/completions`、`/v1/responses`、`/v1/models`
- **Claude 原生透传**：直接支持 `/v1/messages` 与 `/v1/messages/count_tokens`
- **流式输出支持**：同时支持 OpenAI 与 Claude 原生格式的 SSE 流
- **思考 / 推理支持**：将 `reasoning_effort` 映射到 Claude 的 extended thinking
- **工具调用支持**：完整支持 function calling，包括流式与非流式
- **图片支持**：将 `image_url` 内容转发到 Claude 视觉接口
- **单账号模式**：仅使用一个 Claude OAuth 账号，并带有 cooldown、自动刷新与健康状态追踪
- **自动 token 刷新**：在 OAuth token 过期前自动刷新
- **Claude Code 兼容**：同时接受 `Authorization: Bearer` 和 `x-api-key`
- **管理状态接口**：通过 `/admin/accounts` 查看账号健康状态
- **安全性**：使用 timing-safe API key 校验、每 IP 每分钟 60 次限流、仅允许 localhost 浏览器 CORS

## 运行要求

- Node.js 20+
- 一个 Claude 账号（推荐 Claude Max）

## 安装

```bash
git clone https://github.com/AmazingAng/auth2api
cd auth2api
npm install
npm run build
```

## 登录

### 自动模式（需要本地浏览器）

```bash
node dist/index.js --login
```

程序会输出一个浏览器 URL。完成授权后，回调会自动处理。

### 手动模式（适合远程服务器）

```bash
node dist/index.js --login --manual
```

在浏览器中打开输出的链接。授权完成后，浏览器会跳转到一个 `localhost` 地址，这个页面可能无法打开；请把地址栏中的完整 URL 复制回终端。

## 启动服务

```bash
node dist/index.js
```

默认监听地址为 `http://127.0.0.1:8317`。首次启动时，如果 `config.yaml` 中没有配置 API key，会自动生成并写入该文件。

如果上游因为限流导致当前账号进入 cooldown，auth2api 会返回 `429 Rate limited on the configured account`，而不是通用的 `503`。

## 配置

复制 `config.example.yaml` 为 `config.yaml`，然后按需修改：

```yaml
host: ""          # 绑定地址，空字符串表示 127.0.0.1
port: 8317

auth-dir: "~/.auth2api"   # OAuth token 存储目录

api-keys:
  - "your-api-key-here"   # 客户端使用这个 key 访问代理

cloaking:
  mode: "auto"            # auto | always | never
  strict-mode: false
  sensitive-words: []
  cache-user-id: false

debug: false
```

## 使用方法

将任意 OpenAI 兼容客户端指向 `http://127.0.0.1:8317`：

```bash
curl http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 1024
  }'
```

### 支持的模型

| 模型 ID | 说明 |
|--------|------|
| `claude-opus-4-6` | Claude Opus 4.6 |
| `claude-sonnet-4-6` | Claude Sonnet 4.6 |
| `claude-haiku-4-5-20251001` | Claude Haiku 4.5 |
| `claude-haiku-4-5` | Claude Haiku 4.5 的别名 |

auth2api 额外支持以下便捷别名：

- `opus` -> `claude-opus-4-6`
- `sonnet` -> `claude-sonnet-4-6`
- `haiku` -> `claude-haiku-4-5-20251001`

### 接口列表

| Endpoint | 说明 |
|----------|------|
| `POST /v1/chat/completions` | OpenAI 兼容聊天接口 |
| `POST /v1/responses` | OpenAI Responses API 兼容接口 |
| `POST /v1/messages` | Claude 原生消息透传 |
| `POST /v1/messages/count_tokens` | Claude token 计数 |
| `GET /v1/models` | 列出可用模型 |
| `GET /admin/accounts` | 查看账号健康状态（需要 API key） |
| `GET /health` | 健康检查 |

## Docker

```bash
# 构建
docker build -t auth2api .

# 运行（挂载配置文件与 token 目录）
docker run -d \
  -p 8317:8317 \
  -v ~/.auth2api:/data \
  -v ./config.yaml:/config/config.yaml \
  auth2api
```

或者使用 docker-compose：

```bash
docker-compose up -d
```

## 与 Claude Code 配合使用

将 `ANTHROPIC_BASE_URL` 指向 auth2api：

```bash
ANTHROPIC_BASE_URL=http://127.0.0.1:8317 \
ANTHROPIC_API_KEY=<your-api-key> \
claude
```

Claude Code 使用的是原生 `/v1/messages` 接口，auth2api 会直接透传。`Authorization: Bearer` 与 `x-api-key` 两种认证头都支持。

## 单账号模式

当前版本仅支持一个 Claude OAuth 账号。

- 再次执行 `--login` 时，如果还是同一个账号，会更新已保存的 token
- 如果本地已保存的是另一个账号，auth2api 会拒绝覆盖，并要求你先删除旧 token
- 如果 token 目录中存在多个 token 文件，auth2api 会直接报错并退出，直到你清理多余文件

## 管理状态

你可以通过 `/admin/accounts` 查看当前账号状态：

```bash
curl http://127.0.0.1:8317/admin/accounts \
  -H "Authorization: Bearer <your-api-key>"
```

返回内容包含账号是否可用、cooldown 截止时间、失败计数、最近刷新时间以及基础请求统计。

## Smoke 测试

仓库内置了一套最小自动化 smoke test，并使用 mocked upstream response，因此不会调用真实 Claude 服务：

```bash
npm run test:smoke
```

## 致谢

- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)
- [sub2api](https://github.com/Wei-Shaw/sub2api)

## License

MIT
