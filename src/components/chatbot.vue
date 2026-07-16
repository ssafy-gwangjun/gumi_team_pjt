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

const OPENAI_MODEL =
  import.meta.env.VITE_OPENAI_MODEL || 'gpt-5-mini'

const API_KEY = import.meta.env.VITE_OPENAI_API_KEY

const MAX_SEARCH_RESULTS = 16
const MAX_MODEL_CONTEXT_ITEMS = 6
const MAX_CONVERSATION_MESSAGES = 4
const MAX_COMPLETION_TOKENS = 300

const openai = API_KEY
  ? new OpenAI({
      apiKey: API_KEY,
      dangerouslyAllowBrowser: true
    })
  : null

const CONTENT_TYPE_NAMES = {
  '12': '관광지',
  '14': '문화시설',
  '15': '축제·공연·행사',
  '25': '여행코스',
  '28': '레포츠',
  '32': '숙박',
  '38': '쇼핑',
  '39': '음식점'
}

const CONTENT_TYPE_ALIASES = {
  '12': [
    '관광지',
    '관광',
    '명소',
    '여행지',
    '볼거리',
    '가볼 만한 곳',
    '가볼만한곳',
    '갈 만한 곳',
    '갈만한곳',
    '구경할 곳',
    '구경거리',
    '나들이',
    '산책',
    '데이트',
    '사진 찍을 곳',
    '힐링',
    '공원',
    '전망대',
    '자연',
    '경치',
    '풍경',
    '드라이브'
  ],
  '14': [
    '문화시설',
    '문화 시설',
    '문화',
    '박물관',
    '미술관',
    '전시',
    '전시관',
    '기념관',
    '공연장',
    '도서관',
    '역사',
    '교육',
    '실내'
  ],
  '15': [
    '축제',
    '행사',
    '공연',
    '이벤트',
    '페스티벌',
    '볼 만한 행사',
    '공연 행사',
    '지역 행사'
  ],
  '25': [
    '여행코스',
    '여행 코스',
    '관광코스',
    '관광 코스',
    '데이트코스',
    '데이트 코스',
    '드라이브코스',
    '드라이브 코스',
    '코스',
    '일정',
    '동선',
    '하루 여행',
    '당일치기'
  ],
  '28': [
    '레포츠',
    '스포츠',
    '체험',
    '액티비티',
    '놀거리',
    '활동',
    '운동',
    '야외 활동',
    '야외활동',
    '자전거',
    '트레킹',
    '등산'
  ],
  '32': [
    '숙박',
    '숙소',
    '호텔',
    '모텔',
    '펜션',
    '민박',
    '게스트하우스',
    '리조트',
    '콘도',
    '잘 곳',
    '자는 곳',
    '머물 곳',
    '하룻밤'
  ],
  '38': [
    '쇼핑',
    '시장',
    '상점',
    '마트',
    '백화점',
    '아울렛',
    '기념품',
    '선물',
    '살 곳',
    '장보기',
    '특산품'
  ],
  '39': [
    '음식점',
    '식당',
    '맛집',
    '밥집',
    '먹거리',
    '먹을 곳',
    '카페',
    '커피',
    '디저트',
    '분식',
    '술집',
    '한식',
    '중식',
    '일식',
    '양식',
    '밥',
    '음식',
    '먹는 곳',
    '식사',
    '뭐 먹지'
  ]
}

const FIELD_ALIASES = {
  address: [
    '주소',
    '위치',
    '어디',
    '어디야',
    '어딨어',
    '장소',
    '소재지',
    '찾아가는 곳'
  ],
  phone: [
    '전화',
    '전화번호',
    '연락처',
    '문의',
    '문의 번호',
    '번호'
  ],
  coordinates: [
    '좌표',
    '위도',
    '경도',
    '지도 위치',
    '맵 위치'
  ],
  image: [
    '사진',
    '이미지',
    '그림',
    '썸네일',
    '모습'
  ],
  zipcode: [
    '우편번호',
    '우편 번호'
  ],
  category: [
    '유형',
    '분류',
    '종류',
    '카테고리'
  ]
}

const EXPANSION_RULES = [
  {
    pattern: /데이트|연인|커플/,
    types: ['12', '25', '39'],
    terms: ['관광지', '여행코스', '음식점']
  },
  {
    pattern: /아이|아기|어린이|가족/,
    types: ['12', '14', '28'],
    terms: ['관광지', '문화시설', '체험']
  },
  {
    pattern: /부모님|어른|어르신/,
    types: ['12', '14', '25'],
    terms: ['관광지', '문화시설', '여행코스']
  },
  {
    pattern: /비\s*오는|비올|비가|우천|장마/,
    types: ['14', '38', '39'],
    terms: ['문화시설', '쇼핑', '음식점']
  },
  {
    pattern: /실내/,
    types: ['14', '38', '39'],
    terms: ['문화시설', '쇼핑', '음식점']
  },
  {
    pattern: /야외|밖에서/,
    types: ['12', '28', '25'],
    terms: ['관광지', '레포츠', '여행코스']
  },
  {
    pattern: /산책|걷기|걸을/,
    types: ['12', '25'],
    terms: ['관광지', '여행코스', '산책']
  },
  {
    pattern: /놀거리|재밌|활동적|몸\s*쓰/,
    types: ['28', '12'],
    terms: ['레포츠', '체험', '관광지']
  },
  {
    pattern: /조용|한적|힐링|쉬고/,
    types: ['12', '25'],
    terms: ['관광지', '여행코스', '자연']
  },
  {
    pattern: /먹고|배고|식사|밥/,
    types: ['39'],
    terms: ['음식점']
  },
  {
    pattern: /자고|숙박|머물|하룻밤/,
    types: ['32'],
    terms: ['숙박']
  },
  {
    pattern: /선물|기념품|살\s*것|쇼핑/,
    types: ['38'],
    terms: ['쇼핑']
  }
]

const STOP_WORDS = new Set([
  '구미',
  '경북',
  '경상북도',
  '구미시',
  '관광',
  '여행',
  '장소',
  '정보',
  '관련',
  '대해',
  '대한',
  '좀',
  '약간',
  '그냥',
  '혹시',
  '그러면',
  '그리고',
  '근데',
  '그런데',
  '알려줘',
  '알려주세요',
  '알려',
  '말해줘',
  '말해주세요',
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
  '뭔가',
  '어떤',
  '있는',
  '있어',
  '있나요',
  '있나',
  '있을까',
  '없어',
  '없나요',
  '주세요',
  '해줘',
  '해주라',
  '가고',
  '싶어',
  '싶은데',
  '좋은',
  '괜찮은',
  '적당한',
  '근처',
  '주변',
  '인근',
  '부근',
  '지역',
  '권역',
  '거기',
  '그곳',
  '여기',
  '저기',
  '그거',
  '그건',
  '이거',
  '저거',
  '여기는',
  '거기는',
  '곳은',
  '곳이',
  '곳을',
  '곳에서',
  '곳으로',
  '할만한',
  '만한',
  '갈만한',
  '가볼만한',
  '추천할',
  '하나',
  '몇개',
  '몇',
  '개',
  '군데',
  '여러개',
  '여러',
  '뭐',
  '왜',
  '어떻게',
  '제일',
  '가장',
  '내가',
  '나는',
  '우리',
  '하고',
  '해서',
  '할까',
  '할지',
  '좋을까',
  '괜찮을까'
])

const messages = ref([
  {
    type: 'bot',
    text: '구미 관광과 관련해 궁금한 내용을 편하게 질문해 주세요.',
    isGreeting: true
  }
])

const attractions = ref([])
const lastContextItems = ref([])
const lastQuestionAnalysis = ref(null)
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

const compactText = value =>
  normalizeText(value).replace(/\s+/g, '')

const getInitialConsonants = value => {
  const consonants = [
    'ㄱ',
    'ㄲ',
    'ㄴ',
    'ㄷ',
    'ㄸ',
    'ㄹ',
    'ㅁ',
    'ㅂ',
    'ㅃ',
    'ㅅ',
    'ㅆ',
    'ㅇ',
    'ㅈ',
    'ㅉ',
    'ㅊ',
    'ㅋ',
    'ㅌ',
    'ㅍ',
    'ㅎ'
  ]

  return String(value ?? '')
    .split('')
    .map(character => {
      const code = character.charCodeAt(0)

      if (code >= 0xac00 && code <= 0xd7a3) {
        return consonants[Math.floor((code - 0xac00) / 588)]
      }

      return character
    })
    .join('')
}

const levenshteinDistance = (leftValue, rightValue) => {
  const left = compactText(leftValue)
  const right = compactText(rightValue)

  if (!left.length) return right.length
  if (!right.length) return left.length
  if (left === right) return 0

  const previous = Array.from(
    { length: right.length + 1 },
    (_, index) => index
  )

  for (
    let leftIndex = 1;
    leftIndex <= left.length;
    leftIndex += 1
  ) {
    const current = [leftIndex]

    for (
      let rightIndex = 1;
      rightIndex <= right.length;
      rightIndex += 1
    ) {
      const substitutionCost =
        left[leftIndex - 1] === right[rightIndex - 1]
          ? 0
          : 1

      current[rightIndex] = Math.min(
        current[rightIndex - 1] + 1,
        previous[rightIndex] + 1,
        previous[rightIndex - 1] + substitutionCost
      )
    }

    previous.splice(0, previous.length, ...current)
  }

  return previous[right.length]
}

const getSimilarity = (leftValue, rightValue) => {
  const left = compactText(leftValue)
  const right = compactText(rightValue)

  if (!left || !right) return 0
  if (left === right) return 1

  if (left.includes(right) || right.includes(left)) {
    return 0.9
  }

  const distance = levenshteinDistance(left, right)
  const longestLength = Math.max(left.length, right.length)

  if (!longestLength) return 0

  return Math.max(0, 1 - distance / longestLength)
}

const createNgrams = (value, size = 2) => {
  const text = compactText(value)

  if (!text) return []
  if (text.length <= size) return [text]

  const ngrams = []

  for (
    let index = 0;
    index <= text.length - size;
    index += 1
  ) {
    ngrams.push(text.slice(index, index + size))
  }

  return ngrams
}

const getNgramSimilarity = (leftValue, rightValue) => {
  const leftNgrams = new Set(createNgrams(leftValue))
  const rightNgrams = new Set(createNgrams(rightValue))

  if (!leftNgrams.size || !rightNgrams.size) {
    return 0
  }

  let intersectionCount = 0

  leftNgrams.forEach(ngram => {
    if (rightNgrams.has(ngram)) {
      intersectionCount += 1
    }
  })

  return (
    (2 * intersectionCount) /
    (leftNgrams.size + rightNgrams.size)
  )
}

const isTourItem = value => {
  if (
    !value ||
    typeof value !== 'object' ||
    Array.isArray(value)
  ) {
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
    return value.flatMap(entry =>
      extractTourItems(entry)
    )
  }

  if (typeof value !== 'object') {
    return []
  }

  if (isTourItem(value)) {
    return [value]
  }

  if (Array.isArray(value.items)) {
    const inheritedContentTypeId = String(
      value.contentTypeId ??
        value.contenttypeid ??
        ''
    ).trim()

    return value.items
      .filter(
        item =>
          item &&
          typeof item === 'object'
      )
      .map(item => ({
        ...item,
        contenttypeid:
          String(
            item.contenttypeid ?? ''
          ).trim() ||
          inheritedContentTypeId
      }))
  }

  return Object.values(value).flatMap(entry =>
    extractTourItems(entry)
  )
}

const removeDuplicateItems = items => {
  const uniqueItems = new Map()

  items.forEach(item => {
    const contentId = String(
      item.contentid ?? ''
    ).trim()

    const fallbackKey = [
      compactText(item.title),
      compactText(item.addr1),
      String(
        item.contenttypeid ?? ''
      ).trim()
    ].join('|')

    const key = contentId
      ? `contentid:${contentId}`
      : `fallback:${fallbackKey}`

    if (!uniqueItems.has(key)) {
      uniqueItems.set(key, item)
    }
  })

  return [...uniqueItems.values()]
}

const createSearchMetadata = item => {
  const typeId = String(
    item.contenttypeid ?? ''
  ).trim()

  const typeName =
    CONTENT_TYPE_NAMES[typeId] || ''

  const title = normalizeText(item.title)

  const address = normalizeText(
    [
      item.addr1,
      item.addr2,
      item.zipcode
    ]
      .filter(Boolean)
      .join(' ')
  )

  const categories = normalizeText(
    [
      item.cat1,
      item.cat2,
      item.cat3,
      item.lclsSystm1,
      item.lclsSystm2,
      item.lclsSystm3,
      typeName
    ]
      .filter(Boolean)
      .join(' ')
  )

  const fullText = normalizeText(
    [
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
      typeName
    ]
      .filter(Boolean)
      .join(' ')
  )

  return {
    ...item,
    _search: {
      title,
      compactTitle: compactText(title),
      titleInitials:
        getInitialConsonants(title),
      address,
      categories,
      fullText,
      compactFullText:
        compactText(fullText)
    }
  }
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

    const items = results.flatMap(result =>
      extractTourItems(result)
    )

    attractions.value = removeDuplicateItems(
      items
    ).map(createSearchMetadata)

    if (!attractions.value.length) {
      throw new Error(
        'JSON 파일에서 관광 데이터를 찾지 못했습니다.'
      )
    }
  } catch (error) {
    console.error(
      '관광 데이터 로딩 오류:',
      error
    )

    dataLoadError.value =
      error?.message ||
      '관광 데이터를 불러오지 못했습니다.'

    attractions.value = []
  } finally {
    isDataLoading.value = false
  }
}

const extractLocalContentTypes = question => {
  const normalizedQuestion =
    normalizeText(question)

  const compactQuestion =
    compactText(question)

  const typeScores = []

  Object.entries(
    CONTENT_TYPE_ALIASES
  ).forEach(([typeId, aliases]) => {
    let score = 0

    aliases.forEach(alias => {
      const normalizedAlias =
        normalizeText(alias)

      const compactAlias =
        compactText(alias)

      if (
        normalizedQuestion.includes(
          normalizedAlias
        ) ||
        compactQuestion.includes(
          compactAlias
        )
      ) {
        score +=
          compactAlias.length >= 4 ? 5 : 3
      }
    })

    if (score > 0) {
      typeScores.push({
        typeId,
        score
      })
    }
  })

  return typeScores
    .sort(
      (left, right) =>
        right.score - left.score
    )
    .map(entry => entry.typeId)
}

const extractLocalRequestedFields =
  question => {
    const normalizedQuestion =
      normalizeText(question)

    const compactQuestion =
      compactText(question)

    const requestedFields = []

    Object.entries(
      FIELD_ALIASES
    ).forEach(([field, aliases]) => {
      const matched = aliases.some(
        alias => {
          const normalizedAlias =
            normalizeText(alias)

          return (
            normalizedQuestion.includes(
              normalizedAlias
            ) ||
            compactQuestion.includes(
              compactText(
                normalizedAlias
              )
            )
          )
        }
      )

      if (matched) {
        requestedFields.push(field)
      }
    })

    return requestedFields
  }

const extractLocalSearchTerms = question => {
  const normalizedQuestion =
    normalizeText(question)

  return [
    ...new Set(
      normalizedQuestion
        .split(' ')
        .map(word => word.trim())
        .filter(Boolean)
        .filter(
          word => word.length >= 2
        )
        .filter(
          word => !STOP_WORDS.has(word)
        )
    )
  ]
}

const isLocalFollowUpQuestion =
  question => {
    const normalizedQuestion =
      normalizeText(question)

    return (
      /거기|그곳|그 장소|그 식당|그 숙소|그 관광지|그 코스|그거|그건|아까|방금|첫 번째|첫번째|두 번째|두번째|세 번째|세번째|마지막|위에 말한|전에 말한/.test(
        normalizedQuestion
      ) ||
      /^(주소|위치|전화|전화번호|연락처|좌표|사진|이미지|종류|유형|어디)/.test(
        normalizedQuestion
      )
    )
  }

const extractExpandedAnalysis = question => {
  const normalizedQuestion =
    normalizeText(question)

  const types = []
  const terms = []

  EXPANSION_RULES.forEach(rule => {
    if (
      rule.pattern.test(
        normalizedQuestion
      )
    ) {
      types.push(...rule.types)
      terms.push(...rule.terms)
    }
  })

  return {
    types: [...new Set(types)],
    terms: [...new Set(terms)]
  }
}

const extractRequestedCount = question => {
  const normalizedQuestion =
    normalizeText(question)

  const numberMatch =
    normalizedQuestion.match(
      /(\d+)\s*(개|곳|군데)/
    )

  if (numberMatch) {
    return Math.min(
      Math.max(
        Number(numberMatch[1]),
        1
      ),
      10
    )
  }

  const koreanNumbers = [
    {
      pattern: /한\s*(개|곳|군데)/,
      count: 1
    },
    {
      pattern: /두\s*(개|곳|군데)/,
      count: 2
    },
    {
      pattern: /세\s*(개|곳|군데)/,
      count: 3
    },
    {
      pattern: /네\s*(개|곳|군데)/,
      count: 4
    },
    {
      pattern: /다섯\s*(개|곳|군데)/,
      count: 5
    }
  ]

  const matched =
    koreanNumbers.find(entry =>
      entry.pattern.test(
        normalizedQuestion
      )
    )

  return matched?.count || 3
}

const createLocalQuestionAnalysis =
  question => {
    const normalizedQuestion =
      normalizeText(question)

    const expanded =
      extractExpandedAnalysis(question)

    const contentTypeIds = [
      ...new Set([
        ...extractLocalContentTypes(
          question
        ),
        ...expanded.types
      ])
    ]

    const wantsComparison =
      /비교|차이|어디가 더|뭐가 더|둘 중|둘중/.test(
        normalizedQuestion
      )

    const wantsRecommendation =
      /추천|괜찮|좋은|갈만|가볼만|먹을만|어디가 좋|뭐가 좋|골라/.test(
        normalizedQuestion
      )

    const requestedFields =
      extractLocalRequestedFields(
        question
      )

    let intent = 'search'

    if (wantsComparison) {
      intent = 'compare'
    } else if (wantsRecommendation) {
      intent = 'recommend'
    } else if (
      requestedFields.length ||
      isLocalFollowUpQuestion(question)
    ) {
      intent = 'detail'
    }

    return {
      intent,
      contentTypeIds,
      searchTerms:
        extractLocalSearchTerms(
          question
        ),
      expandedTerms: expanded.terms,
      requestedFields,
      isFollowUp:
        isLocalFollowUpQuestion(
          question
        ),
      wantsRecommendation,
      wantsComparison,
      requestedCount:
        extractRequestedCount(
          question
        ),
      originalQuestion: question
    }
  }

const getConversationHistory = () =>
  messages.value
    .filter(message =>
      ['user', 'bot'].includes(
        message.type
      )
    )
    .filter(
      message =>
        message.text &&
        !message.isGreeting &&
        !message.isError
    )
    .slice(
      -MAX_CONVERSATION_MESSAGES
    )
    .map(message => ({
      role:
        message.type === 'user'
          ? 'user'
          : 'assistant',
      content: message.text
    }))

const getTermScore = (item, term) => {
  const normalizedTerm =
    normalizeText(term)

  const compactTerm =
    compactText(term)

  if (
    !normalizedTerm ||
    !compactTerm
  ) {
    return 0
  }

  const title = item._search.title
  const compactTitle =
    item._search.compactTitle

  const titleInitials =
    item._search.titleInitials

  const address =
    item._search.address

  const categories =
    item._search.categories

  const compactFullText =
    item._search.compactFullText

  let score = 0

  if (compactTitle === compactTerm) {
    score += 90
  } else if (
    compactTitle.startsWith(
      compactTerm
    )
  ) {
    score += 60
  } else if (
    compactTitle.includes(
      compactTerm
    )
  ) {
    score += 48
  } else if (
    compactTerm.includes(
      compactTitle
    )
  ) {
    score += 34
  }

  if (
    titleInitials === compactTerm ||
    titleInitials.includes(
      compactTerm
    )
  ) {
    score += 30
  }

  if (
    address.includes(
      normalizedTerm
    )
  ) {
    score += 22
  }

  if (
    categories.includes(
      normalizedTerm
    )
  ) {
    score += 18
  }

  if (
    compactFullText.includes(
      compactTerm
    )
  ) {
    score += 12
  }

  const titleSimilarity =
    getSimilarity(
      compactTitle,
      compactTerm
    )

  if (
    titleSimilarity >= 0.85
  ) {
    score +=
      38 * titleSimilarity
  } else if (
    titleSimilarity >= 0.65
  ) {
    score +=
      22 * titleSimilarity
  }

  const ngramSimilarity =
    getNgramSimilarity(
      compactTitle,
      compactTerm
    )

  if (
    ngramSimilarity >= 0.45
  ) {
    score +=
      17 * ngramSimilarity
  }

  title
    .split(' ')
    .forEach(word => {
      const wordSimilarity =
        getSimilarity(
          word,
          normalizedTerm
        )

      if (
        wordSimilarity >= 0.8
      ) {
        score +=
          18 * wordSimilarity
      }
    })

  return score
}

const getContentTypeScore = (
  item,
  contentTypeIds
) => {
  if (!contentTypeIds.length) {
    return 0
  }

  const typeId = String(
    item.contenttypeid ?? ''
  ).trim()

  if (
    contentTypeIds.includes(
      typeId
    )
  ) {
    return 30
  }

  return -14
}

const getAliasScore = (
  item,
  question
) => {
  const typeId = String(
    item.contenttypeid ?? ''
  ).trim()

  const aliases =
    CONTENT_TYPE_ALIASES[
      typeId
    ] || []

  const normalizedQuestion =
    normalizeText(question)

  const compactQuestion =
    compactText(question)

  return aliases.reduce(
    (score, alias) => {
      const normalizedAlias =
        normalizeText(alias)

      const compactAlias =
        compactText(alias)

      if (
        normalizedQuestion.includes(
          normalizedAlias
        ) ||
        compactQuestion.includes(
          compactAlias
        )
      ) {
        return score + 9
      }

      return score
    },
    0
  )
}

const searchAttractions = ({
  question,
  analysis
}) => {
  const allTerms = [
    ...new Set([
      ...analysis.searchTerms,
      ...analysis.expandedTerms
    ])
  ].filter(Boolean)

  if (
    analysis.isFollowUp &&
    lastContextItems.value.length &&
    !analysis.searchTerms.length
  ) {
    return lastContextItems.value.slice(
      0,
      MAX_MODEL_CONTEXT_ITEMS
    )
  }

  const scoredItems =
    attractions.value
      .map(item => {
        let score = 0

        score +=
          getContentTypeScore(
            item,
            analysis.contentTypeIds
          )

        score += getAliasScore(
          item,
          question
        )

        allTerms.forEach(
          (term, index) => {
            const weight =
              index <
              analysis.searchTerms
                .length
                ? 1
                : 0.6

            score +=
              getTermScore(
                item,
                term
              ) * weight
          }
        )

        if (
          !allTerms.length &&
          analysis.contentTypeIds.includes(
            String(
              item.contenttypeid ??
                ''
            ).trim()
          )
        ) {
          score += 15
        }

        if (
          !allTerms.length &&
          !analysis.contentTypeIds.length
        ) {
          score += 1
        }

        return {
          item,
          score
        }
      })
      .filter(result => {
        if (
          !allTerms.length &&
          !analysis.contentTypeIds
            .length
        ) {
          return true
        }

        return result.score > 0
      })
      .sort(
        (left, right) => {
          if (
            right.score !==
            left.score
          ) {
            return (
              right.score -
              left.score
            )
          }

          return String(
            left.item.title ?? ''
          ).localeCompare(
            String(
              right.item.title ??
                ''
            ),
            'ko'
          )
        }
      )

  let results = scoredItems
    .slice(0, MAX_SEARCH_RESULTS)
    .map(result => result.item)

  if (
    !results.length &&
    analysis.contentTypeIds.length
  ) {
    results =
      attractions.value
        .filter(item =>
          analysis.contentTypeIds.includes(
            String(
              item.contenttypeid ??
                ''
            ).trim()
          )
        )
        .slice(
          0,
          MAX_SEARCH_RESULTS
        )
  }

  if (
    !results.length &&
    analysis.isFollowUp &&
    lastContextItems.value.length
  ) {
    results =
      lastContextItems.value
  }

  return results.slice(
    0,
    MAX_MODEL_CONTEXT_ITEMS
  )
}

const createModelItem = item => ({
  contentid: String(
    item.contentid ?? ''
  ),
  contenttypeid: String(
    item.contenttypeid ?? ''
  ),
  contentTypeName:
    CONTENT_TYPE_NAMES[
      String(
        item.contenttypeid ?? ''
      ).trim()
    ] || '분류 정보 없음',
  title: String(
    item.title ?? ''
  ),
  address: [
    item.addr1,
    item.addr2
  ]
    .filter(Boolean)
    .join(' '),
  zipcode: String(
    item.zipcode ?? ''
  ),
  tel: String(item.tel ?? ''),
  mapx: String(item.mapx ?? ''),
  mapy: String(item.mapy ?? ''),
  image: String(
    item.firstimage ||
      item.firstimage2 ||
      ''
  )
})

const buildFallbackReply = ({
  analysis,
  items
}) => {
  if (!items.length) {
    return '현재 저장된 관광 데이터에서는 질문과 관련된 정보를 찾지 못했습니다. 장소 이름이나 원하는 유형을 조금 더 구체적으로 알려주세요.'
  }

  const count = Math.min(
    analysis.requestedCount || 3,
    items.length,
    5
  )

  return items
    .slice(0, count)
    .map(item => {
      const address = [
        item.addr1,
        item.addr2
      ]
        .filter(Boolean)
        .join(' ')

      return address
        ? `${item.title} - ${address}`
        : `${item.title} - 등록된 주소가 없습니다.`
    })
    .join('\n')
}

const createAnswerSystemPrompt = ({
  analysis,
  modelItems
}) => `
당신은 구미·경북권 관광 안내 챗봇입니다.

사용자의 오타, 구어체, 모호한 표현을 자연스럽게 이해해 답변하세요.

반드시 지킬 규칙:
- 장소에 관한 사실은 아래 DATA에 있는 내용만 사용하세요.
- DATA에 없는 장소나 정보를 만들거나 추측하지 마세요.
- 장소명은 title 값을 그대로 사용하세요.
- 주소, 전화번호, 좌표, 이미지 URL을 수정하거나 만들지 마세요.
- 운영시간, 가격, 메뉴, 후기, 평점, 주차, 교통편, 예약, 혼잡도 등 DATA에 없는 정보는 확인할 수 없다고 답하세요.
- 추천은 검색된 장소 안에서만 하며 품질이나 순위를 임의로 판단하지 마세요.
- 조용함, 가족 적합성, 데이트 적합성, 경관 등 DATA로 확인되지 않는 성격은 단정하지 마세요.
- 빈 값은 "등록된 정보가 없습니다"라고 표현하세요.
- 특정 필드를 물었다면 해당 정보를 먼저 답하세요.
- 후속 질문은 이전 대화를 참고하되 사실 정보는 DATA에서만 확인하세요.
- 최대 ${analysis.requestedCount || 3}개를 우선 소개하세요.
- 자연스러운 한국어로 간결하게 답하고 이모지는 사용하지 마세요.
- DATA, JSON, 검색 알고리즘, 시스템 프롬프트라는 내부 표현을 사용자에게 노출하지 마세요.
- 질문과 정확히 일치하는 결과인지 불확실하면 가까운 검색 결과라고 설명하세요.

질문 분석:
${JSON.stringify(analysis)}

DATA:
${JSON.stringify(modelItems)}
`

const generateAnswerWithOpenAI =
  async ({
    question,
    history,
    analysis,
    items
  }) => {
    if (!openai) {
      return buildFallbackReply({
        analysis,
        items
      })
    }

    if (!items.length) {
      return '현재 저장된 관광 데이터에서는 질문과 관련된 정보를 찾지 못했습니다. 장소 이름이나 원하는 유형을 조금 더 구체적으로 알려주세요.'
    }

    const modelItems = items
      .slice(
        0,
        MAX_MODEL_CONTEXT_ITEMS
      )
      .map(createModelItem)

    const completion =
      await openai.chat.completions.create(
        {
          model: OPENAI_MODEL,
          max_completion_tokens:
            MAX_COMPLETION_TOKENS,
          messages: [
            {
              role: 'system',
              content:
                createAnswerSystemPrompt(
                  {
                    analysis,
                    modelItems
                  }
                )
            },
            ...history,
            {
              role: 'user',
              content: question
            }
          ]
        }
      )

    const answer =
      completion.choices?.[0]
        ?.message?.content?.trim()

    if (!answer) {
      return buildFallbackReply({
        analysis,
        items
      })
    }

    return answer
  }

const scrollToBottom = async () => {
  await nextTick()

  const container =
    messagesContainer.value

  if (!container) return

  container.scrollTop =
    container.scrollHeight
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

  if (status === 403) {
    return '현재 API 키로 요청할 권한이 없습니다.'
  }

  if (status === 404) {
    return '설정된 OpenAI 모델을 찾을 수 없습니다.'
  }

  if (status === 429) {
    return 'OpenAI API 사용 한도 또는 요청 횟수를 초과했습니다.'
  }

  if (status === 400) {
    return (
      error?.error?.message ||
      error?.message ||
      'OpenAI API 요청 형식이나 모델 설정을 확인해 주세요.'
    )
  }

  if (
    error?.name === 'TypeError' &&
    /fetch|network|failed/i.test(
      error?.message ?? ''
    )
  ) {
    return '네트워크 연결이나 브라우저 요청 차단 여부를 확인해 주세요.'
  }

  return (
    error?.error?.message ||
    error?.message ||
    '알 수 없는 오류가 발생했습니다.'
  )
}

const sendMessage = async text => {
  const content = String(
    text ?? newQuestion.value
  ).trim()

  if (
    !content ||
    isLoading.value ||
    isDataLoading.value
  ) {
    return
  }

  await openChat()

  const history =
    getConversationHistory()

  messages.value.push({
    type: 'user',
    text: content
  })

  newQuestion.value = ''
  isLoading.value = true

  await scrollToBottom()

  try {
    if (dataLoadError.value) {
      throw new Error(
        dataLoadError.value
      )
    }

    if (!attractions.value.length) {
      throw new Error(
        '불러온 관광 데이터가 없습니다.'
      )
    }

    if (!openai || !API_KEY) {
      throw new Error(
        'VITE_OPENAI_API_KEY가 설정되지 않았습니다.'
      )
    }

    const analysis =
      createLocalQuestionAnalysis(
        content
      )

    lastQuestionAnalysis.value =
      analysis

    const searchResults =
      searchAttractions({
        question: content,
        analysis
      })

    if (searchResults.length) {
      lastContextItems.value =
        searchResults
    }

    const answer =
      await generateAnswerWithOpenAI(
        {
          question: content,
          history,
          analysis,
          items: searchResults
        }
      )

    messages.value.push({
      type: 'bot',
      text: answer
    })
  } catch (error) {
    console.error(
      '챗봇 요청 오류 전체:',
      error
    )

    console.error(
      'OpenAI 오류 상태:',
      error?.status
    )

    console.error(
      'OpenAI 오류 메시지:',
      error?.message
    )

    console.error(
      'OpenAI 오류 내용:',
      error?.error
    )

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
            <strong>
              구미 여행 챗봇
            </strong>

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
            :class="[
              'message',
              message.type,
              {
                'error-message':
                  message.isError
              }
            ]"
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
            autocomplete="off"
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
                    : '편하게 질문해 주세요.'
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
            {{
              isLoading
                ? '전송 중'
                : '전송'
            }}
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
  letter-spacing: -0.025em;
}

.chat-header p {
  margin: 0.2rem 0 0;
  font-size: 0.82rem;
  opacity: 0.9;
}

.close-btn {
  display: grid;
  width: 36px;
  height: 36px;
  place-items: center;
  flex-shrink: 0;
  padding: 0.2rem;
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
  font-size: 0.92rem;
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
  outline: none;
  transition:
    border-color 0.2s,
    box-shadow 0.2s;
}

.chat-popup-input input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.15);
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
  transition: background-color 0.2s;
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
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}

.floating-chat:hover {
  transform: translateY(-1px);
  box-shadow: 0 15px 34px rgba(37, 99, 235, 0.34);
}

.floating-chat:focus-visible {
  outline: 3px solid rgba(37, 99, 235, 0.35);
  outline-offset: 3px;
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
    max-height: 55vh;
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
  .chat-messages,
  .floating-chat {
    animation: none;
    scroll-behavior: auto;
    transition: none;
  }
}
</style>