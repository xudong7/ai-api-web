<template>
  <div class="chat-container">
    <div class="character-select">
      <div
        v-for="char in characters"
        :key="char.id"
        :class="['character-option', { active: currentCharacter.id === char.id }]"
        @click="selectCharacter(char)"
      >
        <img :src="char.avatar" :alt="char.name">
        <span>{{ char.name }}</span>
      </div>
    </div>
    <div class="messages-container">
      <div
        v-for="message in messages"
        :key="message.id"
        :class="['message-wrapper', message.role]"
      >
        <div class="avatar">
          <img
            :src="
              message.role === 'user' ? userAvatarPath : assistantAvatarPath
            "
            :alt="message.role"
          />
        </div>
        <div class="message-content">
          <div v-if="message.role === 'user'" class="user-message">
            {{ message.content }}
          </div>
          <div
            v-else-if="message.role === 'assistant'"
            class="assistant-message"
            v-html="markdownToHtml(message.content)"
          ></div>
          <div v-else class="error-message">
            {{ message.content }}
          </div>
        </div>
      </div>
    </div>
    <div class="input-area">
      <input
        type="text"
        v-model="message"
        :disabled="isLoading"
        @keyup.enter="sendMessage(message)"
      />
      <button @click="sendMessage(message)" :disabled="isLoading">
        {{ isLoading ? "处理中..." : "发送" }}
      </button>
    </div>
  </div>
</template>

<script setup>
import OpenAI from "openai";
import { ref, computed, onUnmounted } from "vue";
import { marked } from "marked";
import { characters } from "/src/config/characters";

const apiKey = import.meta.env.VITE_DEEPSEEK_API_KEY;
const baseURL = import.meta.env.VITE_DEEPSEEK_BASE_URL;

const openai = new OpenAI({
  baseURL: baseURL,
  apiKey: apiKey,
  temperature: 1,
  dangerouslyAllowBrowser: true, // 允许在浏览器中使用
});

const message = ref("");
const messages = ref([]);
const isLoading = ref(false);
const currentCharacter = ref(characters[0]);
const userAvatarPath = "/src/assets/user-avatar.png";
const assistantAvatarPath = computed(() => currentCharacter.value.avatar);

// 添加取消控制器
let currentController = null;

const selectCharacter = (character) => {
  if (currentController) {
    currentController.abort(); // 中断当前请求
  }
  currentCharacter.value = character;
  messages.value = [];
  isLoading.value = false;
};

const sendMessage = async (content) => {
  try {
    // 检查输入是否为空
    if (!content.trim()) return;
    // 设置加载状态
    isLoading.value = true;

    // 创建新的控制器
    currentController = new AbortController();

    // 添加用户消息
    messages.value.push({
      id: messages.value.length,
      content: content,
      role: "user",
    });

    message.value = "";

    // 添加AI响应占位
    const aiMessageId = messages.value.length;
    messages.value.push({
      id: aiMessageId,
      content: "",
      role: "assistant",
    });

    const stream = await openai.chat.completions.create({
      messages: [
        { role: "system", content: currentCharacter.value.systemPrompt },
        { role: "user", content: content },
      ],
      model: "deepseek-chat",
      stream: true,
      signal: currentController.signal, // 添加控制器信号
    });

    // 处理流式响应
    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content || "";
      messages.value[aiMessageId].content += content;
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('请求被中断');
    } else {
      console.error("Error:", error);
      messages.value.push({
        id: messages.value.length,
        content: "抱歉，发生了错误，请稍后重试。",
        role: "error",
      });
    }
  } finally {
    isLoading.value = false;
    currentController = null;
  }
};

// 组件卸载时清理
onUnmounted(() => {
  if (currentController) {
    currentController.abort();
  }
});

// 添加 Markdown 转换函数
const markdownToHtml = (content) => {
  return marked(content);
};
</script>

<style>
/* 全局样式 */
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>

<style scoped>
.chat-container {
  max-width: 700px;
  margin: 0 auto;
  padding: 15px;
  height: 100vh;
  display: flex;
  flex-direction: column;
  font-size: 14px;
  box-sizing: border-box; /* 确保padding计入总高度 */
}

.input-area {
  display: flex;
  gap: 10px;
  padding: 20px 0;
  position: sticky;
  bottom: 0;
  background: white;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 5px 0;
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE and Edge */
}

.messages-container::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}

.message-wrapper {
  display: flex;
  gap: 12px;
  margin: 10px 0;
}

.message-wrapper.user {
  flex-direction: row-reverse;
}

.avatar {
  width: 40px;
  height: 40px;
  flex-shrink: 0;
}

.avatar img {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  object-fit: cover;
}

.message-content {
  max-width: 75%;
  font-size: 14px;
}

.user-message {
  background-color: #007aff;
  color: white;
  padding: 10px 14px;
  border-radius: 16px 16px 4px 16px;
  margin-right: 10px;
}

.assistant-message {
  background-color: #f5f5f5;
  padding: 10px 14px;
  border-radius: 16px 16px 16px 4px;
  margin-left: 10px;
}

.assistant-message :deep(pre) {
  background-color: #2d2d2d;
  color: #fff;
  padding: 10px;
  border-radius: 5px;
  overflow-x: auto;
  margin: 8px 0;
}

.error-message {
  background-color: #ffe6e6;
  color: #ff0000;
  padding: 12px 16px;
  border-radius: 20px;
}

input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 20px;
  font-size: 14px;
}

button {
  padding: 10px 20px;
  border-radius: 20px;
  border: none;
  background-color: #007aff;
  color: white;
  font-size: 14px;
  cursor: pointer;
}

button:disabled {
  background-color: #ccc;
}

/* 响应式设计 */
@media screen and (max-width: 768px) {
  .chat-container {
    max-width: 100%;
    padding: 8px;
    font-size: 13px;
  }

  .message-content {
    max-width: 85%;
  }

  .input-area {
    padding: 10px;
    gap: 5px;
  }

  input {
    padding: 8px;
    font-size: 14px;
  }

  button {
    padding: 8px 16px;
    font-size: 14px;
  }
}

@media screen and (max-width: 480px) {
  .avatar {
    width: 32px;
    height: 32px;
  }

  .message-wrapper {
    gap: 8px;
    margin: 12px 0;
  }

  .user-message,
  .assistant-message,
  .error-message {
    padding: 8px 12px;
    font-size: 13px;
  }

  .assistant-message :deep(pre) {
    padding: 8px;
    font-size: 12px;
  }
}

.character-select {
  display: flex;
  gap: 15px;
  padding: 10px 0;
  border-bottom: 1px solid #eee;
  margin-bottom: 10px;
}

.character-option {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s;
}

.character-option img {
  width: 24px;
  height: 24px;
  border-radius: 50%;
}

.character-option.active {
  background-color: #e6f3ff;
  color: #007aff;
}

.character-option:hover {
  background-color: #f5f5f5;
}
</style>
