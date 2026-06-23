<script setup>
import { computed, nextTick, onBeforeUnmount, ref } from 'vue'

const messages = ref([])
const input = ref('')
const isRunning = ref(false)
const listRef = ref(null)

let abortController = null

const canSend = computed(() => input.value.trim() && !isRunning.value)

const demoScript = [
  {
    user: '帮我2查一下杭州明天天气，并推荐一件适合出行的外套。',
    steps: [
      { type: 'thinking', text: '分析用户意图：需要天气查询 + 穿搭建议' },
      { type: 'tool', name: 'get_weather', args: '{ "city": "杭州", "date": "明天" }' },
      { type: 'tool', name: 'search_products', args: '{ "category": "外套", "weather": "小雨" }' },
    ],
    reply:
      '杭州明天有小雨，气温 12°C–16°C，体感偏凉。\n\n建议穿轻薄防泼水风衣或软壳外套，内搭长袖 T 恤即可。若需长时间户外，可再备一把折叠伞。',
  },
  {
    user: '如果改成上海呢？',
    steps: [{ type: 'thinking', text: '沿用上一轮上下文，切换城市为上海' }],
    reply: '上海明天多云转晴，15°C–22°C，比杭州更暖一些。薄款针织开衫或休闲夹克就够用了。',
  },
]

function scrollToBottom() {
  nextTick(() => {
    const el = listRef.value
    if (el) el.scrollTop = el.scrollHeight
  })
}

function wait(ms, signal) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(resolve, ms)
    signal?.addEventListener('abort', () => {
      clearTimeout(timer)
      reject(new DOMException('Aborted', 'AbortError'))
    })
  })
}

async function streamText(message, text, signal) {
  const chars = [...text]
  for (let i = 0; i < chars.length; i++) {
    await wait(18 + Math.random() * 22, signal)
    message.content += chars[i]
    scrollToBottom()
  }
  message.status = 'done'
}

async function runAgentTurn({ userText, steps, reply }) {
  messages.value.push({
    id: crypto.randomUUID(),
    role: 'user',
    content: userText,
    status: 'done',
  })
  scrollToBottom()

  const assistant = {
    id: crypto.randomUUID(),
    role: 'assistant',
    content: '',
    status: 'running',
    steps: [],
  }
  messages.value.push(assistant)
  scrollToBottom()

  for (const step of steps) {
    assistant.steps.push({ ...step, status: 'running' })
    scrollToBottom()
    await wait(step.type === 'thinking' ? 900 : 1200, abortController.signal)
    assistant.steps[assistant.steps.length - 1].status = 'done'
  }

  await streamText(assistant, reply, abortController.signal)
}

async function sendMessage(text) {
  const trimmed = text.trim()
  if (!trimmed || isRunning.value) return

  isRunning.value = true
  abortController = new AbortController()

  const fallback = {
    user: trimmed,
    steps: [{ type: 'thinking', text: '理解问题并组织回答' }],
    reply: `收到：「${trimmed}」\n\n这是模拟 Agent 回复。你可以点击「播放演示」查看完整的多步对话流（思考 → 工具调用 → 流式输出）。`,
  }

  try {
    await runAgentTurn(fallback)
  } catch (err) {
    if (err.name !== 'AbortError') throw err
  } finally {
    isRunning.value = false
    abortController = null
  }
}

async function playDemo() {
  if (isRunning.value) return
  reset()
  isRunning.value = true
  abortController = new AbortController()

  try {
    for (const turn of demoScript) {
      await runAgentTurn(turn)
      await wait(600, abortController.signal)
    }
  } catch (err) {
    if (err.name !== 'AbortError') throw err
  } finally {
    isRunning.value = false
    abortController = null
  }
}

function handleSend() {
  const text = input.value
  input.value = ''
  sendMessage(text)
}

function reset() {
  abortController?.abort()
  abortController = null
  isRunning.value = false
  messages.value = []
  input.value = ''
}

onBeforeUnmount(() => {
  abortController?.abort()
})
</script>

<template>
  <div class="chat-demo">
    <header class="chat-header">
      <div>
        <h1>Agent 对话流 Demo</h1>
        <p>模拟思考、工具调用与流式回复</p>
      </div>
      <div class="header-actions">
        <button type="button" class="btn btn-ghost" :disabled="isRunning" @click="playDemo">
          播放演示
        </button>
        <button type="button" class="btn btn-ghost" @click="reset">清空</button>
      </div>
    </header>

    <div ref="listRef" class="message-list">
      <div v-if="!messages.length" class="empty">
        <p>发送消息，或点击「播放演示」自动跑一遍对话流。</p>
      </div>

      <article
        v-for="msg in messages"
        :key="msg.id"
        class="message"
        :class="msg.role"
      >
        <div class="avatar">{{ msg.role === 'user' ? '你' : 'AI' }}</div>
        <div class="bubble">
          <template v-if="msg.role === 'assistant' && msg.steps?.length">
            <ul class="steps">
              <li v-for="(step, i) in msg.steps" :key="i" :class="[step.type, step.status]">
                <span class="step-label">
                  {{
                    step.type === 'thinking'
                      ? '思考'
                      : step.type === 'tool'
                        ? `工具 · ${step.name}`
                        : '输出'
                  }}
                </span>
                <span class="step-body">{{ step.text || step.args }}</span>
                <span v-if="step.status === 'running'" class="pulse">...</span>
              </li>
            </ul>
          </template>

          <p v-if="msg.content" class="content">{{ msg.content }}</p>
          <span
            v-else-if="msg.status === 'running' && !msg.steps?.length"
            class="typing"
          >
            正在输入<span class="dots"><i /><i /><i /></span>
          </span>
          <span v-else-if="msg.status === 'running'" class="cursor">▍</span>
        </div>
      </article>
    </div>

    <footer class="composer">
      <textarea
        v-model="input"
        rows="2"
        placeholder="输入消息，Enter 发送，Shift+Enter 换行"
        :disabled="isRunning"
        @keydown.enter.exact.prevent="handleSend"
      />
      <button type="button" class="btn btn-primary" :disabled="!canSend" @click="handleSend">
        发送
      </button>
    </footer>
  </div>
</template>

<style scoped>
.chat-demo {
  display: flex;
  flex-direction: column;
  min-height: 100svh;
  max-width: 860px;
  margin: 0 auto;
  text-align: left;
}

.chat-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  padding: 28px 24px 20px;
  border-bottom: 1px solid var(--border);
}

.chat-header h1 {
  font-size: 28px;
  margin: 0 0 6px;
  letter-spacing: -0.5px;
}

.chat-header p {
  color: var(--text);
  font-size: 15px;
}

.header-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 20px 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.empty {
  margin: auto;
  text-align: center;
  color: var(--text);
  font-size: 15px;
}

.message {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

.message.user {
  flex-direction: row-reverse;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  display: grid;
  place-items: center;
  font-size: 13px;
  font-weight: 600;
  flex-shrink: 0;
  background: var(--code-bg);
  color: var(--text-h);
}

.message.assistant .avatar {
  background: var(--accent-bg);
  color: var(--accent);
}

.bubble {
  max-width: min(100%, 620px);
  padding: 12px 14px;
  border-radius: 14px;
  border: 1px solid var(--border);
  background: var(--bg);
}

.message.user .bubble {
  background: var(--accent-bg);
  border-color: var(--accent-border);
}

.steps {
  list-style: none;
  padding: 0;
  margin: 0 0 10px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.steps li {
  font-size: 13px;
  padding: 8px 10px;
  border-radius: 8px;
  background: var(--code-bg);
  border: 1px solid var(--border);
}

.steps li.running {
  border-color: var(--accent-border);
}

.step-label {
  display: inline-block;
  margin-right: 8px;
  font-weight: 600;
  color: var(--accent);
  font-size: 12px;
}

.step-body {
  color: var(--text);
  word-break: break-word;
}

.pulse {
  margin-left: 4px;
  color: var(--accent);
}

.content {
  margin: 0;
  white-space: pre-wrap;
  line-height: 1.6;
  color: var(--text-h);
}

.cursor {
  color: var(--accent);
  animation: blink 1s step-end infinite;
}

.typing {
  color: var(--text);
  font-size: 14px;
}

.dots {
  display: inline-flex;
  gap: 3px;
  margin-left: 4px;
  vertical-align: middle;
}

.dots i {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: var(--accent);
  animation: bounce 1.2s infinite ease-in-out;
}

.dots i:nth-child(2) {
  animation-delay: 0.15s;
}

.dots i:nth-child(3) {
  animation-delay: 0.3s;
}

.composer {
  display: flex;
  gap: 12px;
  padding: 16px 24px 24px;
  border-top: 1px solid var(--border);
  align-items: flex-end;
}

.composer textarea {
  flex: 1;
  resize: none;
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 12px 14px;
  font: inherit;
  color: var(--text-h);
  background: var(--bg);
  min-height: 52px;
}

.composer textarea:focus {
  outline: 2px solid var(--accent-border);
  outline-offset: 1px;
}

.btn {
  border: none;
  border-radius: 10px;
  padding: 10px 16px;
  font: inherit;
  cursor: pointer;
  transition: opacity 0.2s, box-shadow 0.2s;
}

.btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.btn-primary {
  background: var(--accent);
  color: #fff;
  font-weight: 600;
}

.btn-primary:not(:disabled):hover {
  box-shadow: var(--shadow);
}

.btn-ghost {
  background: var(--code-bg);
  color: var(--text-h);
  border: 1px solid var(--border);
}

.btn-ghost:not(:disabled):hover {
  border-color: var(--accent-border);
}

@keyframes blink {
  50% {
    opacity: 0;
  }
}

@keyframes bounce {
  0%,
  80%,
  100% {
    transform: translateY(0);
    opacity: 0.4;
  }
  40% {
    transform: translateY(-4px);
    opacity: 1;
  }
}

@media (max-width: 640px) {
  .chat-header {
    flex-direction: column;
    padding: 20px 16px 16px;
  }

  .message-list,
  .composer {
    padding-left: 16px;
    padding-right: 16px;
  }

  .composer {
    flex-direction: column;
    align-items: stretch;
  }
}
</style>
