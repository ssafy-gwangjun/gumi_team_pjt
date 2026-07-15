<script setup>
import { ref, onMounted, watch } from 'vue'
import Community from './components/community.vue'

const navItems = [
  { key: 'home', label: '홈' },
  { key: 'places', label: '관광지 안내' },
  { key: 'bookmarks', label: '북마크' }
]

const currentPage = ref('home')
const attractions = ref([])
const bookmarks = ref([])

const messages = ref([
  { type: 'bot', text: '구미 여행에 대해 무엇이 궁금하세요?' }
])
const newQuestion = ref('')
const isChatOpen = ref(false)

function changePage(page) {
  currentPage.value = page
}

function isBookmarked(id) {
  return bookmarks.value.some(b => b.id === id)
}

function toggleBookmark(item) {
  const idx = bookmarks.value.findIndex(b => b.id === item.id)
  if (idx >= 0) {
    bookmarks.value.splice(idx, 1)
  } else {
    bookmarks.value.push({ ...item })
  }
}

function openChat() { isChatOpen.value = true }
function closeChat() { isChatOpen.value = false }

function sendMessage() {
  const text = newQuestion.value.trim()
  if (!text) return
  openChat()
  messages.value.push({ type: 'user', text })
  messages.value.push({
    type: 'bot',
    text: `“${text}”에 대한 정보는 곧 챗봇에서 더 자세히 답변해드릴 수 있어요.`
  })
  newQuestion.value = ''
}

/* localStorage로 북마크 유지 + 초기 JSON 로드 시도 */
onMounted(async () => {
  const saved = localStorage.getItem('gumi_bookmarks')
  if (saved) {
    try { bookmarks.value = JSON.parse(saved) } catch {}
  }

  try {
    const res = await fetch('/data/attractions.json')
    if (res.ok) attractions.value = await res.json()
  } catch (e) {
    // 실패하면 간단한 예시 데이터 사용
    attractions.value = [
      { id: 1, title: '구미 인삼랜드', description: '인삼과 자연을 함께 즐길 수 있는 대표 관광지입니다.', category: '자연·체험' },
      { id: 2, title: '금오산', description: '산책과 전망을 즐기기 좋은 인기 명소입니다.', category: '산책·전망' }
    ]
  }
})

watch(bookmarks, (val) => {
  localStorage.setItem('gumi_bookmarks', JSON.stringify(val))
}, { deep: true })
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <h1>구미 여행 가이드</h1>
      <nav class="nav">
        <button
          v-for="item in navItems"
          :key="item.key"
          class="nav-link"
          :class="{ active: currentPage === item.key }"
          @click="changePage(item.key)"
        >
          {{ item.label }}
        </button>
      </nav>
    </header>

    <main class="main-content">
      <section class="content-panel">
        <div v-if="currentPage === 'home'" class="page-section">
          <h2>구미의 숨은 명소를 만나보세요</h2>
          <p>맛집, 관광지, 사진 명소까지 한 번에 확인할 수 있는 여행 가이드입니다.</p>
          <div class="actions">
            <button type="button" @click="changePage('places')">관광지 안내 보기</button>
            <button type="button" class="secondary" @click="changePage('bookmarks')">북마크 보기</button>
          </div>
        </div>

        <div v-else-if="currentPage === 'places'" class="page-section">
          <div class="page-header">
            <h2>관광지 안내</h2>
            <p>구미의 대표 관광지와 추천 포인트를 확인해보세요.</p>
          </div>

          <div class="card-grid">
            <article v-for="item in attractions" :key="item.id" class="info-card">
              <div class="card-row">
                <div>
                  <h3>{{ item.title }}</h3>
                  <p>{{ item.description }}</p>
                  <span class="tag">{{ item.category }}</span>
                </div>

                <button
                  class="bookmark-btn"
                  :class="{ on: isBookmarked(item.id) }"
                  @click="toggleBookmark(item)"
                  :aria-pressed="isBookmarked(item.id)"
                >
                  <span v-if="isBookmarked(item.id)">★</span>
                  <span v-else>☆</span>
                </button>
              </div>
            </article>
          </div>
        </div>

        <div v-else-if="currentPage === 'bookmarks'" class="page-section">
          <div class="page-header">
            <h2>북마크</h2>
            <p>나중에 다시 가고 싶은 장소를 모아둘 수 있습니다.</p>
          </div>

          <div class="card-grid" v-if="bookmarks.length">
            <article v-for="item in bookmarks" :key="item.id" class="info-card">
              <div class="card-row">
                <div>
                  <h3>{{ item.title }}</h3>
                  <p>{{ item.description }}</p>
                  <span class="tag">{{ item.category }}</span>
                </div>

                <button class="bookmark-btn on" @click="toggleBookmark(item)">★</button>
              </div>
            </article>
          </div>

          <div v-else class="bookmarks-empty">
            북마크된 항목이 없습니다. 관광지에서 별표를 눌러 저장해보세요.
          </div>
        </div>
      </section>
<Community />
      <aside class="chat-panel">
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

        <div class="chat-input-row">
          <input
            v-model="newQuestion"
            @keyup.enter="sendMessage"
            placeholder="질문을 입력해 주세요"
          />
          <button @click="sendMessage">전송</button>
        </div>
      </aside>
    </main>

    <div v-if="isChatOpen" class="chat-popup">
      <div class="chat-popup-header">
        <strong>구미 여행 챗봇</strong>
        <button class="close-btn" @click="closeChat">✕</button>
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
          @keyup.enter="sendMessage"
          placeholder="질문을 입력해 주세요"
        />
        <button @click="sendMessage">전송</button>
      </div>
    </div>

    <button class="floating-chat" aria-label="챗봇 열기" @click="openChat">💬</button>
  </div>
</template>