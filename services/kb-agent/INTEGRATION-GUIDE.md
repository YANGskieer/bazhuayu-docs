# 八爪鱼文档智能问答 Agent 对接与上线指南

本文供研发、运维直接执行。目标是在现有 Mintlify 文档站的“问题解答”页面中启用 DeepSeek 智能问答，同时保证 API Key 不进入浏览器、Git 仓库、静态构建产物或业务日志。

## 1. 当前状态与目标

当前文档站可以先按无模型模式上线：前端使用本地可信索引推荐相关教程，不进行模型推理。

完整模式的请求链路如下：

```text
用户浏览器
  -> https://docs-api.bazhuayu.com/v1/ask
  -> Nginx / 网关
  -> 127.0.0.1:8787 的 Node Agent 服务
  -> 本地可信知识索引
  -> DeepSeek API
  -> 结构化答案返回前端
```

生产环境必须先部署 Agent 后端并完成验收，再将前端公开配置中的 `enabled` 切换为 `true`。后端未就绪时保持 `enabled: false`，避免用户先等待接口失败再进入降级检索。

## 2. 已有代码与数据

关键文件：

| 文件 | 用途 |
| --- | --- |
| `services/kb-agent/server.mjs` | Agent HTTP 服务，唯一允许读取模型 API Key 的组件 |
| `services/kb-agent/lib/agent-core.mjs` | 检索、问题分类、输出解析及 MCP 双重校验 |
| `assets/knowledge-base/agent-index.json` | 供 Agent 使用的可信知识分片 |
| `assets/knowledge-base/search-index.json` | FAQ 意图及前端降级检索索引 |
| `assets/knowledge-base/agent-config.json.txt` | 浏览器可读取的公开服务地址，不得包含密钥 |
| `scripts/data/kb-review-overrides.json` | 人工审核结果源文件 |
| `scripts/build-knowledge-index.mjs` | 根据文档和审核结果重建索引 |

知识边界：

- 采集学院、产品文档、首页概述、MCP、CLI、OpenAPI 文档可进入可信答案索引。
- 人工确认的 FAQ 可进入可信答案索引。
- 已停用 FAQ 不进入索引。
- 未审核或稍后处理 FAQ 只能帮助识别问题意图，旧答案正文不得发送给模型。
- Agent 每次提问都会重新读取索引文件。若服务器直接同步了新的索引文件，无需重启 Node 进程；若采用 Docker 镜像或不可变发布，则需要发布新镜像。

## 3. 运行环境

最低要求：

- Linux 服务器，建议 Ubuntu 22.04 或更高版本。
- Node.js 20 LTS 或更高版本，必须支持原生 `fetch`。
- Nginx、Ingress 或等价 HTTPS 网关。
- 域名 `docs-api.bazhuayu.com` 已解析到服务入口并配置有效 TLS 证书。
- 服务器可以通过 HTTPS 访问 `api.deepseek.com`。
- Agent 进程只监听 `127.0.0.1`，不得直接暴露 8787 端口到公网。

建议目录：

```text
/srv/bazhuayu-docs
/etc/bazhuayu-kb-agent.env
/var/log/nginx/
```

## 4. 模型接口

使用 DeepSeek 的 OpenAI Chat Completions 兼容接口：

```text
POST https://api.deepseek.com/chat/completions
Authorization: Bearer ${DEEPSEEK_API_KEY}
Content-Type: application/json
```

生产模型固定为：

```text
deepseek-v4-flash
```

当前服务会根据问题复杂度使用：

- 简单问题：`thinking.type=disabled`。
- 排错、原因分析、高风险问题：`thinking.type=enabled` 且 `reasoning_effort=high`。
- 输出格式：`response_format={"type":"json_object"}`。
- 单次上游超时：45 秒。
- MCP 问题：第一次生成答案，第二次基于协议资料和八爪鱼产品资料进行事实复核。

服务只读取最终 `message.content`，不得把 `reasoning_content` 返回前端、保存到日志或展示给用户。

官方参考：

- https://api-docs.deepseek.com/
- https://api-docs.deepseek.com/api/create-chat-completion
- https://api-docs.deepseek.com/guides/thinking_mode
- https://api-docs.deepseek.com/guides/json_mode

## 5. 密钥与环境变量

### 5.1 密钥要求

- 生产 Key 必须重新创建并通过服务器密钥管理系统配置。
- 曾经通过聊天、邮件、工单或截图发送过的 Key 均按已泄露处理，必须废弃。
- 不得把真实 Key 写入 `.env` 后提交 Git。
- 不得把 Key 放入 Mintlify 环境、前端 JavaScript、`agent-config.json.txt` 或 Nginx 返回内容。
- 不得记录 `Authorization` 请求头或 DeepSeek 完整错误响应。

### 5.2 生产环境变量

创建 `/etc/bazhuayu-kb-agent.env`，权限设为 `600`，所有者设为运行 Agent 的系统账号：

```dotenv
NODE_ENV=production

DEEPSEEK_API_KEY=<由密钥系统注入，不写入仓库>
DEEPSEEK_API_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-v4-flash

KB_AGENT_HOST=127.0.0.1
KB_AGENT_PORT=8787
KB_AGENT_ALLOWED_ORIGINS=https://www.bazhuayu.com,https://docs.bazhuayu.com
KB_AGENT_TRUST_PROXY=true
KB_AGENT_RATE_LIMIT_PER_HOUR=20
KB_AGENT_RATE_LIMIT_PER_DAY=1000
```

注意：

- `KB_AGENT_ALLOWED_ORIGINS` 只能填写真实文档站 Origin，即协议、域名和端口，不带路径，不带末尾 `/`。
- Mintlify 预览域名如需联调，应逐个加入，不要使用 `*`。
- 只有 Agent 前面确实存在受信任的反向代理时才设置 `KB_AGENT_TRUST_PROXY=true`。
- 可选变量 `KB_AGENT_INDEX_PATH` 和 `KB_AGENT_INTENT_INDEX_PATH` 可用于指定绝对索引路径；默认读取仓库内的文件。

## 6. 构建与部署前检查

在服务器或 CI 中执行：

```bash
cd /srv/bazhuayu-docs
npm ci
npm run kb:build
npm run kb:validate
npm run kb:agent:test
node --check services/kb-agent/server.mjs
```

验收标准：

- `kb:build` 成功生成 `search-index.json` 与 `agent-index.json`。
- `kb:validate` 无未审核正文泄漏、无停用答案进入可信索引。
- Agent 单元测试全部通过。
- 构建日志、产物和浏览器源码中检索不到 `DEEPSEEK_API_KEY` 或真实 Key 内容。

## 7. Systemd 部署示例

先创建专用账号和可写反馈目录：

```bash
sudo useradd --system --home /srv/bazhuayu-docs --shell /usr/sbin/nologin kb-agent
sudo mkdir -p /srv/bazhuayu-docs/services/kb-agent/data
sudo chown -R kb-agent:kb-agent /srv/bazhuayu-docs/services/kb-agent/data
sudo chown root:kb-agent /etc/bazhuayu-kb-agent.env
sudo chmod 600 /etc/bazhuayu-kb-agent.env
```

创建 `/etc/systemd/system/bazhuayu-kb-agent.service`：

```ini
[Unit]
Description=Bazhuayu Documentation Knowledge Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=kb-agent
Group=kb-agent
WorkingDirectory=/srv/bazhuayu-docs
EnvironmentFile=/etc/bazhuayu-kb-agent.env
ExecStart=/usr/bin/node /srv/bazhuayu-docs/services/kb-agent/server.mjs
Restart=on-failure
RestartSec=5
TimeoutStopSec=20
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/srv/bazhuayu-docs/services/kb-agent/data

[Install]
WantedBy=multi-user.target
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bazhuayu-kb-agent
sudo systemctl status bazhuayu-kb-agent
curl --fail http://127.0.0.1:8787/health
```

`/health` 预期返回：

```json
{
  "ok": true,
  "model": "deepseek-v4-flash",
  "mcpProtocolVersion": "2025-11-25"
}
```

## 8. Nginx 反向代理示例

```nginx
server {
    listen 443 ssl http2;
    server_name docs-api.bazhuayu.com;

    ssl_certificate     /etc/letsencrypt/live/docs-api.bazhuayu.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/docs-api.bazhuayu.com/privkey.pem;

    client_max_body_size 20k;

    location = /health {
        proxy_pass http://127.0.0.1:8787/health;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 10s;
    }

    location /v1/ {
        proxy_pass http://127.0.0.1:8787;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        proxy_buffering off;
    }
}
```

配置检查：

```bash
sudo nginx -t
sudo systemctl reload nginx
curl --fail https://docs-api.bazhuayu.com/health
```

若前面还有 CDN 或云负载均衡，需要确保真实客户端 IP 按可信链路写入 `X-Forwarded-For`，并由网关限制只有受信任代理可以连接 Agent 所在主机。

## 9. 前端开关

浏览器读取：

```text
/assets/knowledge-base/agent-config.json.txt
```

后端联调通过前保持：

```json
{
  "enabled": false,
  "endpoint": "https://docs-api.bazhuayu.com/v1/ask",
  "feedbackEndpoint": "https://docs-api.bazhuayu.com/v1/feedback"
}
```

后端通过生产验收后改为：

```json
{
  "enabled": true,
  "endpoint": "https://docs-api.bazhuayu.com/v1/ask",
  "feedbackEndpoint": "https://docs-api.bazhuayu.com/v1/feedback",
  "localEndpoint": "http://127.0.0.1:8787/v1/ask",
  "localFeedbackEndpoint": "http://127.0.0.1:8787/v1/feedback",
  "timeoutMs": 45000
}
```

该文件是公开配置，只能放开关和服务地址。切换后需要重新发布静态文档站，使浏览器获得新配置。

正确发布顺序：

1. 部署 Agent 服务，但前端保持 `enabled: false`。
2. 验证 `/health`、CORS、问答、反馈和限流。
3. 用生产文档域名完成端到端联调。
4. 将 `enabled` 改为 `true` 并发布文档站。
5. 观察错误率、时延和 DeepSeek 用量。

## 10. 接口协议

### 10.1 健康检查

```http
GET /health
```

不依赖模型调用，仅证明 Node 服务正常启动并显示当前模型配置。

### 10.2 提问

```http
POST /v1/ask
Content-Type: application/json
Origin: https://www.bazhuayu.com
```

请求：

```json
{
  "question": "列表采集第二页开始重复第一页的数据，应该怎么排查？",
  "messages": [
    { "role": "user", "content": "上一轮问题" },
    { "role": "assistant", "content": "上一轮最终答案" }
  ],
  "sessionId": "可选，当前服务不做长期会话存储"
}
```

约束：

- `question` 长度为 2 至 1000 个字符。
- 前端最多保留最近 3 轮，即 6 条消息。
- 服务端会清理手机号、邮箱等敏感内容。
- 请求正文超过 16 KB 会被拒绝。

成功响应：

```json
{
  "kind": "answer",
  "answer": "简短结论",
  "steps": ["步骤一", "步骤二"],
  "cautions": ["注意事项"],
  "sources": [
    {
      "title": "教程标题",
      "route": "/zh/academy/...",
      "heading": "相关小节"
    }
  ],
  "followUps": ["推荐追问"],
  "confidence": "high",
  "needsHumanReview": false,
  "usage": {
    "promptTokens": 0,
    "completionTokens": 0
  }
}
```

无可信资料时返回 HTTP 200，但 `kind` 为 `no_trusted_answer`，不会调用模型编造答案。

MCP 问题额外返回：

```json
{
  "validation": {
    "mode": "mcp_double_check",
    "passed": true,
    "protocolVersion": "2025-11-25"
  }
}
```

### 10.3 反馈

```http
POST /v1/feedback
Content-Type: application/json
Origin: https://www.bazhuayu.com
```

请求：

```json
{
  "rating": "helpful",
  "question": "用户问题",
  "sources": ["/zh/academy/..."],
  "note": "可选说明"
}
```

`rating` 仅允许 `helpful` 或 `unhelpful`。服务仅保存脱敏问题指纹、引用、评分和简短备注，不应保存完整对话。

### 10.4 错误码

| HTTP 状态 | error | 含义 |
| --- | --- | --- |
| 400 | `invalid_question` | 问题为空、过短或超过 1000 字 |
| 403 | `origin_not_allowed` | Origin 不在白名单 |
| 429 | `rate_limited` | IP 小时限额或全站日限额已到 |
| 502 | `upstream_error` / `agent_unavailable` | DeepSeek 或响应解析异常 |
| 503 | `configuration_error` | 服务端没有配置 API Key |
| 504 | `agent_unavailable` | 上游请求超时 |

前端遇到任何非 2xx 响应时应切换到本地可信教程检索，不显示空白页。

## 11. 联调命令

### 11.1 模型和密钥

只在服务器安全会话中执行，不要把 Key 写入命令历史：

```bash
curl --fail https://api.deepseek.com/models \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}"
```

确认响应包含 `deepseek-v4-flash`。

### 11.2 Agent 问答

```bash
curl --fail https://docs-api.bazhuayu.com/v1/ask \
  -H 'Origin: https://www.bazhuayu.com' \
  -H 'Content-Type: application/json' \
  --data '{"question":"列表采集如何翻页？","messages":[]}'
```

### 11.3 CORS 拒绝测试

```bash
curl -i https://docs-api.bazhuayu.com/v1/ask \
  -H 'Origin: https://unauthorized.example' \
  -H 'Content-Type: application/json' \
  --data '{"question":"测试问题","messages":[]}'
```

预期返回 403。

### 11.4 MCP 双重校验

```bash
curl --fail https://docs-api.bazhuayu.com/v1/ask \
  -H 'Origin: https://www.bazhuayu.com' \
  -H 'Content-Type: application/json' \
  --data '{"question":"所有 MCP 的连接方式是否完全一样？","messages":[]}'
```

预期 `validation.mode` 为 `mcp_double_check`，且只有复核通过时 `passed` 才为 `true`。

## 12. 生产验收清单

- [ ] 浏览器源码、网络响应、静态资源中不存在 API Key。
- [ ] Agent 只监听 `127.0.0.1:8787`。
- [ ] 公网只开放 HTTPS 443。
- [ ] 生产文档 Origin 可以请求，其他 Origin 返回 403。
- [ ] 未配置 Key 时返回 503，前端正常降级。
- [ ] DeepSeek 超时或异常时前端正常降级。
- [ ] 412 条人工确认 FAQ 均可进入可信检索，129 条停用 FAQ 均不进入索引。
- [ ] 无匹配问题不会生成无来源答案。
- [ ] 金额、套餐、退款、合同、账号权限、版本问题显示人工确认提示。
- [ ] MCP 回答区分协议能力与八爪鱼产品能力。
- [ ] 反馈接口可写入，日志目录权限正确。
- [ ] 限流生效，不同真实客户端 IP 可正确区分。
- [ ] 日志不记录请求正文、完整对话、API Key、Authorization 头或模型思维内容。
- [ ] 记录请求状态、耗时、错误类别和汇总 token 用量。

建议上线前使用至少 100 个真实问题验收，引用正确率不低于 95%，金额、版本、权限和操作步骤不得出现无来源事实。

## 13. 更新知识库

FAQ 审核或文档变更后：

```bash
npm run kb:build
npm run kb:validate
git add scripts/data/kb-review-overrides.json \
        assets/knowledge-base/search-index.json \
        assets/knowledge-base/search-index.json.txt \
        assets/knowledge-base/agent-index.json
git commit -m "update trusted knowledge index"
git push
```

线上更新方式：

- 普通目录部署：拉取新提交并原子替换索引文件，Agent 下一次请求自动读取新索引。
- Docker/容器部署：构建并发布包含新索引的镜像。
- 更新过程中不要先替换一个索引、延迟很久再替换另一个索引；建议通过版本目录或发布包整体切换。

## 14. 监控与告警

至少监控：

- `/health` 可用性。
- `/v1/ask` 请求量、2xx/4xx/5xx 比例。
- P50、P95、P99 响应时间。
- 429、502、503、504 数量。
- DeepSeek prompt/completion token 日用量和预算。
- `no_trusted_answer` 比例。
- MCP 双重校验失败比例。
- 用户 `unhelpful` 反馈比例。

日志只记录请求 ID、时间、耗时、状态码、错误类型、来源数量、置信等级和 token 汇总，不记录原始问题和完整答案。

## 15. 回滚

出现模型异常、费用异常或错误回答时：

1. 立即把前端 `enabled` 改为 `false` 并发布，用户恢复本地可信教程检索。
2. 停止或回滚 `bazhuayu-kb-agent` 服务。
3. 保留反向代理健康检查，但不要把上游错误详情返回浏览器。
4. 回滚到上一份已验证的知识索引。
5. 若怀疑 Key 泄露，立即在 DeepSeek 控制台撤销并更换。

前端开关是首要止损手段；不要通过删除知识库或在前端写死假答案进行回滚。

## 16. 研发交付完成标准

研发完成以下事项即可认为对接完成：

1. Agent 服务按本文方式运行在服务端，Key 仅由服务端密钥系统注入。
2. `docs-api.bazhuayu.com` 已启用 HTTPS、CORS 白名单和反向代理。
3. `/health`、`/v1/ask`、`/v1/feedback` 全部通过联调。
4. DeepSeek 使用 `deepseek-v4-flash`，复杂问题开启思考模式，但不返回思维链。
5. 回答只使用可信索引并返回真实站内来源。
6. MCP 问题完成第二次事实复核。
7. 异常时前端自动降级到本地可信检索。
8. 监控、限流、日志脱敏、密钥轮换和回滚方案已经演练。
9. 最后才把前端 `enabled` 切换为 `true`。
