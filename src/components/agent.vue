<script setup>
import { ref, reactive, nextTick } from 'vue';
import MarkdownIt from 'markdown-it';

const md = new MarkdownIt();
const chatBody = ref(null);

const fallbackPublicApiBaseUrl = 'https://stove-clone-hurray.ngrok-free.dev';
const rawApiBaseUrl = (import.meta.env.VITE_API_BASE_URL || '').trim();
const resolvedApiBaseUrl = (rawApiBaseUrl || fallbackPublicApiBaseUrl).replace(/\/$/, '');
const isBrowser = typeof window !== 'undefined';
const isLocalPreview =
  !isBrowser || ['localhost', '127.0.0.1'].includes(window.location.hostname);
const chatApiUrl = isLocalPreview && !rawApiBaseUrl ? '/api/chat' : `${resolvedApiBaseUrl}/chat`;

const quickQuestions = [
  '我的订单还没收到，怎么查询物流进度？',
  '商品有质量问题，如何申请退换货？',
  '设备不会连接网络，应该怎么排查？',
  '保修期多久，需要准备哪些售后凭证？',
  '发票开错了，能重新申请吗？',
  '人工客服什么时候在线？'
];

const servicePromises = [
  { label: '7 x 12 在线答疑', desc: '常见售后问题快速响应' },
  { label: '退换流程指引', desc: '一步步说明申请和审核流程' },
  { label: '故障排查建议', desc: '优先帮助用户先定位问题' }
];

const messages = ref([
  {
    role: 'assistant',
    content:
      '你好，我是售后问答助手，可以帮你处理物流查询、退换货、保修说明、故障排查和发票问题。请直接描述你遇到的情况。',
    thinking: false,
    uiData: null
  }
]);

const userInput = ref('');

const scrollToBottom = async () => {
  await nextTick();
  if (chatBody.value) {
    chatBody.value.scrollTop = chatBody.value.scrollHeight;
  }
};

const parseMessageContent = (messageObj) => {
  const fullText = messageObj.content || '';
  const regex = /---JSON_UI_BEGIN---([\s\S]*?)---JSON_UI_END---/g;
  const cards = [];

  let match;
  while ((match = regex.exec(fullText)) !== null) {
    try {
      cards.push(JSON.parse(match[1].trim()));
    } catch (error) {
      console.error('卡片 JSON 解析失败:', error);
    }
  }

  messageObj.uiData = cards.length > 0 ? cards : null;
  messageObj.content = fullText.replace(regex, '').trim();
};

const typeEffect = (messageObj, cleanText, pendingCards = null) => {
  if (!cleanText) {
    messageObj.thinking = false;
    messageObj.uiData = pendingCards;
    return;
  }

  let index = 0;
  messageObj.content = '';
  messageObj.thinking = false;
  messageObj.uiData = null;

  const timer = setInterval(() => {
    if (index < cleanText.length) {
      messageObj.content += cleanText.charAt(index);
      index += 1;
      scrollToBottom();
      return;
    }

    clearInterval(timer);
    if (pendingCards) {
      messageObj.uiData = pendingCards;
      nextTick(() => {
        scrollToBottom();
      });
    }
  }, 24);
};

function executeStyleAction(actionStr) {
  const match = actionStr.match(/\[ACTION\]([^:]+):?(.*)/);
  if (!match) return false;

  const actionName = match[1];
  const params = match[2] ? match[2].split(':') : [];

  switch (actionName) {
    case 'change_chatbox_color':
      changeChatboxColor(params[0]);
      return true;
    case 'change_chatbox_text_color':
      changeChatboxTextColor(params[0]);
      return true;
    case 'reset_chatbox_style':
      resetChatboxStyle();
      return true;
    default:
      return false;
  }
}

function changeChatboxColor(color) {
  const chatWindow = document.querySelector('.chat-shell');
  if (chatWindow) {
    chatWindow.style.background =
      color === 'black' ? 'rgba(10, 18, 32, 0.96)' : `rgba(${getRgbFromColor(color)}, 0.96)`;
  }

  const mainScroll = document.querySelector('.main-scroll');
  if (mainScroll) {
    mainScroll.style.background =
      color === 'black' ? 'rgba(7, 14, 26, 0.72)' : `rgba(${getRgbFromColor(color)}, 0.18)`;
  }

  document.documentElement.style.setProperty('--chat-bg', color);
}

function changeChatboxTextColor(color) {
  const bubbles = document.querySelectorAll('.assistant .bubble');
  bubbles.forEach((bubble) => {
    bubble.style.color = color;
  });
  document.documentElement.style.setProperty('--chat-text', color);
}

function resetChatboxStyle() {
  const chatWindow = document.querySelector('.chat-shell');
  if (chatWindow) {
    chatWindow.style.background = 'rgba(255, 255, 255, 0.96)';
  }

  const mainScroll = document.querySelector('.main-scroll');
  if (mainScroll) {
    mainScroll.style.background = 'linear-gradient(180deg, #f8fafc 0%, #eef4ff 100%)';
  }

  const bubbles = document.querySelectorAll('.assistant .bubble');
  bubbles.forEach((bubble) => {
    bubble.style.color = '#1f2937';
  });

  document.documentElement.style.setProperty('--chat-bg', '#ffffff');
  document.documentElement.style.setProperty('--chat-text', '#1f2937');
}

function getRgbFromColor(color) {
  const colorMap = {
    black: '10,18,32',
    white: '255,255,255',
    red: '220,38,38',
    blue: '37,99,235',
    green: '22,163,74',
    yellow: '245,158,11',
    orange: '249,115,22',
    gray: '107,114,128'
  };
  return colorMap[color] || '255,255,255';
}

const sendMessage = async () => {
  if (!userInput.value.trim()) return;

  const currentPrompt = userInput.value.trim();
  messages.value.push({ role: 'user', content: currentPrompt });
  userInput.value = '';

  const assistantMsg = reactive({
    role: 'assistant',
    content: '',
    thinking: true,
    uiData: null
  });
  messages.value.push(assistantMsg);
  await scrollToBottom();

  const finishAssistantMessage = (rawText) => {
    if (!rawText) {
      assistantMsg.thinking = false;
      assistantMsg.content = '没有收到后端返回内容，请检查接口返回格式。';
      return;
    }

    if (rawText.includes('[ACTION]')) {
      executeStyleAction(rawText);
      const isPureAction = rawText.trim().match(/^\[ACTION[^\]]+\](?::[^:]+)?$/);
      if (isPureAction) {
        assistantMsg.thinking = false;
        assistantMsg.content = '';
        scrollToBottom();
        return;
      }
    }

    const tempProcessor = { content: rawText, uiData: null };
    parseMessageContent(tempProcessor);
    typeEffect(assistantMsg, tempProcessor.content, tempProcessor.uiData);
  };

  try {
    const response = await fetch(chatApiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt: currentPrompt })
    });

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(
        `HTTP ${response.status} ${response.statusText}${errorText ? `: ${errorText}` : ''}`
      );
    }

    const contentType = response.headers.get('content-type') || '';
    const isSse = contentType.includes('text/event-stream');

    if (!response.body || !response.body.getReader || !isSse) {
      const fallbackText = await response.text();
      try {
        const jsonData = JSON.parse(fallbackText);
        finishAssistantMessage(jsonData.content || jsonData.message || fallbackText);
      } catch {
        finishAssistantMessage(fallbackText);
      }
      return;
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let finalContent = '';
    let handledDone = false;

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value, { stream: true });
      const lines = chunk.split('\n');

      lines.forEach((line) => {
        if (!line.startsWith('data: ')) return;

        try {
          const data = JSON.parse(line.slice(6));
          if (data.content) {
            finalContent = data.content;
          }

          if (data.status === 'done') {
            handledDone = true;
            finishAssistantMessage(data.content || finalContent);
          }
        } catch (error) {
          console.error('解析数据包失败:', error);
        }
      });
    }

    if (!handledDone) {
      finishAssistantMessage(finalContent);
    }
  } catch (error) {
    assistantMsg.thinking = false;
    assistantMsg.content = `连接失败，请检查接口地址或后端返回格式。${
      error.message ? ` (${error.message})` : ''
    }`;
    console.error('Fetch Error:', error);
  }
};
</script>

<template>
  <div class="app-container">
    <div class="bg-grid"></div>
    <div class="bg-glow glow-left"></div>
    <div class="bg-glow glow-right"></div>

    <aside class="service-panel">
      <div class="panel-badge">After-Sales Support</div>
      <h1>售后问答助手</h1>
      <p class="panel-copy">
        面向物流、退换货、保修、安装与发票问题的智能客服入口，让用户先得到清晰可执行的答案。
      </p>

      <div class="promise-list">
        <div v-for="item in servicePromises" :key="item.label" class="promise-card">
          <strong>{{ item.label }}</strong>
          <span>{{ item.desc }}</span>
        </div>
      </div>

      <div class="quick-card">
        <div class="quick-header">
          <span class="quick-dot"></span>
          <span>常见售后问题</span>
        </div>
        <div class="quick-list">
          <button
            v-for="question in quickQuestions"
            :key="question"
            class="quick-item"
            @click="userInput = question"
          >
            {{ question }}
          </button>
        </div>
      </div>
    </aside>

    <section class="chat-shell">
      <header class="header">
        <div class="header-info">
          <div class="brand-mark">售后</div>
          <div class="title-group">
            <strong class="main-title">智能售后服务台</strong>
            <span class="sub-title">优先回答处理流程，再引导用户补充订单或设备信息</span>
          </div>
        </div>
        <div class="header-status">
          <span class="status-dot"></span>
          <span>在线响应中</span>
        </div>
      </header>

      <main ref="chatBody" class="main-scroll">
        <div class="notice-strip">
          支持的问题类型：物流进度、退换货申请、故障排查、保修政策、发票与人工客服转接
        </div>

        <div v-for="(msg, i) in messages" :key="i" :class="['message-row', msg.role]">
          <div v-if="msg.role === 'assistant'" class="avatar assistant-avatar">AI</div>

          <div class="bubble-container">
            <div class="role-label">{{ msg.role === 'assistant' ? '售后助手' : '用户' }}</div>
            <div class="bubble">
              <div v-if="msg.role === 'assistant' && msg.thinking" class="thinking-bubble">
                <span>正在整理售后处理建议</span>
                <div class="dot-ani"></div>
                <div class="dot-ani"></div>
                <div class="dot-ani"></div>
              </div>

              <div v-else class="markdown-body" v-html="md.render(msg.content)"></div>

              <div v-if="msg.uiData && msg.uiData.length > 0" class="ui-cards-wrapper">
                <div v-for="(card, index) in msg.uiData" :key="index" class="card-item">
                  <div v-if="card.type === 'scenery-card'" class="service-card">
                    <div class="card-image">
                      <img :src="card.data.image" :alt="card.data.name" @load="scrollToBottom" />
                    </div>
                    <div class="card-body">
                      <h4 class="card-name">{{ card.data.name }}</h4>
                      <div class="card-tags">
                        <span v-for="tag in card.data.tags" :key="tag" class="tag-item">{{ tag }}</span>
                      </div>
                      <p class="card-desc">{{ card.data.desc }}</p>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div v-if="msg.role === 'user'" class="avatar user-avatar">我</div>
        </div>
      </main>

      <footer class="footer">
        <div class="input-wrapper">
          <input
            v-model="userInput"
            placeholder="请输入你的售后问题，例如：耳机左边没有声音，怎么处理？"
            @keyup.enter="sendMessage"
          />
          <button class="send-btn" :disabled="!userInput.trim()" @click="sendMessage">发送</button>
        </div>
      </footer>
    </section>
  </div>
</template>

<style scoped>
.app-container {
  position: fixed;
  inset: 0;
  display: grid;
  grid-template-columns: 340px minmax(0, 1fr);
  gap: 28px;
  padding: 28px;
  overflow: hidden;
  background:
    radial-gradient(circle at top left, rgba(14, 165, 233, 0.22), transparent 32%),
    radial-gradient(circle at bottom right, rgba(249, 115, 22, 0.18), transparent 28%),
    linear-gradient(135deg, #eaf1f8 0%, #f6f8fc 46%, #eef4fb 100%);
  font-family: 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
}

.bg-grid {
  position: absolute;
  inset: 0;
  background-image:
    linear-gradient(rgba(148, 163, 184, 0.08) 1px, transparent 1px),
    linear-gradient(90deg, rgba(148, 163, 184, 0.08) 1px, transparent 1px);
  background-size: 28px 28px;
  mask-image: linear-gradient(180deg, rgba(0, 0, 0, 0.55), transparent 88%);
  pointer-events: none;
}

.bg-glow {
  position: absolute;
  border-radius: 999px;
  filter: blur(90px);
  opacity: 0.5;
  pointer-events: none;
}

.glow-left {
  top: 8%;
  left: -6%;
  width: 260px;
  height: 260px;
  background: rgba(14, 165, 233, 0.3);
}

.glow-right {
  right: -4%;
  bottom: 12%;
  width: 320px;
  height: 320px;
  background: rgba(249, 115, 22, 0.22);
}

.service-panel,
.chat-shell {
  position: relative;
  z-index: 1;
}

.service-panel {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 30px 26px;
  border: 1px solid rgba(255, 255, 255, 0.75);
  border-radius: 28px;
  background: rgba(9, 21, 38, 0.88);
  color: #f8fafc;
  box-shadow: 0 24px 70px rgba(15, 23, 42, 0.22);
  backdrop-filter: blur(22px);
}

.panel-badge {
  align-self: flex-start;
  padding: 8px 12px;
  border: 1px solid rgba(125, 211, 252, 0.32);
  border-radius: 999px;
  background: rgba(14, 165, 233, 0.12);
  color: #7dd3fc;
  font-size: 12px;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.service-panel h1 {
  margin: 0;
  font-size: 36px;
  line-height: 1.12;
  letter-spacing: 0.02em;
}

.panel-copy {
  color: rgba(226, 232, 240, 0.78);
  line-height: 1.75;
}

.promise-list {
  display: grid;
  gap: 12px;
}

.promise-card {
  display: grid;
  gap: 6px;
  padding: 16px;
  border: 1px solid rgba(148, 163, 184, 0.18);
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.04);
}

.promise-card strong {
  font-size: 15px;
}

.promise-card span {
  color: rgba(226, 232, 240, 0.7);
  font-size: 13px;
}

.quick-card {
  padding: 18px;
  border-radius: 22px;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.06), rgba(255, 255, 255, 0.03));
}

.quick-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 14px;
  color: #e2e8f0;
  font-size: 14px;
  font-weight: 600;
}

.quick-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #fb923c;
  box-shadow: 0 0 10px rgba(251, 146, 60, 0.8);
}

.quick-list {
  display: grid;
  gap: 10px;
}

.quick-item {
  padding: 12px 14px;
  border: 1px solid rgba(148, 163, 184, 0.14);
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.04);
  color: #f8fafc;
  text-align: left;
  cursor: pointer;
  transition:
    transform 0.25s ease,
    border-color 0.25s ease,
    background 0.25s ease;
}

.quick-item:hover {
  transform: translateX(6px);
  border-color: rgba(125, 211, 252, 0.42);
  background: rgba(14, 165, 233, 0.12);
}

.chat-shell {
  display: flex;
  flex-direction: column;
  min-width: 0;
  height: calc(100vh - 56px);
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.85);
  border-radius: 30px;
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 28px 80px rgba(30, 41, 59, 0.16);
  backdrop-filter: blur(24px);
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  flex-shrink: 0;
  padding: 22px 26px;
  border-bottom: 1px solid #e5edf5;
  background:
    linear-gradient(90deg, rgba(14, 165, 233, 0.08), transparent 42%),
    linear-gradient(180deg, #ffffff, #fbfdff);
}

.header-info {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: 0;
}

.brand-mark {
  display: grid;
  place-items: center;
  width: 48px;
  height: 48px;
  border-radius: 16px;
  background: linear-gradient(135deg, #0ea5e9, #2563eb);
  color: #fff;
  font-size: 14px;
  font-weight: 700;
  box-shadow: 0 14px 30px rgba(37, 99, 235, 0.22);
}

.title-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.main-title {
  color: #0f172a;
  font-size: 18px;
  font-weight: 700;
}

.sub-title {
  color: #64748b;
  font-size: 12px;
}

.header-status {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #475569;
  font-size: 13px;
  white-space: nowrap;
}

.status-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #22c55e;
  box-shadow: 0 0 10px rgba(34, 197, 94, 0.85);
}

.main-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 22px 24px 14px;
  background: linear-gradient(180deg, #f8fafc 0%, #eef4ff 100%);
  scroll-behavior: smooth;
}

.notice-strip {
  margin-bottom: 18px;
  padding: 12px 14px;
  border: 1px solid #dbeafe;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.82);
  color: #33517a;
  font-size: 13px;
}

.message-row {
  display: flex;
  align-items: flex-start;
  margin-bottom: 18px;
  animation: fadeIn 0.36s ease;
}

.message-row.user {
  justify-content: flex-end;
}

.avatar {
  display: grid;
  place-items: center;
  flex-shrink: 0;
  width: 42px;
  height: 42px;
  border-radius: 15px;
  font-size: 13px;
  font-weight: 700;
}

.assistant-avatar {
  margin-right: 12px;
  background: linear-gradient(135deg, #dbeafe, #e0f2fe);
  color: #1d4ed8;
}

.user-avatar {
  margin-left: 12px;
  background: linear-gradient(135deg, #fdeddc, #ffedd5);
  color: #c2410c;
}

.bubble-container {
  max-width: min(78%, 760px);
}

.role-label {
  margin-bottom: 6px;
  color: #64748b;
  font-size: 12px;
}

.message-row.user .role-label {
  text-align: right;
}

.bubble {
  padding: 14px 18px;
  border-radius: 18px;
  font-size: 15px;
  line-height: 1.7;
  word-break: break-word;
}

.assistant .bubble {
  border: 1px solid rgba(203, 213, 225, 0.72);
  border-top-left-radius: 6px;
  background: rgba(255, 255, 255, 0.96);
  color: #1f2937;
  box-shadow: 0 12px 28px rgba(148, 163, 184, 0.12);
}

.user .bubble {
  border-top-right-radius: 6px;
  background: linear-gradient(135deg, #1d4ed8 0%, #0ea5e9 100%);
  color: #fff;
  box-shadow: 0 14px 28px rgba(37, 99, 235, 0.2);
}

.thinking-bubble {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #0f766e;
  font-weight: 600;
}

.dot-ani {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #0ea5e9;
  animation: dotFlash 1.4s infinite;
}

.dot-ani:nth-child(2) {
  animation-delay: 0.2s;
}

.dot-ani:nth-child(3) {
  animation-delay: 0.4s;
}

.footer {
  padding: 18px 24px 22px;
  border-top: 1px solid #e5edf5;
  background: rgba(255, 255, 255, 0.92);
}

.input-wrapper {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 10px 10px 18px;
  border: 1px solid #d8e2ef;
  border-radius: 22px;
  background: #f8fafc;
  transition:
    border-color 0.24s ease,
    box-shadow 0.24s ease,
    background 0.24s ease;
}

.input-wrapper:focus-within {
  border-color: #7dd3fc;
  background: #fff;
  box-shadow: 0 0 0 4px rgba(14, 165, 233, 0.1);
}

input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  color: #0f172a;
  font-size: 15px;
}

input::placeholder {
  color: #94a3b8;
}

.send-btn {
  height: 44px;
  padding: 0 22px;
  border: none;
  border-radius: 16px;
  background: linear-gradient(135deg, #f97316, #ea580c);
  color: #fff;
  font-weight: 700;
  cursor: pointer;
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease,
    opacity 0.2s ease;
  box-shadow: 0 12px 24px rgba(234, 88, 12, 0.24);
}

.send-btn:hover {
  transform: translateY(-1px);
}

.send-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.main-scroll::-webkit-scrollbar {
  width: 8px;
}

.main-scroll::-webkit-scrollbar-thumb {
  border-radius: 999px;
  background: rgba(148, 163, 184, 0.72);
}

.main-scroll::-webkit-scrollbar-track {
  background: transparent;
}

.ui-cards-wrapper {
  display: flex;
  flex-direction: column;
  gap: 14px;
  margin-top: 14px;
  animation: cardSlideUp 0.45s ease both;
}

.service-card {
  overflow: hidden;
  border: 1px solid rgba(203, 213, 225, 0.7);
  border-radius: 18px;
  background: #fff;
  box-shadow: 0 16px 30px rgba(148, 163, 184, 0.16);
}

.card-image {
  width: 100%;
  height: 190px;
  overflow: hidden;
  background: #e2e8f0;
}

.card-image img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.card-body {
  padding: 16px;
}

.card-name {
  margin: 0 0 10px;
  color: #0f172a;
  font-size: 18px;
}

.card-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 10px;
}

.tag-item {
  padding: 4px 10px;
  border-radius: 999px;
  background: #eff6ff;
  color: #2563eb;
  font-size: 12px;
}

.card-desc {
  margin: 0;
  color: #475569;
  font-size: 14px;
  line-height: 1.65;
}

@media (max-width: 1180px) {
  .app-container {
    grid-template-columns: 1fr;
    padding: 18px;
  }

  .service-panel {
    gap: 16px;
    padding: 22px;
  }

  .chat-shell {
    height: auto;
    min-height: calc(100vh - 280px);
  }
}

@media (max-width: 768px) {
  .app-container {
    padding: 0;
    gap: 0;
    background: linear-gradient(180deg, #eff5fb 0%, #ffffff 100%);
  }

  .bg-grid,
  .bg-glow {
    display: none;
  }

  .service-panel {
    border: none;
    border-radius: 0;
    box-shadow: none;
    padding: 20px 16px 16px;
  }

  .service-panel h1 {
    font-size: 28px;
  }

  .chat-shell {
    height: auto;
    min-height: 0;
    border: none;
    border-radius: 24px 24px 0 0;
    box-shadow: none;
  }

  .header {
    padding: 18px 16px;
  }

  .header-status {
    display: none;
  }

  .main-scroll {
    padding: 16px;
  }

  .bubble-container {
    max-width: 88%;
  }

  .footer {
    padding: 14px 16px calc(14px + env(safe-area-inset-bottom));
  }

  .input-wrapper {
    padding-left: 14px;
  }
}

@media (max-width: 480px) {
  .promise-list {
    gap: 10px;
  }

  .quick-item,
  .notice-strip,
  .bubble,
  input,
  .send-btn {
    font-size: 14px;
  }

  .avatar {
    width: 36px;
    height: 36px;
  }

  .bubble-container {
    max-width: 92%;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes dotFlash {
  0%,
  100% {
    opacity: 0.25;
  }

  50% {
    opacity: 1;
  }
}

@keyframes cardSlideUp {
  from {
    opacity: 0;
    transform: translateY(14px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
