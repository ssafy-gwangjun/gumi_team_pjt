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
const replyingToCommentId = ref(null)
const editingCommentTarget = ref(null)
const editingCommentDraft = ref('')

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

      <div v-if="getCommentTree(selectedPost).length" class="comment-list">
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
              <button type="button" @click="deleteComment(selectedPost.id, comment.id)">삭제</button>
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
                    <button type="button" @click="deleteComment(selectedPost.id, reply.id)">삭제</button>
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
.comment-item {
  background: #f9fafb;
  padding: 10px;
  border-radius: 8px;
}

.reply-item {
  margin-left: 20px;
  background: #f5f7fa;
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
.comment-content {
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

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
  font-size: 0.95rem;
}

.edit-box {
  margin-top: 8px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
</style>