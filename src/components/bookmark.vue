<template>
  <section class="bookmark-page">
    <div class="bookmark-header">
      <h2>내 즐겨찾기</h2>
      <p>북마크한 장소와 좋아요한 장소를 한눈에 확인하세요.</p>
    </div>

    <div class="bookmark-section">
      <h3>즐겨찾기</h3>
      <div v-if="bookmarks.length" class="bookmark-list">
        <article v-for="item in bookmarks" :key="item.contentid" class="bookmark-card">
          <img
            :src="item.firstimage2 || item.firstimage || placeholderImage"
            :alt="item.title"
          />
          <div class="bookmark-info">
            <h4>{{ item.title }}</h4>
            <p>{{ item.addr1 }} {{ item.addr2 }}</p>
          </div>
          <div class="bookmark-actions">
            <button class="button remove" @click="$emit('toggle-bookmark', item)">
              제거
            </button>
            <button class="button like" @click="$emit('toggle-like', item)">
              좋아요
            </button>
          </div>
        </article>
      </div>

      <div v-else class="empty-state">
        저장된 즐겨찾기가 없습니다.
      </div>
    </div>

    <div class="bookmark-section">
      <h3>좋아요한 장소</h3>
      <div v-if="likes.length" class="bookmark-list">
        <article v-for="item in likes" :key="item.contentid" class="bookmark-card">
          <img
            :src="item.firstimage2 || item.firstimage || placeholderImage"
            :alt="item.title"
          />
          <div class="bookmark-info">
            <h4>{{ item.title }}</h4>
            <p>{{ item.addr1 }} {{ item.addr2 }}</p>
          </div>
          <div class="bookmark-actions">
            <button class="button like" @click="$emit('toggle-like', item)">
              좋아요 취소
            </button>
          </div>
        </article>
      </div>

      <div v-else class="empty-state">
        좋아요한 장소가 없습니다.
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

.bookmark-header h2 {
  margin: 0;
  font-size: 1.4rem;
}

.bookmark-header p {
  margin: 0.5rem 0 1rem;
  color: #555;
}

.bookmark-section {
  margin-top: 1.5rem;
}

.bookmark-list {
  display: grid;
  gap: 1rem;
}

.bookmark-card {
  display: grid;
  grid-template-columns: 120px 1fr auto;
  gap: 1rem;
  padding: 1rem;
  border-radius: 16px;
  background: #f8fafc;
  align-items: center;
}

.bookmark-card img {
  width: 120px;
  height: 90px;
  object-fit: cover;
  border-radius: 14px;
}

.bookmark-info h4 {
  margin: 0 0 0.4rem;
  font-size: 1rem;
}

.bookmark-info p {
  margin: 0;
  color: #555;
  font-size: 0.95rem;
}

.bookmark-actions {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.button {
  border: none;
  border-radius: 12px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  font-size: 0.95rem;
}

.button.remove {
  background: #ca1515;
  color: white;
}

.button.like {
  background: #2563eb;
  color: white;
}

.empty-state {
  padding: 1rem;
  color: #6b7280;
}
</style>