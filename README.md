# 智旅通前端

这是一个基于 `Vue 3 + Vite` 的聊天前端。

## 本地开发

安装依赖后运行：

```bash
npm install
npm run dev
```

本地开发默认请求 `/api/chat`，再由 [vite.config.js](/d:/python/AI大模型RAG与智能体开发/Agent小项目前端/demo1/vite.config.js:1) 代理到本地后端 `http://localhost:8000/chat`。

## Vercel 前端联调本地后端

直接把部署在 Vercel 的前端页面请求到 `localhost` 是做不到的。

原因很简单：

1. Vercel 的 `rewrites` 运行在 Vercel 服务器上，不是在你的电脑上。
2. `127.0.0.1` 对 Vercel 来说是它自己的机器，不是你本地的后端。
3. 浏览器访问的是公网页面，所以后端也必须有一个公网可访问的地址，最好是 `https`。

可行方案是：

1. 本地启动后端，例如监听 `http://127.0.0.1:8000`
2. 用隧道工具把本地端口暴露到公网，例如 `ngrok`、`Cloudflare Tunnel`、`localtunnel`
3. 在 Vercel 项目里配置环境变量 `VITE_API_BASE_URL`
4. 把它设置成你的公网后端地址，例如 `https://abc123.ngrok-free.app`
5. 重新部署前端

前端会自动把聊天接口拼成：

```text
${VITE_API_BASE_URL}/chat
```

例如：

```text
https://abc123.ngrok-free.app/chat
```

如果没有配置 `VITE_API_BASE_URL`，前端会退回到默认的 `/api/chat`。

## 推荐联调方式

如果你想继续保持“前端在 Vercel，后端在本机”的测试流程，推荐这样做：

```bash
cloudflared tunnel --url http://127.0.0.1:8000
```

或者：

```bash
ngrok http 8000
```

拿到公网 `https` 地址后：

1. 打开 Vercel 项目设置
2. 添加环境变量 `VITE_API_BASE_URL`
3. 重新部署
4. 确认你的后端允许来自 Vercel 域名的 CORS

## 后端需要注意

如果前端直接请求隧道地址，后端需要允许跨域，至少包括：

1. `Access-Control-Allow-Origin`
2. `Access-Control-Allow-Headers: Content-Type`
3. `Access-Control-Allow-Methods: POST, OPTIONS`

如果你的后端走的是流式返回，建议继续使用：

```text
Content-Type: text/event-stream
```
