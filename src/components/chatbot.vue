<script setup>
import { ref } from 'vue'
import OpenAI from 'openai'

const props = defineProps({
  attractions: {
    type: Array,
    default: () => []
  }
})

const openai = new OpenAI({
  apiKey: import.meta.env.VITE_OPENAI_API_KEY,
  dangerouslyAllowBrowser: true
})

const messages = ref([
  { type: 'bot', text: '구미 여행에 대해 무엇이 궁금하세요?' }
])
const newQuestion = ref('')
const isChatOpen = ref(false)
const isLoading = ref(false)

const openChat = () => { isChatOpen.value = true }
const closeChat = () => { isChatOpen.value = false }

const sendMessage = async (text) => {
  const content = (text ?? newQuestion.value).trim()
  if (!content) return

  openChat()
  messages.value.push({ type: 'user', text: content })
  isLoading.value = true

  try {
    const completion = await openai.chat.completions.create({
      model: 'gpt-5-mini',
      messages: [
        {
          role: 'system',
          content: `당신은 구미 여행 가이드 챗봇입니다. 친절하고 정확하게 답변하세요.
          
구미 관광 정보:
${JSON.stringify(props.attractions, null, 2)}

위 데이터를 참고하여 사용자의 질문에 답변하세요.`
        },
        {
          role: 'user',
          content: content
        }
      ]
    })

    const reply = completion.choices[0].message.content
    messages.value.push({
      type: 'bot',
      text: reply
    })
  } catch (err) {
    console.error(err)
    messages.value.push({
      type: 'bot',
      text: '죄송합니다. 답변을 불러오지 못했습니다.'
    })
  } finally {
    isLoading.value = false
    newQuestion.value = ''
  }
}

const openWithQuestion = (text) => { sendMessage(text) }

defineExpose({ openWithQuestion })
</script>

<template>
 <transition name="chat-popup">
  <div v-if="isChatOpen" class="chat-overlay" @click.self="closeChat">
   <div class="chat-popup" @click.stop>
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
    </div>
    </transition>

<button class="floating-chat" aria-label="챗봇 열기" @click="openChat">
    💬
  </button>
</template>