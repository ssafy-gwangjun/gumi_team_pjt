<template>
  <section class="bookmark-page">
    <div class="bookmark-header">
      <h2>⭐내 즐겨찾기</h2>
      <p>북마크한 장소와 좋아요한 장소를 한눈에 확인하세요.</p>
    </div>

    <div class="bookmark-layout">
      <div class="bookmark-panel">
        <h3>즐겨찾기</h3>

        <div v-if="bookmarks.length" class="bookmark-list">
          <article
            v-for="item in bookmarks"
            :key="item.contentid"
            class="bookmark-card"
          >
            <img
              :src="item.firstimage2 || item.firstimage || placeholderImage"
              :alt="item.title"
            />
            <div class="bookmark-info">
              <h4>{{ item.title }}</h4>
              <p>{{ item.addr1 }} {{ item.addr2 }}</p>
            </div>
            <button
              class="button remove"
              @click="$emit('toggle-bookmark', item)"
            >
              삭제
            </button>
          </article>
        </div>

        <div v-else class="empty-state">
          저장된 즐겨찾기가 없습니다.
        </div>
      </div>

      <div class="bookmark-panel">
        <h3>좋아요한 장소</h3>

        <div v-if="likes.length" class="bookmark-list">
          <article
            v-for="item in likes"
            :key="item.contentid"
            class="bookmark-card"
          >
            <img
              :src="item.firstimage2 || item.firstimage || placeholderImage"
              :alt="item.title"
            />
            <div class="bookmark-info">
              <h4>{{ item.title }}</h4>
              <p>{{ item.addr1 }} {{ item.addr2 }}</p>
            </div>
            <button class="button remove" @click="$emit('toggle-like', item)">
              삭제
            </button>
          </article>
        </div>

        <div v-else class="empty-state">
          좋아요한 장소가 없습니다.
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
const props = defineProps({
  bookmarks: {
    type: Array,
    default: () => [],
  },
  likes: {
    type: Array,
    default: () => [],
  },
})

const placeholderImage = 'https://via.placeholder.com/240x160?text=No+Image'
</script>

<style scoped>
.bookmark-page {
  background: #fff;
  border-radius: 20px;
  padding: 1.5rem;
  box-shadow: 0 18px 40px rgba(0, 0, 0, 0.08);
}

.bookmark-header {
  margin-bottom: 1.25rem;
  padding: 1.25rem 1.5rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
}

.bookmark-header h2 {
  margin: 0;
  font-size: 1.4rem;
  color: #111827;
}

.bookmark-header p {
  margin: 0.5rem 0 0;
  color: #555;
}

.bookmark-layout {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
}

.bookmark-panel {
  background: #f8fafc;
  border-radius: 20px;
  padding: 1rem;
  min-height: 260px;
}

.bookmark-panel h3 {
  margin: 0 0 1rem;
  font-size: 1.1rem;
}

.bookmark-list {
  display: grid;
  gap: 1rem;
}

.bookmark-card {
  display: grid;
  grid-template-columns: 100px minmax(0, 1fr) auto;
  gap: 1rem;
  padding: 1rem;
  border-radius: 16px;
  background: #fff;
  align-items: center;
  border: 1px solid #e2e8f0;
}

.bookmark-card img {
  width: 100px;
  height: 80px;
  object-fit: cover;
  border-radius: 14px;
}

.bookmark-info {
  min-width: 0;
}

.bookmark-info h4 {
  margin: 0 0 0.35rem;
  font-size: 1rem;
}

.bookmark-info p {
  margin: 0;
  color: #555;
  font-size: 0.95rem;
  line-height: 1.4;
}

.button {
  border: none;
  border-radius: 12px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  font-size: 0.95rem;
  background: #ef4444;
  color: #fff;
}

.empty-state {
  padding: 1rem;
  color: #6b7280;
}

@media (max-width: 900px) {
  .bookmark-layout {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 640px) {
  .bookmark-card {
    grid-template-columns: 1fr;
    align-items: start;
  }

  .bookmark-card img {
    width: 100%;
    height: 140px;
  }

  .button {
    width: 100%;
  }
}
</style>