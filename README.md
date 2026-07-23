# OpenViking 快速部署（Docker）

本项目使用 Docker Compose 一键拉起 OpenViking 及其向量嵌入模型（Ollama），无需在宿主机安装任何依赖。

## 前置要求

- 已安装 Docker 和 Docker Compose
- 可访问 [ghcr.io](https://ghcr.io)（拉取 OpenViking 镜像）和 [Docker Hub](https://hub.docker.com)（拉取 Ollama 镜像）
- 拥有一个 VLM API Key（默认配置使用 DeepSeek，可按需替换为其他 OpenAI 兼容服务）

> **推荐部署路径**：本项目建议放在用户主目录下，例如 `~/.openviking`。该路径与容器内挂载的工作区（`/app/.openviking`）对应，OpenViking CLI 及相关工具默认会读取此位置，可免去额外的环境变量配置。

## 部署步骤

### 1. 配置 `ov.conf`

从示例配置复制一份，并填入你自己的 VLM API Key 和 root API Key：

```bash
cp ov.example.conf ov.conf
```

编辑 `ov.conf`，重点修改以下字段：

```json
{
  "server": {
    "root_api_key": "<你的 root_api_key>"   // 管理员密钥，请改成强随机串
  },
  "vlm": {
    "api_key": "<你的 VLM api_key>",         // 例如 DeepSeek: sk-xxxxxxxxxxxx
    "model": "deepseek-v4-flash"             // 按所选服务支持的模型填写
  }
}
```

> 嵌入模型默认使用容器内的 Ollama（`nomic-embed-text`），无需额外配置。

### 2. 启动本地容器

```bash
docker compose up -d
```

启动后 OpenViking 会监听宿主机的 `1933` 端口。

### 3. 拉取向量嵌入模型

在 Ollama 容器中拉取嵌入模型（首次部署执行一次即可）：

```bash
docker exec ov-ollama ollama pull nomic-embed-text
```

### 4. 检查可用性

等待服务健康后，访问健康检查接口：

```bash
curl http://localhost:1933/health
```

返回正常即说明 OpenViking 已就绪。

### 5. 创建用户

使用 root API Key 创建一个管理员用户，并为其签发访问密钥：

```bash
ROOT=<你的 root_api_key>

# 创建管理员用户
curl -s -XPOST localhost:1933/api/v1/admin/accounts/default/users \
  -H "x-api-key: $ROOT" -H 'content-type: application/json' \
  -d '{"user_id":"me","role":"admin"}'

# 为用户签发 API Key
curl -s -XPOST localhost:1933/api/v1/admin/accounts/default/users/me/key \
  -H "x-api-key: $ROOT"
```

第二条命令会返回该用户的 API Key，请妥善保存——后续调用 OpenViking 的业务接口时使用它，而不是 root 密钥。

## 完成

部署到此结束。具体使用方法、API 说明与进阶配置请参考官方文档：

🔗 https://www.openviking.ai/

## 附：常用运维命令

```bash
# 查看服务状态
docker compose ps

# 查看实时日志
docker compose logs -f openviking

# 停止服务
docker compose down

# 升级镜像（拉取最新版后重启）
docker compose pull && docker compose up -d
```

## 数据与安全说明

- OpenViking 的工作区数据持久化在 `./data` 目录，Ollama 模型存储在命名卷 `ollama_models` 中，`docker compose down` 不会删除数据；如需彻底清理请追加 `-v`。
- `ov.conf` 中包含 root API Key 与 VLM API Key，**请勿提交到公开代码仓库**（本仓库的 `.gitignore` 已默认忽略 `ov.conf`）。
