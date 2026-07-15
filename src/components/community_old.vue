<script setup>
import { ref, computed, onMounted } from 'vue'

const STORAGE_KEY = 'localhub-posts'
const posts = ref([])
const form = ref({ title: '', content: '', password: '' })
const editingId = ref(null)

// 모달 상태
const isModalOpen = ref(false)
const selectedPost = ref(null)

// 댓글 폼
const commentForm = ref({ author: '', content: '', password: '' })
const replyingToCommentId = ref(null)
const replyForm = ref({})

// 수정 관련
const editingCommentId = ref(null)
const editingCommentContent = ref('')
const postPasswordInput = ref('')
const commentPasswordInputs = ref({})

function normalizeComments(comments = []) {
  const result = []

  for (const comment of Array.isArray(comments) ? comments : []) {
    const baseComment = {
      id: comment.id ?? Date.now() + Math.random(),
      author: comment.author ?? '익명',
      content: comment.content ?? '',
      password: comment.password ?? '',
      createdAt: comment.createdAt || new Date().toISOString(),
      updatedAt: comment.updatedAt || comment.createdAt || new Date().toISOString(),
      parentId: comment.parentId ?? null
    }

    result.push(baseComment)

    if (Array.isArray(comment.replies)) {
      for (const reply of comment.replies) {
        result.push({
          id: reply.id ?? Date.now() + Math.random(),
          author: reply.author ?? '익명',
          content: reply.content ?? '',
          password: reply.password ?? '',
          createdAt: reply.createdAt || new Date().toISOString(),
          updatedAt: reply.updatedAt || reply.createdAt || new Date().toISOString(),
          parentId: baseComment.id
        })
      }
    }
  }

  return result
}

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      posts.value = parsed.map(post => ({
        ...post,
        comments: normalizeComments(post.comments),
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
    commentForms.value[postId] = { author: '', content: '', password: '' }
  }
  return commentForms.value[postId]
}

function getReplyForm(commentId) {
  if (!replyForms.value[commentId]) {
    replyForms.value[commentId] = { author: '', content: '', password: '' }
  }
  return replyForms.value[commentId]
}

function getCommentTree(post) {
  if (!post) return []

  const comments = Array.isArray(post.comments) ? post.comments : []
  return comments
    .filter(comment => comment.parentId === null)
    .map(comment => ({
      ...comment,
      replies: comments.filter(reply => reply.parentId === comment.id)
    }))
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
  const author = formData.author.trim()
  const content = formData.content.trim()
  const password = formData.password.trim()

  if (!author || !content || !password) {
    alert('작성자, 내용, 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  post.comments = post.comments || []
  post.comments.unshift({
    id: Date.now() + Math.random(),
    author,
    content,
    password,
    parentId: null,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  formData.author = ''
  formData.content = ''
  formData.password = ''
}

function toggleReplyForm(commentId) {
  replyingToCommentId.value = replyingToCommentId.value === commentId ? null : commentId
}

function addReply(postId, parentCommentId) {
  const formData = getReplyForm(parentCommentId)
  const author = formData.author.trim()
  const content = formData.content.trim()
  const password = formData.password.trim()

  if (!author || !content || !password) {
    alert('작성자, 내용, 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  const parentComment = (post.comments || []).find(item => item.id === parentCommentId)
  if (!parentComment) return

  if (parentComment.parentId !== null) {
    alert('대댓글에는 다시 대댓글을 달 수 없습니다.')
    return
  }

  post.comments = post.comments || []
  post.comments.push({
    id: Date.now() + Math.random(),
    author,
    content,
    password,
    parentId: parentCommentId,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  formData.author = ''
  formData.content = ''
  formData.password = ''
  replyingToCommentId.value = null
}

function startEditComment(postId, commentId) {
  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  const comment = (post.comments || []).find(item => item.id === commentId)
  if (!comment) return

  const key = `${postId}-${commentId}`
  const enteredPassword = (commentPasswordInputs.value[key] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (comment.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    commentPasswordInputs.value[key] = ''
    return
  }

  editingCommentTarget.value = { postId, commentId }
  editingCommentDraft.value = comment.content
  commentPasswordInputs.value[key] = ''
}

function saveEditComment() {
  if (!editingCommentTarget.value) return

  const { postId, commentId } = editingCommentTarget.value
  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  const comment = (post.comments || []).find(item => item.id === commentId)
  if (!comment) return

  const content = editingCommentDraft.value.trim()
  if (!content) {
    alert('내용을 입력해주세요.')
    return
  }

  comment.content = content
  comment.updatedAt = new Date().toISOString()
  savePosts()

  editingCommentTarget.value = null
  editingCommentDraft.value = ''
}

function cancelEditComment() {
  editingCommentTarget.value = null
  editingCommentDraft.value = ''
}

function deleteComment(postId, commentId) {
  const post = posts.value.find(item => item.id === postId)
  if (!post) return

  const comment = (post.comments || []).find(item => item.id === commentId)
  if (!comment) return

  const key = `${postId}-${commentId}`
  const enteredPassword = (commentPasswordInputs.value[key] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 먼저 입력하세요.')
    return
  }

  if (comment.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    commentPasswordInputs.value[key] = ''
    return
  }

  post.comments = (post.comments || []).filter(item => item.id !== commentId)
  savePosts()

  delete commentPasswordInputs.value[key]

  if (
    editingCommentTarget.value &&
    editingCommentTarget.value.postId === postId &&
    editingCommentTarget.value.commentId === commentId
  ) {
    editingCommentTarget.value = null
    editingCommentDraft.value = ''
  }

  alert('댓글이 삭제되었습니다.')
}

function isEditingComment(postId, commentId) {
  return (
    editingCommentTarget.value &&
    editingCommentTarget.value.postId === postId &&
    editingCommentTarget.value.commentId === commentId
  )
}
</script>

<template>
  <section class="card">
    <h3>📝 익명 커뮤니티</h3>
    <p class="hint">
      브라우저 localStorage에 저장됩니다. 게시글, 댓글, 대댓글은 모두 로컬에 저장됩니다.
    </p>

    <!-- 글쓰기 폼 (항상 상단에 표시) -->
    <form @submit.prevent="addOrUpdatePost" class="post-form">
      <input v-model="form.title" placeholder="제목" />
      <textarea v-model="form.content" placeholder="내용을 입력하세요"></textarea>
      <input v-model="form.password" type="password" placeholder="비밀번호(수정/삭제용)" />

      <div class="form-actions">
        <button type="submit">{{ editingId ? '수정 완료' : '작성' }}</button>
        <button v-if="editingId" type="button" @click="resetForm">취소</button>
      </div>
    </form>

    <!-- 목록 및 상세 분할 레이아웃 -->
    <div class="container-split">
      <!-- 왼쪽: 게시글 목록 -->
      <div class="list-section">
        <h4 class="section-title">게시글 목록</h4>
        <div class="post-list">
          <div v-if="posts.length === 0" class="empty">
            게시글이 없습니다. 첫 번째 글을 작성해주세요!
          </div>

          <button
            v-for="post in posts"
            :key="post.id"
            @click="selectPost(post)"
            :class="['post-list-item', { active: selectedPost?.id === post.id }]"
          >
            <div class="post-list-content">
              <div class="post-list-title">{{ post.title }}</div>
              <div class="post-list-date">{{ formatDate(post.createdAt) }}</div>
            </div>
            <div class="post-list-arrow">›</div>
          </button>
        </div>
      </div>

      <!-- 오른쪽: 상세 내용 -->
      <div v-if="selectedPost" class="detail-section">
        <div class="detail-header">
          <h4>{{ selectedPost.title }}</h4>
          <button class="close-btn" @click="selectedPostId = null">✕</button>
        </div>

        <div class="detail-meta">
          <span>작성: {{ formatDate(selectedPost.createdAt) }}</span>
          <span v-if="selectedPost.updatedAt !== selectedPost.createdAt">
            수정: {{ formatDate(selectedPost.updatedAt) }}
          </span>
        </div>

        <div class="detail-content">
          {{ selectedPost.content }}
        </div>

        <!-- 수정/삭제 액션 -->
        <div class="detail-actions">
          <input
            :value="passwordInputs[selectedPost.id] || ''"
            @input="passwordInputs[selectedPost.id] = $event.target.value"
            type="password"
            placeholder="비밀번호"
            class="password-input"
          />
          <button type="button" @click="startEdit(selectedPost)">수정</button>
          <button type="button" @click="deletePost(selectedPost)" class="btn-danger">삭제</button>
        </div>

        <!-- 댓글 작성 폼 -->
        <div class="comment-form">
          <h5>댓글 작성</h5>
          <input
            v-model="getCommentForm(selectedPost.id).author"
            placeholder="작성자 이름"
          />
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

        <!-- 댓글 목록 -->
        <div v-if="getCommentTree(selectedPost).length" class="comment-list">
          <h5>댓글 ({{ getCommentTree(selectedPost).length }})</h5>
          <div
            v-for="comment in getCommentTree(selectedPost)"
            :key="comment.id"
            class="comment-item"
          >
            <div class="comment-card">
              <div class="comment-header">
                <strong>{{ comment.author }}</strong>
                <span>{{ formatDate(comment.createdAt) }}</span>
              </div>

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
                <button type="button" @click="deleteComment(selectedPost.id, comment.id)" class="btn-danger">삭제</button>
                <button type="button" @click="toggleReplyForm(comment.id)">대댓글</button>
              </div>

              <div v-if="replyingToCommentId === comment.id" class="reply-form">
                <input
                  v-model="getReplyForm(comment.id).author"
                  placeholder="작성자 이름"
                />
                <textarea
                  v-model="getReplyForm(comment.id).content"
                  placeholder="대댓글 내용을 입력하세요"
                ></textarea>
                <input
                  v-model="getReplyForm(comment.id).password"
                  type="password"
                  placeholder="대댓글 비밀번호"
                />
                <button type="button" @click="addReply(selectedPost.id, comment.id)">대댓글 등록</button>
              </div>

              <div v-if="isEditingComment(selectedPost.id, comment.id)" class="edit-box">
                <textarea v-model="editingCommentDraft"></textarea>
                <div class="form-actions">
                  <button type="button" @click="saveEditComment">수정 완료</button>
                  <button type="button" @click="cancelEditComment">취소</button>
                </div>
              </div>

              <div v-if="comment.replies && comment.replies.length" class="reply-list">
                <div
                  v-for="reply in comment.replies"
                  :key="reply.id"
                  class="comment-item reply-item"
                >
                  <div class="comment-card">
                    <div class="comment-header">
                      <strong>{{ reply.author }}</strong>
                      <span>{{ formatDate(reply.createdAt) }}</span>
                    </div>

                    <p class="comment-content">{{ reply.content }}</p>

                    <div class="comment-actions">
                      <input
                        :value="commentPasswordInputs[`${selectedPost.id}-${reply.id}`] || ''"
                        @input="commentPasswordInputs[`${selectedPost.id}-${reply.id}`] = $event.target.value"
                        type="password"
                        placeholder="비밀번호"
                        class="password-input"
                      />
                      <button type="button" @click="startEditComment(selectedPost.id, reply.id)">수정</button>
                      <button type="button" @click="deleteComment(selectedPost.id, reply.id)" class="btn-danger">삭제</button>
                    </div>

                    <div v-if="isEditingComment(selectedPost.id, reply.id)" class="edit-box">
                      <textarea v-model="editingCommentDraft"></textarea>
                      <div class="form-actions">
                        <button type="button" @click="saveEditComment">수정 완료</button>
                        <button type="button" @click="cancelEditComment">취소</button>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div v-else class="empty">
          아직 댓글이 없습니다. 첫 번째 댓글을 작성해주세요!
        </div>
      </div>

      <!-- 상세 내용 미선택 -->
      <div v-else class="detail-section empty-state">
        <p>📌 게시글을 선택하면 상세 내용을 볼 수 있습니다.</p>
      </div>
    </div>
  </section>
</template>

<style scoped>
.card {
  border: 1px solid #e5e7eb;
  padding: 20px;
  border-radius: 12px;
  margin-bottom: 16px;
  background: #ffffff;
}

h3 {
  margin: 0 0 8px 0;
  font-size: 1.25rem;
  color: #1f2937;
}

.hint {
  font-size: 0.85rem;
  color: #6b7280;
  margin-bottom: 16px;
}

/* 글쓰기 폼 */
.post-form,
.comment-form,
.reply-form {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 20px;
  padding: 12px;
  background: #f3f4f6;
  border-radius: 8px;
}

input,
textarea {
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-family: inherit;
  font-size: 0.95rem;
  transition: border-color 0.2s;
}

input:focus,
textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

textarea {
  min-height: 80px;
  resize: vertical;
}

button {
  padding: 8px 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 500;
  transition: all 0.2s;
  background: #3b82f6;
  color: white;
}

button:hover {
  background: #2563eb;
}

.btn-danger {
  background: #ef4444;
}

.btn-danger:hover {
  background: #dc2626;
}

.form-actions,
.comment-actions,
.detail-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
}

/* 2단 분할 레이아웃 */
.container-split {
  display: grid;
  grid-template-columns: 1fr 1.5fr;
  gap: 20px;
  margin-top: 20px;
}

@media (max-width: 768px) {
  .container-split {
    grid-template-columns: 1fr;
  }
}

.list-section,
.detail-section {
  display: flex;
  flex-direction: column;
}

.section-title {
  font-size: 1rem;
  font-weight: 600;
  color: #1f2937;
  margin: 0 0 12px 0;
  padding-bottom: 8px;
  border-bottom: 2px solid #e5e7eb;
}

/* 게시글 목록 */
.post-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 600px;
  overflow-y: auto;
  padding-right: 4px;
}

.post-list::-webkit-scrollbar {
  width: 6px;
}

.post-list::-webkit-scrollbar-track {
  background: #f3f4f6;
  border-radius: 3px;
}

.post-list::-webkit-scrollbar-thumb {
  background: #d1d5db;
  border-radius: 3px;
}

.post-list::-webkit-scrollbar-thumb:hover {
  background: #9ca3af;
}

.post-list-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  text-align: left;
  cursor: pointer;
  transition: all 0.2s;
}

.post-list-item:hover {
  background: #f9fafb;
  border-color: #3b82f6;
}

.post-list-item.active {
  background: #dbeafe;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.post-list-content {
  flex: 1;
  min-width: 0;
}

.post-list-title {
  font-weight: 600;
  color: #1f2937;
  margin-bottom: 4px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.post-list-date {
  font-size: 0.8rem;
  color: #9ca3af;
}

.post-list-arrow {
  font-size: 1.5rem;
  color: #d1d5db;
  margin-left: 8px;
  transition: color 0.2s;
}

.post-list-item.active .post-list-arrow {
  color: #3b82f6;
}

/* 상세 영역 */
.detail-section {
  padding: 16px;
  background: #f9fafb;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
}

.detail-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 12px;
  gap: 12px;
}

.detail-header h4 {
  font-size: 1.1rem;
  font-weight: 700;
  color: #1f2937;
  margin: 0;
  word-break: break-word;
}

.close-btn {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  padding: 0;
  background: #e5e7eb;
  color: #6b7280;
  font-size: 1.2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
}

.close-btn:hover {
  background: #d1d5db;
  color: #1f2937;
}

.detail-meta {
  display: flex;
  gap: 12px;
  font-size: 0.85rem;
  color: #6b7280;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #e5e7eb;
}

.detail-content {
  padding: 12px;
  background: white;
  border-radius: 6px;
  margin-bottom: 12px;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-word;
  color: #374151;
  min-height: 80px;
}

.detail-actions {
  margin-bottom: 12px;
  padding: 12px;
  background: white;
  border-radius: 6px;
}

.password-input {
  min-width: 120px;
  flex-shrink: 0;
}

/* 댓글 */
.comment-form {
  border: 1px solid #e5e7eb;
}

.comment-form h5 {
  margin: 0 0 8px 0;
  font-size: 0.95rem;
  color: #1f2937;
}

.comment-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 16px;
}

.comment-list h5 {
  margin: 0 0 12px 0;
  font-size: 0.95rem;
  color: #1f2937;
}

.comment-item {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  padding: 12px;
}

.comment-card {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.9rem;
  color: #1f2937;
}

.comment-header strong {
  font-weight: 600;
  color: #1f2937;
}

.comment-header span {
  color: #9ca3af;
  font-size: 0.8rem;
}

.comment-content {
  margin: 6px 0;
  padding: 8px;
  background: #f9fafb;
  border-radius: 4px;
  white-space: pre-wrap;
  word-break: break-word;
  color: #374151;
  line-height: 1.5;
}

.comment-actions {
  padding: 8px 0;
  border-top: 1px solid #e5e7eb;
  padding-top: 8px;
}

.reply-form {
  margin-top: 8px;
  padding: 8px;
  background: #f3f4f6;
  border-radius: 6px;
}

.reply-list {
  margin-top: 12px;
  padding-left: 12px;
  border-left: 2px solid #d1d5db;
}

.reply-item {
  background: #f9fafb;
  border: 1px solid #d1d5db;
}

.edit-box {
  margin-top: 8px;
  padding: 8px;
  background: #fef3c7;
  border-radius: 6px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* 빈 상태 */
.empty {
  text-align: center;
  padding: 20px;
  color: #9ca3af;
  font-size: 0.95rem;
}

.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 300px;
  background: linear-gradient(135deg, #f3f4f6 0%, #f9fafb 100%);
}

.empty-state p {
  font-size: 1rem;
  color: #6b7280;
}
</style>