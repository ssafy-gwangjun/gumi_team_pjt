<script setup>
import { ref, onMounted, watch } from 'vue'

import Chatbot from './components/chatbot.vue'


const navItems = [
  { key: 'map', label: '관광 지도 안내' },
  { key: 'community', label: '익명 커뮤니티' },
  { key: 'bookmarks', label: '내 저장목록' }
]

const currentPage = ref('map')
const isChatOpen = ref(false)
const bookmarks = ref([])

const chatQuery = ref('')
const chatbotRef = ref(null)

function openChatWithQuestion() {
  const text = chatQuery.value.trim()
  if (!text) return
  chatbotRef.value?.openWithQuestion(text)
  chatQuery.value = ''
}

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
      <div>
        <h1>구미 여행 가이드</h1>
        <p>현재는 지도 페이지만 정상 확인용으로 연결되어 있습니다.</p>
      </div>

      <nav class="nav">
        <button
          v-for="item in navItems"
          :key="item.key"
          :class="['nav-link', { active: currentPage === item.key }]"
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

        <div v-else-if="currentPage === 'bookmarks'" class="placeholder-panel">
          <h2>내 저장목록</h2>
          <p>팀원 개발 중인 북마크 화면 자리입니다.</p>
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
            v-model="chatQuery"
            @keyup.enter="openChatWithQuestion"
            placeholder="질문을 입력하면 챗봇이 열립니다"
          />
          <button @click="openChatWithQuestion">챗봇으로 이동</button>
        </div>
      </section>
    </main>
  
   <Chatbot ref="chatbotRef" />
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: #f3f6fb;
  padding: 1rem;
}

.topbar {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
  margin-bottom: 1rem;
  padding: 1rem 1.25rem;
  background: #fff;
  border-radius: 18px;
  box-shadow: 0 16px 35px rgba(15, 23, 42, 0.08);
}

.topbar h1 {
  margin: 0;
  font-size: 1.5rem;
}

.topbar p {
  margin: 0.35rem 0 0;
  color: #6b7280;
}

.nav {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.nav-link {
  padding: 0.75rem 1rem;
  border-radius: 999px;
  border: 1px solid transparent;
  background: #eff6ff;
  color: #1d4ed8;
  cursor: pointer;
}

.nav-link.active {
  background: #1d4ed8;
  color: #fff;
  border-color: #1d4ed8;
}

.main-content {
  display: grid;
  gap: 1rem;
}

.content-panel {
  min-height: calc(100vh - 160px);
}

.placeholder-panel {
  background: #fff;
  border-radius: 20px;
  padding: 2rem;
  box-shadow: 0 20px 40px rgba(15, 23, 42, 0.08);
  text-align: center;
  color: #475569;
}

.floating-chat {
  position: fixed;
  left: 1.25rem;
  bottom: 1.25rem;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  border: none;
  background: #1d4ed8;
  color: #fff;
  font-size: 1.4rem;
  cursor: pointer;
  box-shadow: 0 18px 30px rgba(15, 23, 42, 0.18);
}

.chat-popup {
  position: fixed;
  left: 1.25rem;
  bottom: 5.5rem;
  width: min(360px, calc(100vw - 2.5rem));
  background: #fff;
  border-radius: 22px;
  box-shadow: 0 24px 60px rgba(15, 23, 42, 0.18);
}

.chat-popup-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.1rem;
  border-bottom: 1px solid #e5e7eb;
}

.chat-popup-body {
  padding: 1rem;
  color: #334155;
}
.close-btn {
  border: none;
  background: transparent;
  font-size: 1.1rem;
  cursor: pointer;
}
</style>