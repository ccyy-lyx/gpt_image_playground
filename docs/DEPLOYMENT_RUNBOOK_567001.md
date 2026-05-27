# GPT Image Playground 567001 部署与更新手册

本文档整理当前这套服务在新环境上的可重复部署方式，以及后续更新流程。

目标：
- 部署 `gpt-image-playground`
- 部署 `image-sse-proxy`
- 配置 `systemd service`
- 配置 `api.567001.xyz` 的 OpenResty/1Panel 代理规则
- 提供后续更新步骤
- 明确避免把任何 token 提交到 GitHub

本文档不包含任何真实 token、PAT、密码、`gh hosts.yml` 内容。

## 1. 当前架构

当前线上链路分两段：

1. `image.567001.xyz`
   - 提供前端页面
   - 反代到本机 `gpt-image-playground` Docker 容器
   - 容器对外映射 `8081 -> 80`

2. `api.567001.xyz`
   - 提供 OpenAI 兼容接口
   - 普通请求反代到本机 `newapi`，端口 `3000`
   - `POST /v1/images/generations` 单独转到本机 `image-sse-proxy`，端口 `3101`

这样做的原因：
- `gpt-image-2` 的长图生成请求在 Cloudflare 链路上容易出现“上游最终成功，但浏览器提前断开”
- `image-sse-proxy` 会立刻返回 `text/event-stream`，先发首包，再周期性发心跳，等上游最终 JSON 返回后再转成流式完成事件

## 2. 目录与端口约定

当前使用如下路径和端口：

- 应用代码目录：`/opt/gpt_image_playground_update`
- `image-sse-proxy` 目录：`/opt/image-sse-proxy`
- systemd 文件：`/etc/systemd/system/image-sse-proxy.service`
- 1Panel 站点配置目录：`/opt/1panel/www/sites/api.567001.xyz/proxy`
- `gpt-image-playground` 容器映射端口：`8081`
- `newapi` 端口：`3000`
- `image-sse-proxy` 端口：`3101`

如果新环境和这里不同，把下面命令里的路径和端口替换掉即可。

## 3. 前置条件

需要具备：

- Docker
- OpenResty 或 Nginx
- systemd
- Node.js
- Git
- GitHub CLI `gh`

推荐 Node 版本：
- `v22.x`

当前机器使用：
- `/root/.nvm/versions/node/v22.22.1/bin/node`

如果新环境 Node 路径不同，记得同步修改 systemd 文件里的 `ExecStart`。

## 4. Git 与仓库约定

推荐使用你自己的 fork 作为部署源码来源。

当前 fork：
- `https://github.com/ccyy-lyx/gpt_image_playground`

推荐新环境初始化方式：

```bash
gh auth login
gh auth setup-git

gh repo clone ccyy-lyx/gpt_image_playground /opt/gpt_image_playground_update
cd /opt/gpt_image_playground_update
git remote add upstream https://github.com/CookSleep/gpt_image_playground.git
```

不要做的事：

- 不要把 PAT 写进 `git remote`
- 不要把 `/root/.config/gh/hosts.yml` 提交到 GitHub
- 不要把 `.env`、token、cookie、私钥写进仓库
- 不要把临时命令里的 token 留在 shell history 或文档里

如果你曾经误把 token 放进 remote URL，立刻改回普通 URL：

```bash
git remote set-url origin https://github.com/ccyy-lyx/gpt_image_playground.git
```

## 5. 部署 gpt-image-playground

### 5.1 构建镜像

在源码目录执行：

```bash
cd /opt/gpt_image_playground_update
docker build -f deploy/Dockerfile -t gpt-image-playground:timeoutfix .
```

### 5.2 启动容器

当前线上容器运行参数：

- `DEFAULT_API_URL=https://api.567001.xyz/v1`
- `ENABLE_API_PROXY=false`
- `LOCK_API_PROXY=false`
- 端口映射：`8081:80`
- 网络：`sub2api_sub2api-network`
- 重启策略：`unless-stopped`

部署命令：

```bash
docker rm -f gpt-image-playground || true

docker run -d \
  --name gpt-image-playground \
  --restart unless-stopped \
  --network sub2api_sub2api-network \
  -p 8081:80 \
  -e DEFAULT_API_URL=https://api.567001.xyz/v1 \
  -e ENABLE_API_PROXY=false \
  -e LOCK_API_PROXY=false \
  gpt-image-playground:timeoutfix
```

说明：

- 这里显式关闭了容器内置 `API_PROXY`
- 图片站前端默认直接请求 `https://api.567001.xyz/v1`
- 这样可以避免再经过 `image.567001.xyz` 自己的 `/api-proxy` 链路

## 6. 部署 image-sse-proxy

### 6.1 创建目录

```bash
mkdir -p /opt/image-sse-proxy
```

### 6.2 写入 `/opt/image-sse-proxy/server.js`

把以下完整内容写入：

```js
const http = require('http')
const { URL } = require('url')

const PORT = Number(process.env.PORT || 3101)
const UPSTREAM_BASE = (process.env.UPSTREAM_BASE || 'http://127.0.0.1:3000').replace(/\/+$/, '')
const HEARTBEAT_MS = Number(process.env.HEARTBEAT_MS || 15000)
const UPSTREAM_TIMEOUT_MS = Number(process.env.UPSTREAM_TIMEOUT_MS || 600000)

function writeJson(res, statusCode, payload) {
  const body = JSON.stringify(payload)
  res.writeHead(statusCode, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(body),
    'Cache-Control': 'no-store',
  })
  res.end(body)
}

function buildHeaders(req, bodyLength) {
  const headers = { ...req.headers }
  delete headers.host
  headers.connection = 'keep-alive'
  if (bodyLength != null) headers['content-length'] = String(bodyLength)
  return headers
}

function normalizeImageResultPayload(payload) {
  if (!payload || typeof payload !== 'object') return null
  const data = Array.isArray(payload.data) ? payload.data : null
  if (!data || data.length === 0) return null
  const first = data[0] && typeof data[0] === 'object' ? data[0] : {}
  const event = {
    type: 'image_generation.completed',
  }
  if (typeof first.b64_json === 'string') event.b64_json = first.b64_json
  if (typeof first.revised_prompt === 'string') event.revised_prompt = first.revised_prompt
  if (typeof payload.size === 'string') event.size = payload.size
  if (typeof payload.quality === 'string') event.quality = payload.quality
  if (typeof payload.output_format === 'string') event.output_format = payload.output_format
  if (typeof payload.moderation === 'string') event.moderation = payload.moderation
  if (typeof payload.output_compression === 'number') event.output_compression = payload.output_compression
  return event
}

async function readRequestBody(req) {
  const chunks = []
  for await (const chunk of req) chunks.push(chunk)
  return Buffer.concat(chunks)
}

async function handleImageGeneration(req, res) {
  let body
  try {
    body = await readRequestBody(req)
  } catch (err) {
    writeJson(res, 400, { error: 'failed to read request body' })
    return
  }

  let payload
  try {
    payload = JSON.parse(body.toString('utf8') || '{}')
  } catch {
    writeJson(res, 400, { error: 'invalid json body' })
    return
  }

  const requestPayload = { ...payload }
  delete requestPayload.stream
  delete requestPayload.partial_images

  const upstreamUrl = new URL('/v1/images/generations', UPSTREAM_BASE)
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), UPSTREAM_TIMEOUT_MS)

  req.on('close', () => controller.abort())

  res.writeHead(200, {
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-cache, no-transform',
    Connection: 'keep-alive',
    'X-Accel-Buffering': 'no',
    'Access-Control-Allow-Origin': req.headers.origin || '*',
    'Access-Control-Allow-Credentials': 'true',
  })
  res.write(': stream-start\\n\\n')

  const heartbeat = setInterval(() => {
    if (!res.writableEnded) res.write(': ping\\n\\n')
  }, HEARTBEAT_MS)

  try {
    const upstreamRes = await fetch(upstreamUrl, {
      method: 'POST',
      headers: buildHeaders(req, Buffer.byteLength(JSON.stringify(requestPayload))),
      body: JSON.stringify(requestPayload),
      signal: controller.signal,
    })

    if (!upstreamRes.ok) {
      const errorText = await upstreamRes.text()
      res.write(`data: ${JSON.stringify({ error: { message: errorText || `HTTP ${upstreamRes.status}` } })}\\n\\n`)
      res.end('data: [DONE]\\n\\n')
      return
    }

    const rawText = await upstreamRes.text()
    let json
    try {
      json = JSON.parse(rawText)
    } catch {
      res.write(`data: ${JSON.stringify({ error: { message: 'upstream returned invalid json' } })}\\n\\n`)
      res.end('data: [DONE]\\n\\n')
      return
    }

    const event = normalizeImageResultPayload(json)
    if (!event || !event.b64_json) {
      res.write(`data: ${JSON.stringify({ error: { message: 'upstream did not return image data' } })}\\n\\n`)
      res.end('data: [DONE]\\n\\n')
      return
    }

    res.write(`data: ${JSON.stringify(event)}\\n\\n`)
    res.end('data: [DONE]\\n\\n')
  } catch (err) {
    const message = err && typeof err === 'object' && 'name' in err && err.name === 'AbortError'
      ? 'request aborted'
      : (err instanceof Error ? err.message : String(err))
    if (!res.writableEnded) {
      res.write(`data: ${JSON.stringify({ error: { message } })}\\n\\n`)
      res.end('data: [DONE]\\n\\n')
    }
  } finally {
    clearInterval(heartbeat)
    clearTimeout(timeout)
  }
}

const server = http.createServer(async (req, res) => {
  if (!req.url) {
    writeJson(res, 400, { error: 'missing url' })
    return
  }

  const url = new URL(req.url, `http://${req.headers.host || 'localhost'}`)

  if (req.method === 'OPTIONS') {
    res.writeHead(204, {
      'Access-Control-Allow-Origin': req.headers.origin || '*',
      'Access-Control-Allow-Credentials': 'true',
      'Access-Control-Allow-Headers': req.headers['access-control-request-headers'] || '*',
      'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
      'Access-Control-Max-Age': '43200',
    })
    res.end()
    return
  }

  if (req.method === 'POST' && url.pathname === '/v1/images/generations') {
    await handleImageGeneration(req, res)
    return
  }

  const upstreamUrl = new URL(url.pathname + url.search, UPSTREAM_BASE)
  const body = ['GET', 'HEAD'].includes(req.method || 'GET') ? undefined : await readRequestBody(req)
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), UPSTREAM_TIMEOUT_MS)
  req.on('close', () => controller.abort())

  try {
    const upstreamRes = await fetch(upstreamUrl, {
      method: req.method,
      headers: buildHeaders(req, body?.length),
      body,
      signal: controller.signal,
    })
    const headers = Object.fromEntries(upstreamRes.headers.entries())
    res.writeHead(upstreamRes.status, headers)
    if (upstreamRes.body) {
      for await (const chunk of upstreamRes.body) res.write(chunk)
    }
    res.end()
  } catch (err) {
    writeJson(res, 502, { error: err instanceof Error ? err.message : String(err) })
  } finally {
    clearTimeout(timeout)
  }
})

server.listen(PORT, '127.0.0.1', () => {
  console.log(`image-sse-proxy listening on 127.0.0.1:${PORT}`)
})
```

## 7. systemd service

把以下内容写入 `/etc/systemd/system/image-sse-proxy.service`：

```ini
[Unit]
Description=Image SSE Proxy for Cloudflare-safe image generation
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/image-sse-proxy
ExecStart=/root/.nvm/versions/node/v22.22.1/bin/node /opt/image-sse-proxy/server.js
Restart=always
RestartSec=3
Environment=PORT=3101
Environment=UPSTREAM_BASE=http://127.0.0.1:3000
Environment=HEARTBEAT_MS=15000
Environment=UPSTREAM_TIMEOUT_MS=600000

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
systemctl daemon-reload
systemctl enable --now image-sse-proxy
systemctl status --no-pager image-sse-proxy
```

如果 Node 路径不同，修改 `ExecStart=` 后再执行上面三条命令。

## 8. api.567001.xyz 代理配置

当前 1Panel 站点代理配置文件：

- `/opt/1panel/www/sites/api.567001.xyz/proxy/root.conf`

请确保内容至少包含以下规则：

```nginx
location = /v1/images/generations {
    proxy_pass http://127.0.0.1:3101;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "";
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_request_buffering off;
    gzip off;
    add_header X-Accel-Buffering no always;
    add_header Cache-Control "no-cache, no-transform" always;
    add_header Strict-Transport-Security "max-age=31536000";
    proxy_connect_timeout 60s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
}

location ^~ / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_http_version 1.1;
    add_header X-Cache $upstream_cache_status;
    proxy_ssl_server_name off;
    proxy_ssl_name $proxy_host;
    proxy_buffering off;
    proxy_request_buffering off;
    gzip off;
    add_header X-Accel-Buffering no always;
    add_header Strict-Transport-Security "max-age=31536000";
    proxy_connect_timeout 60s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
}
```

重载 OpenResty：

```bash
docker exec 9203948e8da4 sh -lc 'openresty -t && openresty -s reload'
```

如果新环境不是这个容器 ID，请替换成你自己的 OpenResty/Nginx 容器或服务。

## 9. image.567001.xyz 站点约定

`image.567001.xyz` 站点应继续反代到：

- `http://127.0.0.1:8081`

不需要再启用 `gpt-image-playground` 容器内的 `/api-proxy` 作为主链路。

## 10. 部署验证

### 10.1 验证前端站点

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://image.567001.xyz/
```

期望：

```text
200
```

### 10.2 验证 `OPTIONS /v1/images/generations`

```bash
curl -isk \
  -X OPTIONS https://api.567001.xyz/v1/images/generations \
  -H 'Origin: https://image.567001.xyz' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: authorization,content-type'
```

期望：

- `HTTP 204`
- 存在 `Access-Control-Allow-Origin`

### 10.3 验证 SSE 首包

```bash
timeout 12s curl -iskN \
  https://api.567001.xyz/v1/images/generations \
  -H 'Origin: https://image.567001.xyz' \
  -H 'Authorization: Bearer INVALID' \
  -H 'Content-Type: application/json' \
  --data '{"model":"gpt-image-2","prompt":"test","size":"1024x1024","output_format":"png","moderation":"auto"}'
```

期望：

- 返回 `HTTP 200`
- `Content-Type: text/event-stream`
- 首屏马上出现：

```text
: stream-start
```

只要首包能马上出来，说明 Cloudflare 链路已经不再等最终整包才出响应。

## 11. 更新流程

更新分两部分：

1. 应用代码更新
2. 部署链路确认

### 11.1 更新你的 fork

在本地开发机或服务器上：

```bash
cd /opt/gpt_image_playground_update
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

如果你是先在修复分支上开发，就先合并修复分支到你自己的 `main`，再继续同步上游。

### 11.2 服务器拉取最新代码

```bash
cd /opt/gpt_image_playground_update
git fetch origin
git checkout main
git pull --ff-only origin main
```

### 11.3 重新构建并替换 `gpt-image-playground`

```bash
cd /opt/gpt_image_playground_update
docker build -f deploy/Dockerfile -t gpt-image-playground:timeoutfix .

docker rm -f gpt-image-playground || true

docker run -d \
  --name gpt-image-playground \
  --restart unless-stopped \
  --network sub2api_sub2api-network \
  -p 8081:80 \
  -e DEFAULT_API_URL=https://api.567001.xyz/v1 \
  -e ENABLE_API_PROXY=false \
  -e LOCK_API_PROXY=false \
  gpt-image-playground:timeoutfix
```

### 11.4 如果 `image-sse-proxy` 改过，重启它

```bash
systemctl restart image-sse-proxy
systemctl status --no-pager image-sse-proxy
```

### 11.5 检查 1Panel/OpenResty 配置有没有被覆盖

重点检查：

- `/opt/1panel/www/sites/api.567001.xyz/proxy/root.conf`

因为 1Panel 面板操作、站点重建、代理规则调整都有可能覆盖这个文件。

检查完后执行：

```bash
docker exec 9203948e8da4 sh -lc 'openresty -t && openresty -s reload'
```

### 11.6 最后做验证

至少执行这三条：

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://image.567001.xyz/
curl -sk -o /dev/null -w '%{http_code}\n' -X OPTIONS https://api.567001.xyz/v1/images/generations -H 'Origin: https://image.567001.xyz' -H 'Access-Control-Request-Method: POST'
systemctl is-active image-sse-proxy
```

期望：

- `200`
- `204`
- `active`

## 12. 回滚流程

如果新版本异常，按下面方式回滚：

1. 回到上一个稳定 commit
2. 重新构建 `gpt-image-playground`
3. 保持 `image-sse-proxy` 和 OpenResty 配置不动

示例：

```bash
cd /opt/gpt_image_playground_update
git log --oneline -n 20
git checkout <stable_commit>
docker build -f deploy/Dockerfile -t gpt-image-playground:timeoutfix .
docker rm -f gpt-image-playground || true
docker run -d \
  --name gpt-image-playground \
  --restart unless-stopped \
  --network sub2api_sub2api-network \
  -p 8081:80 \
  -e DEFAULT_API_URL=https://api.567001.xyz/v1 \
  -e ENABLE_API_PROXY=false \
  -e LOCK_API_PROXY=false \
  gpt-image-playground:timeoutfix
```

## 13. 不要上传到 GitHub 的内容

下面这些内容不要提交：

- 任何 GitHub token
- 任何 OpenAI / NewAPI / Cloudflare token
- `/root/.config/gh/hosts.yml`
- 带 token 的 remote URL
- 临时导出的 `GITHUB_TOKEN`
- 包含真实密钥的 `.env`
- 面板导出的完整站点配置备份（如果里面含敏感信息）

推荐做法：

- 认证只用 `gh auth login`
- 命令示例里只放占位符
- 所有 secret 只保留在服务器本机
- 提交前执行：

```bash
git diff --check
git status
```

## 14. 建议

这份文档可以安全放进 GitHub，因为它不包含 token。

如果后面还要进一步固化，建议再补两样：

1. 把 `image-sse-proxy` 独立成仓库内一个 `deploy-extra/` 目录
2. 再做一个只含占位符的 `.env.example` 和 `docker-compose.example.yml`

但就当前需求来说，这一个文件已经足够用于新环境重复部署与后续更新。
