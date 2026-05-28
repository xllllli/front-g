<script setup>
import { ref, reactive, nextTick } from 'vue';
import MarkdownIt from 'markdown-it';

const md = new MarkdownIt();
const chatBody = ref(null);

const rawApiBaseUrl = (import.meta.env.VITE_API_BASE_URL || '').trim();
const normalizedApiBaseUrl = rawApiBaseUrl.replace(/\/$/, '');
const chatApiUrl = normalizedApiBaseUrl ? `${normalizedApiBaseUrl}/chat` : '/api/chat';

const messages = ref([
  {
    role: 'assistant',
    content: '你好！我是智旅通 AI 助手，有什么可以帮你的？',
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
  }, 30);
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
  const chatWindow = document.querySelector('.chat-window');
  if (chatWindow) {
    chatWindow.style.background =
      color === 'black' ? 'rgba(0, 0, 0, 0.95)' : `rgba(${getRgbFromColor(color)}, 0.96)`;
  }

  const mainScroll = document.querySelector('.main-scroll');
  if (mainScroll) {
    mainScroll.style.background =
      color === 'black' ? 'rgba(0, 0, 0, 0.3)' : `rgba(${getRgbFromColor(color)}, 0.3)`;
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
  const chatWindow = document.querySelector('.chat-window');
  if (chatWindow) {
    chatWindow.style.background = 'rgba(255, 255, 255, 0.96)';
  }

  const mainScroll = document.querySelector('.main-scroll');
  if (mainScroll) {
    mainScroll.style.background = 'rgba(249, 251, 255, 0.5)';
  }

  const bubbles = document.querySelectorAll('.assistant .bubble');
  bubbles.forEach((bubble) => {
    bubble.style.color = '#333';
  });

  document.documentElement.style.setProperty('--chat-bg', 'rgba(249, 251, 255, 0.5)');
  document.documentElement.style.setProperty('--chat-text', '#333');
}

function getRgbFromColor(color) {
  const colorMap = {
    black: '0,0,0',
    white: '255,255,255',
    red: '255,0,0',
    blue: '0,0,255',
    green: '0,255,0',
    yellow: '255,255,0',
    purple: '128,0,128',
    pink: '255,192,203'
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
      assistantMsg.content = '未收到后端返回内容，请检查接口返回格式。';
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
      throw new Error(`HTTP ${response.status} ${response.statusText}`);
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
    assistantMsg.content = `连接失败，请检查接口地址或后端返回格式。${error.message ? ` (${error.message})` : ''}`;
    console.error('Fetch Error:', error);
  }
};
</script>

<template>
  <div class="app-container">
    <div class="bg-decoration top-left"></div>
    <div class="bg-decoration bottom-right"></div>

    <div class="inspiration-sidebar">
      <div class="section-title">
        <span class="icon">✨</span>
        <span>探索灵感</span>
      </div>
      <div class="tag-cloud">
        <div class="tag-card" @click="userInput = '推荐一个南昌适合看日落的地方'">南昌日落</div>
        <div class="tag-card" @click="userInput = '武功山两天一夜攻略'">武功山攻略</div>
        <div class="tag-card" @click="userInput = '景德镇陶瓷厂怎么玩？'">景德镇陶瓷</div>
        <div class="tag-card" @click="userInput = '婺源现在油菜花开了吗？'">婺源花海</div>
        <div class="tag-card" @click="userInput = '江西有哪些不累的避暑胜地？'">轻松避暑</div>
      </div>
    </div>

    <div class="chat-window">
      <header class="header">
        <div class="header-info">
          <span class="status-dot"></span>
          <div class="title-group">
            <strong class="main-title">智旅通 · 江西金牌向导</strong>
            <span class="sub-title">基于实时气象与 RAG 的行程规划</span>
          </div>
        </div>
      </header>

      <main ref="chatBody" class="main-scroll">
        <div v-for="(msg, i) in messages" :key="i" :class="['message-row', msg.role]">
          <div v-if="msg.role === 'assistant'" class="avatar assistant-avatar">AI</div>

          <div class="bubble-container">
            <div class="bubble">
              <div v-if="msg.role === 'assistant' && msg.thinking" class="thinking-bubble">
                <span>正在为你规划行程</span>
                <div class="dot-ani"></div>
                <div class="dot-ani"></div>
                <div class="dot-ani"></div>
              </div>

              <div v-else class="markdown-body" v-html="md.render(msg.content)"></div>

              <div v-if="msg.uiData && msg.uiData.length > 0" class="ui-cards-wrapper">
                <div v-for="(card, index) in msg.uiData" :key="index" class="card-item">
                  <div v-if="card.type === 'scenery-card'" class="scenery-card">
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
            placeholder="输入目的地，开启你的江西之旅..."
            @keyup.enter="sendMessage"
          />
          <button class="send-btn" :disabled="!userInput.trim()" @click="sendMessage">发送</button>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  background: #f0f4f8 linear-gradient(135deg, #eef2f3 0%, #8e9eab 100%);
  font-family: "PingFang SC", "Helvetica Neue", Helvetica, Arial, sans-serif;
}

.bg-decoration {
  position: absolute;
  z-index: 0;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.3;
  pointer-events: none;
}

.top-left {
  top: -10%;
  left: -5%;
  width: 400px;
  height: 400px;
  background: #1890ff;
}

.bottom-right {
  right: -5%;
  bottom: -10%;
  width: 500px;
  height: 500px;
  background: #52c41a;
}

.inspiration-sidebar {
  position: absolute;
  top: 20%;
  left: 40px;
  z-index: 10;
  display: flex;
  flex-direction: column;
  gap: 15px;
  width: 200px;
}

@media (max-width: 1150px) {
  .inspiration-sidebar {
    display: none;
  }
}

.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 5px;
  font-size: 16px;
  font-weight: 600;
  color: #2c3e50;
}

.tag-cloud {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.tag-card {
  padding: 12px 16px;
  border: 1px solid rgba(255, 255, 255, 0.5);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.8);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.02);
  color: #555;
  cursor: pointer;
  backdrop-filter: blur(10px);
  transition: all 0.3s ease;
}

.tag-card:hover {
  transform: translateX(10px);
  border-color: #1890ff;
  background: #fff;
  box-shadow: 0 8px 15px rgba(24, 144, 255, 0.1);
  color: #1890ff;
}

.chat-window {
  position: relative;
  z-index: 20;
  display: flex;
  flex-direction: column;
  width: 90%;
  max-width: 850px;
  height: 85vh;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.4);
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.12);
  backdrop-filter: blur(20px);
}

.header {
  display: flex;
  align-items: center;
  flex-shrink: 0;
  height: 70px;
  padding: 0 25px;
  border-bottom: 1px solid #f0f0f0;
  background: #fff;
}

.header-info {
  display: flex;
  align-items: center;
  gap: 12px;
}

.status-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #52c41a;
  box-shadow: 0 0 8px #52c41a;
}

.title-group {
  display: flex;
  flex-direction: column;
}

.main-title {
  font-size: 17px;
  font-weight: 600;
  color: #333;
}

.sub-title {
  font-size: 12px;
  color: #999;
}

.main-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 25px;
  background: rgba(249, 251, 255, 0.5);
  scroll-behavior: smooth;
}

.message-row {
  display: flex;
  align-items: flex-start;
  margin-bottom: 20px;
  animation: fadeIn 0.4s ease;
}

.message-row.user {
  justify-content: flex-end;
}

.message-row.assistant {
  justify-content: flex-start;
}

.avatar {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  font-size: 14px;
  font-weight: 600;
}

.assistant-avatar {
  margin-right: 12px;
  background: #e6f7ff;
}

.user-avatar {
  margin-left: 12px;
  background: #f0f0f0;
}

.bubble-container {
  max-width: 75%;
}

.bubble {
  padding: 12px 18px;
  border-radius: 16px;
  font-size: 15px;
  line-height: 1.6;
  word-wrap: break-word;
}

.assistant .bubble {
  border: 1px solid #ebedf0;
  border-top-left-radius: 4px;
  background: #fff;
  color: #333;
}

.user .bubble {
  border-top-right-radius: 4px;
  background: linear-gradient(135deg, #1890ff 0%, #096dd9 100%);
  box-shadow: 0 4px 12px rgba(24, 144, 255, 0.2);
  color: #fff;
}

.thinking-bubble {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #1890ff;
  font-weight: 500;
}

.dot-ani {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #1890ff;
  animation: dotFlash 1.4s infinite;
}

.dot-ani:nth-child(2) {
  animation-delay: 0.2s;
}

.dot-ani:nth-child(3) {
  animation-delay: 0.4s;
}

.footer {
  padding: 20px 25px;
  border-top: 1px solid #f0f0f0;
  background: #fff;
}

.input-wrapper {
  display: flex;
  align-items: center;
  padding: 8px 8px 8px 20px;
  border-radius: 30px;
  background: #f4f5f7;
  transition: all 0.3s;
}

.input-wrapper:focus-within {
  background: #fff;
  box-shadow: 0 0 0 2px #1890ff;
}

input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  font-size: 15px;
  color: #333;
}

.send-btn {
  height: 40px;
  padding: 0 22px;
  border: none;
  border-radius: 20px;
  background: #1890ff;
  color: #fff;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.send-btn:hover {
  background: #40a9ff;
  transform: scale(1.05);
}

.send-btn:disabled {
  background: #bfbfbf;
  cursor: not-allowed;
  transform: none;
}

.main-scroll::-webkit-scrollbar {
  width: 6px;
}

.main-scroll::-webkit-scrollbar-thumb {
  border-radius: 10px;
  background: #e0e0e0;
}

.main-scroll::-webkit-scrollbar-track {
  background: transparent;
}

.ui-cards-wrapper {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-top: 14px;
  animation: cardSlideUp 0.5s cubic-bezier(0.25, 0.46, 0.45, 0.94) both;
}

.scenery-card {
  max-width: 100%;
  overflow: hidden;
  border: 1px solid rgba(0, 0, 0, 0.04);
  border-radius: 18px;
  background: #fff;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.05);
  transition: all 0.3s ease;
}

.scenery-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 15px 35px rgba(24, 144, 255, 0.1);
}

.card-image {
  position: relative;
  width: 100%;
  height: 190px;
  overflow: hidden;
  background: #f0f2f5;
}

.card-image img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.5s ease;
}

.scenery-card:hover .card-image img {
  transform: scale(1.08);
}

.card-body {
  padding: 16px;
  text-align: left;
}

.card-name {
  margin: 0 0 10px;
  color: #262626;
  font-size: 18px;
  font-weight: 600;
  letter-spacing: 0.5px;
}

.card-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 12px;
}

.tag-item {
  padding: 4px 10px;
  border: 1px solid #bae7ff;
  border-radius: 6px;
  background: #e6f7ff;
  color: #1890ff;
  font-size: 12px;
  font-weight: 500;
}

.card-desc {
  margin: 0;
  color: #595959;
  font-size: 14px;
  line-height: 1.6;
}

@media (max-width: 1024px) {
  .chat-window {
    width: 95%;
    height: 90vh;
  }

  .bubble-container {
    max-width: 85%;
  }

  .inspiration-sidebar {
    left: 20px;
    width: 180px;
  }
}

@media (max-width: 768px) {
  .bg-decoration {
    display: none;
  }

  .app-container {
    align-items: flex-start;
    background: #fff;
  }

  .chat-window {
    width: 100%;
    max-width: none;
    height: 100vh;
    height: 100dvh;
    border: none;
    border-radius: 0;
    box-shadow: none;
  }

  .header {
    height: 60px;
    padding: 0 15px;
    background: #f8f9fa;
    border-bottom: 1px solid #eee;
  }

  .main-title {
    font-size: 15px;
  }

  .sub-title {
    font-size: 10px;
  }

  .main-scroll {
    padding: 16px;
  }

  .avatar {
    width: 32px;
    height: 32px;
    font-size: 12px;
  }

  .bubble {
    padding: 8px 12px;
    font-size: 14px;
  }

  .bubble-container {
    max-width: 90%;
  }

  .footer {
    padding: 10px 15px env(safe-area-inset-bottom);
  }

  .input-wrapper {
    padding: 4px 4px 4px 16px;
  }

  input {
    font-size: 12px;
  }

  .send-btn {
    height: 36px;
    padding: 0 16px;
    font-size: 14px;
  }

  .tag-card {
    padding: 8px 12px;
    font-size: 12px;
  }

  .card-image {
    height: 150px;
  }

  .card-name {
    font-size: 16px;
  }

  .card-desc {
    font-size: 13px;
  }
}

@media (max-width: 480px) {
  .bubble-container {
    max-width: 95%;
  }

  .message-row {
    margin-bottom: 12px;
  }

  .avatar {
    width: 28px;
    height: 28px;
  }

  .bubble {
    padding: 6px 10px;
    font-size: 13px;
  }

  .tag-card {
    padding: 6px 10px;
    font-size: 11px;
  }

  input {
    font-size: 11px;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(5px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes dotFlash {
  0%,
  100% {
    opacity: 0.2;
  }

  50% {
    opacity: 1;
  }
}

@keyframes cardSlideUp {
  from {
    opacity: 0;
    transform: translateY(15px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
