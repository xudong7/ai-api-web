<template>
  <div>
    <h1>请输入你的问题...</h1>
    <input type="text" v-model="message" :disabled="isLoading"/>
    <button @click="sendMessage(message)" :disabled="isLoading">
      {{ isLoading ? '处理中...' : '发送' }}
    </button>
    <div v-for="message in messages" :key="message.id" :class="message.role">
      <div v-if="message.role === 'user'" class="user-message">
        {{ message.content }}
      </div>
      <div v-else-if="message.role === 'assistant'" class="assistant-message" v-html="markdownToHtml(message.content)">
      </div>
      <div v-else class="error-message">
        {{ message.content }}
      </div>
    </div>
  </div>
</template>

<script setup>
import OpenAI from "openai";
import { ref } from "vue";
import { marked } from 'marked';

const apiKey = import.meta.env.VITE_DEEPSEEK_API_KEY;
const baseURL = import.meta.env.VITE_DEEPSEEK_BASE_URL;

const openai = new OpenAI({
  baseURL: baseURL,
  apiKey: apiKey,
  temperature: 1,
  dangerouslyAllowBrowser: true, // 允许在浏览器中使用
});

// 代码生成/数学解题   	0.0
// 数据抽取/分析	1.0
// 通用对话	1.3
// 翻译	1.3
// 创意类写作/诗歌创作	1.5

const message = ref("");
const messages = ref([]);
const isLoading = ref(false);

const sendMessage = async (content) => {
  try {
    // 检查输入是否为空
    if (!content.trim()) return;
    // 设置加载状态
    isLoading.value = true;

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
      messages: [{ role: "user", content: content }],
      model: "deepseek-chat",
      stream: true,
    });

    // 处理流式响应
    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content || "";
      messages.value[aiMessageId].content += content;
    }

  } catch (error) {
    console.error("Error:", error);
    messages.value.push({
      id: messages.value.length,
      content: "抱歉，发生了错误，请稍后重试。",
      role: "error",
    });
  } finally {
    isLoading.value = false;
  }
};

// 添加 Markdown 转换函数
const markdownToHtml = (content) => {
  return marked(content);
};
</script>

<style scoped>
div {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

input {
  width: 80%;
  padding: 8px;
  margin-right: 10px;
}

button {
  padding: 8px 16px;
}

.user-message {
  background-color: #e6f3ff;
  padding: 10px;
  margin: 5px 0;
  border-radius: 5px;
}

.assistant-message {
  background-color: #f5f5f5;
  padding: 10px;
  margin: 5px 0;
  border-radius: 5px;
}

.assistant-message :deep(pre) {
  background-color: #2d2d2d;
  color: #fff;
  padding: 10px;
  border-radius: 5px;
  overflow-x: auto;
}

.error-message {
  background-color: #ffe6e6;
  color: #ff0000;
  padding: 10px;
  margin: 5px 0;
  border-radius: 5px;
}
</style>
