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
  { type: 'bot', text: '구미, 경북 여행에 대해 무엇이 궁금하세요?' }
])
const newQuestion = ref('')
const isChatOpen = ref(false)
const isLoading = ref(false)

const openChat = () => { isChatOpen.value = true }
const closeChat = () => { isChatOpen.value = false }

// 질문 키워드 감지 함수
const keywordMap = {
  region: /권역|지역|구역|동네|인근|주변|쪽|부근|코스/,
  festival: /축제|행사|공연|이벤트|페스티벌|개최|일정|기간|날짜|개막|폐막|스케줄/,
  restaurant: /맛집|음식점|식당|밥집|먹거리|카페|디저트|분식|술집|대표 메뉴|추천 메뉴|식도락/,
  accommodation: /숙소|호텔|모텔|펜션|민박|게스트하우스|숙박|예약|체크인|체크아웃/,
  attraction: /관광지|명소|여행지|볼거리|추천지|여행 코스|산책로|전망|스팟/,
  sports: /레포츠|체험|액티비티|스포츠|야외|트레킹|자전거|카약|클라이밍|모험/,
  shopping: /쇼핑|시장|상점|기념품|몰|아울렛|백화점|선물/,
  community: /커뮤니티|게시판|게시글|글|후기|리뷰|추천글|질문|꿀팁|경험담|정보/
}

const detectIntent = (question) => {
  const normalized = question.toLowerCase()
  for (const [intent, pattern] of Object.entries(keywordMap)) {
    if (pattern.test(normalized)) {
      return intent
    }
  }
  return 'general'
}

const getDataForIntent = (intent) => {
  const categoryKeywords = {
    region: ['지역', '권역', '구역', '동네', '인근', '주변'],
    festival: ['축제', '행사', '공연', '이벤트', '페스티벌'],
    restaurant: ['맛집', '음식점', '식당', '밥집', '카페', '디저트'],
    accommodation: ['숙소', '호텔', '모텔', '펜션', '민박', '게스트하우스'],
    shopping: ['쇼핑', '시장', '상점', '기념품', '몰', '아울렛', '백화점'],
    sports: ['레포츠', '체험', '액티비티', '스포츠', '야외']
  }

  const keywords = categoryKeywords[intent]
  if (!keywords) {
    return props.attractions
  }

  const matchedItems = props.attractions.filter(item => {
    const searchable = `${item.title ?? ''} ${item.description ?? ''} ${item.category ?? ''}`
    return keywords.some(keyword => searchable.includes(keyword))
  })

  return matchedItems.length ? matchedItems : props.attractions
}

const sendMessage = async (text) => {
  const content = (text ?? newQuestion.value).trim()
  if (!content || isLoading.value) return

  openChat()
  messages.value.push({ type: 'user', text: content })
  newQuestion.value = ''
  isLoading.value = true
  messages.value.push({ type: 'typing', text: '챗봇이 입력을 이해하는 중입니다.' })

  try {
    const intent = detectIntent(content)
    const relevantData = getDataForIntent(intent)

    const completion = await openai.chat.completions.create({
      model: 'gpt-5-mini',
      messages: [
        {
          role: 'system',
          content: `
당신은 10년차 구미, 경북권 여행 전문가입니다.
- 답변은 최대 5줄 이내로 간단하게 작성하세요.
- 목록 형태 등을 활용해 핵심을 전달하세요.
- 불필요한 배경 설명은 빼고 핵심만 요약하세요.
- 줄바꿈을 활용해 가독성을 높이세요.
- 방문 팁 등 조언을 할 때는 줄글로 제안하고, 
  구체적인 장소나 추천 메뉴 등은 목록으로 제시하세요.

만약 질문한 정보가 데이터에 없으면, 아래와 유사한 형식으로 답변하세요:
- 현재 구미 페스티벌 관련 등록 정보가 없습니다.
- 대신 이런 축제 유형을 추천할 수 있어요:
  1. 음식 축제: 인삼랜드
  2. 자연/야외 축제: 금오산
- 방문 시기:
  - 6월 말 → 야외형
  - 가을 → 산책+전통형

사용자 질문 의도: ${intent}

관련 데이터:
${JSON.stringify(relevantData, null, 2)}`
        },
        { role: 'user', content }
      ]
    })

    const reply = completion.choices[0]?.message?.content ?? '죄송합니다. 답변을 불러오지 못했습니다.'
    const typingIndex = messages.value.findIndex(m => m.type === 'typing')
    if (typingIndex !== -1) messages.value.splice(typingIndex, 1)
    messages.value.push({ type: 'bot', text: reply })
  } catch (err) {
    console.error(err)
    const typingIndex = messages.value.findIndex(m => m.type === 'typing')
    if (typingIndex !== -1) messages.value.splice(typingIndex, 1)
    messages.value.push({
      type: 'bot',
      text: `죄송합니다. 답변을 불러오지 못했습니다. (${err?.message ?? 'Unknown error'})`
    })
  } finally {
    isLoading.value = false
  }
}

const openWithQuestion = (text) => { sendMessage(text) }

defineExpose({ openWithQuestion })
</script>

<template>
  <transition name="chat-popup">
    <div v-if="isChatOpen" class="chat-overlay" @click.self="closeChat">
      <div class="chat-popup" @click.stop>
        <div class="chat-header">
          <div>
            <strong>구미 여행 챗봇</strong>
            <p>여행 질문을 바로 입력해보세요.</p>
          </div>
          <button class="close-btn" @click="closeChat" aria-label="챗봇 닫기">✕</button>
        </div>

        <div class="chat-messages">
          <div
            v-for="(message, index) in messages"
            :key="index"
            :class="['message', message.type, message.type === 'typing' ? 'typing-dot' : '']"
          >
            {{ message.text }}
          </div>

          <div v-if="isLoading" class="message bot typing-indicator" aria-live="polite">
            <span></span><span></span><span></span>
          </div>
        </div>

        <div class="chat-popup-input">
          <input
            v-model="newQuestion"
            :disabled="isLoading"
            @keyup.enter.prevent="sendMessage()"
            :placeholder="isLoading ? '전송 중입니다...' : '질문을 입력해 주세요'"
          />
          <button @click="sendMessage()" :disabled="isLoading">
            {{ isLoading ? '전송 중...' : '전송' }}
          </button>
        </div>
      </div>
    </div>
  </transition>

  <button class="floating-chat" aria-label="챗봇 열기" @click="openChat">💬</button>
</template>

<style scoped>
.chat-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.35);
  display: flex;
  justify-content: flex-end;
  align-items: flex-end;
  padding: 1rem;
  z-index: 100;
}

.chat-popup {
  width: min(430px, 100%);
  max-height: min(760px, 92vh);
  display: flex;
  flex-direction: column;
  border-radius: 20px;
  overflow: hidden;
  background: #fff;
  box-shadow: 0 18px 50px rgba(15, 23, 42, 0.18);
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.95rem 1rem;
  border-bottom: 1px solid #e5e7eb;
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: white;
}

.chat-header strong {
  display: block;
  font-size: 1rem;
}

.chat-header p {
  margin: 0.2rem 0 0;
  font-size: 0.84rem;
  opacity: 0.95;
}

.close-btn {
  border: none;
  background: transparent;
  color: white;
  font-size: 1rem;
  cursor: pointer;
}

.chat-messages {
  padding: 0.9rem;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  overflow-y: auto;
  background: #f8fafc;
  min-height: 280px;
  max-height: 480px;
}

.message {
  width: fit-content;
  max-width: 88%;
  padding: 0.7rem 0.85rem;
  border-radius: 14px;
  line-height: 1.45;
  white-space: pre-line;
  word-break: keep-all;
  box-shadow: 0 2px 8px rgba(15, 23, 42, 0.05);
}

.message.user {
  align-self: flex-end;
  background: #dbeafe;
  color: #1e3a8a;
  border-bottom-right-radius: 6px;
}

.message.bot {
  align-self: flex-start;
  background: white;
  color: #111827;
  border: 1px solid #e5e7eb;
  border-bottom-left-radius: 6px;
}

.typing-indicator {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  padding: 0.65rem 0.8rem;
}

.typing-indicator span {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #94a3b8;
  animation: bounce 1.2s infinite ease-in-out;
}

.typing-indicator span:nth-child(2) {
  animation-delay: 0.15s;
}

.typing-indicator span:nth-child(3) {
  animation-delay: 0.3s;
}

@keyframes bounce {
  0%, 80%, 100% { transform: scale(0.8); opacity: 0.6; }
  40% { transform: scale(1); opacity: 1; }
}

.chat-popup-input {
  display: flex;
  gap: 0.6rem;
  padding: 0.8rem;
  border-top: 1px solid #e5e7eb;
  background: #fff;
}

.chat-popup-input input {
  flex: 1;
  min-height: 44px;
  border: 1px solid #d1d5db;
  border-radius: 999px;
  padding: 0 0.95rem;
  font-size: 0.95rem;
}

.chat-popup-input button {
  border: none;
  border-radius: 999px;
  padding: 0 0.95rem;
  min-height: 44px;
  background: #2563eb;
  color: white;
  cursor: pointer;
}

.floating-chat {
  width: 56px;
  height: 56px;
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: white;
  font-size: 1.4rem;
  box-shadow: 0 12px 30px rgba(37, 99, 235, 0.3);
  cursor: pointer;
}

@media (max-width: 768px) {
  .chat-overlay {
    padding: 0.5rem;
    align-items: flex-end;
    justify-content: center;
  }

  .chat-popup {
    width: 100%;
    max-height: 85vh;
  }

  .chat-messages {
    min-height: 220px;
    max-height: 60vh;
  }

  .chat-popup-input {
    flex-direction: column;
  }

  .chat-popup-input button {
    width: 100%;
  }

  .floating-chat {
    width: 52px;
    height: 52px;
  }
}
</style>