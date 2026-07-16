<script setup>
import { ref, computed, onMounted, nextTick } from "vue";

// =====================================================
// 기본 상태
// =====================================================

const open = ref(false);
const input = ref("");
const loading = ref(false);
const chatBody = ref(null);

const messages = ref([
  {
    role: "assistant",
    content:
      "안녕하세요 😊 구미·경북권 여행 정보를 도와드리는 챗봇이에요.\n관광지, 축제, 모범음식점, 숙박, 여행코스 등 궁금한 걸 편하게 물어보세요!",
  },
]);

// 초기 화면에서 보여줄 추천 질문 (예시 질의 유형을 그대로 반영)
const suggestedPrompts = [
  "아이랑 갈만한 관광지 추천해줘",
  "이번 달 열리는 축제 알려줘",
  "구미 모범음식점 위치 알려줘",
  "하루 코스로 돌아볼만한 여행코스 있어?",
];

function useSuggestion(text) {
  input.value = text;
  sendMessage();
}

// =====================================================
// JSON 데이터 설정 & 로딩
// =====================================================

const CATEGORY_LIST = [
  "관광지",
  "문화시설",
  "축제",
  "레포츠",
  "숙박",
  "쇼핑",
  "음식점",
  "여행코스",
];

const jsonFiles = [
  { category: "관광지", path: "/data/구미_경북권_관광지.json" },
  { category: "문화시설", path: "/data/구미_경북권_문화시설.json" },
  { category: "축제", path: "/data/구미_경북권_축제공연행사.json" },
  { category: "레포츠", path: "/data/구미_경북권_레포츠.json" },
  { category: "숙박", path: "/data/구미_경북권_숙박.json" },
  { category: "쇼핑", path: "/data/구미_경북권_쇼핑.json" },
  { category: "음식점", path: "/data/구미_경북권_음식점.json" },
  { category: "여행코스", path: "/data/구미_경북권_여행코스.json" },
];

const regionDB = ref([]);
const dbReady = ref(false);

// TourAPI 4.0 원본 필드 → 챗봇이 실제로 쓰는 필드로 정규화.
// 원본에는 cat1~3, lclsSystm 같은 분류 코드와 createdtime 등
// 검색/답변에 쓸모없는 값이 많아서, 여기서 필요한 것만 추려낸다.
// (설명/일정/요금 같은 텍스트 필드는 원본에 아예 없음 — SCHEMA.md 기준)
function normalizeItem(raw, category) {
  const address = [raw.addr1, raw.addr2].filter(Boolean).join(" ").trim();
  return {
    id: raw.contentid,
    category,
    name: raw.title || "",
    address,
    tel: raw.tel || "",
    image: raw.firstimage2 || raw.firstimage || "",
    lat: raw.mapy ? Number(raw.mapy) : null,
    lng: raw.mapx ? Number(raw.mapx) : null,
  };
}

onMounted(async () => {
  const result = await Promise.all(
    jsonFiles.map(async (file) => {
      try {
        const response = await fetch(file.path);
        const json = await response.json();
        // 최상위가 배열이 아니라 { region, items:[...] } 형태의 객체
        const items = Array.isArray(json) ? json : json.items || [];
        return items.map((item) => normalizeItem(item, file.category));
      } catch (error) {
        console.error("[지역 데이터 로딩 실패]", file.path, error);
        return [];
      }
    })
  );

  regionDB.value = result.flat();
  dbReady.value = true;
  console.log("지역 데이터 로딩 완료:", regionDB.value.length, "건");
});

// =====================================================
// OpenAI 공통 호출 헬퍼
// =====================================================

async function callOpenAI({ system, user, history = [], json = false, model = "gpt-5-mini" }) {
  const body = {
    model,
    messages: [
      { role: "system", content: system },
      ...history,
      { role: "user", content: user },
    ],
  };

  if (json) {
    body.response_format = { type: "json_object" };
  }

  const response = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    const errText = await response.text();
    throw new Error(`OpenAI API 오류 (${response.status}): ${errText}`);
  }

  const data = await response.json();
  return data.choices[0].message.content;
}

// =====================================================
// 1단계: 질문 분석 (자연어 → 구조화된 검색 조건)
// =====================================================
// 단순 카테고리 하나가 아니라, 복합 질문("축제랑 근처 맛집")도
// 어느 정도 대응할 수 있도록 카테고리를 배열로 받고,
// 질의 유형(queryType)까지 뽑아서 답변 톤을 다르게 가져간다.

async function analyzeQuestion(question, historyForLLM) {
  const system = `
너는 "구미·경북권 지역 정보 챗봇"의 질문 분석기다.
사용자의 질문을 분석해서 아래 JSON 형식으로만 답한다. 설명 문장은 절대 추가하지 않는다.

{
  "inScope": true,
  "categories": ["관광지"],
  "keywords": ["아이", "체험", "가족"],
  "queryType": "recommend",
  "clarifyNeeded": false,
  "clarifyQuestion": ""
}

중요: 이 데이터베이스의 각 항목에는 "이름(장소/행사명)", "주소", "전화번호"만 있다.
설명, 소개글, 날짜/기간, 요금, 프로그램, 대상 연령 같은 텍스트 정보는 전혀 없다.
따라서 keywords는 이름이나 주소에 실제로 등장할 법한 "고유한 단어"만 뽑아야 한다.
(장소명 일부, 음식/축제 테마 단어, 동/읍/면/거리 이름 등)
"아이랑", "가족끼리", "힐링", "분위기 좋은", "10월에" 처럼 이름/주소에 나타나지 않을
추상적 수식어나 날짜는 keywords에 넣지 말고 비워 둔다(빈 배열). 그런 요청은 카테고리
필터만으로 넓게 찾은 뒤 답변에서 자연스럽게 풀어주면 된다.

필드 설명:
- inScope: 아래 categories 중 하나라도 관련이 있으면 true.
  단, "커뮤니티 게시글", "회원 후기", "로그인", "예약 내역" 등
  이 데이터베이스와 무관한 요청이면 false로 표시한다.
- categories: 아래 목록 중에서만 1~2개 선택 (관련 있는 것만, 애매하면 가장 유력한 1개만)
  ["관광지","문화시설","축제","레포츠","숙박","쇼핑","음식점","여행코스"]
  ※ "모범음식점", "맛집", "우수업소" 관련 질문은 category "음식점"으로 매핑
    (단, "모범업소"라는 단어가 실제 상호명에 들어있는 경우는 드무니 keywords에는 넣지 않는다)
- keywords: 이름/주소에 실제로 나타날 만한 고유 단어 0~5개. 없으면 빈 배열([])로 둔다.
- queryType: 아래 중 하나
  "recommend" (추천), "schedule" (일정/기간 문의), "location" (위치/찾아가는 법),
  "course" (코스/동선 구성), "info" (기타 상세 정보)
- clarifyNeeded: 지역 정보 질문이 맞지만 너무 모호해서 되물어야 하면 true
- clarifyQuestion: clarifyNeeded가 true일 때만, 사용자에게 되물을 한국어 질문 한 문장

예시)
질문: "아이와 갈만한 곳 추천해줘"
{"inScope":true,"categories":["관광지"],"keywords":[],"queryType":"recommend","clarifyNeeded":false,"clarifyQuestion":""}

질문: "구미 라면축제 어디서 해?"
{"inScope":true,"categories":["축제"],"keywords":["라면"],"queryType":"location","clarifyNeeded":false,"clarifyQuestion":""}

질문: "10월에 하는 축제 있어?"
{"inScope":true,"categories":["축제"],"keywords":[],"queryType":"schedule","clarifyNeeded":false,"clarifyQuestion":""}

질문: "모범음식점 알려줘"
{"inScope":true,"categories":["음식점"],"keywords":[],"queryType":"recommend","clarifyNeeded":false,"clarifyQuestion":""}
`.trim();

  const raw = await callOpenAI({
    system,
    user: question,
    history: historyForLLM,
    json: true,
    model: "gpt-5-mini",
  });

  try {
    return JSON.parse(raw);
  } catch (e) {
    console.error("intent 파싱 실패:", raw);
    return {
      inScope: true,
      categories: [],
      keywords: [],
      queryType: "info",
      clarifyNeeded: false,
      clarifyQuestion: "",
    };
  }
}

// =====================================================
// 2단계: JSON 검색 (키워드 매칭 스코어링)
// =====================================================

function scoreItem(item, keywords) {
  const name = (item.name || "").toLowerCase();
  const address = (item.address || "").toLowerCase();
  let score = 0;
  const matched = [];

  keywords.forEach((kw) => {
    const k = String(kw).toLowerCase().trim();
    if (!k) return;

    if (name.includes(k)) {
      score += 3; // 이름에 들어간 키워드가 훨씬 중요 (예: "라면" → "구미라면 축제")
      matched.push(kw);
    } else if (address.includes(k)) {
      score += 1; // 주소(동네 이름 등) 매칭은 보조적으로만 반영
      matched.push(kw);
    }
  });

  return { score, matched };
}

function shuffle(arr) {
  const copy = [...arr];
  for (let i = copy.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [copy[i], copy[j]] = [copy[j], copy[i]];
  }
  return copy;
}

function searchRegion(intent) {
  const categories =
    intent.categories && intent.categories.length
      ? intent.categories.filter((c) => CATEGORY_LIST.includes(c))
      : CATEGORY_LIST;

  let pool = regionDB.value.filter((item) => categories.includes(item.category));

  const keywords = intent.keywords || [];

  if (keywords.length === 0) {
    // 키워드가 없으면(아주 포괄적인 추천 요청) 카테고리 내에서 랜덤 샘플링
    return shuffle(pool).slice(0, 3);
  }

  const scored = pool
    .map((item) => {
      const { score, matched } = scoreItem(item, keywords);
      return { item, score, matched };
    })
    .filter((x) => x.score > 0)
    .sort((a, b) => b.score - a.score);

  // 키워드 매칭 결과가 없으면, 최소한 카테고리 안에서라도 몇 개 보여준다
  const finalList = scored.length > 0 ? scored : pool.map((item) => ({ item, score: 0, matched: [] }));

  return finalList.slice(0, 3).map((x) => ({ ...x.item, _matchedKeywords: x.matched }));
}

// =====================================================
// 3단계: 자연어 답변 생성
// =====================================================

const QUERY_TYPE_GUIDE = {
  recommend:
    "각 장소는 이름에서 느껴지는 테마를 짧은 명사구로 자신있게 끝맺어줘. 예: '중화요리 전문점.', '강변을 따라 걷기 좋은 공원.', '카페형 음식점.' 처럼. '이름으로 보면', '~일 가능성이 있어요' 같은 헤징 표현은 쓰지 마.",
  schedule:
    "정확한 날짜·기간 정보가 데이터에 없다는 사실을 길게 설명하지 말고, 곧바로 '정확한 일정과 세부 내용은 아래 각 축제의 전화번호로 문의해 확인해 보시길 권해드려요' 정도로 간결하게 안내해.",
  location:
    "주소를 문장 안에 자연스럽게 녹여서 설명해. 예: '~에 위치한'. 좌표(mapx/mapy)는 사람이 읽기 불편하니 문장으로 옮기지 마.",
  course:
    "여러 장소를 묶어서 이동 흐름이 느껴지도록 순서를 제안하듯 설명해. 각 장소를 짧게 소개하며 다음 장소로 자연스럽게 이어지는 느낌으로 써.",
  info: "질문에서 궁금해한 포인트를 우선적으로 짚어주고, 나머지는 보조적으로 설명해.",
};

async function createAnswer(question, intent, searchResult, historyForLLM) {
  const guide = QUERY_TYPE_GUIDE[intent.queryType] || QUERY_TYPE_GUIDE.info;

  const system = `
너는 구미·경북권 지역을 안내하는 친절한 여행 가이드 챗봇이다.

아래로 전달되는 각 장소/행사 데이터에는 다음 필드만 있다: name(이름), address(주소), tel(전화번호).
이 데이터베이스에는 설명글, 정확한 날짜/기간, 요금, 프로그램, 편의시설, 대상 연령 같은 정보가 전혀 없다.

절대 규칙:
1. name/address/tel에 실제로 있는 값만 사실로 언급한다. 날짜, 요금, 프로그램 내용 등 데이터에 없는 사실을 지어내지 않는다.
2. 이름에서 유추 가능한 테마는 짧고 단정적인 명사구로 끝맺는다 (예: "중화요리 전문점.", "카페형 음식점.", "강변을 따라 걷기 좋은 공원."). "~이에요/예요" 문장체는 받침 유무에 따라 표기가 갈려서 문법 오류가 나기 쉬우니 장소 특징을 설명하는 부분에서는 아예 쓰지 않는다. "이름으로 보면", "~일 가능성이 있어요" 같은 헤징 표현도 쓰지 않는다. 단, "몇 회째", "몇 명 참여" 같은 구체적 수치·사실은 절대 지어내지 않는다.
3. 이름과 주소만 나열하는 리스트 형태로 답하지 않는다. 반드시 자연스러운 문장/문단으로 설명한다.
4. 장소가 여러 개면 번호(1, 2, 3)나 소제목 정도는 써도 되지만, 그 안의 설명은 완전한 문장으로 쓴다.
5. 존댓말(해요체)로, 실제 사람 가이드가 설명해주듯 따뜻하고 자연스럽게 쓴다.
6. image, lat, lng, id 같은 필드는 답변 문장에서 언급하지 않는다 (화면에 별도로 표시됨).
7. 간결하게 답한다. 데이터의 한계(없는 정보)를 설명해야 할 때도 장황한 배경 설명 없이 한 문장 이내로 짧게 언급하고 바로 본론으로 들어간다. 서론 없이 첫 문장부터 실질적인 정보를 준다.
8. 마지막에 더 필요한 정보가 있으면 물어보라는 한 줄 정도의 짧은 마무리만 덧붙인다.
9. tel 필드가 데이터에 없는 장소는 전화번호에 대해 아무 말도 하지 않는다. "전화번호 정보는 제공되지 않았어요" 같은 문장도 쓰지 않는다 — 그냥 언급 자체를 생략한다.

이번 질문 유형은 "${intent.queryType}"이다. ${guide}
`.trim();

  const user = `
사용자 질문: ${question}

검색된 JSON 데이터 (이 안의 정보만 활용할 것):
${JSON.stringify(
  searchResult.map(({ name, address, tel, category }) => {
    const item = { name, address, category };
    if (tel) item.tel = tel; // 전화번호가 없는 항목은 필드 자체를 빼서, 모델이 "없다"고 언급할 거리를 없앤다
    return item;
  }),
  null,
  2
)}
`.trim();

  return await callOpenAI({ system, user, history: historyForLLM });
}

// =====================================================
// 대화 기록을 LLM 컨텍스트로 변환 (최근 몇 턴만)
// =====================================================

function buildHistoryForLLM() {
  return messages.value
    .filter((m) => m.role === "user" || m.role === "assistant")
    .slice(-6)
    .map((m) => ({ role: m.role, content: m.content }));
}

// =====================================================
// 메시지 전송 (오케스트레이션)
// =====================================================

async function sendMessage() {
  const question = input.value.trim();
  if (!question || loading.value) return;

  const historyForLLM = buildHistoryForLLM();

  messages.value.push({ role: "user", content: question });
  input.value = "";
  loading.value = true;
  await scrollBottom();

  try {
    if (!dbReady.value || regionDB.value.length === 0) {
      messages.value.push({
        role: "assistant",
        content: "지역 정보 데이터를 아직 불러오는 중이에요. 잠시 후 다시 물어봐 주세요!",
      });
      return;
    }

    const intent = await analyzeQuestion(question, historyForLLM);

    // 이 챗봇의 데이터 범위를 벗어난 질문 (예: 커뮤니티 게시글 검색 등)
    if (!intent.inScope) {
      messages.value.push({
        role: "assistant",
        content:
          "앗, 그 부분은 제가 가진 관광지·축제·음식점 등 지역 정보 데이터로는 답해드리기 어려워요.\n혹시 관광지, 축제, 모범음식점, 숙박, 여행코스 같은 지역 정보가 궁금하시면 편하게 물어봐 주세요!",
      });
      return;
    }

    // 질문이 너무 모호해서 되물어야 하는 경우
    if (intent.clarifyNeeded && intent.clarifyQuestion) {
      messages.value.push({ role: "assistant", content: intent.clarifyQuestion });
      return;
    }

    const searchResult = searchRegion(intent);

    if (searchResult.length === 0) {
      messages.value.push({
        role: "assistant",
        content: "말씀하신 조건에 딱 맞는 정보를 아직 찾지 못했어요 😢 지역이나 키워드를 조금 다르게 말씀해주시면 다시 찾아볼게요!",
      });
      return;
    }

    const answer = await createAnswer(question, intent, searchResult, historyForLLM);
    const sources = searchResult
      .filter((item) => item.image)
      .map((item) => ({ name: item.name, image: item.image, address: item.address }));

    messages.value.push({ role: "assistant", content: answer, sources });
  } catch (error) {
    console.error(error);
    messages.value.push({
      role: "assistant",
      content: "답변을 만드는 중에 오류가 발생했어요. 잠시 후 다시 시도해 주세요.",
    });
  } finally {
    loading.value = false;
    await scrollBottom();
  }
}

async function scrollBottom() {
  await nextTick();
  if (chatBody.value) {
    chatBody.value.scrollTop = chatBody.value.scrollHeight;
  }
}

const showSuggestions = computed(() => messages.value.length <= 1 && !loading.value);
</script>

<template>
  <button class="floating-chat" @click="open = !open" :aria-label="open ? '챗봇 닫기' : '챗봇 열기'">
    💬
  </button>

  <Transition name="chat-popup">
    <div v-if="open" class="chat-overlay" @click.self="open = false">
      <div class="chat-popup" role="dialog" aria-label="구미 지역 정보 챗봇">
        <div class="chat-popup-header">
          <div class="chat-header">🧭 구미·경북권 지역 정보 챗봇</div>
          <button class="close-btn" @click="open = false" aria-label="닫기">✕</button>
        </div>

        <div class="chat-messages" ref="chatBody">
          <div
            v-for="(msg, idx) in messages"
            :key="idx"
            :class="['msg-group', msg.role === 'user' ? 'align-user' : 'align-bot']"
          >
            <div :class="['message', msg.role === 'user' ? 'user' : 'bot']">{{ msg.content }}</div>

            <div v-if="msg.sources && msg.sources.length" class="source-cards">
              <div v-for="(s, si) in msg.sources" :key="si" class="source-card">
                <img :src="s.image" :alt="s.name" loading="lazy" />
                <span class="source-card-name">{{ s.name }}</span>
              </div>
            </div>
          </div>

          <div v-if="loading" class="message typing">
            답변을 준비하고 있어요<span class="typing-dot"></span>
          </div>

          <div v-if="showSuggestions" class="actions" style="flex-wrap: wrap; margin-top: 4px;">
            <button
              v-for="(prompt, i) in suggestedPrompts"
              :key="i"
              class="secondary"
              @click="useSuggestion(prompt)"
            >
              {{ prompt }}
            </button>
          </div>
        </div>

        <div class="chat-popup-input">
          <input
            v-model="input"
            type="text"
            placeholder="예) 아이랑 갈만한 관광지 추천해줘"
            :disabled="loading"
            @keyup.enter="sendMessage"
          />
          <button @click="sendMessage" :disabled="loading || !input.trim()">전송</button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<style scoped>
/* 메시지 버블 + 참고 카드를 하나의 묶음으로 정렬하기 위한 래퍼.
   기존 style.css의 .message 정렬(align-self)은 그대로 두고,
   여기서는 버블과 카드를 같은 쪽으로 묶어주는 역할만 한다. */
.msg-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-width: 85%;
}
.msg-group.align-user {
  align-self: flex-end;
  align-items: flex-end;
}
.msg-group.align-bot {
  align-self: flex-start;
  align-items: flex-start;
}
.msg-group .message {
  max-width: 100%;
}

.source-cards {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  padding-bottom: 2px;
}

.source-card {
  flex: 0 0 auto;
  width: 96px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  background: #f8fafc;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 6px;
  text-align: center;
}

.source-card img {
  width: 100%;
  height: 64px;
  object-fit: cover;
  border-radius: 6px;
  background: #e5e7eb;
}

.source-card-name {
  font-size: 11px;
  color: #374151;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}
</style>
