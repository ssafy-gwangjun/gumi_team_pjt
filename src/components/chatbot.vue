<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import OpenAI from 'openai'

const props = defineProps({
  attractions: {
    type: Array,
    default: () => []
  }
})

const HISTORY_KEY = 'gumi-chat-history'
const COMMUNITY_KEY = 'localhub-posts'
const openai = import.meta.env.VITE_OPENAI_API_KEY
  ? new OpenAI({
      apiKey: import.meta.env.VITE_OPENAI_API_KEY,
      dangerouslyAllowBrowser: true
    })
  : null

const messages = ref([
  { type: 'bot', text: '구미, 경북 여행에 대해 무엇이 궁금하세요? 축제, 맛집, 관광지, 숙소 등에 대해 물어보세요!' }
])
const newQuestion = ref('')
const isChatOpen = ref(false)
const isLoading = ref(false)
const communityPosts = ref([])

// 전달받은 attractions 데이터가 원본 TourAPI 통째 구조({ items: [...] })일 경우와 단순 배열일 경우를 모두 대응하기 위한 안전 가드
const rawAttractions = computed(() => {
  if (!props.attractions) return []
  if (Array.isArray(props.attractions)) {
    // 만약 배열 내부에 다시 items가 들어있는 경우 등 2차 예외 처리
    return props.attractions.flatMap(item => {
      if (item && item.items && Array.isArray(item.items)) return item.items
      return item
    })
  } else if (typeof props.attractions === 'object') {
    if (Array.isArray(props.attractions.items)) {
      return props.attractions.items
    }
  }
  return []
})

const openChat = () => { isChatOpen.value = true }
const closeChat = () => { isChatOpen.value = false }

// 키워드 정규식 맵핑 개선
const keywordMap = {
  region: /권역|지역|구역|동네|인근|주변|쪽|부근|코스/,
  festival: /축제|행사|공연|이벤트|페스티벌|개최|일정|기간|날짜|개막|폐막|스케줄|엑스포|라디오|콘서트|체맥|불교/,
  restaurant: /맛집|음식점|식당|밥집|먹거리|카페|디저트|분식|술집|대표 메뉴|추천 메뉴|식도락|라면|푸드/,
  accommodation: /숙소|호텔|모텔|펜션|민박|게스트하우스|숙박|예약|체크인|체크아웃/,
  attraction: /관광지|명소|여행지|볼거리|추천지|산책로|전망|스팟|가볼만한곳|공원/,
  sports: /레포츠|체험|액티비티|스포츠|야외|트레킹|자전거|카약|클라이밍|모험|워터/,
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

const normalizeText = (value = '') => String(value).normalize('NFKC').toLowerCase()

const getSchemaFields = (item = {}) => {
  const title = String(item.title ?? '').trim()
  const category = String(item.category ?? item.cat3 ?? item.cat2 ?? item.cat1 ?? '').trim()
  const addr1 = String(item.addr1 ?? '').trim()
  const addr2 = String(item.addr2 ?? '').trim()
  const description = String(item.description ?? item.overview ?? item.summary ?? '').trim()
  const contentTypeId = String(item.contenttypeid ?? item.contentTypeId ?? '')
  const firstImage = String(item.firstimage ?? item.firstimage2 ?? '').trim()
  const tel = String(item.tel ?? '').trim()
  const location = [addr1, addr2].filter(Boolean).join(' ')

  return {
    title,
    category,
    addr1,
    addr2,
    description: description.replace(/\s+/g, ' '),
    contentTypeId,
    location,
    firstImage,
    tel
  }
}

const getContentTypeGroup = (item) => {
  const fields = getSchemaFields(item)
  const combined = normalizeText(`${fields.title} ${fields.category} ${fields.description} ${fields.location}`)

  // contenttypeid 매칭 혹은 제목 및 상세 키워드 매칭 우선
  if (fields.contentTypeId === '15' || /(축제|행사|페스티벌|공연|이벤트|페스타|엑스포|문화제)/.test(combined)) return 'festival'
  if (fields.contentTypeId === '39' || /(맛집|음식점|식당|카페|디저트|분식|푸드)/.test(combined)) return 'restaurant'
  if (fields.contentTypeId === '32' || /(숙소|호텔|모텔|펜션|민박|게스트하우스)/.test(combined)) return 'accommodation'
  if (fields.contentTypeId === '12' || /(관광지|명소|여행지|볼거리|전망|산책|공원)/.test(combined)) return 'attraction'
  if (fields.contentTypeId === '28' || /(레포츠|체험|액티비티|스포츠|트레킹|자전거|카약|클라이밍)/.test(combined)) return 'sports'
  if (fields.contentTypeId === '38' || /(쇼핑|시장|상점|백화점|아울렛|기념품)/.test(combined)) return 'shopping'
  if (fields.contentTypeId === '14' || /(문화|전시|박물관|센터)/.test(combined)) return 'culture'
  return 'general'
}

const getDataForIntent = (intent, question) => {
  const categoryKeywords = {
    region: ['지역', '권역', '구역', '동네', '인근', '주변', '코스'],
    festival: ['축제', '행사', '공연', '이벤트', '페스티벌', '문화제', '페스타', '엑스포'],
    restaurant: ['맛집', '음식점', '식당', '밥집', '카페', '디저트', '음식', '라면', '푸드'],
    accommodation: ['숙소', '호텔', '모텔', '펜션', '민박', '게스트하우스'],
    attraction: ['관광지', '명소', '여행지', '볼거리', '추천지', '산책로', '전망', '스팟', '관광', '공원'],
    sports: ['레포츠', '체험', '액티비티', '스포츠', '야외', '트레킹'],
    shopping: ['쇼핑', '시장', '상점', '기념품', '몰', '아울렛', '백화점', '선물'],
    community: ['커뮤니티', '게시글', '후기', '리뷰', '질문', '꿀팁', '경험담', '정보']
  }

  const allItems = rawAttractions.value
  const allPosts = Array.isArray(communityPosts.value) ? communityPosts.value : []
  const keywords = categoryKeywords[intent] || []
  const normalizedQuestion = normalizeText(question)

  if (intent === 'community') {
    const matchedPosts = allPosts.filter((post) => {
      const searchable = `${post.title ?? ''} ${post.content ?? ''}`.toLowerCase()
      return keywords.some((keyword) => searchable.includes(keyword))
        || normalizedQuestion.split(/\s+/).some((word) => word.length > 1 && searchable.includes(word))
    })

    return {
      attractions: allItems.slice(0, 10),
      communityPosts: matchedPosts.length ? matchedPosts.slice(0, 10) : allPosts.slice(0, 10),
      intent
    }
  }

  // 매칭 로직 고도화: 의도 유형과 직접적인 문자열 일치 여부를 모두 체크
  let filteredItems = allItems.filter((item) => {
    const group = getContentTypeGroup(item)
    const fields = getSchemaFields(item)
    const searchable = `${fields.title} ${fields.description} ${fields.category} ${fields.location}`.toLowerCase()

    // 1. 구체적인 키워드가 제목이나 상세정보에 직접 들어있는 경우 최우선 매칭 (예: '라면' 축제를 검색한 경우)
    const directMatch = normalizedQuestion.length > 1 && searchable.includes(normalizedQuestion)
    if (directMatch) return true

    // 2. 의도에 근거한 카테고리 매칭
    if (intent === 'festival') return group === 'festival'
    if (intent === 'restaurant') return group === 'restaurant'
    if (intent === 'accommodation') return group === 'accommodation'
    if (intent === 'attraction') return group === 'attraction'
    if (intent === 'sports') return group === 'sports'
    if (intent === 'shopping') return group === 'shopping'

    // 3. 포괄적인 키워드 매칭
    return keywords.some((keyword) => searchable.includes(keyword))
  })

  // 만약 필터링된 결과가 없다면 질문에 들어있는 단어 단위 매칭 시도
  if (filteredItems.length === 0) {
    const words = normalizedQuestion.split(/\s+/).filter(w => w.length > 1)
    filteredItems = allItems.filter(item => {
      const fields = getSchemaFields(item)
      const searchable = `${fields.title} ${fields.location}`.toLowerCase()
      return words.some(word => searchable.includes(word))
    })
  }

  return {
    attractions: filteredItems.length ? filteredItems.slice(0, 12) : allItems.slice(0, 12),
    communityPosts: allPosts.slice(0, 5),
    intent,
    fallback: filteredItems.length === 0
  }
}

const buildStructuredFallbackReply = (question, intent, relevantData) => {
  const attractions = (relevantData.attractions || []).slice(0, 3)
  const topic = intent === 'festival' ? '축제·행사' : '여행 정보'

  const candidates = attractions.map((item) => {
    const fields = getSchemaFields(item)
    return {
      name: fields.title || '추천 장소',
      category: fields.category || '일반',
      location: fields.location || '위치 정보 미등록',
      summary: fields.description ? fields.description.slice(0, 100) : '추가 설명 없음'
    }
  })

  if (intent === 'festival') {
    return `구미/경북 여행 데이터를 분석한 결과입니다.
정확한 상세 일정 정보를 찾기 어려운 경우, 아래 등록된 축제 정보를 참고해주세요!

${candidates.map((item) => `📢 **${item.name}**\n📍 위치: ${item.location}\nℹ️ 설명: ${item.summary}`).join('\n\n')}`
  }

  return `{
  "region": "구미",
  "intent": "${intent}",
  "topic": "${topic}",
  "status": "partial",
  "message": "현재 등록된 데이터에서 원하시는 상세 정보를 직접 분류하기 어려워 추천 후보들을 전해드립니다.",
  "candidates": ${JSON.stringify(candidates, null, 2)},
  "next_steps": ["축제 이름", "지역", "기간"]
}`
}

const buildDataReply = (question, intent, relevantData) => {
  const attractions = (relevantData.attractions || []).slice(0, 8)
  const posts = (relevantData.communityPosts || []).slice(0, 3)
  const normalizedQuestion = normalizeText(question)

  // 특정 키워드 차단식(hasKeyword)을 완화하여, 질문 의도가 매칭되었거나 데이터가 존재하는 경우 바로 답변이 조립되도록 변경
  if (intent === 'community' && posts.length) {
    const lines = posts.map((post) => `- ${post.title ?? '커뮤니티 글'}`)
    return `💬 커뮤니티에서 발견한 관련 여행 후기글입니다.\n${lines.join('\n')}\n\n더 자세한 정보나 다른 후기가 궁금하시다면 구체적인 키워드로 알려주세요!`
  }

  if (attractions.length) {
    if (intent === 'festival') {
      const lines = attractions.map((item) => {
        const fields = getSchemaFields(item)
        const name = fields.title || '이름 미상'
        const location = fields.location ? `\n📍 위치: ${fields.location}` : ''
        const tel = fields.tel ? `\n📞 문의처: ${fields.tel}` : ''
        const summary = fields.description ? `\nℹ️ 설명: ${fields.description.slice(0, 110)}...` : ''
        return `🎉 **${name}**${location}${tel}${summary}`
      })

      return `구미/경북 권역의 대표적인 **축제·행사 정보**를 안내해 드립니다.\n\n${lines.join('\n\n')}\n\n💡 더 관심이 가는 특정 축제의 이름을 말씀해주시면 연관 세부 사항을 추가로 찾아볼게요!`
    }

    const lines = attractions.map((item) => {
      const fields = getSchemaFields(item)
      const title = fields.title || '추천 장소'
      const location = fields.location ? ` (${fields.location})` : ''
      const summary = fields.description ? `: ${fields.description.slice(0, 80)}` : ''
      return `📍 **${title}**${location}${summary}`
    })

    const prefix = intent === 'restaurant'
      ? '😋 구미_경북의 대표 추천 맛집·식도락 리스트입니다.'
      : intent === 'accommodation'
        ? '🛌 편안한 구미 여행을 위한 숙박 후보지입니다.'
        : intent === 'shopping'
          ? '🛍️ 구미의 전통시장 및 쇼핑 추천 명소입니다.'
          : intent === 'sports'
            ? '🏃 신나는 액티비티를 체험할 수 있는 장소들입니다.'
            : intent === 'attraction'
              ? '📸 구미에서 가볼 만한 인기 관광지 목록입니다.'
              : '🗺️ 문의하신 조건에 어울리는 추천 여행 장소입니다.'

    return `${prefix}\n\n${lines.join('\n')}\n\n원하시는 특정 권역(원평동, 송정동 등)이나 추가 정보가 필요하시면 편하게 질문해 주세요.`
  }

  return buildStructuredFallbackReply(question, intent, relevantData)
}

const loadHistory = () => {
  try {
    const saved = localStorage.getItem(HISTORY_KEY)
    if (saved) {
      const parsed = JSON.parse(saved)
      if (Array.isArray(parsed) && parsed.length) {
        messages.value = parsed
      }
    }
  } catch (error) {
    console.error('챗봇 히스토리 로드 실패:', error)
  }
}

const loadCommunityPosts = () => {
  try {
    const saved = localStorage.getItem(COMMUNITY_KEY)
    if (saved) {
      const parsed = JSON.parse(saved)
      communityPosts.value = Array.isArray(parsed) ? parsed : []
    }
  } catch (error) {
    console.error('커뮤니티 게시글 로드 실패:', error)
  }
}

onMounted(() => {
  loadHistory()
  loadCommunityPosts()
})

watch(messages, (value) => {
  localStorage.setItem(HISTORY_KEY, JSON.stringify(value))
}, { deep: true })

const sendMessage = async (text) => {
  const content = (text ?? newQuestion.value).trim()
  if (!content || isLoading.value) return

  openChat()
  messages.value.push({ type: 'user', text: content })
  newQuestion.value = ''
  isLoading.value = true

  try {
    const intent = detectIntent(content)
    const relevantData = getDataForIntent(intent, content)
    const dataReply = buildDataReply(content, intent, relevantData)

    let reply = dataReply
    const shouldPolish = Boolean(openai) && (intent === 'general' || !relevantData.attractions?.length || relevantData.fallback)

    if (reply && shouldPolish && !reply.trim().startsWith('{')) {
      const completion = await openai.chat.completions.create({
        model: 'gpt-5-mini', // gpt-5-mini 등 부정확한 모델 네임 예방을 위해 보편적인 gpt-4o-mini 사용 권장
        messages: [
          {
            role: 'system',
            content: `
당신은 대한민국 경상북도 구미 여행 가이드 전문가입니다.
- 제공된 구미/경북의 JSON 데이터를 최우선 기반으로 사용하여 사용자 질문에 성실히 답변하세요.
- 각 분류에 해당하는 JSON 데이터를 활용해 가독성 높고 품격 있는 문체로 가공해 제공하세요.
- 데이터 항목 중 축제 및 행사 등이 있다면 상세 내용(주소, 전화번호 등)을 적극 참고하여 정성껏 다듬어 주세요.
- 리스트와 명확한 이모지 가이드를 사용해 한눈에 들어오도록 작성해야 합니다.

사용자 질문 의도: ${intent}

JSON 데이터:
${JSON.stringify(relevantData, null, 2)}

기본 정보 텍스트:
${reply}`
          },
          { role: 'user', content }
        ]
      })

      reply = completion.choices[0]?.message?.content ?? reply
    }

    if (!reply) {
      reply = '죄송합니다. 현재 제공 중인 구미 여행 데이터 내에서는 구체적인 내용을 찾지 못했습니다.\n"구미라면 축제", "구미푸드페스티벌", "대구치맥페스티벌"과 같은 다양한 축제와 맛집, 관광지 키워드로 다시 한 번 물어봐주세요!'
    }

    messages.value.push({ type: 'bot', text: reply })
  } catch (err) {
    console.error(err)
    messages.value.push({
      type: 'bot',
      text: `죄송합니다. 답변을 처리하는 과정에 문제가 발생했습니다. (${err?.message ?? '오류 발생'})`
    })
  } finally {
    isLoading.value = false
  }
}

const openWithQuestion = (text) => { sendMessage(text) }

defineExpose({ openWithQuestion })
</script>

<template>
  <div class="chat-shell">
    <transition name="chat-slide">
      <div v-if="isChatOpen" class="chat-panel" @click.stop>
        <div class="chat-header">
          <div>
            <strong>구미 여행 챗봇 (GumiTour Bot)</strong>
            <p>축제 정보, 맛집, 가볼 만한 곳을 질문해보세요!</p>
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
            :placeholder="isLoading ? '전송 중입니다...' : '예: 구미라면 축제 언제 어디서 해?'"
          />
          <button @click="sendMessage()" :disabled="isLoading">
            {{ isLoading ? '전송 중...' : '전송' }}
          </button>
        </div>
      </div>
    </transition>

    <button class="floating-chat" aria-label="챗봇 열기" @click="openChat">💬</button>
  </div>
</template>

<style scoped>
.chat-shell {
  position: fixed;
  right: 1rem;
  bottom: 1rem;
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.7rem;
}

.chat-panel {
  width: min(520px, calc(100vw - 1.5rem));
  max-height: min(780px, calc(100vh - 1.5rem));
  display: flex;
  flex-direction: column;
  border-radius: 18px;
  overflow: hidden;
  background: #fff;
  box-shadow: 0 16px 40px rgba(15, 23, 42, 0.16);
  border: 1px solid rgba(148, 163, 184, 0.2);
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.95rem 1rem;
  border-bottom: 1px solid #e5e7eb;
  background: linear-gradient(135deg, #1d4ed8, #3b82f6);
  color: white;
}

.chat-header strong {
  display: block;
  font-size: 1rem;
  letter-spacing: -0.025em;
}

.chat-header p {
  margin: 0.2rem 0 0;
  font-size: 0.82rem;
  opacity: 0.9;
}

.close-btn {
  border: none;
  background: transparent;
  color: white;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 0.2rem;
}

.chat-messages {
  padding: 0.95rem;
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
  overflow-y: auto;
  background: #f8fafc;
  min-height: 320px;
  max-height: 560px;
}

.message {
  width: fit-content;
  max-width: 88%;
  padding: 0.7rem 0.85rem;
  border-radius: 14px;
  line-height: 1.5;
  white-space: pre-line;
  word-break: keep-all;
  font-size: 0.92rem;
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

.chat-slide-enter-active,
.chat-slide-leave-active {
  transition: all 0.22s ease;
}

.chat-slide-enter-from,
.chat-slide-leave-to {
  opacity: 0;
  transform: translateY(14px) scale(0.98);
}

.chat-slide-enter-to,
.chat-slide-leave-from {
  opacity: 1;
  transform: translateY(0) scale(1);
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
  padding: 0 1.1rem;
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s;
}

.chat-popup-input input:focus {
  border-color: #3b82f6;
}

.chat-popup-input button {
  border: none;
  border-radius: 999px;
  padding: 0 1.3rem;
  min-height: 44px;
  background: #2563eb;
  color: white;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.chat-popup-input button:hover {
  background: #1d4ed8;
}

.floating-chat {
  width: 60px;
  height: 60px;
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: white;
  font-size: 1.4rem;
  box-shadow: 0 12px 30px rgba(37, 99, 235, 0.3);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.2s;
}

.floating-chat:hover {
  transform: scale(1.05);
}

@media (max-width: 768px) {
  .chat-shell {
    right: 0.7rem;
    bottom: 0.7rem;
  }

  .chat-panel {
    width: min(100%, calc(100vw - 1.2rem));
    max-height: 80vh;
  }

  .chat-messages {
    min-height: 260px;
    max-height: 55vh;
  }

  .floating-chat {
    width: 52px;
    height: 52px;
  }
}
</style>