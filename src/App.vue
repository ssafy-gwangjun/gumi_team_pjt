<script setup>
import { ref, onMounted, watch } from 'vue'
import MapView from './components/map.vue'
import Chatbot from './components/chatbot.vue'
import Community from './components/community.vue'
import BookmarkView from './components/bookmark.vue'

const navItems = [
  { key: 'map', label: '관광 지도 안내' },
  { key: 'community', label: '익명 커뮤니티' },
  { key: 'bookmarks', label: '내 저장 목록' }
]

const currentPage = ref('map')
const bookmarks = ref([])
const likes = ref([])
const attractions = ref([])
const chatbotRef = ref(null)

function changePage(page) {
  currentPage.value = page
}

function toggleBookmark(item) {
  const idx = bookmarks.value.findIndex(b => String(b.contentid) === String(item.contentid))
  if (idx >= 0) {
    bookmarks.value.splice(idx, 1)
  } else {
    bookmarks.value.push({ ...item })
  }
}

function toggleLike(item) {
  const idx = likes.value.findIndex(l => String(l.contentid) === String(item.contentid))
  if (idx >= 0) {
    likes.value.splice(idx, 1)
  } else {
    likes.value.push({ ...item })
  }
}

onMounted(() => {
  const savedBookmarks = localStorage.getItem('gumi_bookmarks')
  if (savedBookmarks) {
    try { bookmarks.value = JSON.parse(savedBookmarks) } catch {}
  }

  const savedLikes = localStorage.getItem('gumi_likes')
  if (savedLikes) {
    try { likes.value = JSON.parse(savedLikes) } catch {}
  }
})

watch(bookmarks, (val) => {
  localStorage.setItem('gumi_bookmarks', JSON.stringify(val))
}, { deep: true })

watch(likes, (val) => {
  localStorage.setItem('gumi_likes', JSON.stringify(val))
}, { deep: true })
</script>

<template>
  <div class="app-shell">
    <header class="app-header">
      <div class="brand-panel">
        <div class="brand-mark">G</div>
        <div>
          <p class="eyebrow">LocalHub Gumi</p>
          <h1>구미 관광 가이드</h1>
          <p class="brand-desc">
            금오산부터 구미 문화 시설, 맛집, 숙박까지 한 번에 확인하는 로컬 여행 허브입니다.
          </p>
        </div>
      </div>

      <nav class="page-nav">
        <button
          v-for="item in navItems"
          :key="item.key"
          @click="changePage(item.key)"
          :class="['tab-btn', { active: currentPage === item.key }]"
        >
          {{ item.label }}
        </button>
      </nav>
    </header>

    <main class="app-main">
      <section class="page-panel">
        <MapView
          v-if="currentPage === 'map'"
          @toggle-bookmark="toggleBookmark"
          @toggle-like="toggleLike"
          :bookmarked-ids="bookmarks.map(b => b.contentid)"
          :liked-ids="likes.map(l => l.contentid)"
        />

        <Community v-else-if="currentPage === 'community'" />

        <BookmarkView
          v-else-if="currentPage === 'bookmarks'"
          :bookmarks="bookmarks"
          :likes="likes"
          @toggle-bookmark="toggleBookmark"
          @toggle-like="toggleLike"
        />
      </section>
    </main>

    <Chatbot :attractions="attractions" ref="chatbotRef" />
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: #eef2ff;
  padding: 1rem;
  box-sizing: border-box;
}

.app-header {
  max-width: 1600px;
  margin: 0 auto 1rem;
  padding: 1.15rem 1.25rem;
  border-radius: 24px;
  background: linear-gradient(135deg, #0f766e, #22c55e);
  color: white;
  display: flex;
  flex-direction: column;
  gap: 1.2rem;
  box-shadow: 0 25px 70px rgba(15, 23, 42, 0.18);
}

.brand-panel {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.brand-mark {
  width: 58px;
  height: 58px;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.2);
  display: grid;
  place-items: center;
  font-size: 1.4rem;
  font-weight: 700;
  color: white;
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.18);
}

.eyebrow {
  margin: 0 0 0.35rem;
  text-transform: uppercase;
  letter-spacing: 0.18em;
  font-size: 0.78rem;
  opacity: 0.9;
}

.brand-desc {
  margin: 0.35rem 0 0;
  font-size: 0.99rem;
  max-width: 740px;
  line-height: 1.65;
  opacity: 0.92;
}

.page-nav {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.tab-btn {
  min-width: 140px;
  border: none;
  border-radius: 999px;
  padding: 0.9rem 1.2rem;
  cursor: pointer;
  background: rgba(255, 255, 255, 0.18);
  color: white;
  transition: all 0.18s ease;
  font-weight: 600;
}

.tab-btn.active {
  background: white;
  color: #0f766e;
  box-shadow: 0 16px 35px rgba(15, 23, 42, 0.14);
}

.app-main {
  max-width: 1600px;
  margin: 0 auto;
}

.page-panel {
  min-height: calc(100vh - 160px);
}

@media (max-width: 900px) {
  .app-header {
    padding: 1rem;
  }
  .page-nav {
    justify-content: center;
  }
}
</style>