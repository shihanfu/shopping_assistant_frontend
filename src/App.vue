<template>
  <!-- Chat interface -->
  <main>
    <div class="toolbar" style="display:flex; gap:8px; justify-content:flex-end;">
      <n-button size="small" tertiary @click="reloadFromServer" :disabled="!sessionId || isLoading">Reload</n-button>
      <n-button size="small" tertiary @click="clearChat" :disabled="isLoading">Clear Chat</n-button>
    </div>
    <div class="chat-container">
      <div class="row" v-for="message in visibleMessages" :key="message.id">
        <div class="space" v-if="message.role == 'user'"></div>
        <div :class="['message', message.role]" v-if="message.role !== 'system'">
          <div v-for="(item, index) in message.content" :key="index">
            <!-- User messages: always text -->
            <p v-if="item.type === 'text' && message.role === 'user'">
              {{ item.text }}
            </p>
            <!-- Assistant messages: text or card -->
            <div v-else-if="item.type === 'text' && message.role === 'assistant'" v-html="renderMarkdown(item.text)" class="message-text"></div>
            <div v-else-if="item.type === 'tool_use' && message.role === 'assistant'" class="message-text">
              <p v-if="item.tool === 'visit_product'">🔍 Looking into product details...</p>
              <p v-else-if="item.tool === 'search'">🔎 Searching for products...</p>
              <p v-else>⚙️ {{ item.tool.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase()) }}...</p>
              <!-- <p>The assistant is trying to {{ item.tool }}</p> -->
              <!-- <p>Input: {{ item.input }}</p> -->
            </div>
            <div v-else-if="item.type === 'unfinished-card' && message.role === 'assistant'" class="unfinished-card-container">
              <div class="unfinished-card">
                <div class="unfinished-card-content">
                  <div class="unfinished-card-header">
                    <span class="unfinished-card-title">Agent is working on your product recommendation</span>
                  </div>
                  <div class="unfinished-card-body">
                    <div class="loading-dots">
                      <span></span>
                      <span></span>
                      <span></span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div v-else-if="item.type === 'card' && message.role === 'assistant'" class="pc-card-container">
              <div v-for="(product, productIndex) in item.card.data" :key="productIndex" class="pc-product-card">
                <div class="pc-card-image">
                  <img :src="product.image" :alt="product.name" @error="handleImageError" />
                </div>
                <div class="pc-card-content">
                  <h3 class="pc-product-title">
                    <a :href="product.url" target="_top" rel="noopener noreferrer">
                      {{ product.name }}
                    </a>
                  </h3>
                  <div class="pc-price">{{ product.price }}</div>
                  <div class="pc-rating-section">
                    <div class="pc-stars">
                      <div class="pc-stars-filled" :style="{ width: (product.rating / 5 * 100) + '%' }"></div>
                    </div>
                    <span class="pc-rating-text">{{ product.rating.toFixed(1) }}</span>
                    <span class="pc-review-count">({{ formatReviewCount(product.review_count) }})</span>
                  </div>
                  <p class="pc-reason">{{ product.reason }}</p>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="space" v-if="message.role == 'assistant'"></div>
      </div>
      <!-- Loading indicator
      <div class="row">
        <div v-if="isAssistantTyping" class="message assistant">
          <p>Assistant is typing...</p>
        </div>
        <div class="space"></div>
      </div> -->
    </div>
    <div class="input-container">
      <n-input-group>
        <n-input
          size="large"
          v-model:value="userInput"
          placeholder="Type your message..."
          type="textarea"
          style="height: 50px; --n-caret-color: rgb(0, 122, 255); --n-border-hover: 1px solid rgb(0, 122, 255); --n-border-focus: 1px solid rgb(0, 122, 255);"
          @keydown.enter.prevent="sendMessage()"
        />
        <n-button
          :loading="isLoading"
          type="primary"
          size="large"
          @click="sendMessage"
          style="height: 50px; --n-color: rgb(0, 122, 255); --n-color-hover: rgb(0, 100, 220); --n-color-pressed: rgb(0, 80, 180); --n-color-focus: rgb(0, 100, 220); --n-color-disabled: rgb(0, 122, 255); --n-ripple-color: rgb(0, 122, 255); --n-border: 1px solid rgb(0, 122, 255); --n-border-hover: 1px solid rgb(0, 100, 220); --n-border-pressed: 1px solid rgb(0, 80, 180); --n-border-focus: 1px solid rgb(0, 100, 220); --n-border-disabled: 1px solid rgb(0, 122, 255);"
          :disabled="userInput.trim() === '' || isLoading"
        >Send</n-button>
      </n-input-group>
    </div>
  </main>
</template>

<script setup lang="ts">
import { nextTick, ref, onMounted, onUnmounted, computed } from 'vue'
import MarkdownIt from 'markdown-it'
const md = new MarkdownIt()

// const SERVER_URL = "http://localhost:5000"
// const SERVER_URL = "http://52.91.223.130:5000"
// const SERVER_URL = "http://52.91.223.130/api"
import { SERVER_URL } from './config'

// ============ Local storage keys ============
const LS_KEYS = {
  sessionId: 'CSA_SESSION_ID',
  parentUrl: 'CSA_PARENT_URL'
} as const

// === Parent page URL logic (keep your implementation) ===
const currentUrl = ref<string | null>(localStorage.getItem(LS_KEYS.parentUrl))
let resolveParentUrlReady!: () => void
const parentUrlReady = new Promise<void>((res) => (resolveParentUrlReady = res))
function getUrlForSend(): string | null {
  return currentUrl.value || (window as any).__CSA_PARENT_URL__ || localStorage.getItem(LS_KEYS.parentUrl)
}

// ========= Type definitions (retain original) =========
interface ProductItem {
  name: string; url: string; image: string; price: string;
  rating: number; review_count: number; reason: string
}
interface ProductCardJSON { type: 'product_card'; version: '1.0'; data: ProductItem[] }
type MessageItem = { type:'text'; text:string } | { type:'card'; card: ProductCardJSON } | { type:'tool_use'; tool:string; input:string } | { type:'unfinished-card'; text:string }
interface Message { id: number; role: 'user' | 'assistant' | 'system'; content: MessageItem[] }

// ========= State =========
const userInput = ref('')
const messages = ref<Message[]>([])
const sessionId = ref<string | null>(null)
const isLoading = ref(true)
const isAssistantTyping = ref(false)

// Only expose non-hidden messages to the template
const visibleMessages = computed(() => {
  const allMessages = (messages.value as any[]) ?? []
  const visible = allMessages.filter(m => !m?.hidden)
  
  console.log('🔍 visibleMessages computed:', {
    total: allMessages.length,
    visible: visible.length,
    hidden: allMessages.length - visible.length
  })
  
  // Debug: show which messages are being hidden
  const hiddenMessages = allMessages.filter(m => m?.hidden)
  if (hiddenMessages.length > 0) {
    console.log('🔍 Hidden messages:', hiddenMessages.map(m => ({ role: m.role, text: m.content?.[0]?.text?.substring(0, 50) + '...' })))
  }
  
  return visible
})

// ========= Utility functions (retain original) =========
const validateProductCard = (obj: any): obj is ProductCardJSON => {
  if (!obj || typeof obj !== 'object') return false
  if (obj.type !== 'product_card') return false
  if (obj.version !== '1.0') return false
  if (!Array.isArray(obj.data)) return false
  return obj.data.every((item: any) =>
    item && typeof item === 'object' &&
    typeof item.name === 'string' &&
    typeof item.url === 'string' &&
    typeof item.image === 'string' &&
    typeof item.price === 'string' &&
    typeof item.rating === 'number' && item.rating >= 0 && item.rating <= 5 &&
    typeof item.review_count === 'number' && item.review_count >= 0 && Number.isInteger(item.review_count) &&
    typeof item.reason === 'string'
  )
}
const safeJsonParse = (t: string): any => { try { return JSON.parse(t) } catch { return null } }
const extractFencedContent = (text: string): string | null => {
  const m1 = text.match(/```json\s*([\s\S]*?)\s*```/); if (m1) return m1[1].trim()
  const m2 = text.match(/```product_card\s*([\s\S]*?)\s*```/); if (m2) return m2[1].trim()
  return null
}
const parseMessageContent = (text: string): MessageItem[] => {
  const frags: MessageItem[] = []
  let idx = 0
  
  while (idx < text.length) {
    const fenced = text.slice(idx).match(/```(?:json|product_card)\s*([\s\S]*?)\s*```/)
    if (fenced) {
      const start = idx + (fenced.index ?? 0)
      const end = start + fenced[0].length
      if (start > idx) {
        const before = text.slice(idx, start)
        if (before.trim()) frags.push({ type: 'text', text: before })
      }
      const body = extractFencedContent(fenced[0])
      if (body) {
        const parsed = safeJsonParse(body)
        if (parsed && validateProductCard(parsed)) frags.push({ type: 'card', card: parsed })
        else frags.push({ type: 'text', text: fenced[0] })
      } else {
        frags.push({ type: 'text', text: fenced[0] })
      }
      idx = end
    } else {
      // Check for incomplete product card at the end
      const incompleteCardMatch = text.slice(idx).match(/```(?:json|product_card)\s*([\s\S]*)$/)
      if (incompleteCardMatch) {
        const start = idx + (incompleteCardMatch.index ?? 0)
        if (start > idx) {
          const before = text.slice(idx, start)
          if (before.trim()) frags.push({ type: 'text', text: before })
        }
        // Add the incomplete card
        const incompleteCardText = text.slice(start)
        frags.push({ type: 'unfinished-card', text: incompleteCardText })
        //debugger
        return frags
      }
      
      // No more fenced blocks, add remaining text
      const rest = text.slice(idx)
      if (rest.trim()) frags.push({ type: 'text', text: rest })
      break
    }
  }
  return frags
}
const formatReviewCount = (n: number) => n >= 1000 ? (n / 1000).toFixed(1).replace(/\.0$/, '') + 'k' : String(n)
const handleImageError = (e: Event) => {
  (e.target as HTMLImageElement).src =
    'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMTAwIiBoZWlnaHQ9IjEwMCIgdmlld0JveD0iMCAwIDEwMCAxMDAiIGZpbGw9Im5vbmUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHJlY3Qgd2lkdGg9IjEwMCIgaGVpZ2h0PSIxMDAiIGZpbGw9IiNGNUY1RjUiLz48cGF0aCBkPSJNMzAgMzBINzBWNzBIMzBWMzBaIiBmaWxsPSIjRDdEN0Q3Ii8+PHBhdGggZD0iTTM1IDM1TDUwIDUwTDUwIDM1TDM1IDM1WiIgZmlsbD0iI0E5QTlBOSIvPjwvc3ZnPg=='
}
const renderMarkdown = (t: string): string => {
  return md.render(t)
}
  // t.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
  //  .replace(/\*(.*?)\*/g, '<em>$1</em>')
  //  .replace(/`(.*?)`/g, '<code>$1</code>')
  //  .replace(/```([\s\S]*?)```/g, '<pre><code>$1</code></pre>')
  //  .replace(/^### (.*$)/gim, '<h3>$1</h3>')
  //  .replace(/^## (.*$)/gim, '<h2>$1</h2>')
  //  .replace(/^# (.*$)/gim, '<h1>$1</h1>')
  //  .replace(/^- (.*$)/gim, '<li>$1</li>')
  //  .replace(/\n/g, '<br>')

// ========= Parent page URL listener =========
const handleParentMessage = (event: MessageEvent) => {
  if (event.data?.type === 'PARENT_URL') {
    const url: string = event.data.url
    console.log('[IFRAME] got parent url:', url)
    currentUrl.value = url
    ;(window as any).__CSA_PARENT_URL__ = url
    localStorage.setItem(LS_KEYS.parentUrl, url)
    resolveParentUrlReady?.()
  }
}
onMounted(() => window.addEventListener('message', handleParentMessage))
onUnmounted(() => window.removeEventListener('message', handleParentMessage))

// ========= Core: initialization & history fetch =========
onMounted(async () => {
  await initSessionAndLoadHistory()
})

async function initSessionAndLoadHistory() {
  isLoading.value = true
  try {
    // 1) First check if session_id exists in local storage
    const saved = localStorage.getItem(LS_KEYS.sessionId)
    if (saved) {
      sessionId.value = saved
      const ok = await reloadFromServer()
      if (ok) {
        console.log('✅ Restored session from localStorage:', saved)
        isLoading.value = false
        return
      }
      // Local session is invalid; start a new session
      console.warn('⚠️ Saved session invalid, creating new session…')
    }

    // 2) Create a new session
    await createSession()
    // 3) A new session has no history, no need to fetch; if the backend
    //    already created it with history, you could call again:
    // await reloadFromServer()
  } catch (e) {
    console.error('init/load error:', e)
  } finally {
    isLoading.value = false
  }
}

// Create session (store id in localStorage)
async function createSession() {
  const resp = await fetch(`${SERVER_URL}/create-session`, { method: 'POST', headers: { 'Content-Type': 'application/json' } })
  if (!resp.ok) {
    console.error(`❌ Failed to create session: ${await resp.text()}`)
    return
  }
  const data = await resp.json()
  sessionId.value = data.session_id
  localStorage.setItem(LS_KEYS.sessionId, data.session_id) // Persist
  console.log(`✅ Session created: ${sessionId.value}`)
}

async function reloadFromServer(): Promise<boolean> {
  if (!sessionId.value) return false
  try {
    const resp = await fetch(`${SERVER_URL}/sessions/${sessionId.value}/messages`, { method: 'GET' })
    if (!resp.ok) {
      console.warn('load history failed:', resp.status)
      return false // ← Return false on 404 to trigger creating a new session upstream
    }
    // ... keep your original logic
    const data = await resp.json()
    if (!data.success || !Array.isArray(data.messages)) return false

    // Convert backend {role, text} to frontend Message[]
    const flat = data.messages as Array<{ role: string; text: string; createdAt?: string; hidden?: boolean }>
    console.log('🔍 Raw messages from backend:', flat)
    
    const mapped: Message[] = flat
      .filter(m => m.role === 'user' || m.role === 'assistant' || m.role === 'system')
      .map((m, idx) => {
        const role = m.role as Message['role']
        const id = Date.now() + idx
        const baseMessage = { id, role, hidden: m.hidden || false }
        
        // Debug log for messages with "current url"
        if (m.text && m.text.includes('current url')) {
          console.log('🔍 Found current url message:', { text: m.text, hidden: m.hidden, willBeHidden: baseMessage.hidden })
        }
        
        if (role === 'assistant') {
          return { ...baseMessage, content: parseMessageContent(m.text ?? '') }
        }
        // Treat user/system as plain text blocks
        return { ...baseMessage, content: [{ type: 'text', text: m.text ?? '' }] }
      })

    console.log('🔍 Mapped messages:', mapped)
    console.log('🔍 Hidden messages count:', mapped.filter(m => m.hidden).length)

    messages.value = mapped
    // Scroll to bottom
    nextTick(() => {
      const el = document.querySelector('.chat-container')
      el?.scrollTo({ top: el.scrollHeight, behavior: 'auto' })
    })
    return true
  } catch (e) {
    console.error('reloadFromServer error:', e)
    return false
  }
}


async function sendMessage() {
  if (userInput.value.trim() === '') return

  // Fallback: automatically create a session if missing
  if (!sessionId.value) {
    await createSession()
    if (!sessionId.value) {
      console.error('❌ No session available')
      return
    }
  }

  const messageText = userInput.value.trim()
  messages.value.push({ id: Date.now(), role: 'user', content: [{ type: 'text', text: messageText }] })
  userInput.value = ''
  nextTick(() => document.querySelector('.chat-container')?.scrollTo({ top: 9e9, behavior: 'smooth' }))

  isLoading.value = true
  isAssistantTyping.value = true


  const createEventSource = async () => {
    // Get parent url
    await Promise.race([parentUrlReady, new Promise(r => setTimeout(r, 1500))])
    const urlForSend = getUrlForSend()
    console.log('[IFRAME] /chat-stream current_url =', urlForSend)
    
    // Create EventSource URL with query parameters
    const params = new URLSearchParams({
      session_id: sessionId.value!,
      message: messageText,
      current_url: urlForSend || ''
    })
    
    return new EventSource(`${SERVER_URL}/chat-stream?${params.toString()}`)
  }

  try {

    
    // Create initial assistant message for streaming
    const currentMessageId = Date.now() + 1
    const assistantMessage: Message = {
      id: currentMessageId,
      role: 'assistant',
      content: [{ type: 'text', text: 'Assistant is typing...' }]
    }
    messages.value.push(assistantMessage)
    nextTick(() => document.querySelector('.chat-container')?.scrollTo({ top: 9e9, behavior: 'smooth' }))

    // Use EventSource for streaming
    const eventSource = await createEventSource()
    
    let currentText = ''

    // Handle message events
    eventSource.addEventListener('message', (event) => {
      try {
        const parsed = JSON.parse(event.data)
        console.log('parsed', parsed)
        
        if (parsed.type === 'text') {
          currentText += parsed.content
          messages.value[messages.value.length - 1].content = parseMessageContent(currentText + "  ...  ")
          // console.log(JSON.stringify(messages.value));
          // console.log(JSON.stringify(visibleMessages.value));
          nextTick(() => document.querySelector('.chat-container')?.scrollTo({ top: 9e9, behavior: 'smooth' }))
        } else if (parsed.type === 'done') {
          // Stream is complete, parse the final content for cards
          const finalFragments = parseMessageContent(currentText)
          messages.value[messages.value.length - 1].content = finalFragments
          nextTick(() => document.querySelector('.chat-container')?.scrollTo({ top: 9e9, behavior: 'smooth' }))
          eventSource.close()
          isLoading.value = false
          isAssistantTyping.value = false
        } else if (parsed.type === 'error') {
          messages.value[messages.value.length - 1].content = [{ type: 'text', text: "ERROR: " + parsed.content }]
          eventSource.close()
          isLoading.value = false
          isAssistantTyping.value = false
        } else if (parsed.type === 'tool_use') {
          messages.value[messages.value.length - 1].content = [{ type: 'text', text: currentText }]
          messages.value.push({ id: Date.now(), role: 'assistant', content: [{ type: 'tool_use', tool: parsed.tool, input: parsed.input }] })
          currentText = ""
          messages.value.push({ id: Date.now(), role: 'assistant', content: [{ type: 'text', text: "..." }] })
          // messages.value[messages.value.length - 1].content = [{ type: 'text', text: "TOOL: " + parsed.content }]
        }
        nextTick(() => document.querySelector('.chat-container')?.scrollTo({ top: 9e9, behavior: 'smooth' }))
      } catch (e) {
        console.warn('Failed to parse SSE data:', event.data, e)
      }
    })

    // Handle errors
     eventSource.addEventListener('error', () => {
       // console.error('EventSource error:', event)
       eventSource.close()
       isLoading.value = false
       isAssistantTyping.value = false
     })

    // Handle connection open
    eventSource.addEventListener('open', () => {
      console.log('EventSource connection opened')
    })

  } catch (e) {
    console.error('sendMessage error:', e)
    isLoading.value = false
    isAssistantTyping.value = false
  }
}

// ========= Clear chat (new session from frontend perspective) =========
async function clearChat() {
  if (!sessionId.value) return
  try {
    isLoading.value = true
    const resp = await fetch(`${SERVER_URL}/cleanup-session`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ session_id: sessionId.value })
    })
    const data = await resp.json()
    if (resp.ok && data.success) {
      console.log('✅ Chat cleared:', data.message)

      // 1) Clear the frontend view
      messages.value = []

      // 2) Reset local session and remove it from localStorage
      sessionId.value = null
      localStorage.removeItem(LS_KEYS.sessionId)

      // 3) Immediately create a brand-new session (avoid subsequent /chat-stream 404)
      await createSession()

      // (Optional) add a system message
      messages.value.push({
        id: Date.now(),
        role: 'system',
        content: [{ type: 'text', text: 'Started a new session.' }]
      })
    } else {
      console.error('❌ Failed to cleanup session:', data.error || resp.status)
    }
  } catch (e) {
    console.error('clearChat error:', e)
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped lang="scss">
/* No styles here, styles are in main.css and base.css */
</style>

<style>
body {
  line-height: 1.5;
  font-size: 22px;
}
</style>

<style scoped lang="scss">
.row {
  display: flex;
  flex-direction: row;
}
.space {
  flex-grow: 1;
  flex-shrink: 1;
}
.message {
  flex-shrink: 0;
  max-width: 100%;
}
.message:has(.n-input) {
  width: 100%;
}

/* Chat styles */
.chat-container {
  flex: 1;
  padding: 10px;
  overflow-y: auto;
  min-height: 0;
  margin: 10px 0;
  background-color: #ffffff;
  border-radius: 8px;
}

.message {
  margin-bottom: 15px;
  padding: 4px 10px;
  border-radius: 20px 20px 0px 20px;
  color: var(--color-text); // Ensure text color is set

  p {
    margin: 0;
  }

  &.system {
    background-color: #e9ecef;
    color: #333333;
  }

  &.user {
    background-color: rgb(0, 122, 255);
    font-weight: bold;
    color: white;
  }

  &.assistant {
    background-color: white;
    text-align: left;
    
    // Markdown styling
    strong {
      font-weight: bold;
    }
    
    em {
      font-style: italic;
    }
    
    code {
      background-color: rgba(0, 0, 0, 0.1);
      padding: 2px 4px;
      border-radius: 3px;
      font-family: monospace;
    }
    
    pre {
      background-color: rgba(0, 0, 0, 0.05);
      padding: 10px;
      border-radius: 5px;
      overflow-x: auto;
      
      code {
        background-color: transparent;
        padding: 0;
      }
    }
    
    h1, h2, h3 {
      margin: 10px 0 5px 0;
      font-weight: bold;
    }
    
    h1 {
      font-size: 1.5em;
    }
    
    h2 {
      font-size: 1.3em;
    }
    
    h3 {
      font-size: 1 em;
    }
    
    li {
      margin: 2px 0;
      padding-left: 10px;
    }
  }
}

.input-container {
  display: flex;
  gap: 10px;
  margin-top: 10px;
  align-items: center;
}

/* Product Card Styles */
.pc-card-container {
  margin: 10px 0;
}

.pc-product-card {
  display: flex;
  background: white;
  border-radius: 16px;
  border: 1px solid #e0e0e0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin-bottom: 12px;
  overflow: hidden;
  transition: box-shadow 0.2s ease;
  
  &:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  }
  
  @media (max-width: 768px) {
    flex-direction: row;
  }
}

.pc-card-image {
  flex: 0 0 40%;   /* Left image takes 40% */
  max-width: 40%;
  
  img {
    width: 100%;
    height: auto;
    object-fit: contain;
    display: block;
  }
}

.pc-card-content {
  flex: 0 0 60%;   /* Right text takes 60% */
  max-width: 60%;
  padding: 8px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.pc-product-title {
  margin: 0 0 8px 0;
  font-size: 14px;
  font-weight: 600;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
  
  a {
    color: #2563eb;
    text-decoration: none;
    
    &:hover {
      text-decoration: underline;
    }
  }
}

.pc-price {
  font-size: 18px;
  font-weight: 700;
  color: #059669;
  margin-bottom: 8px;
}

.pc-rating-section {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.pc-stars {
  position: relative;
  width: 80px;
  height: 16px;
  background: #e5e7eb;
  border-radius: 2px;
  overflow: hidden;
}

.pc-stars-filled {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: #fbbf24;
  border-radius: 2px;
  transition: width 0.3s ease;
}

.pc-rating-text {
  font-weight: 600;
  color: #374151;
  font-size: 12px;
}

.pc-review-count {
  color: #6b7280;
  font-size: 12px;
}

.pc-reason {
  margin: 0;
  color: #4b5563;
  font-size: 12px;
  line-height: 1.4;
}
.message-text{
  :deep(ul) {
    list-style-type: disc;
    list-style-position: inside;
  }
}

/* Unfinished Card Styles */
.unfinished-card-container {
  margin: 10px 0;
}

.unfinished-card {
  display: flex;
  background: #f8f9fa;
  border-radius: 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  margin-bottom: 12px;
  overflow: hidden;
  border: 2px dashed #dee2e6;
  min-height: 120px;
}

.unfinished-card-content {
  flex: 1;
  padding: 16px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.unfinished-card-header {
  margin-bottom: 16px;
}

.unfinished-card-title {
  font-size: 16px;
  font-weight: 600;
  color: #6c757d;
}

.unfinished-card-body {
  display: flex;
  justify-content: center;
  align-items: center;
}

.loading-dots {
  display: flex;
  gap: 4px;
}

.loading-dots span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #6c757d;
  animation: loading-dots 1.4s infinite ease-in-out both;
}

.loading-dots span:nth-child(1) {
  animation-delay: -0.32s;
}

.loading-dots span:nth-child(2) {
  animation-delay: -0.16s;
}

.loading-dots span:nth-child(3) {
  animation-delay: 0s;
}

@keyframes loading-dots {
  0%, 80%, 100% {
    transform: scale(0);
    opacity: 0.5;
  }
  40% {
    transform: scale(1);
    opacity: 1;
  }
}
</style>
