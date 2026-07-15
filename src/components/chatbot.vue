```vue
<script setup>
import { nextTick, onMounted, ref } from 'vue'
import OpenAI from 'openai'

const DATA_FILES = [
  '구미_경북권_관광지.json',
  '구미_경북권_문화시설.json',
  '구미_경북권_축제공연행사.json',
  '구미_경북권_레포츠.json',
  '구미_경북권_숙박.json',
  '구미_경북권_쇼핑.json',
  '구미_경북권_음식점.json',
  '구미_경북권_여행코스.json'
]

const MAX_CONTEXT_ITEMS = 20
const MAX_HISTORY_MESSAGES = 6
const OPENAI_MODEL =
  import.meta.env.VITE_OPENAI_MODEL || 'gpt-5-mini'

const apiKey = import.meta.env.VITE_OPENAI_API_KEY

const openai = apiKey
  ? new OpenAI({
      apiKey,
      dangerouslyAllowBrowser: true
    })
  : null

const CONTENT_TYPE_NAMES = {
  '12': '관광지',
  '14': '문화시설',
  '15': '축제공연행사',
  '25': '여행코스',
  '28': '레포츠',
  '32': '숙박',
  '38': '쇼핑',
  '39': '음식점'
}

const CONTENT_TYPE_PATTERNS = [
  {
    id: '39',
    pattern:
      /맛집|음식점|식당|밥집|먹거리|카페|커피|디저트|분식|술집|한식|중식|일식|양식|메뉴/
  },
  {
    id: '32',
    pattern:
      /숙소|호텔|모텔|펜션|민박|게스트하우스|숙박|리조트|콘도|체크인|체크아웃/
  },
  {
    id: '15',
    pattern:
      /축제|행사|공연|이벤트|페스티벌|개최|개막|폐막/
  },
  {
    id: '38',
    pattern:
      /쇼핑|시장|상점|기념품|아울렛|백화점|마트|선물/
  },
  {
    id: '28',
    pattern:
      /레포츠|체험|액티비티|스포츠|트레킹|자전거|카약|클라이밍|모험/
  },
  {
    id: '14',
    pattern:
      /문화시설|문화 시설|박물관|미술관|전시관|기념관|공연장|도서관/
  },
  {
    id: '25',
    pattern:
      /여행\s*코스|관광\s*코스|데이트\s*코스|드라이브\s*코스|코스/
  },
  {
    id: '12',
    pattern:
      /관광지|명소|여행지|볼거리|산책로|전망대|관광 명소|가볼\s*만한\s*곳/
  }
]

const STOP_WORDS = new Set([
  '구미',
  '경북',
  '경상북도',
  '관광',
  '여행',
  '장소',
  '정보',
  '관련',
  '대해',
  '대한',
  '알려줘',
  '알려주세요',
  '알려',
  '보여줘',
  '보여주세요',
  '찾아줘',
  '찾아주세요',
  '추천해줘',
  '추천해주세요',
  '추천',
  '어디야',
  '어디',
  '무엇',
  '뭐가',
  '뭐야',
  '어떤',
  '있는',
  '있어',
  '있나요',
  '있을까',
  '주세요',
  '해줘',
  '해주라',
  '가고',
  '싶어',
  '좋은',
  '괜찮은',
  '근처',
  '주변',
  '인근',
  '부근',
  '지역',
  '권역',
  '관광지',
  '명소',
  '여행지',
  '볼거리',
  '맛집',
  '음식점',
  '식당',
  '밥집',
  '카페',
  '숙소',
  '숙박',
  '호텔',
  '모텔',
  '펜션',
  '축제',
  '행사',
  '공연',
  '쇼핑',
  '시장',
  '레포츠',
  '체험',
  '문화시설',
  '문화',
  '시설',
  '코스',
  '주소',
  '위치',
  '전화',
  '전화번호',
  '연락처',
  '좌표',
  '위도',
  '경도',
  '지도',
  '사진',
  '이미지',
  '우편번호',
  '유형',
  '분류',
  '카테고리',
  '거기',
  '그곳',
  '여기'
])

const messages = ref([
  {
    type: 'bot',
    text: '구미 관광 데이터에 대해 무엇이 궁금한가요?',
    isGreeting: true
  }
])

const attractions = ref([])
const lastContextItems = ref([])
const newQuestion = ref('')
const isChatOpen = ref(false)
const isLoading = ref(false)
const isDataLoading = ref(true)
const dataLoadError = ref('')
const messagesContainer = ref(null)

const normalizeText = value =>
  String(value ?? '')
    .toLowerCase()
    .normalize('NFKC')
    .replace(/[^\p{L}\p{N}\s]/gu, ' ')
    .replace(/\s+/g, ' ')
    .trim()

const joinSearchableFields = values =>
  normalizeText(values.filter(Boolean).join(' '))

const isTourItem = value => {
  if (!value || typeof value !== 'object' || Array.isArray(value)) {
    return false
  }

  return Boolean(
    value.contentid ||
      value.contenttypeid ||
      value.title ||
      value.addr1 ||
      value.mapx ||
      value.mapy
  )
}

const extractTourItems = value => {
  if (!value) return []

  if (Array.isArray(value)) {
    return value.flatMap(entry => extractTourItems(entry))
  }

  if (typeof value !== 'object') {
    return []
  }

  if (isTourItem(value)) {
    return [value]
  }

  if (Array.isArray(value.items)) {
    const inheritedContentTypeId = String(
      value.contentTypeId ?? ''
    ).trim()

    return value.items
      .filter(item => item && typeof item === 'object')
      .map(item => ({
        ...item,
        contenttypeid:
          String(item.contenttypeid ?? '').trim() ||
          inheritedContentTypeId
      }))
  }

  return Object.values(value).flatMap(entry =>
    extractTourItems(entry)
  )
}

const removeDuplicateItems = items => {
  const uniqueItems = new Map()

  for (const item of items) {
    const contentId = String(item.contentid ?? '').trim()

    const fallbackKey = [
      normalizeText(item.title),
      normalizeText(item.addr1),
      String(item.contenttypeid ?? '').trim()
    ].join('|')

    const key = contentId
      ? `contentid:${contentId}`
      : `fallback:${fallbackKey}`

    if (!uniqueItems.has(key)) {
      uniqueItems.set(key, item)
    }
  }

  return [...uniqueItems.values()]
}

const loadTourData = async () => {
  isDataLoading.value = true
  dataLoadError.value = ''

  try {
    const results = await Promise.all(
      DATA_FILES.map(async fileName => {
        const response = await fetch(
          encodeURI(`/data/${fileName}`)
        )

        if (!response.ok) {
          throw new Error(
            `${fileName} 불러오기 실패: ${response.status}`
          )
        }

        return response.json()
      })
    )

    const allItems = results.flatMap(data =>
      extractTourItems(data)
    )

    attractions.value = removeDuplicateItems(allItems)

    if (!attractions.value.length) {
      throw new Error(
        'JSON 파일에서 관광지 항목을 찾지 못했습니다.'
      )
    }
  } catch (error) {
    console.error('관광 데이터 로딩 오류:', error)
    dataLoadError.value =
      error?.message ||
      '관광 데이터를 불러오지 못했습니다.'
    attractions.value = []
  } finally {
    isDataLoading.value = false
  }
}

const detectContentTypeId = question => {
  const normalizedQuestion = normalizeText(question)

  for (const entry of CONTENT_TYPE_PATTERNS) {
    if (entry.pattern.test(normalizedQuestion)) {
      return entry.id
    }
  }

  return null
}

const extractSearchTerms = question => {
  const normalizedQuestion = normalizeText(question)

  return [
    ...new Set(
      normalizedQuestion
        .split(' ')
        .map(word => word.trim())
        .filter(Boolean)
        .filter(word => word.length >= 2)
        .filter(word => !STOP_WORDS.has(word))
    )
  ]
}

const detectRequestedFields = question => {
  const normalizedQuestion = normalizeText(question)
  const requestedFields = []

  if (/주소|위치|어디|가는 곳/.test(normalizedQuestion)) {
    requestedFields.push('주소')
  }

  if (/전화|전화번호|연락처|문의/.test(normalizedQuestion)) {
    requestedFields.push('전화번호')
  }

  if (/좌표|위도|경도|지도/.test(normalizedQuestion)) {
    requestedFields.push('좌표')
  }

  if (/사진|이미지|썸네일/.test(normalizedQuestion)) {
    requestedFields.push('이미지')
  }

  if (/우편번호/.test(normalizedQuestion)) {
    requestedFields.push('우편번호')
  }

  if (/종류|유형|분류|카테고리/.test(normalizedQuestion)) {
    requestedFields.push('콘텐츠 유형')
  }

  return requestedFields
}

const isFollowUpQuestion = question => {
  const normalizedQuestion = normalizeText(question)

  return (
    /거기|그곳|여기|그 장소|그 관광지|그 식당|그 숙소/.test(
      normalizedQuestion
    ) ||
    /^(주소|위치|전화번호|연락처|좌표|사진|이미지|우편번호|메뉴|어디)/.test(
      normalizedQuestion
    )
  )
}

const calculateSearchScore = (item, terms) => {
  if (!terms.length) return 0

  const title = normalizeText(item.title)

  const address = joinSearchableFields([
    item.addr1,
    item.addr2,
    item.zipcode
  ])

  const classifications = joinSearchableFields([
    item.cat1,
    item.cat2,
    item.cat3,
    item.lclsSystm1,
    item.lclsSystm2,
    item.lclsSystm3,
    CONTENT_TYPE_NAMES[String(item.contenttypeid ?? '')]
  ])

  const fullText = joinSearchableFields([
    item.title,
    item.addr1,
    item.addr2,
    item.zipcode,
    item.tel,
    item.cat1,
    item.cat2,
    item.cat3,
    item.lclsSystm1,
    item.lclsSystm2,
    item.lclsSystm3,
    CONTENT_TYPE_NAMES[String(item.contenttypeid ?? '')]
  ])

  return terms.reduce((score, term) => {
    let termScore = 0

    if (title === term) {
      termScore += 20
    } else if (title.includes(term)) {
      termScore += 10
    }

    if (address.includes(term)) {
      termScore += 6
    }

    if (classifications.includes(term)) {
      termScore += 3
    }

    if (fullText.includes(term) && termScore === 0) {
      termScore += 1
    }

    return score + termScore
  }, 0)
}

const searchAttractions = question => {
  const allItems = attractions.value
  const contentTypeId = detectContentTypeId(question)
  const searchTerms = extractSearchTerms(question)
  const requestedFields = detectRequestedFields(question)

  if (
    isFollowUpQuestion(question) &&
    lastContextItems.value.length
  ) {
    return {
      items: lastContextItems.value,
      totalAvailable: allItems.length,
      matchedContentTypeId: null,
      searchTerms,
      requestedFields
    }
  }

  const typeFilteredItems = contentTypeId
    ? allItems.filter(
        item =>
          String(item.contenttypeid ?? '').trim() ===
          contentTypeId
      )
    : allItems

  if (!searchTerms.length) {
    return {
      items: typeFilteredItems.slice(0, MAX_CONTEXT_ITEMS),
      totalAvailable: allItems.length,
      matchedContentTypeId: contentTypeId,
      searchTerms,
      requestedFields
    }
  }

  const scoredItems = typeFilteredItems
    .map(item => ({
      item,
      score: calculateSearchScore(item, searchTerms)
    }))
    .filter(result => result.score > 0)
    .sort((a, b) => {
      if (b.score !== a.score) {
        return b.score - a.score
      }

      return String(a.item.title ?? '').localeCompare(
        String(b.item.title ?? ''),
        'ko'
      )
    })

  return {
    items: scoredItems
      .slice(0, MAX_CONTEXT_ITEMS)
      .map(result => result.item),
    totalAvailable: allItems.length,
    matchedContentTypeId: contentTypeId,
    searchTerms,
    requestedFields
  }
}

const createModelContext = items =>
  items.map(item => ({
    contentid: String(item.contentid ?? ''),
    contenttypeid: String(item.contenttypeid ?? ''),
    contentTypeName:
      CONTENT_TYPE_NAMES[
        String(item.contenttypeid ?? '')
      ] || '알 수 없음',
    title: String(item.title ?? ''),
    addr1: String(item.addr1 ?? ''),
    addr2: String(item.addr2 ?? ''),
    zipcode: String(item.zipcode ?? ''),
    tel: String(item.tel ?? ''),
    mapx: String(item.mapx ?? ''),
    mapy: String(item.mapy ?? ''),
    firstimage: String(item.firstimage ?? ''),
    firstimage2: String(item.firstimage2 ?? ''),
    cat1: String(item.cat1 ?? ''),
    cat2: String(item.cat2 ?? ''),
    cat3: String(item.cat3 ?? ''),
    lclsSystm1: String(item.lclsSystm1 ?? ''),
    lclsSystm2: String(item.lclsSystm2 ?? ''),
    lclsSystm3: String(item.lclsSystm3 ?? ''),
    createdtime: String(item.createdtime ?? ''),
    modifiedtime: String(item.modifiedtime ?? '')
  }))

const getConversationHistory = () =>
  messages.value
    .filter(message =>
      ['user', 'bot'].includes(message.type)
    )
    .filter(
      message =>
        message.text &&
        !message.isGreeting &&
        !message.isError
    )
    .slice(-MAX_HISTORY_MESSAGES)
    .map(message => ({
      role:
        message.type === 'user'
          ? 'user'
          : 'assistant',
      content: message.text
    }))

const createSystemPrompt = ({
  modelContext,
  matchedContentTypeId,
  searchTerms,
  requestedFields
}) => `
당신은 제공된 TOUR_DATA만 조회하는 구미 관광 데이터 안내 챗봇입니다.

반드시 지켜야 하는 규칙:
1. 답변에는 TOUR_DATA에 명시된 사실만 사용하세요.
2. 모델의 사전 지식, 외부 정보, 일반 상식, 추측을 관광지의 실제 정보처럼 사용하지 마세요.
3. 웹 검색을 했다고 말하거나 외부 출처를 인용하지 마세요.
4. 장소명은 TOUR_DATA의 title 값과 정확히 일치하게 작성하세요.
5. 주소, 전화번호, 좌표, 이미지 URL, 우편번호를 임의로 만들거나 수정하지 마세요.
6. 데이터에 없는 운영시간, 영업 여부, 휴무일, 입장료, 예약 정보, 메뉴, 가격, 후기, 평점, 인기도, 혼잡도, 주차, 교통편, 소요 시간, 날씨, 방문 적기, 행사 일정을 만들지 마세요.
7. 특정 정보가 빈 문자열이거나 제공되지 않았다면 "등록된 정보가 없습니다"라고 설명하세요.
8. 추천을 요청받으면 TOUR_DATA에서 확인되는 콘텐츠 유형, 장소명, 주소만을 근거로 제시하세요.
9. 데이터만으로 장소의 품질, 분위기, 경관, 가족 적합성 등을 판단할 수 없다면 판단할 수 없다고 명시하세요.
10. 질문에 필요한 정보가 TOUR_DATA에 없으면 "현재 저장된 관광 데이터에서는 해당 정보를 확인할 수 없습니다."라고 답하세요.
11. 최대 5줄 이내로 간결하게 한국어로 답하세요.
12. 이모지를 사용하지 마세요.
13. 같은 장소를 중복해서 소개하지 마세요.
14. 사용자가 이전 답변의 장소를 지칭하면 대화 기록을 참고하되 사실 정보는 TOUR_DATA에서 확인하세요.

감지된 콘텐츠 유형:
${
  matchedContentTypeId
    ? `${CONTENT_TYPE_NAMES[matchedContentTypeId]} (${matchedContentTypeId})`
    : '지정되지 않음'
}

검색어:
${searchTerms.length ? searchTerms.join(', ') : '없음'}

요청 필드:
${
  requestedFields.length
    ? requestedFields.join(', ')
    : '특정 필드 지정 없음'
}

TOUR_DATA:
${JSON.stringify(modelContext, null, 2)}
`

const scrollToBottom = async () => {
  await nextTick()

  const container = messagesContainer.value

  if (!container) return

  container.scrollTop = container.scrollHeight
}

const openChat = async () => {
  isChatOpen.value = true
  await scrollToBottom()
}

const closeChat = () => {
  isChatOpen.value = false
}

const getErrorMessage = error => {
  const status = error?.status

  if (status === 401) {
    return 'OpenAI API 키가 올바르지 않거나 사용할 수 없습니다.'
  }

  if (status === 429) {
    return 'API 사용 한도 또는 요청 횟수를 초과했습니다.'
  }

  if (status === 400) {
    return 'API 요청 형식 또는 모델 설정을 확인해 주세요.'
  }

  if (
    error?.name === 'TypeError' &&
    /fetch|network/i.test(error?.message ?? '')
  ) {
    return '네트워크 연결 또는 브라우저의 API 요청 차단 여부를 확인해 주세요.'
  }

  return error?.message || '알 수 없는 오류가 발생했습니다.'
}

const sendMessage = async text => {
  const content = String(
    text ?? newQuestion.value
  ).trim()

  if (!content || isLoading.value || isDataLoading.value) {
    return
  }

  await openChat()

  const previousHistory = getConversationHistory()

  messages.value.push({
    type: 'user',
    text: content
  })

  newQuestion.value = ''
  isLoading.value = true

  await scrollToBottom()

  try {
    if (dataLoadError.value) {
      throw new Error(dataLoadError.value)
    }

    if (!attractions.value.length) {
      throw new Error(
        '불러온 관광 데이터가 없습니다.'
      )
    }

    if (!openai || !apiKey) {
      throw new Error(
        'VITE_OPENAI_API_KEY가 설정되지 않았습니다.'
      )
    }

    const searchResult = searchAttractions(content)

    if (!searchResult.items.length) {
      messages.value.push({
        type: 'bot',
        text: '현재 저장된 관광 데이터에서는 해당 정보를 확인할 수 없습니다.'
      })

      return
    }

    lastContextItems.value = searchResult.items

    const modelContext = createModelContext(
      searchResult.items
    )

    const systemPrompt = createSystemPrompt({
      modelContext,
      matchedContentTypeId:
        searchResult.matchedContentTypeId,
      searchTerms: searchResult.searchTerms,
      requestedFields: searchResult.requestedFields
    })

    const completion =
      await openai.chat.completions.create({
        model: OPENAI_MODEL,
        messages: [
          {
            role: 'system',
            content: systemPrompt
          },
          ...previousHistory,
          {
            role: 'user',
            content
          }
        ]
      })

    const reply =
      completion.choices?.[0]?.message?.content?.trim()

    messages.value.push({
      type: 'bot',
      text:
        reply ||
        '죄송합니다. 응답 내용을 불러오지 못했습니다.'
    })
  } catch (error) {
    console.error('챗봇 요청 오류:', error)

    messages.value.push({
      type: 'bot',
      text: `답변을 불러오지 못했습니다. ${getErrorMessage(error)}`,
      isError: true
    })
  } finally {
    isLoading.value = false
    await scrollToBottom()
  }
}

const openWithQuestion = text => {
  sendMessage(text)
}

onMounted(() => {
  loadTourData()
})

defineExpose({
  openWithQuestion,
  openChat,
  closeChat
})
</script>

<template>
  <div class="chat-shell">
    <transition name="chat-slide">
      <section
        v-if="isChatOpen"
        class="chat-panel"
        role="dialog"
        aria-label="구미 여행 챗봇"
        @click.stop
      >
        <header class="chat-header">
          <div>
            <strong>구미 여행 챗봇</strong>
            <p v-if="isDataLoading">
              관광 데이터를 불러오고 있습니다.
            </p>
            <p v-else-if="dataLoadError">
              관광 데이터 로딩에 실패했습니다.
            </p>
            <p v-else>
              {{ attractions.length }}개의 관광 데이터를 기준으로 답변합니다.
            </p>
          </div>

          <button
            type="button"
            class="close-btn"
            aria-label="챗봇 닫기"
            @click="closeChat"
          >
            ×
          </button>
        </header>

        <div
          ref="messagesContainer"
          class="chat-messages"
          aria-live="polite"
        >
          <div
            v-for="(message, index) in messages"
            :key="index"
            :class="['message', message.type]"
          >
            {{ message.text }}
          </div>

          <div
            v-if="isDataLoading"
            class="message bot"
          >
            관광 데이터를 준비하고 있습니다.
          </div>

          <div
            v-else-if="dataLoadError"
            class="message bot error-message"
          >
            {{ dataLoadError }}
          </div>

          <div
            v-if="isLoading"
            class="message bot typing-indicator"
            aria-label="답변 생성 중"
          >
            <span></span>
            <span></span>
            <span></span>
          </div>
        </div>

        <form
          class="chat-popup-input"
          @submit.prevent="sendMessage()"
        >
          <input
            v-model="newQuestion"
            type="text"
            :disabled="
              isLoading ||
              isDataLoading ||
              Boolean(dataLoadError)
            "
            :placeholder="
              isDataLoading
                ? '관광 데이터를 불러오고 있습니다.'
                : dataLoadError
                  ? '관광 데이터를 불러오지 못했습니다.'
                  : isLoading
                    ? '답변을 생성하고 있습니다.'
                    : '질문을 입력해 주세요.'
            "
            aria-label="관광 질문 입력"
          />

          <button
            type="submit"
            :disabled="
              isLoading ||
              isDataLoading ||
              Boolean(dataLoadError) ||
              !newQuestion.trim()
            "
          >
            {{ isLoading ? '전송 중' : '전송' }}
          </button>
        </form>
      </section>
    </transition>

    <button
      type="button"
      class="floating-chat"
      aria-label="챗봇 열기"
      @click="openChat"
    >
      대화
    </button>
  </div>
</template>

<style scoped>
.chat-shell {
  position: fixed;
  right: 1rem;
  bottom: 1rem;
  z-index: 1000;
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
  overflow: hidden;
  border: 1px solid rgba(148, 163, 184, 0.2);
  border-radius: 18px;
  background: #ffffff;
  box-shadow: 0 16px 40px rgba(15, 23, 42, 0.16);
}

.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-shrink: 0;
  padding: 0.95rem 1rem;
  border-bottom: 1px solid #e5e7eb;
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: #ffffff;
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
  display: grid;
  width: 36px;
  height: 36px;
  place-items: center;
  flex-shrink: 0;
  border: none;
  border-radius: 50%;
  background: transparent;
  color: #ffffff;
  font-size: 1.5rem;
  line-height: 1;
  cursor: pointer;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.15);
}

.close-btn:focus-visible {
  outline: 2px solid #ffffff;
  outline-offset: 2px;
}

.chat-messages {
  min-height: 320px;
  max-height: 560px;
  padding: 0.95rem;
  display: flex;
  flex: 1;
  flex-direction: column;
  gap: 0.65rem;
  overflow-y: auto;
  scroll-behavior: smooth;
  background: #f8fafc;
}

.message {
  width: fit-content;
  max-width: 88%;
  padding: 0.7rem 0.85rem;
  border-radius: 14px;
  line-height: 1.5;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  word-break: keep-all;
  box-shadow: 0 2px 8px rgba(15, 23, 42, 0.05);
}

.message.user {
  align-self: flex-end;
  border-bottom-right-radius: 6px;
  background: #dbeafe;
  color: #1e3a8a;
}

.message.bot {
  align-self: flex-start;
  border: 1px solid #e5e7eb;
  border-bottom-left-radius: 6px;
  background: #ffffff;
  color: #111827;
}

.error-message {
  border-color: #fecaca;
  background: #fef2f2;
  color: #991b1b;
}

.typing-indicator {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  min-width: 54px;
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
  0%,
  80%,
  100% {
    opacity: 0.6;
    transform: scale(0.8);
  }

  40% {
    opacity: 1;
    transform: scale(1);
  }
}

.chat-slide-enter-active,
.chat-slide-leave-active {
  transition:
    opacity 0.22s ease,
    transform 0.22s ease;
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
  flex-shrink: 0;
  gap: 0.6rem;
  padding: 0.8rem;
  border-top: 1px solid #e5e7eb;
  background: #ffffff;
}

.chat-popup-input input {
  min-width: 0;
  min-height: 44px;
  flex: 1;
  padding: 0 0.95rem;
  border: 1px solid #d1d5db;
  border-radius: 999px;
  background: #ffffff;
  color: #111827;
  font: inherit;
  font-size: 0.95rem;
}

.chat-popup-input input:focus {
  border-color: #2563eb;
  outline: 2px solid rgba(37, 99, 235, 0.15);
}

.chat-popup-input input:disabled {
  background: #f3f4f6;
  cursor: not-allowed;
}

.chat-popup-input button {
  min-height: 44px;
  flex-shrink: 0;
  padding: 0 1rem;
  border: none;
  border-radius: 999px;
  background: #2563eb;
  color: #ffffff;
  font: inherit;
  font-weight: 600;
  cursor: pointer;
}

.chat-popup-input button:hover:not(:disabled) {
  background: #1d4ed8;
}

.chat-popup-input button:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

.chat-popup-input button:focus-visible {
  outline: 2px solid #1d4ed8;
  outline-offset: 2px;
}

.floating-chat {
  min-width: 60px;
  height: 60px;
  padding: 0 1rem;
  border: none;
  border-radius: 999px;
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: #ffffff;
  font: inherit;
  font-weight: 700;
  box-shadow: 0 12px 30px rgba(37, 99, 235, 0.3);
  cursor: pointer;
}

.floating-chat:hover {
  transform: translateY(-1px);
}

.floating-chat:focus-visible {
  outline: 3px solid rgba(37, 99, 235, 0.35);
  outline-offset: 3px;
}

@media (max-width: 768px) {
  .chat-shell {
    right: 0.6rem;
    bottom: 0.6rem;
    left: 0.6rem;
    align-items: stretch;
  }

  .chat-panel {
    width: 100%;
    max-height: 85vh;
  }

  .chat-messages {
    min-height: 260px;
    max-height: 60vh;
  }

  .chat-popup-input {
    flex-direction: column;
  }

  .chat-popup-input button {
    width: 100%;
  }

  .floating-chat {
    min-width: 68px;
    height: 52px;
    align-self: flex-end;
  }
}

@media (prefers-reduced-motion: reduce) {
  .chat-slide-enter-active,
  .chat-slide-leave-active,
  .typing-indicator span,
  .chat-messages {
    animation: none;
    scroll-behavior: auto;
    transition: none;
  }
}
</style>
```
