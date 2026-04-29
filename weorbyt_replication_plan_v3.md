# Orbyt 风格 Agent 平台复刻方案 V3（修订版）
> 基于实地调研 + OpenSandbox 源码核查 · Python 后端 · 阿里云基建适配  
> 版本：2026-04-29 · 作者：内部技术调研

---

## ⚠️ 前版本 Bug 清单（已全部修正）

| # | 错误描述 | 前版本说法 | 修正后 |
|---|---------|-----------|--------|
| 1 | **SDK 包名混用** | `e2b.Sandbox.create()` | `from opensandbox import Sandbox` |
| 2 | **预览 URL 生成方式错误** | `sbx.get_hostname(8080)` (E2B API) | Ingress Gateway 路由：`{sandbox_id}-8080.{domain}` |
| 3 | **续约 API 不存在** | `sbx.keep_alive()` | 通过 Ingress 的 `renew-intent` Redis 事件机制续约（OSEP-0009） |
| 4 | **OSSFS 平台限制漏写** | "直接挂载 OSS" | 要求宿主机 Linux + ossfs 已安装 + FUSE 启用；Docker on Windows 不支持 |
| 5 | **OSSFS 认证方式受限漏写** | "OSS AK 直接配置" | 当前只支持 inline AK/SK，不支持 RAM Role / STS Token |
| 6 | **Ingress 不是自动 SLB** | "自动配置 SLB 规则" | Ingress Gateway 是独立 Go 服务，需单独部署；K8s 模式需手动配置 Ingress CR 绑定 SLB |
| 7 | **冷启动时间误导** | "~150ms 秒级拉起" | 150ms 是 E2B MicroVM 数据；OpenSandbox Docker 约 2-5s；K8s 含 Pod 调度延迟 |
| 8 | **async API 未加 await** | `sandbox.commands.run(...)` | OpenSandbox Python SDK 全异步，必须 `await sandbox.commands.run(...)` |
| 9 | **ZIP 打包逻辑错误** | `shutil.make_archive` 本地打包 | 文件在沙箱内，需先 `await sandbox.files.read_file(path)` 逐一取回再打包 |
| 10 | **Vercel AI SDK 与 Python 不兼容** | "用 Vercel AI SDK 处理流" | Vercel AI SDK 是 Node.js 库；Python FastAPI 应使用 `StreamingResponse` + SSE |
| 11 | **Ingress Docker/K8s 模式差异未说明** | 统一描述为 Ingress | Docker 模式读 `sandbox.opensandbox.io/endpoints` annotation；K8s 读 `status.serviceFQDN`；配置完全不同 |
| 12 | **`react-arborist` 未验证可用性** | 直接推荐 | 该库长期未更新，建议用 radix-ui/react-collapsible 或 shadcn/ui 自实现树组件 |

---

## 一、系统总体架构

```
┌───────────────────────────────────────────────────────┐
│                     用户浏览器                         │
│   Left Panel: Task List   Center: Code/Preview/Terminal│
│   (React + Tailwind)      (Monaco + iframe + xterm.js) │
└──────────────────┬────────────────────────────────────┘
                   │ SSE (text/event-stream)
┌──────────────────▼────────────────────────────────────┐
│              FastAPI 后端 (Python 3.11+)               │
│  POST /api/task       - 创建任务，返回 task_id         │
│  GET  /api/stream/{id} - SSE 推送执行状态              │
│  GET  /api/download/{id} - 下载 ZIP（从沙箱取回文件）  │
└──────┬───────────────────┬───────────────────────────┘
       │                   │
┌──────▼──────┐   ┌────────▼──────────────────────────┐
│  LangGraph  │   │  OpenSandbox Server                │
│  Agent Loop │──▶│  (Docker runtime / ACK K8s)        │
│  (Python)   │   │  commands.run() / files.*          │
└─────────────┘   └───────────┬────────────────────────┘
                              │
                  ┌───────────▼───────────────────────┐
                  │  Ingress Gateway (独立 Go 服务)    │
                  │  Header 模式：{id}-8080.domain     │
                  │  URI 模式：/ingress/{id}/8080/     │
                  │  Auto-renew via Redis (OSEP-0009)  │
                  └───────────┬───────────────────────┘
                              │
                  ┌───────────▼───────────────────────┐
                  │  阿里云基建                         │
                  │  OSS (OSSFS 挂载 /workspace)        │
                  │  SLB + ALB Ingress Controller       │
                  │  ACK (K8s 调度沙箱 Pod)             │
                  └───────────────────────────────────┘
```

---

## 二、后端核心实现（Python）

### 2.1 OpenSandbox 正确 API 用法

```python
# pip install opensandbox opensandbox-code-interpreter langgraph fastapi uvicorn

import asyncio, os, zipfile, tempfile
from datetime import timedelta
from opensandbox import Sandbox
from opensandbox.models import WriteEntry

SANDBOX_DOMAIN   = os.getenv("SANDBOX_DOMAIN", "localhost:8080")
SANDBOX_PROTOCOL = os.getenv("SANDBOX_PROTOCOL", "http")
INGRESS_DOMAIN   = os.getenv("INGRESS_DOMAIN", "preview.yourdomain.com")


async def create_sandbox(session_id: str) -> Sandbox:
    """创建沙箱。生产环境可加 volumes= 参数挂载 OSS（见第五节约束）"""
    return await Sandbox.create(
        image="python:3.12",           # 或自定义镜像
        timeout=timedelta(minutes=30),
        env={"SESSION_ID": session_id},
        domain=SANDBOX_DOMAIN,
        protocol=SANDBOX_PROTOCOL,
    )


async def write_file(sandbox: Sandbox, path: str, content: str) -> None:
    """写文件到沙箱内"""
    await sandbox.files.write_files([
        WriteEntry(path=path, data=content, mode=0o644)
    ])


async def run_command(sandbox: Sandbox, cmd: str) -> str:
    """执行命令，全程 await（OpenSandbox Python SDK 全异步）"""
    result = await sandbox.commands.run(cmd)
    stdout = result.logs.stdout
    return stdout[0].text if stdout else ""


async def package_session_files(sandbox: Sandbox, paths: list[str]) -> bytes:
    """
    ✅ 正确打包方式：文件在沙箱内，需逐一 read_file 取回再压缩
    ❌ 错误方式：shutil.make_archive（只能打包本地文件）
    """
    buf = tempfile.NamedTemporaryFile(suffix=".zip", delete=False)
    with zipfile.ZipFile(buf.name, "w", zipfile.ZIP_DEFLATED) as zf:
        for fpath in paths:
            content = await sandbox.files.read_file(fpath)
            zf.writestr(fpath.lstrip("/"), content)
    with open(buf.name, "rb") as f:
        return f.read()


def get_preview_url(sandbox_id: str, port: int = 8080) -> str:
    """
    ✅ 预览 URL = {sandbox_id}-{port}.{INGRESS_DOMAIN}
    依赖：Ingress Gateway 已部署 + 通配符 DNS *.INGRESS_DOMAIN → Ingress Service
    ❌ E2B 的 get_hostname() 在 OpenSandbox 不存在
    """
    return f"https://{sandbox_id}-{port}.{INGRESS_DOMAIN}"
```

### 2.2 FastAPI SSE 流式推送

```python
import json
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from typing import AsyncGenerator

app = FastAPI()


async def event_generator(task_id: str) -> AsyncGenerator[str, None]:
    """从 Redis 队列消费 Agent 执行事件，推送为 SSE"""
    queue = get_task_queue(task_id)   # 由 LangGraph Node 写入 Redis
    while True:
        try:
            event = await asyncio.wait_for(queue.get(), timeout=60.0)
        except asyncio.TimeoutError:
            yield "event: ping\ndata: {}\n\n"
            continue
        yield f"data: {json.dumps(event, ensure_ascii=False)}\n\n"
        if event.get("type") == "done":
            break


@app.get("/api/stream/{task_id}")
async def stream_task(task_id: str):
    """
    ✅ Python 后端用 StreamingResponse + SSE
    ❌ Vercel AI SDK 是 Node.js 库，Python 项目不可用
    """
    return StreamingResponse(
        event_generator(task_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",   # 阻止 Nginx 缓冲 SSE 数据
            "Connection": "keep-alive",
        },
    )
```

### 2.3 LangGraph Agent 编排

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class AgentState(TypedDict):
    task: str
    sandbox_id: str
    file_map: dict[str, str]   # {path: content}
    steps: List[dict]
    error: str | None


async def architect_node(state: AgentState) -> AgentState:
    """调用 LLM 将任务分解为子步骤，输出 MANIFEST"""
    ...

async def coder_node(state: AgentState) -> AgentState:
    """生成代码文件并写入沙箱，推送 write_file 事件给前端更新 VFS"""
    sandbox = await get_sandbox(state["sandbox_id"])
    for path, code in state["file_map"].items():
        await write_file(sandbox, path, code)
        await push_event(state["sandbox_id"], {"type": "write_file", "path": path, "content": code})
    ...

async def qa_node(state: AgentState) -> AgentState:
    """用 Playwright 在沙箱内截图做视觉验证（需镜像包含 playwright）"""
    ...

graph = StateGraph(AgentState)
graph.add_node("architect", architect_node)
graph.add_node("coder", coder_node)
graph.add_node("qa", qa_node)
graph.set_entry_point("architect")
graph.add_edge("architect", "coder")
graph.add_edge("coder", "qa")
graph.add_edge("qa", END)
agent = graph.compile()
```

---

## 三、Ingress Gateway 部署（预览功能核心）

> **关键认知**：Ingress Gateway 是 OpenSandbox 独立的 Go 服务，**不是** K8s Ingress Controller，需单独部署后再用 K8s Ingress 或 SLB 把流量转进来。

### 3.1 Docker 本地模式

```bash
# Docker 模式：Ingress 从 BatchSandbox annotation 读取 Pod endpoints
docker run -d -p 28888:28888 opensandbox/ingress:latest \
  --provider-type batchsandbox \
  --mode header \
  --port 28888

# 请求示例（通过 Host header 路由到指定沙箱端口）
curl -H "Host: {sandbox_id}-8080.localhost" http://localhost:28888/
```

### 3.2 ACK K8s 生产模式

```yaml
# Step 1：部署 Ingress Gateway（独立 Deployment）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opensandbox-ingress
  namespace: opensandbox
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: ingress
        image: opensandbox/ingress:latest
        args:
        - --provider-type=agent-sandbox   # K8s 模式读 status.serviceFQDN
        - --mode=header
        - --port=8080
        - --renew-intent-enabled          # ✅ 用 Redis 续约，替代不存在的 keep_alive()
        - --renew-intent-redis-dsn=redis://redis:6379/0
        - --renew-intent-min-interval=300 # 最短续约间隔 300s（OSEP-0009 限制）
---
# Step 2：用 K8s Ingress 绑定阿里云 ALB，把通配符域名流量打到 Gateway
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "alb"  # 阿里云 ALB Ingress Controller
spec:
  rules:
  - host: "*.preview.yourdomain.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: opensandbox-ingress
            port:
              number: 8080
```

---

## 四、前端实现要点

### 4.1 文件树组件

```tsx
// ❌ 不推荐：react-arborist（长期未维护，存在兼容性风险）
// ✅ 推荐：基于 @radix-ui/react-collapsible 自实现，或 shadcn/ui Tree primitive

// SSE 实时同步 VFS
const [vfs, setVfs] = useState<Map<string, string>>(new Map())

useEffect(() => {
  const es = new EventSource(`/api/stream/${taskId}`)
  es.onmessage = (e) => {
    const event = JSON.parse(e.data)
    if (event.type === "write_file") {
      setVfs(prev => new Map(prev).set(event.path, event.content))
    }
  }
  return () => es.close()
}, [taskId])
```

### 4.2 Preview iframe

```tsx
// 预览 URL：https://{sandbox_id}-8080.preview.yourdomain.com
// 前置条件：
//   1. Ingress Gateway 已部署
//   2. 通配符 DNS *.preview.yourdomain.com → SLB
//   3. 通配符 TLS 证书（Let's Encrypt 或阿里云证书服务）
//   4. 沙箱内 HTTP Server 响应头设置 X-Frame-Options: ALLOWALL

const previewUrl = `https://${sandboxId}-8080.${INGRESS_DOMAIN}`

<iframe
  src={previewUrl}
  sandbox="allow-scripts allow-same-origin allow-forms"
  className="w-full h-full border-0"
/>
```

---

## 五、OSS 工作区持久化（含前置约束）

```python
from opensandbox.models.sandboxes import OSSFS, Volume

# ⚠️ 使用前必须确认的前置条件（缺一不可）：
# 1. OpenSandbox Server 运行在 Linux 宿主机（Windows 不支持）
# 2. 宿主机已安装 ossfs（apt/yum install ossfs）
# 3. 宿主机已加载 fuse 模块（modprobe fuse）
# 4. Server toml 配置了 storage.ossfs_mount_root（默认 /mnt/ossfs）
#
# ⚠️ 当前认证限制：只支持 inline AK/SK，不支持 STS / RAM Role
# 安全措施：使用 K8s Secret 注入，赋予最小权限子账号

volume = Volume(
    name="workspace",
    ossfs=OSSFS(
        bucket=os.getenv("OSS_BUCKET"),
        endpoint=f"oss-cn-{os.getenv('OSS_REGION')}.aliyuncs.com",
        accessKeyId=os.getenv("OSS_AK"),       # K8s Secret 注入
        accessKeySecret=os.getenv("OSS_SK"),   # K8s Secret 注入
        # version 默认 "2.0"（使用 ossfs2 命令），也可显式指定 "1.0"
    ),
    mountPath=f"/workspace/{session_id}",
    readOnly=False,
)
```

---

## 六、分阶段 Roadmap

| 阶段 | 目标 | 关键技术 | 预估工时 |
|------|------|---------|---------|
| **MVP** | 对话框 + 脚本执行 + 静态结果（PNG） | FastAPI + OpenSandbox Docker | 1 周 |
| **IDE** | Monaco 编辑器 + 文件树 + SSE 同步 | VFS + SSE + Monaco | 2 周 |
| **Preview** | 沙箱 HTTP Server + iframe 实时预览 | Ingress Gateway + 通配符域名 + TLS | 1 周 |
| **生产化** | ACK 部署 + OSS 持久化 + Credit 系统 | ACK + ALB + Redis + PostgreSQL | 2 周 |
| **增强** | Playwright 视觉校验 + 多模型切换 | Playwright-in-sandbox + 模型路由 | 1 周 |

---

## 七、风险与注意事项

> [!WARNING]
> **OSSFS 密钥安全**：当前版本强制 inline AK/SK 明文传入 API。生产环境必须使用专用子账号 + K8s Secret 管理，等 OpenSandbox 支持 STS 后迁移。

> [!WARNING]
> **沙箱续约**：`keep_alive()` 不存在。续约通过 Ingress Gateway 的 `renew-intent` Redis 机制（OSEP-0009），最短 300s 间隔，最长 86400s。未访问的沙箱到 timeout 后自动销毁。

> [!CAUTION]
> **Docker 模式不适合生产**：单机无法横向扩展，宿主机重启沙箱消失，OSSFS 挂载也会失效。生产环境必须使用 ACK K8s 模式。

> [!NOTE]
> **地图任务最优方案**：Orbyt 实测使用静态 PNG（Matplotlib/Folium）。进阶方案：让 Agent 生成 Folium HTML，沙箱起 `python -m http.server 8080`，前端 iframe 渲染，可获得可缩放交互地图，体验超越 Orbyt 现状。
