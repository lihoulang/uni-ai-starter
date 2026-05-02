<template>
  <view class="chat-page">
    <view class="sidebar-mask" v-if="showSidebar" @click="showSidebar = false"></view>

    <view class="sidebar" :class="{ 'sidebar-open': showSidebar }">
      <view class="sidebar-header">
        <text class="sidebar-title">历史会话</text>
        <view class="sidebar-new-btn" @click="createConversation">+ 新对话</view>
      </view>
      <view class="sidebar-search">
        <input class="sidebar-search-input" type="text" v-model="searchKeyword" placeholder="搜索会话..." />
      </view>
      <scroll-view class="sidebar-list" scroll-y>
        <view
          class="sidebar-item"
          v-for="conv in filteredConversations"
          :key="conv.id"
          :class="{ 'sidebar-item-active': conv.id === currentConvId }"
          @click="switchConversation(conv.id)"
        >
          <view class="sidebar-copy">
            <text class="sidebar-item-title">{{ conv.title }}</text>
            <text class="sidebar-item-date">{{ conv.created_at }}</text>
          </view>
          <view class="sidebar-actions">
            <text class="sidebar-action" @click.stop="togglePin(conv.id)">{{ conv.pinned ? '📌' : '📍' }}</text>
            <text class="sidebar-action sidebar-action-delete" @click.stop="deleteConversation(conv.id)">✕</text>
          </view>
        </view>
      </scroll-view>
    </view>

    <view class="topbar">
      <view class="menu-btn" @click="showSidebar = !showSidebar">
        <view class="menu-line"></view>
        <view class="menu-line"></view>
        <view class="menu-line"></view>
      </view>
      <view class="header-copy" @click="showModelPicker = !showModelPicker">
        <text class="header-title">AI Chat</text>
        <text class="header-subtitle">{{ currentModelLabel }}</text>
      </view>
      <view class="balance-pill" @click="goProfile">
        <text class="balance-icon">⚡</text>
        <text class="balance-value">{{ headerBalance }}</text>
      </view>
    </view>

    <view class="model-sheet" v-if="showModelPicker">
      <view
        class="model-row"
        v-for="m in modelOptions"
        :key="m.id"
        :class="{ 'model-row-active': currentModel === m.id }"
        @click="switchModel(m.id)"
      >
        <text class="model-icon">{{ m.icon }}</text>
        <view class="model-copy">
          <text class="model-name">{{ m.name }}</text>
          <text class="model-desc">{{ m.desc }}</text>
        </view>
        <text class="model-check" v-if="currentModel === m.id">✓</text>
      </view>
    </view>

    <scroll-view class="message-scroll" scroll-y :scroll-top="scrollTop" scroll-with-animation @click="closeMenu">
      <view class="message-stack">
        <view
          class="message-row"
          v-for="(msg, index) in messages"
          :key="index"
          :class="msg.role === 'user' ? 'message-row-user' : 'message-row-ai'"
          v-show="msg.role !== 'system'"
        >
          <view class="message-avatar" v-if="msg.role === 'ai'">AI</view>
          <view class="message-bubble-wrap">
            <view class="message-bubble" :class="msg.role === 'user' ? 'message-bubble-user' : 'message-bubble-ai'" @longpress="onLongPress(index, msg)">
              <image v-if="msg.image && msg.role === 'user'" :src="msg.image" class="chat-img" mode="widthFix" />
              <image v-if="msg.aiImage" :src="msg.aiImage" class="chat-img" mode="widthFix" />
              <video v-if="msg.video" :src="msg.video" controls class="chat-video"></video>
              <view v-if="msg.role === 'ai' && !msg.content && isLoading" class="typing-indicator">
                <view class="dot"></view>
                <view class="dot"></view>
                <view class="dot"></view>
              </view>
              <rich-text v-if="msg.content" :nodes="renderMarkdown(msg.content)"></rich-text>
            </view>
            <view class="longpress-menu" v-if="menuIndex === index" @click.stop>
              <view class="menu-opt" @click="copyText(msg.content); closeMenu();">复制</view>
              <view class="menu-opt" v-if="msg.role === 'ai'" @click="reportMessage(msg); closeMenu();">举报</view>
              <view class="menu-opt menu-danger" @click="deleteMsg(index)">删除</view>
            </view>
          </view>
        </view>
      </view>
    </scroll-view>

    <view class="input-panel">
      <view class="input-shell">
        <input class="chat-input" type="text" v-model="inputText" placeholder="输入消息..." @confirm="sendMessage" />
        <view class="input-actions">
          <button class="image-btn" @click="chooseImage">＋</button>
          <view class="send-btn" @click="sendMessage" :class="{ 'send-btn-disabled': isLoading }">
            <text class="send-icon">➤</text>
          </view>
        </view>
      </view>
      <view class="image-preview" v-if="selectedImagePreview">
        <image :src="selectedImagePreview" mode="aspectFill" class="preview-img" />
        <view class="remove-img-btn" @click="removeImage">×</view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { computed, nextTick, ref } from 'vue';
import { onLoad, onShow } from '@dcloudio/uni-app';
import { marked } from 'marked';
import { API_BASE } from '../../config.js';
import { ensureLoggedIn, getAuthHeaders, redirectToLogin, requestWithAuth } from '../../utils/auth.js';

const inputText = ref('');
const isLoading = ref(false);
const scrollTop = ref(0);
const userId = ref(null);
const selectedImagePreview = ref('');
const selectedImageBase64 = ref(null);
const showSidebar = ref(false);
const conversations = ref([]);
const currentConvId = ref(0);
const searchKeyword = ref('');
const menuIndex = ref(-1);
const headerBalance = ref(0);
const showModelPicker = ref(false);
const currentModel = ref('qwen');

const modelOptions = [
  { id: 'deepseek', icon: '🧠', name: 'DeepSeek', desc: '强推理、写代码' },
  { id: 'qwen', icon: '🌟', name: 'Qwen-Max', desc: '通义千问，均衡全能' },
  { id: 'doubao', icon: '🚀', name: 'Doubao', desc: '火山引擎，响应更快' },
  { id: 'gemini', icon: '✦', name: 'Gemini', desc: 'Google 模型，综合能力强' },
];

const currentModelLabel = computed(() => {
  const current = modelOptions.find((item) => item.id === currentModel.value);
  return current ? current.name : '';
});

const filteredConversations = computed(() => {
  const kw = searchKeyword.value.trim().toLowerCase();
  if (!kw) return conversations.value;
  return conversations.value.filter((conv) => conv.title.toLowerCase().includes(kw));
});

const renderer = new marked.Renderer();
renderer.code = function(tokenOrCode, language) {
  const codeText = typeof tokenOrCode === 'object' ? tokenOrCode.text : tokenOrCode;
  const lang = typeof tokenOrCode === 'object' ? (tokenOrCode.lang || 'text') : (language || 'text');
  const safe = (codeText || '').replace(/</g, '&lt;').replace(/>/g, '&gt;');
  const formatted = safe.replace(/\n/g, '<br/>').replace(/ /g, '&nbsp;');
  return `<div style="background:#1f2430;border-radius:10px;margin:10px 0;overflow:hidden;">
    <div style="padding:8px 12px;background:#2d3342;color:#b7bfcd;font-size:12px;">${lang}</div>
    <div style="padding:12px 14px;color:#edf1f7;font-family:Consolas,monospace;font-size:13px;line-height:1.7;">${formatted}</div>
  </div>`;
};
marked.setOptions({ renderer });

const renderMarkdown = (text) => {
  if (!text) return '';
  let content = text.replace(/\[IMAGE:[^\]]*$/, '').replace(/\[VIDEO:[^\]]*$/, '').replace(/@@@[\s\S]*?(@@@|$)/g, '');
  let html = marked.parse(content);
  html = html.replace(/<table/g, '<table style="border-collapse:collapse;width:100%;margin:10px 0;font-size:13px;"');
  html = html.replace(/<th/g, '<th style="border:1px solid #ebeef5;padding:8px;background:#f6f7fb;text-align:left;"');
  html = html.replace(/<td/g, '<td style="border:1px solid #ebeef5;padding:8px;"');
  return html;
};

const systemPrompt = { role: 'system', content: 'You are AI Assistant.' };
const welcomeMsg = { role: 'ai', content: '你好！有什么我可以帮助你的吗？' };
const messages = ref([systemPrompt, { ...welcomeMsg }]);

const scrollToBottom = () => {
  nextTick(() => {
    scrollTop.value = scrollTop.value === 99999 ? 99998 : 99999;
  });
};

const switchModel = (id) => {
  currentModel.value = id;
  showModelPicker.value = false;
  uni.showToast({ title: `已切换：${modelOptions.find((o) => o.id === id).name}`, icon: 'none' });
};

const onLongPress = (index, msg) => {
  if (msg.role === 'system' || !msg.content) return;
  menuIndex.value = index;
};

const closeMenu = () => {
  menuIndex.value = -1;
};

const deleteMsg = (index) => {
  messages.value.splice(index, 1);
  closeMenu();
};

const copyText = (text) => {
  if (!text) return;
  uni.setClipboardData({ data: text, success: () => uni.showToast({ title: '已复制', icon: 'success' }) });
};

const normalizeAIMessage = (item) => {
  if (!item?.content) return item;
  const videoMatch = item.content.match(/\[VIDEO:([\s\S]+?)\]/);
  if (videoMatch) {
    item.video = videoMatch[1].trim();
    item.content = item.content.replace(/\[VIDEO:[\s\S]+?\]/g, '').trim();
  }
  const imageMatch = item.content.match(/\[IMAGE:([\s\S]+?)\]/);
  if (imageMatch) {
    item.aiImage = imageMatch[1].trim();
    item.content = item.content.replace(/\[IMAGE:[\s\S]+?\]/g, '').trim();
  }
  item.content = item.content.replace(/@@@[\s\S]*?(@@@|$)/g, '').trim();
  return item;
};

const fileToBase64 = (filePath) => new Promise((resolve, reject) => {
  // #ifdef H5
  const reader = new FileReader();
  fetch(filePath)
    .then((r) => r.blob())
    .then((blob) => {
      reader.onload = () => resolve(reader.result);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    })
    .catch(reject);
  // #endif

  // #ifndef H5
  const fs = uni.getFileSystemManager();
  fs.readFile({
    filePath,
    encoding: 'base64',
    success: (res) => resolve(`data:image/jpeg;base64,${res.data}`),
    fail: reject,
  });
  // #endif
});

const chooseImage = () => {
  uni.chooseImage({
    count: 1,
    sizeType: ['compressed'],
    success: async (res) => {
      selectedImagePreview.value = res.tempFilePaths[0];
      try {
        selectedImageBase64.value = await fileToBase64(res.tempFilePaths[0]);
      } catch (e) {
        selectedImagePreview.value = '';
        selectedImageBase64.value = null;
        uni.showToast({ title: '图片读取失败', icon: 'none' });
      }
    },
  });
};

const removeImage = () => {
  selectedImagePreview.value = '';
  selectedImageBase64.value = null;
};

const fetchBalance = async () => {
  if (!userId.value) return;
  try {
    const res = await requestWithAuth({ url: `${API_BASE}/user/balance/${userId.value}` });
    if (res.data.code === 200) headerBalance.value = res.data.balance;
  } catch (e) {}
};

const goProfile = () => {
  uni.switchTab({ url: '/pages/profile/profile' });
};

const loadConversations = async () => {
  try {
    const res = await requestWithAuth({ url: `${API_BASE}/conversations/${userId.value}` });
    if (res.data.code === 200) conversations.value = res.data.data;
  } catch (e) {}
};

const createConversation = async () => {
  try {
    const res = await requestWithAuth({ url: `${API_BASE}/conversation/create/${userId.value}`, method: 'POST' });
    if (res.data.code === 200) {
      currentConvId.value = res.data.conversation_id;
      messages.value = [systemPrompt, { ...welcomeMsg }];
      await loadConversations();
      showSidebar.value = false;
    }
  } catch (e) {}
};

const switchConversation = async (convId) => {
  currentConvId.value = convId;
  showSidebar.value = false;
  messages.value = [systemPrompt];
  await loadHistory();
};

const togglePin = async (convId) => {
  try {
    await requestWithAuth({ url: `${API_BASE}/conversation/pin/${convId}`, method: 'PUT' });
    await loadConversations();
  } catch (e) {}
};

const deleteConversation = async (convId) => {
  uni.showModal({
    title: '删除会话',
    content: '确认删除？',
    success: async (res) => {
      if (!res.confirm) return;
      await requestWithAuth({ url: `${API_BASE}/conversation/${convId}`, method: 'DELETE' });
      await loadConversations();
      if (convId === currentConvId.value) {
        conversations.value.length > 0 ? await switchConversation(conversations.value[0].id) : await createConversation();
      }
    },
  });
};

const loadHistory = async () => {
  try {
    const res = await requestWithAuth({ url: `${API_BASE}/history/${userId.value}?conversation_id=${currentConvId.value}` });
    if (res.data.code === 200 && res.data.data.length > 0) {
      const data = res.data.data.map((item) => {
        if (item.role === 'assistant') item.role = 'ai';
        return normalizeAIMessage(item);
      });
      messages.value = [systemPrompt, ...data];
      scrollToBottom();
    } else {
      messages.value = [systemPrompt, { ...welcomeMsg }];
    }
  } catch (e) {}
};

const reportMessage = (msg) => {
  uni.showActionSheet({
    itemList: ['色情/裸露', '暴力/违法', '仇恨/骚扰', '事实错误/误导'],
    success: async ({ tapIndex }) => {
      const reason = ['色情/裸露', '暴力/违法', '仇恨/骚扰', '事实错误/误导'][tapIndex];
      try {
        const res = await requestWithAuth({
          url: `${API_BASE}/ai/report`,
          method: 'POST',
          data: {
            user_id: userId.value,
            conversation_id: currentConvId.value,
            reason,
            message_content: msg.content || '',
          },
        });
        uni.showToast({ title: res.data.msg || '已提交', icon: 'none' });
      } catch (e) {}
    },
  });
};

const sendMessage = async () => {
  const text = inputText.value.trim();
  if ((!text && !selectedImageBase64.value) || isLoading.value) return;

  const userMsg = { role: 'user', content: text, image: selectedImagePreview.value || null };
  messages.value.push(userMsg);
  inputText.value = '';
  const imgBase64 = selectedImageBase64.value;
  removeImage();

  messages.value.push({ role: 'ai', content: '' });
  isLoading.value = true;
  scrollToBottom();
  const aiIndex = messages.value.length - 1;

  try {
    // #ifdef H5
    const response = await fetch(`${API_BASE}/chat`, {
      method: 'POST',
      headers: getAuthHeaders({ 'Content-Type': 'application/json' }),
      body: JSON.stringify({
        user_id: userId.value,
        message: text,
        image_base64: imgBase64,
        conversation_id: currentConvId.value,
        model: currentModel.value,
      }),
    });

    if (response.status === 401) {
      redirectToLogin();
      return;
    }
    if (!response.ok || !response.body) throw new Error('REQUEST_FAILED');

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value, { stream: true });
      messages.value[aiIndex].content += chunk;
      normalizeAIMessage(messages.value[aiIndex]);
      scrollToBottom();
    }
    // #endif

    // #ifndef H5
    const response = await requestWithAuth({
      url: `${API_BASE}/chat`,
      method: 'POST',
      header: { 'Content-Type': 'application/json' },
      data: {
        user_id: userId.value,
        message: text,
        image_base64: imgBase64,
        conversation_id: currentConvId.value,
        model: currentModel.value,
      },
    });
    messages.value[aiIndex].content = typeof response.data === 'string' ? response.data : '服务返回异常';
    normalizeAIMessage(messages.value[aiIndex]);
    // #endif

    normalizeAIMessage(messages.value[aiIndex]);
    fetchBalance();
  } catch (e) {
    messages.value[aiIndex].content = '网络错误，请检查后端服务是否运行。';
  } finally {
    isLoading.value = false;
    scrollToBottom();
  }
};

onLoad(() => {
  if (!ensureLoggedIn()) return;
  const storedId = uni.getStorageSync('user_id');
  if (!storedId) {
    redirectToLogin('请先登录');
    return;
  }
  userId.value = storedId;
  initApp();
});

onShow(() => {
  if (!ensureLoggedIn()) return;
  fetchBalance();
  const pending = uni.getStorageSync('pending_prompt');
  if (pending) {
    uni.removeStorageSync('pending_prompt');
    inputText.value = pending;
    nextTick(() => sendMessage());
  }
});

const initApp = async () => {
  await fetchBalance();
  await loadConversations();
  if (conversations.value.length > 0) {
    currentConvId.value = conversations.value[0].id;
    await loadHistory();
  } else {
    await createConversation();
  }
};
</script>

<style scoped>
.chat-page {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #fff;
  display: flex;
  flex-direction: column;
  padding-bottom: calc(56px + env(safe-area-inset-bottom));
}

.sidebar-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.28);
  z-index: 40;
}

.sidebar {
  position: fixed;
  top: 0;
  left: -280px;
  width: 280px;
  height: 100vh;
  background: #fff;
  border-right: 1px solid #eceff4;
  box-shadow: 6px 0 18px rgba(32, 43, 67, 0.06);
  transition: left 0.28s ease;
  z-index: 41;
  display: flex;
  flex-direction: column;
}

.sidebar-open {
  left: 0;
}

.sidebar-header {
  height: 60px;
  padding: 8px 16px 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #eceff4;
}

.sidebar-title {
  font-size: 16px;
  font-weight: 600;
  color: #232939;
}

.sidebar-new-btn {
  font-size: 12px;
  color: #6670e8;
  border: 1px solid #d7dcfb;
  border-radius: 999px;
  padding: 6px 10px;
}

.sidebar-search {
  padding: 12px 16px;
}

.sidebar-search-input {
  width: 100%;
  height: 36px;
  border-radius: 12px;
  background: #f6f8fb;
  padding: 0 12px;
  font-size: 13px;
  color: #232939;
}

.sidebar-list {
  flex: 1;
}

.sidebar-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  border-bottom: 1px solid #f0f2f6;
}

.sidebar-item-active {
  background: #f8f9fc;
}

.sidebar-copy {
  flex: 1;
  min-width: 0;
}

.sidebar-item-title {
  display: block;
  color: #232939;
  font-size: 14px;
  font-weight: 500;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.sidebar-item-date {
  display: block;
  margin-top: 4px;
  color: #9aa3b3;
  font-size: 11px;
}

.sidebar-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.sidebar-action {
  color: #a6adbb;
  font-size: 14px;
}

.sidebar-action-delete {
  color: #c7ceda;
}

.topbar {
  height: 60px;
  padding: 8px 14px 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #fff;
  border-bottom: 1px solid #eceff4;
  flex-shrink: 0;
}

.menu-btn {
  width: 36px;
  height: 36px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
}

.menu-line {
  width: 15px;
  height: 2px;
  border-radius: 2px;
  background: #61697b;
}

.header-copy {
  flex: 1;
  min-width: 0;
  text-align: center;
}

.header-title {
  display: block;
  font-size: 18px;
  font-weight: 700;
  color: #1f2430;
}

.header-subtitle {
  display: block;
  margin-top: 2px;
  font-size: 12px;
  color: #9aa3b3;
}

.balance-pill {
  min-width: 70px;
  height: 34px;
  padding: 0 12px;
  border-radius: 10px;
  border: 1px solid #dde3ec;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.balance-icon,
.balance-value {
  color: #2b3140;
  font-size: 14px;
  font-weight: 600;
}

.model-sheet {
  padding: 8px 14px 10px;
  background: #fff;
  border-bottom: 1px solid #eceff4;
  flex-shrink: 0;
}

.model-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 14px;
}

.model-row-active {
  background: #f7f8fc;
}

.model-icon {
  font-size: 18px;
}

.model-copy {
  flex: 1;
}

.model-name {
  display: block;
  font-size: 14px;
  color: #232939;
  font-weight: 600;
}

.model-desc {
  display: block;
  margin-top: 3px;
  font-size: 12px;
  color: #99a1b0;
}

.model-check {
  color: #6670e8;
  font-size: 14px;
  font-weight: 700;
}

.message-scroll {
  flex: 1;
  min-height: 0;
  background: #fff;
}

.message-stack {
  padding: 18px 16px 24px;
}

.message-row {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 16px;
}

.message-row-user {
  justify-content: flex-end;
}

.message-avatar {
  width: 30px;
  height: 30px;
  flex-shrink: 0;
  border-radius: 50%;
  background: linear-gradient(135deg, #6670e8 0%, #5d62c9 100%);
  color: #fff;
  font-size: 13px;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 2px;
}

.message-bubble-wrap {
  position: relative;
  max-width: 84%;
}

.message-bubble {
  border-radius: 16px;
  padding: 14px 16px;
  font-size: 15px;
  line-height: 1.65;
  word-break: break-word;
}

.message-bubble-ai {
  background: #fff;
  border: 1px solid #dfe4ee;
  color: #232939;
}

.message-bubble-user {
  background: linear-gradient(135deg, #6670e8 0%, #5d62c9 100%);
  color: #fff;
}

.chat-img,
.chat-video {
  max-width: 100%;
  border-radius: 12px;
  margin: 6px 0;
}

.typing-indicator {
  display: flex;
  gap: 4px;
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #b3b9c5;
  animation: bounce 1.2s infinite ease-in-out;
}

.dot:nth-child(2) {
  animation-delay: 0.15s;
}

.dot:nth-child(3) {
  animation-delay: 0.3s;
}

@keyframes bounce {
  0%, 80%, 100% { transform: scale(0.5); opacity: 0.6; }
  40% { transform: scale(1); opacity: 1; }
}

.longpress-menu {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  min-width: 92px;
  background: #fff;
  border: 1px solid #e6e9ef;
  border-radius: 12px;
  box-shadow: 0 10px 18px rgba(32, 43, 67, 0.08);
  overflow: hidden;
  z-index: 8;
}

.menu-opt {
  padding: 10px 14px;
  color: #232939;
  font-size: 13px;
  border-bottom: 1px solid #f1f3f7;
}

.menu-opt:last-child {
  border-bottom: none;
}

.menu-danger {
  color: #fa6a67;
}

.input-panel {
  padding: 14px 14px 16px;
  background: #fff;
  border-top: 1px solid #eceff4;
  flex-shrink: 0;
}

.input-shell {
  height: 50px;
  border-radius: 16px;
  border: 1px solid #dfe4ee;
  background: #fff;
  display: flex;
  align-items: center;
  padding: 0 6px 0 14px;
}

.chat-input {
  flex: 1;
  height: 100%;
  font-size: 15px;
  color: #232939;
}

.input-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.image-btn {
  width: 34px;
  height: 34px;
  border: none;
  background: transparent;
  color: #8892a4;
  font-size: 24px;
  line-height: 1;
  padding: 0;
}

.send-btn {
  width: 34px;
  height: 34px;
  border-radius: 10px;
  background: linear-gradient(135deg, #6670e8 0%, #5d62c9 100%);
  display: flex;
  align-items: center;
  justify-content: center;
}

.send-btn-disabled {
  opacity: 0.5;
}

.send-icon {
  color: #fff;
  font-size: 16px;
}

.image-preview {
  margin-top: 10px;
  display: inline-flex;
  align-items: flex-start;
  position: relative;
}

.preview-img {
  width: 60px;
  height: 60px;
  border-radius: 12px;
}

.remove-img-btn {
  position: absolute;
  top: -6px;
  right: -6px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #fa6a67;
  color: #fff;
  font-size: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
