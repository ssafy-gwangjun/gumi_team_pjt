<script setup>
import { ref } from 'vue'
import MapView from './components/map.vue'

const navItems = [
  { key: 'map', label: '관광 지도 안내' },
  { key: 'community', label: '익명 커뮤니티' },
  { key: 'bookmarks', label: '내 저장목록' }
]

const currentPage = ref('map')
const isChatOpen = ref(false)
const bookmarks = ref([])

function changePage(page) {
  currentPage.value = page
}

function toggleChat() {
  isChatOpen.value = !isChatOpen.value
}
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
        <MapView v-if="currentPage === 'map'" />

        <div v-else-if="currentPage === 'community'" class="placeholder-panel">
          <h2>익명 커뮤니티</h2>
          <p>팀원 개발 중인 커뮤니티 화면 자리입니다.</p>
        </div>

        <div v-else-if="currentPage === 'bookmarks'" class="placeholder-panel">
          <h2>내 저장목록</h2>
          <p>팀원 개발 중인 북마크 화면 자리입니다.</p>
        </div>
      </section>
    </main>

    <button class="floating-chat" @click="toggleChat">💬 챗봇</button>

    <div v-if="isChatOpen" class="chat-popup">
      <div class="chat-popup-header">
        <strong>챗봇(임시)</strong>
        <button class="close-btn" @click="toggleChat">✕</button>
      </div>
      <div class="chat-popup-body">
        <p>챗봇은 팀원 개발 완료 후 연결됩니다.</p>
      </div>
    </div>
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