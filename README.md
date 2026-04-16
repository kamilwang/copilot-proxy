# copilot-proxy
services:
  copilot-proxy:
    image: kamilwang/copilot-proxy:0.1.15
    container_name: copilot-proxy
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3000" # host 3001 -> container 3000 (container will bind to 127.0.0.1)
    environment:
      - PORT=3000
      - BIND_ADDR=0.0.0.0
      - OPENCLAW_STATE_DIR=/var/lib/openclaw-state
      - SKIP_PORT_PROBE=1
      - PROXY_STRICT_PASSTHROUGH=1
      # Container will listen on all interfaces; host maps 127.0.0.1:3001 -> container:3000
      # Optional: set a proxy API key to protect the endpoint
      # - PROXY_API_KEY=your_api_key_here
      # Optional: provide a GitHub token for Copilot token resolution (or use device-flow endpoints)
      # - COPILOT_GITHUB_TOKEN=ghp_xxx
    volumes:
      - /var/lib/openclaw-state:/var/lib/openclaw-state:rw
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O - http://127.0.0.1:3000/health >/dev/null 2>&1 || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 5s




################以下为说明###########################################
      copilot-proxy-service

目的
- 从 OpenClaw 源码抽取并独立运行 GitHub Copilot 的 device-flow 授权与 Copilot runtime token 交换逻辑。
- 提供一个轻量 HTTP proxy，供本机上其它程序（例如 OpenClaw 实例、脚本或服务）通过统一接口获取 Copilot runtime token 并转发请求到 Copilot API。

快速概览（中文）
- 目录：/opt/copilot-proxy-service
- 默认绑定端口：1111（可用 PORT 环境变量覆盖）

环境变量（重要）
- PORT: HTTP 端口（默认 1111）
- OPENCLAW_STATE_DIR: 状态目录，缓存与 auth-profiles 存放位置（默认 ~/.openclaw）
- PROXY_API_KEY: 可选，启用后要求所有请求通过 x-api-key header 或 ?api_key= 值进行验证（生产环境推荐开启）
- PROXY_EDITOR_VERSION: 覆盖默认的 Editor-Version header（默认：vscode/1.96.2）
- PROXY_USER_AGENT: 覆盖默认的 User-Agent header（默认：GitHubCopilotChat/0.26.7）
- COPILOT_GITHUB_TOKEN: 可选，直接提供 GitHub token（替代从 auth-profiles 读取）

主要端点与使用说明

1) GET /health
- 返回服务存活信息：{ ok: true }

2) GET /models
- 返回 proxy 接受的模型列表（结构化），包含 reasoning 和 contextWindow 字段，方便上游发现能力。
- 示例返回（简化）:
  {
    "default": "gpt-5-mini",
    "supported": [
      { "id": "gpt-5-mini", "name": "gpt-5-mini", "contextWindow": 128000, "reasoning": false },
      { "id": "gpt-5.2", "name": "gpt-5.2", "contextWindow": 128000, "reasoning": true }
    ]
  }

3) POST /resolve-token
- 功能：使用 GitHub token（请求体的 githubToken 优先）向 Copilot token endpoint 交换 runtime token，并把结果缓存到 $OPENCLAW_STATE_DIR/credentials/github-copilot.token.json。若请求未提供 githubToken，proxy 会尝试读取 $OPENCLAW_STATE_DIR/agents/main/agent/auth-profiles.json 中第一个 provider === "github-copilot" 的明文 token。
- 请求体示例：{ "githubToken": "ghp_xxx" }
- 返回示例：{ "token": "<opaque>", "expiresAt": 1710000000000, "source": "fetched:...", "baseUrl": "https://api.individual.githubcopilot.com" }

4) POST /proxy
- 功能：通用转发接口。请求体说明：
  {
    "githubToken"?: string,    // 可选，覆盖 auth-profiles
    "method": "POST",        // 可选，默认 POST
    "path": "/chat/completions", // 要转发到 Copilot baseUrl 的路径
    "headers"?: { ... },      // 可选，覆盖默认 headers
    "body"?: { ... }         // 请求的 JSON 负载，proxy 会确保把 model 字段写入 body
  }
- proxy 会：
  - 调用 /resolve-token 逻辑获取 runtime token，注入 Authorization: Bearer <copilot-runtime-token>
  - 注入默认 Copilot headers（可被请求中的 headers 覆盖）：
    - Editor-Version: (默认 vscode/1.96.2)
    - User-Agent: (默认 GitHubCopilotChat/0.26.7)
    - X-Github-Api-Version: 2025-04-01
  - 确保 JSON body 中包含 model 字段（从 body、query、或 top-level model 字段读取，或使用代理默认值）

- curl 调用示例：
  curl -X POST http://127.0.0.1:1111/proxy \
    -H "Content-Type: application/json" \
    -d '{"method":"POST","path":"/chat/completions","body":{"model":"gpt-5-mini","messages":[{"role":"user","content":"Hello"}]}}'

- 使用 API key（若启用 PROXY_API_KEY）：
  curl -X POST http://127.0.0.1:1111/proxy \
    -H "Content-Type: application/json" \
    -H "x-api-key: secretkey" \
    -d '{...}'

5) /v1/* passthrough
- 功能：把收到的 /v1/* 请求尽量原样转发到 Copilot baseUrl 的相对路径（例如 /v1/responses -> https://api.../responses）。
- 适用于想以 OpenAI Responses 兼容接口直接发起请求的场景。proxy 会：
  - 优先保留传入的 headers（但会替换/注入 Authorization/editor-version/user-agent/x-github-api-version）
  - 若是有 body 的请求，会把 model 写入 JSON body（若已存在则保留或覆盖为上游指定的 modelToUse）

- 示例：把 OpenAI-style 请求直接发给 proxy：
  curl -X POST http://127.0.0.1:1111/v1/responses \
    -H "Content-Type: application/json" \
    -d '{"model":"gpt-5-mini","input":"Hello"}'

鉴权与安全
- PROXY_API_KEY：在容器启动时设置 -e PROXY_API_KEY=secretkey，会要求客户端在 x-api-key header 或 ?api_key=secretkey 中提供相同值。
- 建议仅绑定本机（-p 127.0.0.1:1111:1111）并启用 API key 在需要远程访问时。

示例客户端代码

1) Node.js (fetch)

const fetch = require('node-fetch');

async function example(){
  const res = await fetch('http://127.0.0.1:1111/proxy', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'x-api-key': 'secretkey' },
    body: JSON.stringify({ method: 'POST', path: '/chat/completions', body: { model: 'gpt-5-mini', messages: [{ role: 'user', content: 'Hello' }] } })
  });
  console.log(await res.text());
}

2) Python (requests)

import requests
r = requests.post('http://127.0.0.1:1111/proxy', json={
  'method':'POST', 'path':'/chat/completions',
  'body':{ 'model':'gpt-5-mini','messages':[{'role':'user','content':'Hello'}] }
}, headers={'x-api-key':'secretkey'})
print(r.status_code, r.text)

3) OpenClaw / 其它程序（概念性说明）
- 任何能发出 HTTP 请求的程序都可以使用上述 /proxy 或 /v1/* 接口。
- 建议：
  - 如果程序希望以 OpenAI-兼容的方式调用 Copilot（例如直接调用 /v1/responses），用 /v1/* passthrough。
  - 如果程序希望以更细粒度控制 header/model 等，使用 /proxy 并在 body.headers 里覆盖。

容器运行示例
  docker run -d --name copilot-proxy -p 127.0.0.1:1111:1111 \
    -e OPENCLAW_STATE_DIR=/var/lib/openclaw-state \
    -v /var/lib/openclaw-state:/var/lib/openclaw-state \
    -e PROXY_API_KEY=secretkey \
    copilot-proxy-service:local

故障排查
- "githubToken required": 表示既未在请求中提供 githubToken，也没有在容器可见的 auth-profiles.json 中找到明文 token；请检查容器是否正确挂载了 OPENCLAW_STATE_DIR，或在请求中直接提供 githubToken，或设置 COPILOT_GITHUB_TOKEN 环境变量。
- 日志：容器启动时会打印当前生效的 Editor-Version、User-Agent、API_KEY 是否设置与 STATE_DIR 路径，便于排查。

注
- 本 README 以示例为主；生产环境请务必做好网络与凭据隔离。

---

Docker Hub 拉取并通过 Docker Compose 部署（新增）

我已在本目录生成 docker-compose.hub.yml，用于从 Docker Hub 拉取镜像并启动服务。文件路径： /opt/copilot-proxy-service/docker-compose.hub.yml

快速部署示例（在主机上执行）：

1) 拉取最新镜像（可选，docker compose up 会自动拉取）：
   docker pull kamilwang/copilot-proxy:latest

2) 使用 docker compose 启动服务：
   docker compose -f /opt/copilot-proxy-service/docker-compose.hub.yml up -d

3) 查看日志/健康检查：
   docker compose -f /opt/copilot-proxy-service/docker-compose.hub.yml logs -f
   curl -sS http://127.0.0.1:3000/health

持久化挂载与授权（重要）

- 挂载 OPENCLAW_STATE_DIR：
  本 compose 文件中已把 /var/lib/openclaw-state 挂载到容器的 /var/lib/openclaw-state（read-write）。此目录用于缓存 Copilot runtime token（github-copilot.token.json）与 auth-profiles。如果你已经有现成的 state 目录，请确保把其路径映射到宿主的 /var/lib/openclaw-state 或修改 compose 的卷路径。

- GitHub 登录授权（两种方式）：
  1) 设备/交互登录（推荐在带浏览器的机器上完成）：
     - 启动容器后，使用 proxy 的 /auth/login 与 /auth/poll 端点进行设备流（如果代理实现了相应 UI/端点）。这会在 /var/lib/openclaw-state/credentials 中写入 token。
  2) 直接提供 GitHub Token：
     - 在 docker-compose.hub.yml 中设置环境变量 COPILOT_GITHUB_TOKEN=ghp_xxx（注意：请仅在受信任环境中使用）。
     - 或事先把解析得到的 github-copilot.token.json 放在宿主的 /var/lib/openclaw-state/credentials/ 路径下，容器会读取并使用它。

- API Key：
  - 若你想限制对这个代理的访问，建议在环境变量中设置 PROXY_API_KEY 并在客户端请求时通过 x-api-key header 传入该值。

安全建议

- 默认为本机监听（127.0.0.1:3000），若需要外网访问，请放在内网或在前端加反向代理与访问控制。
- 镜像和配置中不要把实际的 secrets（GitHub tokens / API keys）写入仓库或长期文件。
