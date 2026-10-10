# 电商智能分析助手 · 前端

配套 [ecommerce-deep-agent](https://github.com/Supereggcat/ecommerce-deep-agent)（DeepAgents 多智能体后端）的对话式前端：提交分析任务，实时查看执行过程，并下载生成的报告。

## 功能

- **对话式交互**：提问即触发后端多智能体协作，回答以 Markdown 渲染（含表格、代码块）。
- **执行过程可视化**：实时展示调用了哪个子智能体、执行了哪个工具，每步参数按需展开。
- **会话文件面板**：自动列出本次任务产出的文档，支持一键下载。
- **文件上传**：可随提问一起上传 Markdown / Word / PDF / Excel 供智能体分析。
- **明暗双主题**：一键切换并记忆，首次访问跟随系统偏好，首屏无闪烁。
- **响应式布局**：窄屏下文件面板自动切换为抽屉。

## 技术栈

Vue 3（`<script setup>` + TypeScript）· Vite · axios · marked

## 快速开始

```bash
npm install
cp .env.example .env      # 按需修改后端地址
npm run dev               # http://localhost:5173
```

> 需要先启动后端：`python -m uvicorn api.server:app --port 8000`

## 环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `VITE_API_BASE` | `http://localhost:8000` | 后端地址，WebSocket 地址会由它自动推导 |

## 与后端的接口约定

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/api/task` | 提交任务 `{ query, thread_id }` |
| `POST` | `/api/upload` | 上传文件（multipart：`files` + `thread_id`） |
| `GET` | `/api/files?path=...` | 列出会话产出文件 |
| `GET` | `/api/download?path=...` | 下载产出文件 |
| `WS` | `/ws/{thread_id}` | 实时事件推送 |

WebSocket 事件：

```json
{
  "type": "monitor_event",
  "event": "session_created | assistant_call | tool_start | task_result | error",
  "message": "人类可读的描述",
  "data": { },
  "timestamp": "2026-01-01T00:00:00"
}
```

## 目录结构

```
src/
├── App.vue      # 页面结构、状态与交互逻辑
├── main.ts      # 应用入口
└── style.css    # 设计令牌（明/暗主题）、基础样式、Markdown 排版
```

## 构建

```bash
npm run build      # vue-tsc 类型检查 + 产物构建到 dist/
npm run preview    # 本地预览构建产物
```

