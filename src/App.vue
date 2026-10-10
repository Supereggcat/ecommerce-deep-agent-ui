<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import axios from 'axios'
import { marked } from 'marked'

/* 接口地址：可通过 .env 中的 VITE_API_BASE 覆盖 */
const API_BASE = (import.meta.env.VITE_API_BASE as string | undefined) || 'http://localhost:8000'
const WS_BASE = API_BASE.replace(/^http/, 'ws')

interface LogItem {
  type: string
  title: string
  details?: unknown
  timestamp: string
}

interface FileItem {
  name: string
  path: string
  url: string
  size?: number
}

interface Message {
  role: 'user' | 'ai' | 'system'
  content: string
  logs?: LogItem[]
  files?: FileItem[]
  timestamp?: number
}

/* ---------------------------- 状态 ---------------------------- */
const inputQuery = ref('')
const messages = ref<Message[]>([])
const status = ref<'idle' | 'running'>('idle')
const socket = ref<WebSocket | null>(null)
const currentThreadId = ref(crypto.randomUUID())
const currentSessionPath = ref('')
const fileList = ref<FileItem[]>([])
const isSidebarOpen = ref(false)
const copiedIndex = ref<number | null>(null)
const selectedFiles = ref<File[]>([])
const fileInputRef = ref<HTMLInputElement | null>(null)
const textareaRef = ref<HTMLTextAreaElement | null>(null)
const messagesEndRef = ref<HTMLElement | null>(null)

const isWelcomeScreen = computed(() => messages.value.length === 0)
const isRunning = computed(() => status.value === 'running')
const sessionName = computed(
  () => currentSessionPath.value.split(/[\\/]/).filter(Boolean).pop() ?? '',
)

/* ---------------------------- 主题 ---------------------------- */
const isDark = ref(document.documentElement.classList.contains('dark'))

const toggleTheme = () => {
  isDark.value = !isDark.value
  document.documentElement.classList.toggle('dark', isDark.value)
  try {
    localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
  } catch {
    /* 隐私模式下写入可能失败，忽略 */
  }
}

/* ---------------------------- 欢迎页推荐 ---------------------------- */
const suggestions = [
  {
    icon: '📈',
    title: '行业趋势洞察',
    desc: '联网检索并输出趋势报告',
    prompt: '请搜索电商行业最新的发展趋势，并生成一份 Markdown 分析报告',
  },
  {
    icon: '🔍',
    title: '竞品价格调研',
    desc: '对比同类商品的价格与卖点',
    prompt: '搜索并对比主流电商平台同类商品的价格与卖点，生成一份竞品调研报告',
  },
  {
    icon: '📊',
    title: '商品销售分析',
    desc: '基于业务库做数据统计',
    prompt: '统计数据库中各商品类目的销量与销售额，并给出分析结论',
  },
  {
    icon: '📚',
    title: '内部知识问答',
    desc: '检索企业内部知识库',
    prompt: '请检索内部知识库，说明商品售后与退换货的政策规定',
  },
]

const useSuggestion = (prompt: string) => {
  inputQuery.value = prompt
  void sendMessage()
}

/* ---------------------------- 工具函数 ---------------------------- */
const scrollToBottom = async () => {
  await nextTick()
  messagesEndRef.value?.scrollIntoView({ behavior: 'smooth', block: 'end' })
}

const formatSize = (bytes?: number) => {
  if (bytes === undefined || bytes === null) return ''
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`
}

const renderMarkdown = (text: string) => marked(text) as string

const logEmoji = (type: string) => {
  if (type === 'agent') return '🤖'
  if (type === 'error') return '⚠️'
  if (type === 'success') return '✅'
  return '🔧'
}

const copyMessage = async (text: string, index: number) => {
  try {
    await navigator.clipboard.writeText(text)
    copiedIndex.value = index
    setTimeout(() => {
      if (copiedIndex.value === index) copiedIndex.value = null
    }, 1600)
  } catch {
    /* 剪贴板不可用时忽略 */
  }
}

const autoGrow = () => {
  const el = textareaRef.value
  if (!el) return
  el.style.height = 'auto'
  el.style.height = `${Math.min(el.scrollHeight, 200)}px`
}

watch(inputQuery, () => void nextTick(autoGrow))

/* ---------------------------- 文件上传 ---------------------------- */
const triggerFileUpload = () => fileInputRef.value?.click()

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files.length > 0) {
    selectedFiles.value = [...selectedFiles.value, ...Array.from(target.files)]
    target.value = ''
  }
}

const removeFile = (index: number) => selectedFiles.value.splice(index, 1)

/* ---------------------------- 文件列表 ---------------------------- */
const fetchFiles = async () => {
  if (!currentSessionPath.value) return
  try {
    const res = await axios.get(`${API_BASE}/api/files`, {
      params: { path: currentSessionPath.value },
    })
    const incoming: FileItem[] = (res.data.files ?? []).map((f: any) => ({
      name: f.name,
      path: f.path,
      size: f.size,
      url: `${API_BASE}/api/download?path=${encodeURIComponent(f.path)}`,
    }))

    // 新增的文件挂到最近一条 AI 消息上，方便直接在对话里下载
    const known = new Set(fileList.value.map((f) => f.path))
    const fresh = incoming.filter((f) => !known.has(f.path))
    fileList.value = incoming

    if (fresh.length > 0) {
      const lastAiMsg = [...messages.value].reverse().find((m) => m.role === 'ai')
      if (lastAiMsg) lastAiMsg.files = [...(lastAiMsg.files ?? []), ...fresh]
    }
  } catch (e) {
    console.error('获取文件列表失败', e)
  }
}

/* ---------------------------- WebSocket ---------------------------- */
let unmounted = false

const connectWebSocket = () => {
  const ws = new WebSocket(`${WS_BASE}/ws/${currentThreadId.value}`)

  ws.onmessage = (event) => {
    try {
      handleSocketMessage(JSON.parse(event.data))
    } catch (e) {
      console.error('WebSocket 消息解析失败', e)
    }
  }

  ws.onclose = () => {
    if (unmounted) return
    setTimeout(connectWebSocket, 3000)
  }

  socket.value = ws
}

const handleSocketMessage = (data: any) => {
  const { type, event, message, data: eventData } = data
  if (type === 'pong') return

  const lastAiMsg = [...messages.value].reverse().find((m) => m.role === 'ai')

  if (event === 'session_created') {
    currentSessionPath.value = eventData.path
    isSidebarOpen.value = true
    void fetchFiles()
  } else if (event === 'tool_start') {
    if (currentSessionPath.value) {
      void fetchFiles()
      setTimeout(() => void fetchFiles(), 2000)
    }
    if (lastAiMsg) {
      if (!lastAiMsg.logs) lastAiMsg.logs = []
      lastAiMsg.logs.push({
        type: 'tool',
        title: `调用工具 ${eventData.tool_name}`,
        details: eventData.args,
        timestamp: new Date().toLocaleTimeString(),
      })
    }
  } else if (event === 'assistant_call') {
    if (currentSessionPath.value) void fetchFiles()
    if (lastAiMsg) {
      if (!lastAiMsg.logs) lastAiMsg.logs = []
      lastAiMsg.logs.push({
        type: 'agent',
        title: `调用 ${eventData.assistant_name}`,
        details: eventData.args,
        timestamp: new Date().toLocaleTimeString(),
      })
    }
  } else if (event === 'task_result') {
    if (lastAiMsg) {
      lastAiMsg.content = eventData.result
    } else {
      messages.value.push({ role: 'ai', content: eventData.result, timestamp: Date.now() })
    }
    status.value = 'idle'
    void fetchFiles()
  } else if (event === 'error') {
    messages.value.push({
      role: 'system',
      content: `执行出错：${message}`,
      timestamp: Date.now(),
    })
    status.value = 'idle'
  }

  void scrollToBottom()
}

/* ---------------------------- 发送 ---------------------------- */
const sendMessage = async () => {
  if ((!inputQuery.value.trim() && selectedFiles.value.length === 0) || status.value === 'running') {
    return
  }

  const query = inputQuery.value
  inputQuery.value = ''
  status.value = 'running'

  messages.value.push({ role: 'user', content: query, timestamp: Date.now() })
  messages.value.push({ role: 'ai', content: '', logs: [], files: [], timestamp: Date.now() })
  void scrollToBottom()

  const lastAiMsg = messages.value[messages.value.length - 1]

  if (selectedFiles.value.length > 0) {
    const filesToUpload = [...selectedFiles.value]
    if (lastAiMsg) {
      lastAiMsg.logs?.push({
        type: 'info',
        title: `上传 ${filesToUpload.length} 个文件`,
        details: filesToUpload.map((f) => ({ name: f.name, size: f.size })),
        timestamp: new Date().toLocaleTimeString(),
      })
    }

    try {
      const formData = new FormData()
      formData.append('thread_id', currentThreadId.value)
      filesToUpload.forEach((file) => formData.append('files', file))

      await axios.post(`${API_BASE}/api/upload`, formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
      })
      selectedFiles.value = []
      lastAiMsg?.logs?.push({
        type: 'success',
        title: '文件上传成功',
        timestamp: new Date().toLocaleTimeString(),
      })
    } catch (e) {
      lastAiMsg?.logs?.push({
        type: 'error',
        title: '文件上传失败',
        details: e instanceof Error ? e.message : String(e),
        timestamp: new Date().toLocaleTimeString(),
      })
    }
  }

  try {
    const payload: Record<string, string> = { query }
    if (currentThreadId.value) payload.thread_id = currentThreadId.value

    const res = await axios.post(`${API_BASE}/api/task`, payload)
    if (res.data?.thread_id) currentThreadId.value = res.data.thread_id
  } catch (error: any) {
    let errorMsg = '请求发送失败'
    if (error?.message) errorMsg += `（${error.message}）`
    if (error?.response?.data) errorMsg += ` 服务端返回：${JSON.stringify(error.response.data)}`

    messages.value.push({ role: 'system', content: errorMsg, timestamp: Date.now() })
    status.value = 'idle'
  }

  await nextTick()
  autoGrow()
}

onMounted(() => {
  connectWebSocket()
  void nextTick(autoGrow)
})

onBeforeUnmount(() => {
  unmounted = true
  socket.value?.close()
})
</script>

<template>
  <div class="app-shell">
    <!-- 顶栏 -->
    <header class="topbar">
      <div class="brand">
        <span class="brand-mark">
          <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
            <path
              d="M12 2.5 14.2 9l6.5 2.2-6.5 2.2L12 20l-2.2-6.6L3.3 11.2 9.8 9 12 2.5Z"
              fill="currentColor"
            />
          </svg>
        </span>
        <div class="brand-text">
          <h1>电商智能分析助手</h1>
          <p>DeepAgents · 多智能体协作</p>
        </div>
      </div>

      <div class="topbar-actions">
        <button
          class="icon-btn"
          :title="isDark ? '切换到浅色模式' : '切换到深色模式'"
          aria-label="切换主题"
          @click="toggleTheme"
        >
          <svg v-if="isDark" viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
            <circle cx="12" cy="12" r="4.2" />
            <path d="M12 2.5v2M12 19.5v2M2.5 12h2M19.5 12h2M5.2 5.2l1.4 1.4M17.4 17.4l1.4 1.4M18.8 5.2l-1.4 1.4M6.6 17.4l-1.4 1.4" />
          </svg>
          <svg v-else viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M20.5 14.5A8.5 8.5 0 1 1 9.5 3.5a7 7 0 0 0 11 11Z" />
          </svg>
        </button>

        <button
          class="icon-btn"
          :class="{ active: isSidebarOpen }"
          title="会话文件"
          aria-label="会话文件"
          @click="isSidebarOpen = !isSidebarOpen"
        >
          <svg viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M4 6.5A2.5 2.5 0 0 1 6.5 4h3.2l1.8 2.2h6A2.5 2.5 0 0 1 20 8.7v8.8a2.5 2.5 0 0 1-2.5 2.5h-11A2.5 2.5 0 0 1 4 17.5v-11Z" />
          </svg>
        </button>
      </div>
    </header>

    <div class="body">
      <main class="chat-pane">
        <!-- 欢迎页 -->
        <section v-if="isWelcomeScreen" class="welcome">
          <div class="welcome-hero">
            <span class="hero-mark">
              <svg viewBox="0 0 24 24" width="26" height="26" aria-hidden="true">
                <path
                  d="M12 2.5 14.2 9l6.5 2.2-6.5 2.2L12 20l-2.2-6.6L3.3 11.2 9.8 9 12 2.5Z"
                  fill="currentColor"
                />
              </svg>
            </span>
            <h2>今天想分析点什么？</h2>
            <p>我会自动规划任务、调度搜索 / 数据库 / 知识库助手，并输出可下载的分析报告</p>
          </div>

          <div class="suggestion-grid">
            <button
              v-for="item in suggestions"
              :key="item.title"
              class="suggestion-card"
              type="button"
              @click="useSuggestion(item.prompt)"
            >
              <span class="suggestion-icon">{{ item.icon }}</span>
              <span class="suggestion-text">
                <strong>{{ item.title }}</strong>
                <small>{{ item.desc }}</small>
              </span>
              <svg class="suggestion-arrow" viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M5 12h13M13 6l6 6-6 6" />
              </svg>
            </button>
          </div>
        </section>

        <!-- 对话区 -->
        <section v-else class="chat-scroll">
          <div class="chat-container">
            <div v-for="(msg, index) in messages" :key="index" class="message" :class="msg.role">
              <!-- 用户消息 -->
              <div v-if="msg.role === 'user'" class="bubble-user">{{ msg.content }}</div>

              <!-- 助手消息 -->
              <template v-else-if="msg.role === 'ai'">
                <div class="avatar" aria-hidden="true">
                  <svg viewBox="0 0 24 24" width="16" height="16">
                    <path
                      d="M12 2.5 14.2 9l6.5 2.2-6.5 2.2L12 20l-2.2-6.6L3.3 11.2 9.8 9 12 2.5Z"
                      fill="currentColor"
                    />
                  </svg>
                </div>

                <div class="ai-body">
                  <!-- 执行过程 -->
                  <details v-if="msg.logs && msg.logs.length" class="process" open>
                    <summary>
                      <span v-if="isRunning && index === messages.length - 1" class="spinner" />
                      <span v-else class="process-done">✓</span>
                      <span class="process-label">执行过程</span>
                      <span class="process-count">{{ msg.logs.length }} 步</span>
                      <svg class="chevron" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="m6 9 6 6 6-6" />
                      </svg>
                    </summary>

                    <ol class="process-list">
                      <li v-for="(log, i) in msg.logs" :key="i" class="process-item">
                        <span class="process-dot" />
                        <div class="process-content">
                          <details v-if="log.details" class="step">
                            <summary class="process-title">
                              <span class="process-emoji">{{ logEmoji(log.type) }}</span>
                              <span class="step-name">{{ log.title }}</span>
                              <svg class="step-chevron" viewBox="0 0 24 24" width="12" height="12" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                                <path d="m6 9 6 6 6-6" />
                              </svg>
                            </summary>
                            <pre class="process-details">{{ JSON.stringify(log.details, null, 2) }}</pre>
                          </details>
                          <div v-else class="process-title">
                            <span class="process-emoji">{{ logEmoji(log.type) }}</span>
                            <span class="step-name">{{ log.title }}</span>
                          </div>
                        </div>
                      </li>
                    </ol>
                  </details>

                  <!-- 正文 -->
                  <div v-if="!msg.content" class="thinking">
                    <i /><i /><i />
                  </div>
                  <div v-else class="markdown-body" v-html="renderMarkdown(msg.content)" />

                  <!-- 产出文件 -->
                  <div v-if="msg.files && msg.files.length" class="file-cards">
                    <a
                      v-for="file in msg.files"
                      :key="file.path"
                      class="file-card"
                      :href="file.url"
                      target="_blank"
                      rel="noopener"
                      download
                    >
                      <span class="file-card-icon">📄</span>
                      <span class="file-card-text">
                        <strong>{{ file.name }}</strong>
                        <small>{{ formatSize(file.size) || '点击下载' }}</small>
                      </span>
                      <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M12 4v11M7.5 10.5 12 15l4.5-4.5M5 19.5h14" />
                      </svg>
                    </a>
                  </div>

                  <!-- 操作 -->
                  <div v-if="msg.content" class="msg-actions">
                    <button class="ghost-btn" type="button" @click="copyMessage(msg.content, index)">
                      <svg viewBox="0 0 24 24" width="13" height="13" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <rect x="9" y="9" width="11" height="11" rx="2" />
                        <path d="M5 15V5a2 2 0 0 1 2-2h8" />
                      </svg>
                      {{ copiedIndex === index ? '已复制' : '复制' }}
                    </button>
                  </div>
                </div>
              </template>

              <!-- 系统提示 -->
              <div v-else class="alert">{{ msg.content }}</div>
            </div>

            <div ref="messagesEndRef" />
          </div>
        </section>

        <!-- 输入区 -->
        <footer class="composer">
          <div v-if="selectedFiles.length" class="chips">
            <span v-for="(file, i) in selectedFiles" :key="i" class="chip">
              <span>📎</span>
              <span class="chip-name">{{ file.name }}</span>
              <button class="chip-close" type="button" aria-label="移除文件" @click="removeFile(i)">×</button>
            </span>
          </div>

          <div class="composer-box" :class="{ running: isRunning }">
            <input ref="fileInputRef" class="hidden-input" type="file" multiple @change="handleFileChange" />

            <button
              class="icon-btn attach"
              type="button"
              title="上传文件"
              aria-label="上传文件"
              :disabled="isRunning"
              @click="triggerFileUpload"
            >
              <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M21.4 11.05 12.2 20.2a6 6 0 0 1-8.5-8.5l9.2-9.2a4 4 0 0 1 5.7 5.7l-9.2 9.2a2 2 0 0 1-2.8-2.8l8.5-8.5" />
              </svg>
            </button>

            <textarea
              ref="textareaRef"
              v-model="inputQuery"
              rows="1"
              placeholder="输入你的问题，Enter 发送 / Shift + Enter 换行"
              :disabled="isRunning"
              @keydown.enter.exact.prevent="sendMessage"
            />

            <button
              class="send-btn"
              type="button"
              aria-label="发送"
              :disabled="!inputQuery.trim() || isRunning"
              @click="sendMessage"
            >
              <svg v-if="!isRunning" viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2.1" stroke-linecap="round" stroke-linejoin="round">
                <path d="M12 19V5M6 11l6-6 6 6" />
              </svg>
              <span v-else class="spinner light" />
            </button>
          </div>

          <p class="composer-hint">
            <template v-if="isRunning">正在执行任务，请稍候…</template>
            <template v-else>内容由 AI 生成，请自行核对关键数据</template>
          </p>
        </footer>
      </main>

      <!-- 会话文件侧栏 -->
      <div v-if="isSidebarOpen" class="sidebar-backdrop" @click="isSidebarOpen = false" />
      <aside class="sidebar" :class="{ open: isSidebarOpen }" aria-label="会话文件">
        <div class="sidebar-head">
          <div class="sidebar-title">
            <h3>会话文件</h3>
            <p class="sidebar-sub" :title="currentSessionPath">{{ sessionName || '尚未开始' }}</p>
          </div>
          <div class="sidebar-actions">
            <button class="icon-btn sm" type="button" title="刷新" aria-label="刷新" @click="fetchFiles">
              <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20 11.5A8 8 0 1 0 18.4 16M20 5v6h-6" />
              </svg>
            </button>
            <button class="icon-btn sm" type="button" title="关闭" aria-label="关闭" @click="isSidebarOpen = false">
              <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
                <path d="M6 6l12 12M18 6 6 18" />
              </svg>
            </button>
          </div>
        </div>

        <div class="sidebar-body">
          <p v-if="!fileList.length" class="empty">
            还没有生成文件
            <small>任务产出的文档会自动出现在这里</small>
          </p>
          <ul v-else class="file-list">
            <li v-for="file in fileList" :key="file.path">
              <a class="file-row" :href="file.url" target="_blank" rel="noopener" download>
                <span class="file-row-icon">📄</span>
                <span class="file-row-text">
                  <span class="file-row-name" :title="file.name">{{ file.name }}</span>
                  <small>{{ formatSize(file.size) }}</small>
                </span>
              </a>
            </li>
          </ul>
        </div>
      </aside>
    </div>
  </div>
</template>

<style scoped>
.app-shell {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: hsl(var(--background));
}

/* ---------------------------- 顶栏 ---------------------------- */
.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  height: 60px;
  padding: 0 18px;
  border-bottom: 1px solid hsl(var(--border));
  background: hsl(var(--background) / 0.8);
  backdrop-filter: blur(12px);
  flex-shrink: 0;
  z-index: 20;
}

.brand {
  display: flex;
  align-items: center;
  gap: 11px;
  min-width: 0;
}

.brand-mark {
  display: grid;
  place-items: center;
  width: 34px;
  height: 34px;
  border-radius: 10px;
  color: #fff;
  background: linear-gradient(135deg, hsl(262 83% 62%), hsl(221 83% 58%));
  box-shadow: var(--shadow-sm);
  flex-shrink: 0;
}

.brand-text {
  min-width: 0;
}

.brand-text h1 {
  margin: 0;
  font-size: 14.5px;
  font-weight: 600;
  letter-spacing: -0.01em;
  white-space: nowrap;
}

.brand-text p {
  margin: 0;
  font-size: 11.5px;
  color: hsl(var(--muted-foreground));
  white-space: nowrap;
}

.topbar-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* ---------------------------- 通用按钮 ---------------------------- */
.icon-btn {
  display: grid;
  place-items: center;
  width: 34px;
  height: 34px;
  border-radius: 9px;
  color: hsl(var(--muted-foreground));
  transition: background 0.16s, color 0.16s;
}

.icon-btn:hover:not(:disabled) {
  background: hsl(var(--muted));
  color: hsl(var(--foreground));
}

.icon-btn.active {
  background: hsl(var(--primary) / 0.14);
  color: hsl(var(--primary));
}

.icon-btn.sm {
  width: 28px;
  height: 28px;
  border-radius: 7px;
}

.icon-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.ghost-btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border-radius: 7px;
  font-size: 12px;
  color: hsl(var(--muted-foreground));
  transition: background 0.16s, color 0.16s;
}

.ghost-btn:hover {
  background: hsl(var(--muted));
  color: hsl(var(--foreground));
}

/* ---------------------------- 布局 ---------------------------- */
.body {
  flex: 1;
  display: flex;
  min-height: 0;
  position: relative;
  overflow: hidden;
}

.chat-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
}

/* ---------------------------- 欢迎页 ---------------------------- */
.welcome {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 34px;
  padding: 32px 24px;
  overflow-y: auto;
}

.welcome-hero {
  text-align: center;
  animation: fade-in-up 0.4s ease-out both;
}

.hero-mark {
  display: inline-grid;
  place-items: center;
  width: 54px;
  height: 54px;
  margin-bottom: 18px;
  border-radius: 16px;
  color: #fff;
  background: linear-gradient(135deg, hsl(262 83% 62%), hsl(221 83% 58%));
  box-shadow: var(--shadow-md);
}

.welcome-hero h2 {
  margin: 0 0 8px;
  font-size: 26px;
  font-weight: 650;
  letter-spacing: -0.02em;
}

.welcome-hero p {
  margin: 0;
  font-size: 13.5px;
  color: hsl(var(--muted-foreground));
  max-width: 540px;
  text-wrap: pretty;
}

.suggestion-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  width: 100%;
  max-width: 660px;
}

.suggestion-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 15px;
  text-align: left;
  border: 1px solid hsl(var(--border));
  border-radius: var(--radius);
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  transition: border-color 0.16s, transform 0.16s, box-shadow 0.16s;
  animation: fade-in-up 0.4s ease-out both;
}

.suggestion-card:nth-child(1) { animation-delay: 0.04s; }
.suggestion-card:nth-child(2) { animation-delay: 0.08s; }
.suggestion-card:nth-child(3) { animation-delay: 0.12s; }
.suggestion-card:nth-child(4) { animation-delay: 0.16s; }

.suggestion-card:hover {
  border-color: hsl(var(--primary) / 0.55);
  box-shadow: var(--shadow-md);
  transform: translateY(-1px);
}

.suggestion-icon {
  font-size: 19px;
  line-height: 1;
}

.suggestion-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex: 1;
  min-width: 0;
}

.suggestion-text strong {
  font-size: 13.5px;
  font-weight: 600;
}

.suggestion-text small {
  font-size: 11.5px;
  color: hsl(var(--muted-foreground));
}

.suggestion-arrow {
  color: hsl(var(--muted-foreground));
  opacity: 0.45;
  transform: translateX(0);
  transition: opacity 0.16s, transform 0.16s;
}

.suggestion-card:hover .suggestion-arrow {
  opacity: 1;
  transform: translateX(2px);
}

/* ---------------------------- 对话区 ---------------------------- */
.chat-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 26px 20px 8px;
}

.chat-container {
  max-width: 800px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 22px;
}

.message {
  display: flex;
  animation: fade-in-up 0.28s ease-out both;
}

.message.user {
  justify-content: flex-end;
}

.message.ai {
  gap: 12px;
}

.bubble-user {
  max-width: min(78%, 560px);
  padding: 10px 15px;
  border-radius: 16px 16px 4px 16px;
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
  font-size: 14.5px;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-word;
  box-shadow: var(--shadow-sm);
}

.avatar {
  flex-shrink: 0;
  display: grid;
  place-items: center;
  width: 30px;
  height: 30px;
  margin-top: 2px;
  border-radius: 9px;
  color: #fff;
  background: linear-gradient(135deg, hsl(262 83% 62%), hsl(221 83% 58%));
}

.ai-body {
  flex: 1;
  min-width: 0;
}

/* 执行过程 */
.process {
  margin-bottom: 12px;
  border: 1px solid hsl(var(--border));
  border-radius: var(--radius);
  background: hsl(var(--card));
  overflow: hidden;
}

.process > summary {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  font-size: 12.5px;
  color: hsl(var(--muted-foreground));
  cursor: pointer;
  list-style: none;
  user-select: none;
}

.process > summary::-webkit-details-marker {
  display: none;
}

.process > summary:hover {
  background: hsl(var(--muted) / 0.6);
}

.process-done {
  color: hsl(var(--success));
  font-weight: 700;
}

.process-label {
  font-weight: 500;
  color: hsl(var(--foreground));
}

.process-count {
  padding: 1px 7px;
  border-radius: 999px;
  background: hsl(var(--muted));
  font-size: 11px;
}

.chevron {
  margin-left: auto;
  transition: transform 0.2s;
}

.process[open] .chevron {
  transform: rotate(180deg);
}

.process-list {
  margin: 0;
  padding: 4px 12px 10px;
  list-style: none;
  border-top: 1px solid hsl(var(--border));
  max-height: 340px;
  overflow-y: auto;
}

.process-item {
  display: flex;
  gap: 10px;
  padding: 7px 0;
}

.process-dot {
  position: relative;
  flex-shrink: 0;
  width: 7px;
  height: 7px;
  margin-top: 7px;
  border-radius: 50%;
  background: hsl(var(--border));
}

.process-item:not(:last-child) .process-dot::after {
  content: '';
  position: absolute;
  left: 50%;
  top: 9px;
  width: 1px;
  height: calc(100% + 20px);
  transform: translateX(-50%);
  background: hsl(var(--border));
}

.process-content {
  min-width: 0;
  flex: 1;
}

.step > summary {
  list-style: none;
  cursor: pointer;
  border-radius: 6px;
}

.step > summary::-webkit-details-marker {
  display: none;
}

.step > summary:hover {
  background: hsl(var(--muted) / 0.6);
}

.step-chevron {
  margin-left: auto;
  color: hsl(var(--muted-foreground));
  transition: transform 0.2s;
}

.step[open] .step-chevron {
  transform: rotate(180deg);
}

.process-title {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12.5px;
  color: hsl(var(--foreground));
}

.step-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.process-emoji {
  font-size: 12px;
}

.process-details {
  margin: 6px 0 0;
  padding: 8px 10px;
  border-radius: 8px;
  background: hsl(var(--muted) / 0.7);
  border: 1px solid hsl(var(--border));
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 11px;
  line-height: 1.55;
  color: hsl(var(--muted-foreground));
  overflow-x: auto;
  max-height: 190px;
}

/* 思考中 */
.thinking {
  display: flex;
  align-items: center;
  gap: 5px;
  height: 22px;
}

.thinking i {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: hsl(var(--muted-foreground));
  animation: blink 1.3s infinite both;
}

.thinking i:nth-child(2) { animation-delay: 0.18s; }
.thinking i:nth-child(3) { animation-delay: 0.36s; }

.spinner {
  width: 12px;
  height: 12px;
  border: 2px solid hsl(var(--muted-foreground) / 0.35);
  border-top-color: hsl(var(--primary));
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  flex-shrink: 0;
}

.spinner.light {
  width: 14px;
  height: 14px;
  border-color: hsl(var(--primary-foreground) / 0.35);
  border-top-color: hsl(var(--primary-foreground));
}

/* 产出文件卡片 */
.file-cards {
  display: flex;
  flex-wrap: wrap;
  gap: 9px;
  margin-top: 14px;
}

.file-card {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 216px;
  padding: 10px 12px;
  border: 1px solid hsl(var(--border));
  border-radius: var(--radius);
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  text-decoration: none;
  transition: border-color 0.16s, box-shadow 0.16s;
}

.file-card:hover {
  border-color: hsl(var(--primary) / 0.5);
  box-shadow: var(--shadow-sm);
}

.file-card-icon {
  font-size: 17px;
}

.file-card-text {
  display: flex;
  flex-direction: column;
  flex: 1;
  min-width: 0;
}

.file-card-text strong {
  font-size: 12.5px;
  font-weight: 550;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-card-text small {
  font-size: 11px;
  color: hsl(var(--muted-foreground));
}

/* 消息操作 */
.msg-actions {
  display: flex;
  gap: 4px;
  margin-top: 10px;
  opacity: 0;
  transition: opacity 0.16s;
}

.message.ai:hover .msg-actions {
  opacity: 1;
}

/* 系统提示 */
.message.system {
  justify-content: center;
}

.alert {
  max-width: 620px;
  padding: 9px 14px;
  border: 1px solid hsl(var(--danger) / 0.35);
  border-radius: var(--radius);
  background: hsl(var(--danger) / 0.08);
  color: hsl(var(--danger));
  font-size: 12.5px;
  text-align: center;
  word-break: break-word;
}

/* ---------------------------- 输入区 ---------------------------- */
.composer {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 9px;
  padding: 12px 20px 16px;
}

.chips {
  display: flex;
  flex-wrap: wrap;
  gap: 7px;
  width: 100%;
  max-width: 800px;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 5px 9px;
  border: 1px solid hsl(var(--border));
  border-radius: 999px;
  background: hsl(var(--muted));
  font-size: 12px;
  animation: fade-in-up 0.2s ease-out both;
}

.chip-name {
  max-width: 190px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.chip-close {
  color: hsl(var(--muted-foreground));
  font-size: 15px;
  line-height: 1;
  padding: 0 2px;
  border-radius: 4px;
}

.chip-close:hover {
  color: hsl(var(--danger));
}

.composer-box {
  display: flex;
  align-items: flex-end;
  gap: 6px;
  width: 100%;
  max-width: 800px;
  padding: 7px 8px 7px 10px;
  border: 1px solid hsl(var(--border));
  border-radius: 22px;
  background: hsl(var(--card));
  box-shadow: var(--shadow-sm);
  transition: border-color 0.16s, box-shadow 0.16s;
}

.composer-box:focus-within {
  border-color: hsl(var(--primary) / 0.6);
  box-shadow: 0 0 0 3px hsl(var(--primary) / 0.12);
}

.composer-box.running {
  opacity: 0.85;
}

.hidden-input {
  display: none;
}

.attach {
  margin-bottom: 3px;
}

.composer-box textarea {
  flex: 1;
  min-height: 26px;
  max-height: 200px;
  padding: 6px 2px;
  border: none;
  outline: none;
  background: transparent;
  font-size: 14.5px;
  line-height: 1.6;
  resize: none;
}

.composer-box textarea::placeholder {
  color: hsl(var(--muted-foreground) / 0.85);
}

.send-btn {
  display: grid;
  place-items: center;
  width: 34px;
  height: 34px;
  margin-bottom: 1px;
  border-radius: 50%;
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
  flex-shrink: 0;
  transition: opacity 0.16s, transform 0.16s;
}

.send-btn:hover:not(:disabled) {
  transform: scale(1.05);
}

.send-btn:disabled {
  background: hsl(var(--muted));
  color: hsl(var(--muted-foreground));
  cursor: not-allowed;
}

.composer-hint {
  margin: 0;
  font-size: 11.5px;
  color: hsl(var(--muted-foreground));
}

/* ---------------------------- 侧栏 ---------------------------- */
.sidebar-backdrop {
  display: none;
}

/* 桌面端：侧栏占据布局空间，从右侧滑入，不遮挡对话内容 */
.sidebar {
  flex-shrink: 0;
  width: 320px;
  margin-right: -320px;
  display: flex;
  flex-direction: column;
  border-left: 1px solid hsl(var(--border));
  background: hsl(var(--card));
  visibility: hidden;
  transition: margin-right 0.22s cubic-bezier(0.4, 0, 0.2, 1), visibility 0s linear 0.22s;
}

.sidebar.open {
  margin-right: 0;
  visibility: visible;
  transition: margin-right 0.22s cubic-bezier(0.4, 0, 0.2, 1);
}

.sidebar-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 13px 14px;
  border-bottom: 1px solid hsl(var(--border));
}

.sidebar-title {
  min-width: 0;
}

.sidebar-head h3 {
  margin: 0;
  font-size: 13.5px;
  font-weight: 600;
}

.sidebar-sub {
  margin: 2px 0 0;
  font-size: 11px;
  color: hsl(var(--muted-foreground));
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 190px;
}

.sidebar-actions {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.sidebar-body {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
}

.empty {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin: 26px 0;
  text-align: center;
  font-size: 12.5px;
  color: hsl(var(--muted-foreground));
}

.empty small {
  font-size: 11px;
  opacity: 0.8;
}

.file-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.file-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 10px;
  border: 1px solid transparent;
  border-radius: 10px;
  color: hsl(var(--foreground));
  text-decoration: none;
  transition: background 0.16s, border-color 0.16s;
}

.file-row:hover {
  background: hsl(var(--muted));
  border-color: hsl(var(--border));
}

.file-row-icon {
  font-size: 16px;
}

.file-row-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
  flex: 1;
}

.file-row-name {
  font-size: 12.5px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-row-text small {
  font-size: 11px;
  color: hsl(var(--muted-foreground));
}

/* ---------------------------- 响应式 ---------------------------- */
@media (max-width: 820px) {
  /* 窄屏下侧栏改为浮层抽屉，避免挤压对话区域 */
  .sidebar {
    position: fixed;
    top: 60px;
    right: 0;
    bottom: 0;
    z-index: 30;
    margin-right: 0;
    max-width: 88vw;
    transform: translateX(100%);
    transition: transform 0.22s cubic-bezier(0.4, 0, 0.2, 1), visibility 0s linear 0.22s;
    box-shadow: var(--shadow-lg);
  }

  .sidebar.open {
    transform: translateX(0);
  }

  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: 25;
    background: hsl(0 0% 0% / 0.4);
    animation: fade-in-up 0.16s ease-out both;
  }
}

@media (max-width: 720px) {
  .welcome {
    gap: 24px;
    padding: 22px 16px;
  }

  .welcome-hero h2 {
    font-size: 21px;
  }

  .suggestion-grid {
    grid-template-columns: minmax(0, 1fr);
  }

  .brand-text h1 {
    font-size: 13.5px;
  }

  .chat-scroll {
    padding: 18px 14px 8px;
  }

  .composer {
    padding: 10px 14px 14px;
  }

  .bubble-user {
    max-width: 88%;
  }
}
</style>
