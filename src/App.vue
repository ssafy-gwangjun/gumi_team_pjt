<script setup>
import { ref, onMounted, watch } from 'vue'
import MapView from './components/map.vue'
import Chatbot from './components/chatbot.vue'
import Community from './components/community.vue'
import BookmarkView from './components/bookmark.vue'

const navItems = [
  { key: 'map', label: '관광 지도 안내' },
  { key: 'community', label: '익명 커뮤니티' },
  { key: 'bookmarks', label: '내 즐겨찾기' }
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
  const idx = bookmarks.value.findIndex(b => b.contentid === item.contentid)
  if (idx >= 0) bookmarks.value.splice(idx, 1)
  else bookmarks.value.push({ ...item })
}

function toggleLike(item) {
  const idx = likes.value.findIndex(l => l.contentid === item.contentid)
  if (idx >= 0) likes.value.splice(idx, 1)
  else likes.value.push({ ...item })
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
    <header class="topbar">
      <div>
        <h1>구미 여행 가이드</h1>
        <p>지도, 커뮤니티, 즐겨찾기를 확인하세요.</p>
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

    <div class="chat-wrapper">
      <Chatbot :attractions="attractions" ref="chatbotRef" />
    </div>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: #f3f6fb;
  padding: 1rem;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: stretch;
}

.topbar {
  width: 100%;
  max-width: 1600px;
  margin: 0 auto;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  background: #fff;
  border-radius: 14px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.06);
}

.main-content {
  width: 100%;
  max-width: 1600px;
  margin: 1rem auto 0;
}

.content-panel {
  width: 100%;
  min-height: calc(100vh - 160px);
}

.nav {
  display: flex;
  gap: 0.6rem;
}

.nav-link {
  padding: 0.6rem 0.9rem;
  border-radius: 999px;
  background: #eef2ff;
  color: #1d4ed8;
  border: 1px solid transparent;
  cursor: pointer;
}

.nav-link.active {
  background: #1d4ed8;
  color: #fff;
}

.chat-wrapper {
  position: fixed;
  right: 1.25rem;
  bottom: 1.25rem;
  z-index: 60;
}
</style>