<script setup>
import { ref, computed, onMounted } from 'vue'

const STORAGE_KEY = 'localhub-posts'
const posts = ref([])
const form = ref({ title: '', content: '', password: '' })
const editingId = ref(null)
const selectedPostId = ref(null)

const passwordInputs = ref({})
const commentForms = ref({})
const replyForms = ref({})
const commentPasswordInputs = ref({})
const replyPasswordInputs = ref({})
const editingTarget = ref(null)
const editDraft = ref('')

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      posts.value = parsed.map(post => ({
        ...post,
        comments: (post.comments || []).map(comment => ({
          ...comment,
          replies: (comment.replies || []).map(reply => ({ ...reply }))
        })),
        createdAt: post.createdAt || new Date().toISOString(),
        updatedAt: post.updatedAt || post.createdAt || new Date().toISOString()
      }))
    } catch (e) {
      console.error('게시글 불러오기 실패:', e)
      posts.value = []
    }
  }
})

function savePosts() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(posts.value))
}

function resetForm() {
  form.value = { title: '', content: '', password: '' }
  editingId.value = null
}

function formatDate(value) {
  if (!value) return ''
  const date = new Date(value)
  return `${date.getFullYear()}.${String(date.getMonth() + 1).padStart(2, '0')}.${String(date.getDate()).padStart(2, '0')} ${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}`
}

const selectedPost = computed(() => {
  return posts.value.find(post => post.id === selectedPostId.value) || null
})

function selectPost(post) {
  selectedPostId.value = post.id
}

function getCommentForm(postId) {
  if (!commentForms.value[postId]) {
    commentForms.value[postId] = { content: '', password: '' }
  }
  return commentForms.value[postId]
}

function getReplyForm(postId, commentId) {
  const key = `${postId}-${commentId}`
  if (!replyForms.value[key]) {
    replyForms.value[key] = { content: '', password: '' }
  }
  return replyForms.value[key]
}

function addOrUpdatePost() {
  const title = form.value.title.trim()
  const content = form.value.content.trim()
  const password = form.value.password.trim()

  if (!title || !content || !password) {
    alert('제목, 내용, 비밀번호를 모두 입력해주세요.')
    return
  }

  if (editingId.value) {
    const target = posts.value.find(post => post.id === editingId.value)
    if (!target) return

    if (target.password !== password) {
      alert('비밀번호가 일치하지 않습니다.')
      return
    }

    target.title = title
    target.content = content
    target.updatedAt = new Date().toISOString()

    savePosts()
    resetForm()
    alert('게시글이 수정되었습니다.')
    return
  }

  posts.value.unshift({
    id: Date.now(),
    title,
    content,
    password,
    comments: [],
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  selectedPostId.value = posts.value[0].id
  resetForm()
  alert('게시글이 작성되었습니다.')
}

function startEdit(post) {
  const enteredPassword = (passwordInputs.value[post.id] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (post.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    passwordInputs.value[post.id] = ''
    return
  }

  editingId.value = post.id
  form.value.title = post.title
  form.value.content = post.content
  form.value.password = enteredPassword
  selectedPostId.value = post.id
}

function deletePost(post) {
  const enteredPassword = (passwordInputs.value[post.id] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (post.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    passwordInputs.value[post.id] = ''
    return
  }

  posts.value = posts.value.filter(item => item.id !== post.id)
  savePosts()

  if (selectedPostId.value === post.id) {
    selectedPostId.value = null
  }

  if (editingId.value === post.id) {
    resetForm()
  }

  delete passwordInputs.value[post.id]
  alert('게시글이 삭제되었습니다.')
}

function addComment(postId) {
  const formData = getCommentForm(postId)
  const content = formData.content.trim()
  const password = formData.password.trim()

  if (!content || !password) {
    alert('댓글 내용과 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  post.comments = post.comments || []
  post.comments.unshift({
    id: Date.now() + Math.random(),
    content,
    password,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
    replies: []
  })

  savePosts()
  formData.content = ''
  formData.password = ''
}

function addReply(postId, commentId) {
  const formData = getReplyForm(postId, commentId)
  const content = formData.content.trim()
  const password = formData.password.trim()

  if (!content || !password) {
    alert('대댓글 내용과 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  const comment = (post.comments || []).find(item => item.id === commentId)
  if (!comment) return

  comment.replies = comment.replies || []
  comment.replies.push({
    id: Date.now() + Math.random(),
    content,
    password,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  formData.content = ''
  formData.password = ''
}

function findCommentTarget(postId, commentId, replyId = null) {
  const post = posts.value.find(item => item.id === postId)
  if (!post) return null

  const comment = (post.comments || []).find(item => item.id === commentId)
  if (!comment) return null

  if (replyId) {
    const reply = (comment.replies || []).find(item => item.id === replyId)
    if (!reply) return null
    return { post, comment, reply, target: reply }
  }

  return { post, comment, target: comment }
}

function startEditComment(postId, commentId, replyId = null) {
  const targetInfo = findCommentTarget(postId, commentId, replyId)
  if (!targetInfo) return

  const key = replyId
    ? `${postId}-${commentId}-${replyId}`
    : `${postId}-${commentId}`

  const enteredPassword = (replyId
    ? replyPasswordInputs.value[key]
    : commentPasswordInputs.value[key] || ''
  ).trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (targetInfo.target.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    if (replyId) {
      replyPasswordInputs.value[key] = ''
    } else {
      commentPasswordInputs.value[key] = ''
    }
    return
  }

  editingTarget.value = {
    postId,
    commentId,
    replyId,
    type: replyId ? 'reply' : 'comment'
  }
  editDraft.value = targetInfo.target.content
}

function saveEditedComment() {
  if (!editingTarget.value) return

  const { postId, commentId, replyId } = editingTarget.value
  const targetInfo = findCommentTarget(postId, commentId, replyId)

  if (!targetInfo) return

  const content = editDraft.value.trim()
  if (!content) {
    alert('내용을 입력해주세요.')
    return
  }

  targetInfo.target.content = content
  targetInfo.target.updatedAt = new Date().toISOString()
  savePosts()

  editingTarget.value = null
  editDraft.value = ''
}

function cancelEditComment() {
  editingTarget.value = null
  editDraft.value = ''
}

function deleteComment(postId, commentId, replyId = null) {
  const targetInfo = findCommentTarget(postId, commentId, replyId)
  if (!targetInfo) return

  const key = replyId
    ? `${postId}-${commentId}-${replyId}`
    : `${postId}-${commentId}`

  const enteredPassword = (replyId
    ? replyPasswordInputs.value[key]
    : commentPasswordInputs.value[key] || ''
  ).trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (targetInfo.target.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    if (replyId) {
      replyPasswordInputs.value[key] = ''
    } else {
      commentPasswordInputs.value[key] = ''
    }
    return
  }

  if (replyId) {
    targetInfo.comment.replies = (targetInfo.comment.replies || []).filter(item => item.id !== replyId)
  } else {
    targetInfo.post.comments = (targetInfo.post.comments || []).filter(item => item.id !== commentId)
  }

  savePosts()

  if (replyId) {
    delete replyPasswordInputs.value[key]
  } else {
    delete commentPasswordInputs.value[key]
  }

  if (
    editingTarget.value &&
    editingTarget.value.postId === postId &&
    editingTarget.value.commentId === commentId &&
    editingTarget.value.replyId === replyId
  ) {
    editingTarget.value = null
    editDraft.value = ''
  }

  alert('삭제되었습니다.')
}

function isEditingTarget(postId, commentId, replyId = null) {
  return (
    editingTarget.value &&
    editingTarget.value.postId === postId &&
    editingTarget.value.commentId === commentId &&
    editingTarget.value.replyId === replyId
  )
}
</script>

<template>
  <section class="card">
    <h3>📝 익명 커뮤니티</h3>
    <p class="hint">
      브라우저 localStorage에 저장됩니다. 수정/삭제는 비밀번호 확인 후 가능합니다.
    </p>

    <form @submit.prevent="addOrUpdatePost" class="post-form">
      <input v-model="form.title" placeholder="제목" />
      <textarea v-model="form.content" placeholder="내용을 입력하세요"></textarea>
      <input v-model="form.password" type="password" placeholder="비밀번호(수정/삭제용)" />

      <div class="form-actions">
        <button type="submit">{{ editingId ? '수정 완료' : '작성' }}</button>
        <button v-if="editingId" type="button" @click="resetForm">취소</button>
      </div>
    </form>

    <div class="post-list">
      <div v-for="post in posts" :key="post.id" class="post-item">
        <button class="post-title-btn" @click="selectPost(post)">
          <strong>{{ post.title }}</strong>
        </button>

        <p class="post-content">{{ post.content }}</p>

        <div class="post-meta">
          <span>{{ formatDate(post.createdAt) }}</span>
          <div class="post-actions">
            <input
              :value="passwordInputs[post.id] || ''"
              @input="passwordInputs[post.id] = $event.target.value"
              type="password"
              placeholder="비밀번호"
              class="password-input"
            />
            <button type="button" @click="startEdit(post)">수정</button>
            <button type="button" @click="deletePost(post)">삭제</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="selectedPost" class="detail-card">
      <h4>{{ selectedPost.title }}</h4>
      <p>{{ selectedPost.content }}</p>

      <div class="detail-meta">
        <span>작성: {{ formatDate(selectedPost.createdAt) }}</span>
        <span v-if="selectedPost.updatedAt !== selectedPost.createdAt">
          수정: {{ formatDate(selectedPost.updatedAt) }}
        </span>
      </div>

      <div class="comment-form">
        <h5>댓글 작성</h5>
        <textarea
          v-model="getCommentForm(selectedPost.id).content"
          placeholder="댓글 내용을 입력하세요"
        ></textarea>
        <input
          v-model="getCommentForm(selectedPost.id).password"
          type="password"
          placeholder="댓글 비밀번호"
        />
        <button type="button" @click="addComment(selectedPost.id)">댓글 작성</button>
      </div>

      <div v-if="(selectedPost.comments || []).length" class="comment-list">
        <div v-for="comment in selectedPost.comments || []" :key="comment.id" class="comment-item">
          <p class="comment-content">{{ comment.content }}</p>

          <div class="comment-actions">
            <input
              :value="commentPasswordInputs[`${selectedPost.id}-${comment.id}`] || ''"
              @input="commentPasswordInputs[`${selectedPost.id}-${comment.id}`] = $event.target.value"
              type="password"
              placeholder="비밀번호"
              class="password-input"
            />
            <button type="button" @click="startEditComment(selectedPost.id, comment.id)">수정</button>
            <button type="button" @click="deleteComment(selectedPost.id, comment.id)">삭제</button>
          </div>

          <div v-if="isEditingTarget(selectedPost.id, comment.id, null)" class="edit-box">
            <textarea v-model="editDraft"></textarea>
            <div class="form-actions">
              <button type="button" @click="saveEditedComment">수정 완료</button>
              <button type="button" @click="cancelEditComment">취소</button>
            </div>
          </div>

          <div class="reply-list">
            <div v-for="reply in comment.replies || []" :key="reply.id" class="reply-item">
              <p class="reply-content">{{ reply.content }}</p>

              <div class="comment-actions">
                <input
                  :value="replyPasswordInputs[`${selectedPost.id}-${comment.id}-${reply.id}`] || ''"
                  @input="replyPasswordInputs[`${selectedPost.id}-${comment.id}-${reply.id}`] = $event.target.value"
                  type="password"
                  placeholder="비밀번호"
                  class="password-input"
                />
                <button type="button" @click="startEditComment(selectedPost.id, comment.id, reply.id)">수정</button>
                <button type="button" @click="deleteComment(selectedPost.id, comment.id, reply.id)">삭제</button>
              </div>

              <div v-if="isEditingTarget(selectedPost.id, comment.id, reply.id)" class="edit-box">
                <textarea v-model="editDraft"></textarea>
                <div class="form-actions">
                  <button type="button" @click="saveEditedComment">수정 완료</button>
                  <button type="button" @click="cancelEditComment">취소</button>
                </div>
              </div>
            </div>
          </div>

          <div class="reply-form">
            <textarea
              v-model="getReplyForm(selectedPost.id, comment.id).content"
              placeholder="대댓글 내용을 입력하세요"
            ></textarea>
            <input
              v-model="getReplyForm(selectedPost.id, comment.id).password"
              type="password"
              placeholder="대댓글 비밀번호"
            />
            <button type="button" @click="addReply(selectedPost.id, comment.id)">대댓글 작성</button>
          </div>
        </div>
      </div>

      <div v-else class="empty">
        아직 댓글이 없습니다.
      </div>
    </div>

    <div v-else class="empty">
      게시글을 선택하면 상세 내용을 볼 수 있습니다.
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

.hint {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 12px;
}

.post-form,
.comment-form,
.reply-form {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 16px;
}

input,
textarea {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 6px;
}

.form-actions,
.post-actions,
.comment-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
}

.post-list,
.comment-list,
.reply-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.post-item,
.comment-item,
.reply-item {
  background: #f9fafb;
  padding: 10px;
  border-radius: 8px;
}

.post-title-btn {
  background: none;
  border: none;
  padding: 0;
  text-align: left;
  cursor: pointer;
  font-size: 1rem;
}

.post-content,
.comment-content,
.reply-content {
  margin: 6px 0;
  white-space: pre-wrap;
}

.post-meta,
.detail-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
  color: #666;
  gap: 8px;
  flex-wrap: wrap;
}

.detail-card {
  margin-top: 16px;
  padding: 12px;
  background: #eef6ff;
  border-radius: 8px;
}

.empty {
  margin-top: 16px;
  color: #888;
}

.password-input {
  min-width: 120px;
}
</style>