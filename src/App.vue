<script setup>
import { ref, onMounted, watch } from 'vue'
import MapView from './components/map.vue'
import Chatbot from './components/chatbot.vue'
import Community from './components/community.vue'

const navItems = [
  { key: 'map', label: '관광 지도 안내' },
  { key: 'community', label: '익명 커뮤니티' },
  { key: 'bookmarks', label: '내 즐겨찾기' }
]

const currentPage = ref('map')
const attractions = ref([])
const bookmarks = ref([])

function changePage(page) {
  currentPage.value = page
}

function isBookmarked(id) {
  return bookmarks.value.some(b => b.id === id)
}

function toggleBookmark(item) {
  const idx = bookmarks.value.findIndex(b => b.id === item.id)
  if (idx >= 0) bookmarks.value.splice(idx, 1)
  else bookmarks.value.push({ ...item })
}

/* localStorage로 북마크 유지 + 실제 관광 데이터 로드 */
onMounted(async () => {
  const saved = localStorage.getItem('gumi_bookmarks')
  if (saved) {
    try { bookmarks.value = JSON.parse(saved) } catch {}
  }

  try {
    const dataFiles = [
      '구미_경북권_관광지.json',
      '구미_경북권_문화시설.json',
      '구미_경북권_축제공연행사.json',
      '구미_경북권_레포츠.json',
      '구미_경북권_숙박.json',
      '구미_경북권_쇼핑.json',
      '구미_경북권_음식점.json',
      '구미_경북권_여행코스.json'
    ]

    const responses = await Promise.all(
      dataFiles.map(async (fileName) => {
        const res = await fetch(`/data/${fileName}`)
        if (!res.ok) return { items: [] }
        return res.json()
      })
    )

    attractions.value = responses.flatMap((data) => {
      const items = Array.isArray(data?.items) ? data.items : []
      return items.map((item) => ({
        ...item,
        title: item.title || item.name || '이름 없음',
        description: item.addr1 || item.addr2 || item.overview || item.title || '설명 없음',
        category: item.contenttypeid ? '데이터 기반 장소' : (item.category || '기타')
      }))
    })
  } catch (e) {
    console.error('관광 데이터 로드 실패:', e)
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
        <MapView v-if="currentPage === 'map'" />

        <Community v-else-if="currentPage === 'community'" />

        <div v-else-if="currentPage === 'bookmarks'" class="page-section">
          <div class="page-header">
            <h2>내 즐겨찾기</h2>
            <p>나중에 다시 보고 싶은 장소를 저장해두세요.</p>
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

          <div v-else class="empty-state">
            저장된 즐겨찾기가 없습니다.
          </div>
        </div>
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
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  background: #fff;
  border-radius: 14px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.06);
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

.main-content {
  margin-top: 1rem;
}

.content-panel {
  min-height: calc(100vh - 160px);
}

/* Chatbot 위치: 우하단 고정 */
.chat-wrapper {
  position: fixed;
  right: 1.25rem;
  bottom: 1.25rem;
  z-index: 60;
}
</style>