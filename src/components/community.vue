<script setup>
import { ref, onMounted, computed } from 'vue'

const STORAGE_KEY = 'localhub-posts'
const posts = ref([])
const form = ref({ title: '', content: '', password: '' })
const editingId = ref(null)

// 페이지네이션
const searchQuery = ref('')
const currentPage = ref(1)
const itemsPerPage = ref(10)

// 모달 상태
const isModalOpen = ref(false)
const isWriteModalOpen = ref(false)
const selectedPost = ref(null)

// 댓글/대댓글 폼
const commentForm = ref({ author: '', content: '', password: '' })
const replyingToCommentId = ref(null)
const replyForms = ref({})

// 수정 관련
const editingCommentId = ref(null)
const editingCommentContent = ref('')
const postPasswordInput = ref('')
const commentPasswordInputs = ref({})

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      posts.value = parsed.map(post => ({
        ...post,
        comments: Array.isArray(post.comments) ? post.comments : [],
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


// 게시글 추가/수정
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
    isWriteModalOpen.value = false
    if (selectedPost.value && selectedPost.value.id === target.id) {
      selectedPost.value = { ...target }
    }
    alert('게시글이 수정되었습니다.')
    return
  }

  const newPost = {
    id: Date.now(),
    title,
    content,
    password,
    comments: [],
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  }

  posts.value.unshift(newPost)
  savePosts()
  currentPage.value = 1
  resetForm()
  isWriteModalOpen.value = false
  alert('게시글이 작성되었습니다.')
}

// 게시글 수정 시작
function startEditPost() {
  if (!selectedPost.value) return

  const enteredPassword = postPasswordInput.value.trim()
  if (!enteredPassword) {
    alert('비밀번호를 입력하세요.')
    return
  }

  if (selectedPost.value.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    postPasswordInput.value = ''
    return
  }

  editingId.value = selectedPost.value.id
  form.value.title = selectedPost.value.title
  form.value.content = selectedPost.value.content
  form.value.password = enteredPassword
  closeModal()
}

// 게시글 삭제
function deletePost() {
  if (!selectedPost.value) return

  const enteredPassword = postPasswordInput.value.trim()
  if (!enteredPassword) {
    alert('비밀번호를 입력하세요.')
    return
  }

  if (selectedPost.value.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    postPasswordInput.value = ''
    return
  }

  posts.value = posts.value.filter(post => post.id !== selectedPost.value.id)
  savePosts()
  currentPage.value = 1
  closeModal()
  alert('게시글이 삭제되었습니다.')
}

// 모달 열기/닫기
function openModal(post) {
  selectedPost.value = { ...post }
  isModalOpen.value = true
  postPasswordInput.value = ''
  commentForm.value = { author: '', content: '', password: '' }
  replyingToCommentId.value = null
  editingCommentId.value = null
  commentPasswordInputs.value = {}
  replyForms.value = {}
}

function closeModal() {
  isModalOpen.value = false
  selectedPost.value = null
  postPasswordInput.value = ''
  commentForm.value = { author: '', content: '', password: '' }
  replyingToCommentId.value = null
  editingCommentId.value = null
  commentPasswordInputs.value = {}
  replyForms.value = {}
}

// 댓글 추가
function addComment() {
  if (!selectedPost.value) return

  const author = commentForm.value.author.trim()
  const content = commentForm.value.content.trim()
  const password = commentForm.value.password.trim()

  if (!author || !content || !password) {
    alert('작성자, 내용, 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(p => p.id === selectedPost.value.id)
  if (!post) return

  post.comments.push({
    id: Date.now(),
    author,
    content,
    password,
    parentId: null,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  selectedPost.value.comments = [...post.comments]
  commentForm.value = { author: '', content: '', password: '' }
}

// 대댓글 폼 열기
function toggleReplyForm(commentId) {
  if (replyingToCommentId.value === commentId) {
    replyingToCommentId.value = null
    delete replyForms.value[commentId]
  } else {
    replyingToCommentId.value = commentId
    if (!replyForms.value[commentId]) {
      replyForms.value[commentId] = { author: '', content: '', password: '' }
    }
  }
}

// 대댓글 추가
function addReply(parentCommentId) {
  if (!selectedPost.value) return

  const formData = replyForms.value[parentCommentId]
  if (!formData) return

  const author = formData.author.trim()
  const content = formData.content.trim()
  const password = formData.password.trim()

  if (!author || !content || !password) {
    alert('작성자, 내용, 비밀번호를 모두 입력해주세요.')
    return
  }

  const post = posts.value.find(p => p.id === selectedPost.value.id)
  if (!post) return

  // 대댓글 깊이 확인 (부모가 이미 대댓글이면 추가하지 않음)
  const parentComment = post.comments.find(c => c.id === parentCommentId)
  if (parentComment && parentComment.parentId !== null) {
    alert('대댓글은 1단계만 허용됩니다.')
    return
  }

  post.comments.push({
    id: Date.now(),
    author,
    content,
    password,
    parentId: parentCommentId,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  })

  savePosts()
  selectedPost.value.comments = [...post.comments]
  replyingToCommentId.value = null
  delete replyForms.value[parentCommentId]
}

// 댓글 삭제
function deleteComment(commentId) {
  if (!selectedPost.value) return

  const key = `${selectedPost.value.id}-${commentId}`
  const enteredPassword = (commentPasswordInputs.value[key] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 입력하세요.')
    return
  }

  const post = posts.value.find(p => p.id === selectedPost.value.id)
  if (!post) return

  const comment = post.comments.find(c => c.id === commentId)
  if (!comment) return

  if (comment.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    commentPasswordInputs.value[key] = ''
    return
  }

  // 대댓글도 함께 삭제
  post.comments = post.comments.filter(c => c.id !== commentId && c.parentId !== commentId)
  savePosts()
  selectedPost.value.comments = [...post.comments]
  delete commentPasswordInputs.value[key]
  alert('댓글이 삭제되었습니다.')
}

// 댓글 수정
function startEditComment(commentId) {
  if (!selectedPost.value) return

  const key = `${selectedPost.value.id}-${commentId}`
  const enteredPassword = (commentPasswordInputs.value[key] || '').trim()

  if (!enteredPassword) {
    alert('비밀번호를 입력하세요.')
    return
  }

  const post = posts.value.find(p => p.id === selectedPost.value.id)
  if (!post) return

  const comment = post.comments.find(c => c.id === commentId)
  if (!comment) return

  if (comment.password !== enteredPassword) {
    alert('비밀번호가 일치하지 않습니다.')
    commentPasswordInputs.value[key] = ''
    return
  }

  editingCommentId.value = commentId
  editingCommentContent.value = comment.content
  commentPasswordInputs.value[key] = ''
}

function saveEditComment(commentId) {
  if (!selectedPost.value) return

  const content = editingCommentContent.value.trim()
  if (!content) {
    alert('내용을 입력해주세요.')
    return
  }

  const post = posts.value.find(p => p.id === selectedPost.value.id)
  if (!post) return

  const comment = post.comments.find(c => c.id === commentId)
  if (!comment) return

  comment.content = content
  comment.updatedAt = new Date().toISOString()
  savePosts()
  selectedPost.value.comments = [...post.comments]
  editingCommentId.value = null
  editingCommentContent.value = ''
}

function cancelEditComment() {
  editingCommentId.value = null
  editingCommentContent.value = ''
}

// 글쓰기 모달 열기/닫기
function openWriteModal() {
  isWriteModalOpen.value = true
  resetForm()
}

function closeWriteModal() {
  isWriteModalOpen.value = false
  resetForm()
}

// 검색 처리
function handleSearch() {
  currentPage.value = 1
}

// 댓글 트리 구조 (1단계만)
function getCommentTree() {
  if (!selectedPost.value) return []

  const comments = selectedPost.value.comments || []
  return comments
    .filter(c => c.parentId === null)
    .map(c => ({
      ...c,
      replies: comments.filter(r => r.parentId === c.id)
    }))
}
</script>

<template>
  <section class="card">
    <h3>📝 익명 커뮤니티</h3>
    <p class="hint">
      브라우저 localStorage에 저장됩니다. 게시글, 댓글, 대댓글은 모두 로컬에 저장됩니다.
    </p>

    <!-- 게시글 목록 (전체 너비) -->
    <div class="post-list-section">
      <div class="list-header">
        <h4 class="section-title">게시글 목록</h4>
        <button @click="openWriteModal" class="btn-write">
          📝 글쓰기
        </button>
      </div>

      <!-- 검색창 -->
      <div class="search-box">
        <input
          v-model="searchQuery"
          @input="handleSearch"
          @keyup.enter="handleSearch"
          placeholder="게시글 검색어를 입력하세요"
          class="search-input"
        />
        <button @click="handleSearch" class="search-btn">
          🔍 검색
        </button>
      </div>

      <div v-if="posts.length === 0" class="empty-state">
        <p>📌 게시글이 없습니다. 첫 번째 글을 작성해주세요!</p>
      </div>

      <div v-else class="post-table">
        <div class="post-table-header">
          <div class="col-author">작성자</div>
          <div class="col-title">제목</div>
          <div class="col-date">작성일</div>
        </div>

        <button
          v-for="post in paginatedPosts"
          :key="post.id"
          @click="openModal(post)"
          class="post-table-row"
        >
          <div class="col-author">익명</div>
          <div class="col-title">{{ post.title }}</div>
          <div class="col-date">{{ formatDate(post.createdAt) }}</div>
        </button>
      </div>

      <!-- 페이지네이션 -->
      <div class="pagination">
        <button
          @click="currentPage = currentPage - 1"
          :disabled="currentPage === 1"
          class="pagination-btn"
        >
          ← 이전
        </button>

        <button
          v-for="page in totalPages"
          :key="page"
          @click="currentPage = page"
          :class="['pagination-btn', { active: currentPage === page }]"
        >
          {{ page }}
        </button>

        <button
          @click="currentPage = currentPage + 1"
          :disabled="currentPage === totalPages"
          class="pagination-btn"
        >
          다음 →
        </button>
      </div>
    </div>

    <!-- 상세 보기 모달 -->
    <Teleport to="body">
      <div v-if="isModalOpen" class="modal-overlay" @click.self="closeModal">
        <div class="modal-container">
          <!-- 모달 헤더 -->
          <div class="modal-header">
            <h3>{{ selectedPost?.title }}</h3>
            <button class="close-btn" @click="closeModal">✕</button>
          </div>

          <!-- 모달 콘텐츠 (스크롤 가능) -->
          <div class="modal-content">
            <!-- 게시글 정보 -->
            <div class="post-detail-section">
              <div class="post-meta">
                <span><strong>작성자:</strong> 익명</span>
                <span><strong>작성일:</strong> {{ formatDate(selectedPost?.createdAt) }}</span>
                <span v-if="selectedPost?.updatedAt !== selectedPost?.createdAt">
                  <strong>수정일:</strong> {{ formatDate(selectedPost?.updatedAt) }}
                </span>
              </div>

              <div class="post-body">
                {{ selectedPost?.content }}
              </div>

              <!-- 수정/삭제 -->
              <div class="post-actions">
                <input
                  v-model="postPasswordInput"
                  type="password"
                  placeholder="비밀번호"
                  class="password-input"
                />
                <button @click="startEditPost" class="btn-edit">수정</button>
                <button @click="deletePost" class="btn-danger">삭제</button>
              </div>
            </div>

            <!-- 댓글 영역 -->
            <div class="comment-section">
              <h4>💬 댓글 및 대댓글</h4>

              <!-- 댓글 작성 -->
              <div class="comment-write">
                <input
                  v-model="commentForm.author"
                  placeholder="작성자 이름"
                  class="input-small"
                />
                <textarea
                  v-model="commentForm.content"
                  placeholder="댓글을 입력하세요"
                  class="textarea-small"
                ></textarea>
                <input
                  v-model="commentForm.password"
                  type="password"
                  placeholder="비밀번호"
                  class="input-small"
                />
                <button @click="addComment" class="btn-primary">댓글 작성</button>
              </div>

              <!-- 댓글 목록 -->
              <div v-if="getCommentTree().length" class="comment-tree">
                <div v-for="comment in getCommentTree()" :key="comment.id" class="comment-group">
                  <!-- 부모 댓글 -->
                  <div class="comment-item">
                    <div class="comment-header">
                      <span class="comment-author">{{ comment.author }}</span>
                      <span class="comment-date">{{ formatDate(comment.createdAt) }}</span>
                    </div>

                    <div v-if="editingCommentId === comment.id" class="comment-edit">
                      <textarea v-model="editingCommentContent" class="textarea-small"></textarea>
                      <div class="edit-actions">
                        <button @click="saveEditComment(comment.id)" class="btn-primary">
                          수정 완료
                        </button>
                        <button @click="cancelEditComment" class="btn-secondary">취소</button>
                      </div>
                    </div>

                    <div v-else class="comment-body">
                      {{ comment.content }}
                    </div>

                    <div class="comment-footer">
                      <div class="comment-actions">
                        <input
                          :value="commentPasswordInputs[`${selectedPost.id}-${comment.id}`] || ''"
                          @input="
                            commentPasswordInputs[`${selectedPost.id}-${comment.id}`] =
                              $event.target.value
                          "
                          type="password"
                          placeholder="비밀번호"
                          class="password-input-small"
                        />
                        <button
                          @click="startEditComment(comment.id)"
                          class="btn-action"
                          v-if="editingCommentId !== comment.id"
                        >
                          수정
                        </button>
                        <button @click="deleteComment(comment.id)" class="btn-action btn-danger">
                          삭제
                        </button>
                        <button
                          @click="toggleReplyForm(comment.id)"
                          class="btn-action"
                          v-if="editingCommentId !== comment.id"
                        >
                          {{ replyingToCommentId === comment.id ? '취소' : '답글' }}
                        </button>
                      </div>
                    </div>
                  </div>

                  <!-- 대댓글 작성 폼 -->
                  <div v-if="replyingToCommentId === comment.id" class="reply-write">
                    <input
                      v-model="replyForms[comment.id].author"
                      placeholder="작성자 이름"
                      class="input-small"
                    />
                    <textarea
                      v-model="replyForms[comment.id].content"
                      placeholder="대댓글을 입력하세요"
                      class="textarea-small"
                    ></textarea>
                    <input
                      v-model="replyForms[comment.id].password"
                      type="password"
                      placeholder="비밀번호"
                      class="input-small"
                    />
                    <button @click="addReply(comment.id)" class="btn-primary">대댓글 등록</button>
                  </div>

                  <!-- 대댓글 목록 -->
                  <div v-if="comment.replies && comment.replies.length" class="reply-tree">
                    <div v-for="reply in comment.replies" :key="reply.id" class="reply-item">
                      <div class="reply-icon">└</div>
                      <div class="reply-content">
                        <div class="comment-header">
                          <span class="comment-author">{{ reply.author }}</span>
                          <span class="comment-date">{{ formatDate(reply.createdAt) }}</span>
                        </div>

                        <div v-if="editingCommentId === reply.id" class="comment-edit">
                          <textarea
                            v-model="editingCommentContent"
                            class="textarea-small"
                          ></textarea>
                          <div class="edit-actions">
                            <button @click="saveEditComment(reply.id)" class="btn-primary">
                              수정 완료
                            </button>
                            <button @click="cancelEditComment" class="btn-secondary">취소</button>
                          </div>
                        </div>

                        <div v-else class="comment-body">
                          {{ reply.content }}
                        </div>

                        <div class="comment-footer">
                          <div class="comment-actions">
                            <input
                              :value="commentPasswordInputs[`${selectedPost.id}-${reply.id}`] || ''"
                              @input="
                                commentPasswordInputs[`${selectedPost.id}-${reply.id}`] =
                                  $event.target.value
                              "
                              type="password"
                              placeholder="비밀번호"
                              class="password-input-small"
                            />
                            <button
                              @click="startEditComment(reply.id)"
                              class="btn-action"
                              v-if="editingCommentId !== reply.id"
                            >
                              수정
                            </button>
                            <button
                              @click="deleteComment(reply.id)"
                              class="btn-action btn-danger"
                            >
                              삭제
                            </button>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>

              <div v-else class="empty-comments">
                <p>아직 댓글이 없습니다. 첫 번째 댓글을 작성해보세요!</p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Teleport>

    <!-- 글쓰기 모달 -->
    <Teleport to="body">
      <div v-if="isWriteModalOpen" class="modal-overlay" @click.self="closeWriteModal">
        <div class="modal-container">
          <div class="modal-header">
            <h3>{{ editingId ? '글 수정' : '새 글 작성' }}</h3>
            <button class="close-btn" @click="closeWriteModal">✕</button>
          </div>

          <form @submit.prevent="addOrUpdatePost" class="modal-content-form">
            <div class="form-group">
              <label>제목</label>
              <input v-model="form.title" placeholder="제목을 입력하세요" />
            </div>

            <div class="form-group">
              <label>내용</label>
              <textarea v-model="form.content" placeholder="내용을 입력하세요"></textarea>
            </div>

            <div class="form-group">
              <label>비밀번호</label>
              <input v-model="form.password" type="password" placeholder="비밀번호를 입력하세요" />
            </div>

            <div class="modal-footer">
              <button type="submit" class="btn-primary">
                {{ editingId ? '수정 완료' : '작성 완료' }}
              </button>
              <button type="button" @click="closeWriteModal" class="btn-secondary">취소</button>
            </div>
          </form>
        </div>
      </div>
    </Teleport>
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

.btn-edit {
  background: #10b981;
}

.btn-edit:hover {
  background: #059669;
}

.btn-primary {
  background: #3b82f6;
}

.btn-primary:hover {
  background: #2563eb;
}

.btn-secondary {
  background: #6b7280;
}

.btn-secondary:hover {
  background: #4b5563;
}

.btn-action {
  padding: 6px 10px;
  font-size: 0.85rem;
  background: #3b82f6;
}

.btn-action:hover {
  background: #2563eb;
}

/* 목록 섹션 */
.post-list-section {
  margin-top: 20px;
}

.section-title {
  font-size: 1rem;
  font-weight: 600;
  color: #1f2937;
  margin: 0 0 12px 0;
  padding-bottom: 8px;
  border-bottom: 2px solid #e5e7eb;
}

.post-table {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
}

.post-table-header {
  display: grid;
  grid-template-columns: 100px 1fr 150px;
  background: #f9fafb;
  padding: 12px;
  font-weight: 600;
  font-size: 0.9rem;
  color: #6b7280;
  border-bottom: 1px solid #e5e7eb;
  gap: 12px;
}

.post-table-row {
  display: grid;
  grid-template-columns: 100px 1fr 150px;
  padding: 12px;
  border-bottom: 1px solid #e5e7eb;
  background: white;
  border: none;
  text-align: left;
  cursor: pointer;
  transition: background 0.2s;
  gap: 12px;
}

.post-table-row:last-child {
  border-bottom: none;
}

.post-table-row:hover {
  background: #f3f4f6;
}

.col-author,
.col-title,
.col-date {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.col-author {
  font-weight: 500;
  color: #374151;
}

.col-title {
  color: #1f2937;
  font-weight: 500;
}

.col-date {
  color: #9ca3af;
  font-size: 0.85rem;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #9ca3af;
  background: #f9fafb;
  border-radius: 8px;
  border: 2px dashed #e5e7eb;
}

.empty-state p {
  font-size: 1rem;
  margin: 0;
}

/* 페이지네이션 */
.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 8px;
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid #e5e7eb;
  flex-wrap: wrap;
}

.pagination-btn {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  background: white;
  color: #374151;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 500;
  transition: all 0.2s;
}

.pagination-btn:hover:not(:disabled) {
  background: #f3f4f6;
  border-color: #9ca3af;
}

.pagination-btn.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
  font-weight: 600;
}

.pagination-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pagination-btn:disabled:hover {
  background: white;
}

/* 모달 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 50;
  padding: 20px;
}

.modal-container {
  background: white;
  border-radius: 12px;
  width: 100%;
  max-width: 700px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
  animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #e5e7eb;
  gap: 12px;
}

.modal-header h3 {
  margin: 0;
  font-size: 1.1rem;
  color: #1f2937;
  word-break: break-word;
  flex: 1;
}

.close-btn {
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
  flex-shrink: 0;
}

.close-btn:hover {
  background: #d1d5db;
  color: #1f2937;
}

.modal-content {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.post-detail-section {
  padding-bottom: 20px;
  border-bottom: 2px solid #e5e7eb;
}

.post-meta {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.9rem;
  color: #6b7280;
  margin-bottom: 12px;
}

.post-meta span strong {
  color: #1f2937;
}

.post-body {
  padding: 12px;
  background: #f9fafb;
  border-radius: 6px;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-word;
  color: #374151;
  margin-bottom: 12px;
}

.post-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  align-items: center;
}

.password-input {
  min-width: 120px;
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
}

/* 댓글 섹션 */
.comment-section {
  flex: 1;
}

.comment-section h4 {
  font-size: 0.95rem;
  color: #1f2937;
  margin: 0 0 12px 0;
}

.comment-write {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 12px;
  background: #f3f4f6;
  border-radius: 6px;
  margin-bottom: 16px;
}

.input-small,
.textarea-small {
  padding: 8px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-family: inherit;
  font-size: 0.9rem;
}

.textarea-small {
  min-height: 60px;
  resize: vertical;
}

.comment-tree {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.comment-group {
  border-left: 2px solid #e5e7eb;
  padding-left: 12px;
}

.comment-item {
  padding: 10px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
  gap: 8px;
}

.comment-author {
  font-weight: 600;
  color: #1f2937;
}

.comment-date {
  font-size: 0.8rem;
  color: #9ca3af;
}

.comment-body {
  padding: 8px 0;
  color: #374151;
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

.comment-edit {
  margin: 8px 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.edit-actions {
  display: flex;
  gap: 6px;
}

.comment-footer {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #e5e7eb;
}

.comment-actions {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  align-items: center;
}

.password-input-small {
  padding: 6px 8px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 0.85rem;
  min-width: 100px;
}

/* 대댓글 */
.reply-tree {
  margin-top: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.reply-item {
  display: flex;
  gap: 8px;
  padding-left: 12px;
  border-left: 2px solid #f3f4f6;
}

.reply-icon {
  flex-shrink: 0;
  color: #d1d5db;
  font-weight: bold;
  font-size: 0.9rem;
  line-height: 1.5;
}

.reply-content {
  flex: 1;
  padding: 10px;
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
}

.reply-write {
  margin-top: 12px;
  padding-left: 20px;
  border-left: 2px solid #e5e7eb;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.empty-comments {
  text-align: center;
  padding: 16px;
  color: #9ca3af;
}

.empty-comments p {
  margin: 0;
}

/* 목록 헤더 */
.list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  gap: 12px;
}

.btn-write {
  padding: 10px 16px;
  background: #7c3aed;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  font-size: 0.9rem;
  transition: all 0.2s;
  flex-shrink: 0;
}

.btn-write:hover {
  background: #6d28d9;
  box-shadow: 0 4px 6px rgba(124, 58, 237, 0.3);
}

/* 글쓰기 모달 폼 */
.modal-content-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-group label {
  font-weight: 600;
  color: #1f2937;
  font-size: 0.9rem;
}

.form-group input,
.form-group textarea {
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-family: inherit;
  font-size: 0.95rem;
}

.form-group textarea {
  min-height: 120px;
  resize: vertical;
}

.modal-footer {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
  padding: 16px 20px;
  border-top: 1px solid #e5e7eb;
  background: #f9fafb;
}
/* 검색창 */
.search-box {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
}

.search-input {
  flex: 1;
  padding: 10px 14px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.95rem;
  font-family: inherit;
  transition: all 0.2s;
}

.search-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.search-input::placeholder {
  color: #9ca3af;
}

.search-btn {
  padding: 10px 16px;
  background: #06b6d4;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  font-size: 0.9rem;
  transition: all 0.2s;
  white-space: nowrap;
  flex-shrink: 0;
}

.search-btn:hover {
  background: #0891b2;
  box-shadow: 0 2px 4px rgba(6, 182, 212, 0.3);
}

.search-btn:active {
  transform: scale(0.98);
}
</style>