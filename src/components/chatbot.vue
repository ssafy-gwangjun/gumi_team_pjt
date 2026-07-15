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

const sendMessage = (text) => {
  const content = (text ?? newQuestion.value).trim()
  if (!content) return


  openChat()
 messages.value.push({ type: 'user', text: content })
 messages.value.push({
   type: 'bot',
   text: `“${content}”에 대한 정보는 곧 챗봇에서 더 자세히 답변해드릴 수 있어요.`
  })
  
  if (!text) newQuestion.value = ''
}

const openWithQuestion = (text) => {
  sendMessage(text)
}

defineExpose({ openWithQuestion })
</script>

<template>
 <transition name="chat-popup">
  <div v-if="isChatOpen" class="chat-popup">
    <div class="chat-header">챗봇 질문하기</div>

    <div class="chat-messages">
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
        <button @click="sendMessage()">전송</button>
      </div>
    </div>
   </transition>

<button class="floating-chat" aria-label="챗봇 열기" @click="openChat">
    💬
  </button>
</template>