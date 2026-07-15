<script setup>
import { ref } from 'vue'

const messages = ref([
  { type: 'bot', text: '구미 여행에 대해 무엇이 궁금하세요?' }
])
const newQuestion = ref('')
const isChatOpen = ref(false)

const openChat = () => {
  isChatOpen.value = true
}

const closeChat = () => {
  isChatOpen.value = false
}
const isLoading = ref(false)

const sendMessage = async (text) => {
  const content = (text ?? newQuestion.value).trim()
  if (!content) return

  openChat()
  messages.value.push({ type: 'user', text: content })
  isLoading.value = true

  try {
    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: content
      })
    })

    const data = await res.json()
    messages.value.push({
      type: 'bot',
      text: data.reply
    })
  } catch (err) {
    messages.value.push({
      type: 'bot',
      text: '답변을 불러오지 못했습니다. 잠시 후 다시 시도해 주세요.'
    })
  } finally {
    isLoading.value = false
    newQuestion.value = ''
  }
}

const openWithQuestion = (text) => {
  sendMessage(text)
}

defineExpose({ openWithQuestion })
</script>

<template>
  <button class="floating-chat" aria-label="챗봇 열기" @click="openChat">
    💬
  </button>

  <transition name="chat-popup">
    <div v-if="isChatOpen" class="chat-overlay" @click.self="closeChat">
      <div class="chat-popup" @click.stop>
        <div class="chat-popup-header">
          <strong>구미 여행 챗봇</strong>
        </div>

        <div class="chat-popup-messages">
          <div
            v-for="(message, index) in messages"
            :key="index"
            :class="['message', message.type]"
          >
            {{ message.text }}
          </div>
        </div>

        <div class="chat-popup-input">
          <input
            v-model="newQuestion"
            @keyup.enter="sendMessage()"
            placeholder="질문을 입력해 주세요"
          />
          <button type="button" @click="sendMessage()">전송</button>
        </div>
      </div>
    </div>
  </transition>
</template>