<!-- CommunityView.vue -->
<script setup>
import { ref, onMounted } from 'vue'

const posts = ref([])
const form = ref({ title: '', content: '' })

onMounted(() => {
  const saved = localStorage.getItem('localhub-posts')
  if (saved) {
    posts.value = JSON.parse(saved)
  }
})

function addPost() {
  if (!form.value.title.trim() || !form.value.content.trim()) return

  posts.value.unshift({
    id: Date.now(),
    title: form.value.title,
    content: form.value.content
  })

  localStorage.setItem('localhub-posts', JSON.stringify(posts.value))
  form.value.title = ''
  form.value.content = ''
}

function deletePost(id) {
  posts.value = posts.value.filter(post => post.id !== id)
  localStorage.setItem('localhub-posts', JSON.stringify(posts.value))
}
</script>

<template>
  <section class="card">
    <h3>📝 익명 커뮤니티</h3>

    <form @submit.prevent="addPost" class="post-form">
      <input v-model="form.title" placeholder="제목" />
      <textarea v-model="form.content" placeholder="내용을 입력하세요"></textarea>
      <button type="submit">작성</button>
    </form>

    <div v-for="post in posts" :key="post.id" class="post-item">
      <h4>{{ post.title }}</h4>
      <p>{{ post.content }}</p>
      <button @click="deletePost(post.id)">삭제</button>
    </div>
  </section>
</template>

<style scoped>
.card {
  border: 1px solid #ddd;
  padding: 16px;
  border-radius: 12px;
  margin-bottom: 16px;
}
.post-form {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 12px;
}
input, textarea {
  padding: 8px;
}
.post-item {
  background: #f9fafb;
  padding: 10px;
  border-radius: 8px;
  margin-bottom: 8px;
}
</style>