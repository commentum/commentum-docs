# Vue.js Integration Example

This example demonstrates how to integrate Commentum into a Vue.js 3 application using TypeScript and the Composition API.

## Project Setup

### 1. Install Dependencies

```bash
npm install @supabase/supabase-js vue-query pinia @vueuse/core
# or
yarn add @supabase/supabase-js vue-query pinia @vueuse/core
```

### 2. Environment Configuration

```env
# .env.local
VITE_SUPABASE_URL=your_supabase_project_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
VITE_COMMENTUM_API_URL=https://your-project.supabase.co/functions/v1
```

### 3. Supabase Client Configuration

```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY

export const supabase = createClient(supabaseUrl, supabaseAnonKey)

// Commentum API client
export const commentumAPI = {
  baseURL: import.meta.env.VITE_COMMENTUM_API_URL,
  
  async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
    const url = `${this.baseURL}${endpoint}`
    
    const response = await fetch(url, {
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      ...options,
    })
    
    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.error || 'API request failed')
    }
    
    return response.json()
  }
}
```

## Stores and Composables

### 1. Comment Store (Pinia)

```typescript
// src/stores/commentStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { commentumAPI } from '@/lib/supabase'
import type { Comment, CreateCommentData } from '@/types/comment'

export const useCommentStore = defineStore('comments', () => {
  // State
  const comments = ref<Comment[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const commentsByMediaId = computed(() => {
    return (mediaId: number) => comments.value.filter(comment => comment.media_id === mediaId)
  })

  const topLevelComments = computed(() => {
    return (mediaId: number) => 
      commentsByMediaId(mediaId).filter(comment => !comment.parent_id)
  })

  const repliesByParentId = computed(() => {
    const replies: Record<number, Comment[]> = {}
    
    comments.value.forEach(comment => {
      if (comment.parent_id) {
        if (!replies[comment.parent_id]) {
          replies[comment.parent_id] = []
        }
        replies[comment.parent_id].push(comment)
      }
    })
    
    return replies
  })

  // Actions
  const fetchComments = async (mediaId: number) => {
    loading.value = true
    error.value = null

    try {
      const response = await commentumAPI.request<Comment[]>(`/comments?media_id=${mediaId}`)
      comments.value = response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to fetch comments'
      throw err
    } finally {
      loading.value = false
    }
  }

  const createComment = async (data: CreateCommentData) => {
    try {
      const response = await commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'create',
          ...data,
        }),
      })

      comments.value.push(response)
      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to create comment'
      throw err
    }
  }

  const updateComment = async (commentId: number, content: string, userId: number) => {
    try {
      const response = await commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'edit',
          user_id: userId,
          comment_id: commentId,
          content,
        }),
      })

      const index = comments.value.findIndex(c => c.id === commentId)
      if (index !== -1) {
        comments.value[index] = response
      }

      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to update comment'
      throw err
    }
  }

  const deleteComment = async (commentId: number, userId: number) => {
    try {
      const response = await commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'delete',
          user_id: userId,
          comment_id: commentId,
        }),
      })

      const index = comments.value.findIndex(c => c.id === commentId)
      if (index !== -1) {
        comments.value[index] = response
      }

      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to delete comment'
      throw err
    }
  }

  const clearError = () => {
    error.value = null
  }

  return {
    // State
    comments: readonly(comments),
    loading: readonly(loading),
    error: readonly(error),
    
    // Getters
    commentsByMediaId,
    topLevelComments,
    repliesByParentId,
    
    // Actions
    fetchComments,
    createComment,
    updateComment,
    deleteComment,
    clearError,
  }
})
```

### 2. Auth Store

```typescript
// src/stores/authStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { commentumAPI } from '@/lib/supabase'
import type { UserIdentity } from '@/types/auth'

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<UserIdentity | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const isAuthenticated = computed(() => !!user.value)
  const isModerator = computed(() => 
    user.value?.role === 'moderator' || user.value?.role === 'admin' || user.value?.role === 'super_admin'
  )
  const isAdmin = computed(() => 
    user.value?.role === 'admin' || user.value?.role === 'super_admin'
  )

  // Actions
  const resolveIdentity = async (
    clientType: 'anilist' | 'myanimelist' | 'simkl',
    clientUserId: string,
    username: string,
    avatarUrl?: string
  ) => {
    loading.value = true
    error.value = null

    try {
      const response = await commentumAPI.request<UserIdentity>('/identity-resolve', {
        method: 'POST',
        body: JSON.stringify({
          client_type: clientType,
          client_user_id: clientUserId,
          username,
          avatar_url: avatarUrl,
        }),
      })

      if (response.banned) {
        throw new Error('Account is banned')
      }

      if (response.muted_until && new Date(response.muted_until) > new Date()) {
        throw new Error(`Account muted until ${response.muted_until}`)
      }

      user.value = response
      
      // Store in localStorage for persistence
      localStorage.setItem('commentum_user', JSON.stringify({
        user: response,
        expiresAt: Date.now() + (24 * 60 * 60 * 1000), // 24 hours
      }))

      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Authentication failed'
      throw err
    } finally {
      loading.value = false
    }
  }

  const logout = () => {
    user.value = null
    localStorage.removeItem('commentum_user')
  }

  const initializeAuth = () => {
    const stored = localStorage.getItem('commentum_user')
    if (stored) {
      try {
        const data = JSON.parse(stored)
        if (data.expiresAt > Date.now()) {
          user.value = data.user
        } else {
          localStorage.removeItem('commentum_user')
        }
      } catch (err) {
        localStorage.removeItem('commentum_user')
      }
    }
  }

  const clearError = () => {
    error.value = null
  }

  return {
    // State
    user: readonly(user),
    loading: readonly(loading),
    error: readonly(error),
    
    // Getters
    isAuthenticated,
    isModerator,
    isAdmin,
    
    // Actions
    resolveIdentity,
    logout,
    initializeAuth,
    clearError,
  }
})
```

### 3. Voting Composable

```typescript
// src/composables/useVoting.ts
import { ref, computed } from 'vue'
import { commentumAPI } from '@/lib/supabase'
import type { VoteData } from '@/types/voting'

export function useVoting(commentId: number, initialVotes: VoteData, userId?: number) {
  // State
  const votes = ref<VoteData>({ ...initialVotes })
  const loading = ref(false)

  // Getters
  const totalVotes = computed(() => votes.value.upvotes - votes.value.downvotes)
  const hasVoted = computed(() => votes.value.userVote !== 0)

  // Actions
  const vote = async (action: 'upvote' | 'downvote' | 'remove') => {
    if (!userId || loading.value) return

    loading.value = true

    try {
      const response = await commentumAPI.request('/voting', {
        method: 'POST',
        body: JSON.stringify({
          action,
          user_id: userId,
          comment_id: commentId,
        }),
      })

      // Update local state based on action
      if (action === 'remove') {
        votes.value = {
          ...votes.value,
          upvotes: votes.value.userVote === 1 ? votes.value.upvotes - 1 : votes.value.upvotes,
          downvotes: votes.value.userVote === -1 ? votes.value.downvotes - 1 : votes.value.downvotes,
          userVote: 0,
        }
      } else {
        const voteType = action === 'upvote' ? 1 : -1
        
        // Remove previous vote if different
        if (votes.value.userVote === 1 && voteType === -1) {
          votes.value.upvotes--
        } else if (votes.value.userVote === -1 && voteType === 1) {
          votes.value.downvotes--
        }
        
        // Add new vote if different from previous
        if (votes.value.userVote !== voteType) {
          if (voteType === 1) votes.value.upvotes++
          else votes.value.downvotes++
        }
        
        votes.value.userVote = votes.value.userVote === voteType ? 0 : voteType
      }

      return response
    } catch (error) {
      console.error('Vote failed:', error)
      throw error
    } finally {
      loading.value = false
    }
  }

  const upvote = () => vote('upvote')
  const downvote = () => vote('downvote')
  const removeVote = () => vote('remove')

  return {
    // State
    votes: readonly(votes),
    loading: readonly(loading),
    
    // Getters
    totalVotes,
    hasVoted,
    
    // Actions
    upvote,
    downvote,
    removeVote,
  }
}
```

### 4. Toast Composable

```typescript
// src/composables/useToast.ts
import { ref } from 'vue'

interface Toast {
  id: string
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
  duration?: number
}

export function useToast() {
  const toasts = ref<Toast[]>([])

  const addToast = (message: string, type: Toast['type'] = 'info', duration = 5000) => {
    const id = Date.now().toString()
    const toast: Toast = { id, message, type, duration }

    toasts.value.push(toast)

    if (duration > 0) {
      setTimeout(() => {
        removeToast(id)
      }, duration)
    }

    return id
  }

  const removeToast = (id: string) => {
    const index = toasts.value.findIndex(toast => toast.id === id)
    if (index > -1) {
      toasts.value.splice(index, 1)
    }
  }

  const success = (message: string) => addToast(message, 'success')
  const error = (message: string) => addToast(message, 'error')
  const warning = (message: string) => addToast(message, 'warning')
  const info = (message: string) => addToast(message, 'info')

  return {
    toasts: readonly(toasts),
    addToast,
    removeToast,
    success,
    error,
    warning,
    info,
  }
}
```

## Components

### 1. Comment System Component

```vue
<!-- src/components/CommentSystem.vue -->
<template>
  <div class="comment-system">
    <div class="comment-system-header">
      <h3>Comments ({{ topLevelComments.length }})</h3>
      <CommentSort />
    </div>

    <div v-if="authStore.isAuthenticated" class="comment-form-section">
      <CommentForm @success="handleCommentSuccess" />
    </div>

    <div class="comments-section">
      <div v-if="commentStore.loading" class="loading">
        <div class="loading-spinner">Loading comments...</div>
      </div>

      <div v-else-if="commentStore.error" class="error">
        <div class="error-message">
          Failed to load comments: {{ commentStore.error }}
        </div>
        <button @click="loadComments" class="retry-button">Retry</button>
      </div>

      <div v-else-if="topLevelComments.length === 0" class="no-comments">
        <p>No comments yet. Be the first to share your thoughts!</p>
      </div>

      <div v-else class="comments-tree">
        <CommentItem
          v-for="comment in topLevelComments"
          :key="comment.id"
          :comment="comment"
          :replies="repliesByParentId[comment.id] || []"
          @update="loadComments"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, watch } from 'vue'
import { useCommentStore } from '@/stores/commentStore'
import { useAuthStore } from '@/stores/authStore'
import CommentForm from './CommentForm.vue'
import CommentItem from './CommentItem.vue'
import CommentSort from './CommentSort.vue'

interface Props {
  mediaId: number
  mediaType: 'anime' | 'manga' | 'movie' | 'tv'
}

const props = defineProps<Props>()

const commentStore = useCommentStore()
const authStore = useAuthStore()

const topLevelComments = computed(() => 
  commentStore.topLevelComments(props.mediaId)
)

const repliesByParentId = computed(() => commentStore.repliesByParentId)

const loadComments = async () => {
  try {
    await commentStore.fetchComments(props.mediaId)
  } catch (error) {
    console.error('Failed to load comments:', error)
  }
}

const handleCommentSuccess = () => {
  loadComments()
}

// Load comments on mount
onMounted(() => {
  loadComments()
})

// Reload comments when mediaId changes
watch(() => props.mediaId, () => {
  loadComments()
}, { immediate: true })
</script>

<style scoped>
.comment-system {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.comment-system-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 1px solid #e2e8f0;
}

.comment-form-section {
  margin-bottom: 24px;
}

.comments-section {
  min-height: 200px;
}

.loading, .error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  text-align: center;
}

.error-message {
  color: #ef4444;
  margin-bottom: 16px;
}

.retry-button {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
}

.no-comments {
  text-align: center;
  padding: 40px;
  color: #6b7280;
}

.comments-tree {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
</style>
```

### 2. Comment Form Component

```vue
<!-- src/components/CommentForm.vue -->
<template>
  <form @submit.prevent="handleSubmit" class="comment-form">
    <div class="form-group">
      <textarea
        v-model="formData.content"
        :placeholder="placeholder"
        class="comment-textarea"
        :class="{ error: errors.content }"
        rows="4"
        :disabled="submitting"
      />
      <span v-if="errors.content" class="error-message">{{ errors.content }}</span>
    </div>

    <div class="form-actions">
      <div class="form-left">
        <span :class="['character-count', { warning: characterCount > 9000 }]">
          {{ characterCount }}/10000
        </span>
      </div>

      <div class="form-right">
        <button
          v-if="onCancel"
          type="button"
          @click="onCancel"
          :disabled="submitting"
          class="cancel-button"
        >
          Cancel
        </button>
        
        <button
          type="submit"
          :disabled="submitting || !formData.content.trim()"
          class="submit-button"
        >
          {{ submitting ? 'Posting...' : buttonText }}
        </button>
      </div>
    </div>
  </form>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useCommentStore } from '@/stores/commentStore'
import { useAuthStore } from '@/stores/authStore'
import { useToast } from '@/composables/useToast'

interface Props {
  parentId?: number
  onSuccess?: () => void
  onCancel?: () => void
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: 'Share your thoughts...',
  buttonText: 'Post Comment',
})

const emit = defineEmits<{
  success: []
}>()

const commentStore = useCommentStore()
const authStore = useAuthStore()
const { success, error } = useToast()

const formData = ref({
  content: '',
})

const errors = ref<Record<string, string>>({})
const submitting = ref(false)

const characterCount = computed(() => formData.value.content.length)

const validateForm = () => {
  errors.value = {}

  if (!formData.value.content.trim()) {
    errors.value.content = 'Comment cannot be empty'
  } else if (formData.value.content.length > 10000) {
    errors.value.content = 'Comment cannot exceed 10,000 characters'
  }

  return Object.keys(errors.value).length === 0
}

const handleSubmit = async () => {
  if (!validateForm() || !authStore.user) return

  submitting.value = true

  try {
    await commentStore.createComment({
      user_id: authStore.user.user_id,
      media_id: 0, // This should be passed as a prop
      content: formData.value.content.trim(),
      parent_id: props.parentId,
    })

    formData.value.content = ''
    success('Comment posted successfully!')
    emit('success')
    props.onSuccess?.()
  } catch (err) {
    error(err instanceof Error ? err.message : 'Failed to post comment')
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.comment-form {
  background: #f8fafc;
  border-radius: 8px;
  padding: 16px;
}

.form-group {
  margin-bottom: 12px;
}

.comment-textarea {
  width: 100%;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  padding: 12px;
  font-size: 14px;
  line-height: 1.5;
  resize: vertical;
  transition: border-color 0.2s;
}

.comment-textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.comment-textarea.error {
  border-color: #ef4444;
}

.error-message {
  display: block;
  font-size: 12px;
  color: #ef4444;
  margin-top: 4px;
}

.form-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.character-count {
  font-size: 12px;
  color: #6b7280;
}

.character-count.warning {
  color: #f59e0b;
}

.submit-button, .cancel-button {
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.submit-button {
  background: #3b82f6;
  color: white;
  border: none;
}

.submit-button:hover:not(:disabled) {
  background: #2563eb;
}

.submit-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.cancel-button {
  background: transparent;
  color: #6b7280;
  border: 1px solid #d1d5db;
  margin-right: 8px;
}

.cancel-button:hover:not(:disabled) {
  background: #f3f4f6;
}
</style>
```

### 3. Comment Item Component

```vue
<!-- src/components/CommentItem.vue -->
<template>
  <div :class="['comment-item', `level-${level}`]">
    <div class="comment-content">
      <div class="comment-header">
        <div class="comment-author">
          <img
            :src="comment.avatar_url || '/default-avatar.png'"
            :alt="comment.username"
            class="author-avatar"
          />
          <div class="author-info">
            <span class="author-name">{{ comment.username }}</span>
            <span class="comment-time">{{ formattedTime }}</span>
            <span v-if="comment.edited" class="edited-indicator">(edited)</span>
          </div>
        </div>

        <div class="comment-meta">
          <span v-if="comment.pinned" class="pinned-indicator">📌 Pinned</span>
          <span v-if="comment.locked" class="locked-indicator">🔒 Locked</span>
        </div>
      </div>

      <div class="comment-body">
        <p v-if="comment.deleted" class="deleted-message">This comment has been deleted</p>
        <div v-else-if="isEditing" class="edit-form">
          <CommentEditForm
            :comment="comment"
            @save="handleEditSave"
            @cancel="isEditing = false"
          />
        </div>
        <div v-else class="comment-text">{{ comment.content }}</div>
      </div>

      <div v-if="!comment.deleted && !isEditing" class="comment-actions">
        <VotingButtons
          :comment-id="comment.id"
          :initial-votes="{
            upvotes: comment.upvotes,
            downvotes: comment.downvotes,
            userVote: comment.user_vote,
          }"
          @update="$emit('update')"
        />

        <div class="action-buttons">
          <button
            v-if="authStore.isAuthenticated"
            @click="toggleReplyForm"
            class="reply-button"
          >
            Reply
          </button>

          <button
            v-if="canEdit"
            @click="isEditing = true"
            class="edit-button"
          >
            Edit
          </button>

          <CommentActions
            :comment="comment"
            @update="$emit('update')"
          />
        </div>
      </div>
    </div>

    <!-- Replies Section -->
    <div v-if="hasReplies" class="comment-replies">
      <button @click="toggleReplies" class="toggle-replies-button">
        {{ isExpanded ? 'Hide' : 'Show' }} {{ replies.length }} {{ replies.length === 1 ? 'reply' : 'replies' }}
      </button>

      <div v-if="isExpanded" class="replies-list">
        <CommentItem
          v-for="reply in replies"
          :key="reply.id"
          :comment="reply"
          :level="level + 1"
          @update="$emit('update')"
        />
      </div>
    </div>

    <!-- Reply Form -->
    <div v-if="showReplyForm && authStore.isAuthenticated" class="reply-form">
      <CommentForm
        :parent-id="comment.id"
        @success="handleReplySuccess"
        @cancel="showReplyForm = false"
        :button-text="'Reply'"
        :placeholder="'Write a reply...'"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { formatDistanceToNow } from 'date-fns'
import { useAuthStore } from '@/stores/authStore'
import type { Comment } from '@/types/comment'
import VotingButtons from './VotingButtons.vue'
import CommentActions from './CommentActions.vue'
import CommentEditForm from './CommentEditForm.vue'
import CommentForm from './CommentForm.vue'

interface Props {
  comment: Comment
  replies?: Comment[]
  level?: number
}

const props = withDefaults(defineProps<Props>(), {
  replies: () => [],
  level: 0,
})

defineEmits<{
  update: []
}>()

const authStore = useAuthStore()

const isEditing = ref(false)
const showReplyForm = ref(false)
const isExpanded = ref(false)

const formattedTime = computed(() => 
  formatDistanceToNow(new Date(props.comment.created_at), { addSuffix: true })
)

const canEdit = computed(() => 
  authStore.user?.user_id === props.comment.user_id && !props.comment.deleted
)

const hasReplies = computed(() => props.replies.length > 0)

const toggleReplies = () => {
  isExpanded.value = !isExpanded.value
}

const toggleReplyForm = () => {
  showReplyForm.value = !showReplyForm.value
}

const handleEditSave = () => {
  isEditing.value = false
  // Emit update to refresh parent
}

const handleReplySuccess = () => {
  showReplyForm.value = false
  // Emit update to refresh parent
}
</script>

<style scoped>
.comment-item {
  margin-bottom: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
}

.comment-item.level-1 {
  margin-left: 24px;
  border-color: #f3f4f6;
}

.comment-item.level-2 {
  margin-left: 48px;
  border-color: #f9fafb;
}

.comment-content {
  padding: 16px;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 8px;
}

.comment-author {
  display: flex;
  align-items: center;
  gap: 8px;
}

.author-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  object-fit: cover;
}

.author-name {
  font-weight: 600;
  color: #1f2937;
}

.comment-time {
  font-size: 12px;
  color: #6b7280;
  margin-left: 8px;
}

.edited-indicator {
  font-size: 12px;
  color: #9ca3af;
  font-style: italic;
}

.comment-text {
  line-height: 1.6;
  color: #374151;
  white-space: pre-wrap;
}

.deleted-message {
  font-style: italic;
  color: #6b7280;
}

.comment-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #f3f4f6;
}

.action-buttons {
  display: flex;
  gap: 8px;
}

.reply-button, .edit-button {
  background: transparent;
  border: none;
  color: #6b7280;
  font-size: 12px;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  transition: all 0.2s;
}

.reply-button:hover, .edit-button:hover {
  background: #f3f4f6;
  color: #374151;
}

.comment-replies {
  margin-top: 12px;
  padding-left: 16px;
  border-left: 2px solid #e5e7eb;
}

.toggle-replies-button {
  background: transparent;
  border: none;
  color: #3b82f6;
  font-size: 12px;
  cursor: pointer;
  padding: 4px 0;
  margin-bottom: 8px;
}

.toggle-replies-button:hover {
  text-decoration: underline;
}

.replies-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.reply-form {
  margin-top: 12px;
  padding-left: 16px;
}
</style>
```

### 4. Voting Buttons Component

```vue
<!-- src/components/VotingButtons.vue -->
<template>
  <div class="voting-buttons">
    <button
      @click="handleUpvote"
      :disabled="loading || !userId"
      :class="['vote-button', 'upvote', { voted: votes.userVote === 1 }]"
    >
      ▲ {{ votes.upvotes }}
    </button>
    
    <button
      @click="handleDownvote"
      :disabled="loading || !userId"
      :class="['vote-button', 'downvote', { voted: votes.userVote === -1 }]"
    >
      ▼ {{ votes.downvotes }}
    </button>
  </div>
</template>

<script setup lang="ts">
import { useVoting } from '@/composables/useVoting'
import type { VoteData } from '@/types/voting'

interface Props {
  commentId: number
  initialVotes: VoteData
  userId?: number
}

const props = defineProps<Props>()

const emit = defineEmits<{
  update: []
}>()

const { votes, loading, upvote, downvote, removeVote } = useVoting(
  props.commentId,
  props.initialVotes,
  props.userId
)

const handleUpvote = async () => {
  try {
    if (votes.userVote === 1) {
      await removeVote()
    } else {
      await upvote()
    }
    emit('update')
  } catch (error) {
    console.error('Upvote failed:', error)
  }
}

const handleDownvote = async () => {
  try {
    if (votes.userVote === -1) {
      await removeVote()
    } else {
      await downvote()
    }
    emit('update')
  } catch (error) {
    console.error('Downvote failed:', error)
  }
}
</script>

<style scoped>
.voting-buttons {
  display: flex;
  gap: 8px;
}

.vote-button {
  background: transparent;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  padding: 4px 8px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 4px;
}

.vote-button:hover:not(:disabled) {
  background: #f3f4f6;
}

.vote-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.vote-button.voted {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.vote-button.upvote.voted {
  background: #10b981;
  border-color: #10b981;
}

.vote-button.downvote.voted {
  background: #ef4444;
  border-color: #ef4444;
}
</style>
```

## Usage Example

```vue
<!-- src/views/MediaView.vue -->
<template>
  <div class="media-page">
    <header>
      <h1>{{ media?.title }}</h1>
      <p>Type: {{ media?.type }}</p>
    </header>

    <main>
      <section class="media-content">
        <!-- Your media content here -->
      </section>

      <section class="comments-section">
        <CommentSystem
          v-if="media"
          :media-id="media.id"
          :media-type="media.type"
        />
      </section>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useAuthStore } from '@/stores/authStore'
import CommentSystem from '@/components/CommentSystem.vue'

interface Media {
  id: number
  title: string
  type: 'anime' | 'manga' | 'movie' | 'tv'
}

const media = ref<Media | null>(null)
const authStore = useAuthStore()

// Initialize auth on app start
onMounted(() => {
  authStore.initializeAuth()
  fetchMedia()
})

const fetchMedia = async () => {
  // Implement your media fetching logic
  media.value = {
    id: 123,
    title: 'Sample Media Title',
    type: 'anime',
  }
}
</script>

<style scoped>
.media-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.media-content {
  margin-bottom: 40px;
}

.comments-section {
  border-top: 1px solid #e5e7eb;
  padding-top: 20px;
}
</style>
```

## Main App Setup

```typescript
// src/main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { VueQueryPlugin } from '@tanstack/vue-query'
import App from './App.vue'
import './style.css'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.use(VueQueryPlugin)

app.mount('#app')
```

```vue
<!-- src/App.vue -->
<template>
  <div id="app">
    <header>
      <nav>
        <!-- Your navigation here -->
      </nav>
    </header>

    <main>
      <RouterView />
    </main>

    <!-- Toast notifications -->
    <div class="toast-container">
      <div
        v-for="toast in toasts.toasts"
        :key="toast.id"
        :class="['toast', toast.type]"
      >
        {{ toast.message }}
        <button @click="toasts.removeToast(toast.id)" class="toast-close">×</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useToast } from '@/composables/useToast'

const toasts = useToast()
</script>

<style>
/* Global styles */
.toast-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.toast {
  padding: 12px 16px;
  border-radius: 6px;
  color: white;
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 250px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.toast.success {
  background: #10b981;
}

.toast.error {
  background: #ef4444;
}

.toast.warning {
  background: #f59e0b;
}

.toast.info {
  background: #3b82f6;
}

.toast-close {
  background: transparent;
  border: none;
  color: white;
  font-size: 18px;
  cursor: pointer;
  margin-left: auto;
}
</style>
```

This Vue.js integration provides a complete comment system using modern Vue 3 patterns, TypeScript, and Pinia for state management.