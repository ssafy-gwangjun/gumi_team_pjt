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
    <div class="chat-header">챗봇 질문하기</div>

    <div class="chat-messages">
      <div
        v-for="(message, index) in messages"
        :key="index"
        :class="['message', message.type, message.type === 'typing' ? 'typing-dot' : '']"
      >
        {{ message.text }}
      </div>
    </div>

      <div class="chat-popup-input">
        <input
          v-model="newQuestion"
          :disabled="isLoading"
          @keyup.enter="sendMessage()"
          :placeholder="isLoading ? '전송 중입니다...' : '질문을 입력해 주세요'"
        />
        <button @click="sendMessage()" :disabled="isLoading">
          {{ isLoading ? '전송 중...' : '전송' }}
        </button>
      </div>
    </div>
    </div>
    </transition>

<button class="floating-chat" aria-label="챗봇 열기" @click="openChat">
    💬
  </button>
</template>