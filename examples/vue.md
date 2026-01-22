# 🔒 Vue.js Integration Example - Session-Based Authentication

This example demonstrates how to integrate Commentum into a Vue.js 3 application using TypeScript and the Composition API with **secure session-based authentication**.

## 🚨 Security Notice

This example uses **session-based authentication** to prevent identity spoofing attacks. The `user_id` is never passed from the client - it's extracted from the session token on the server.

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

### 3. Secure API Client with Session Management

```typescript
// src/lib/commentum-client.ts
export class CommentumClient {
  private baseURL: string
  private sessionToken: string | null = null

  constructor(baseURL: string) {
    this.baseURL = baseURL
    this.loadSessionToken()
  }

  private loadSessionToken() {
    if (typeof window !== 'undefined') {
      this.sessionToken = localStorage.getItem('commentum_session_token')
    }
  }

  setSessionToken(token: string) {
    this.sessionToken = token
    if (typeof window !== 'undefined') {
      localStorage.setItem('commentum_session_token', token)
    }
  }

  clearSessionToken() {
    this.sessionToken = null
    if (typeof window !== 'undefined') {
      localStorage.removeItem('commentum_session_token')
    }
  }

  private async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
    if (!this.sessionToken) {
      throw new Error('Not authenticated - Please log in first')
    }

    const url = `${this.baseURL}${endpoint}`
    
    const response = await fetch(url, {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.sessionToken}`,
        ...options.headers,
      },
      ...options,
    })
    
    if (response.status === 401) {
      this.clearSessionToken()
      throw new Error('Session expired - Please log in again')
    }
    
    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.error || 'API request failed')
    }
    
    return response.json()
  }

  // Identity Resolution
  async resolveIdentity(clientType: string, token: string) {
    const response = await fetch(`${this.baseURL}/identity-resolve`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${import.meta.env.VITE_SUPABASE_ANON_KEY}`,
      },
      body: JSON.stringify({
        client_type: clientType,
        token: token,
      }),
    })

    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.error || 'Identity resolution failed')
    }

    const data = await response.json()
    this.setSessionToken(data.session_token)
    return data
  }

  // Comments API
  async getComments(mediaId: number) {
    return this.request(`/comments?media_id=${mediaId}`)
  }

  async createComment(data: {
    media_id?: number
    media_info?: {
      external_id: string
      media_type: 'anime' | 'manga' | 'movie' | 'tv' | 'other'
      title: string
      year?: number
      poster_url?: string
    }
    content: string
    parent_id?: number
  }) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...data,
      }),
    })
  }

  async editComment(commentId: number, content: string) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'edit',
        comment_id: commentId,
        content,
      }),
    })
  }

  async deleteComment(commentId: number) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'delete',
        comment_id: commentId,
      }),
    })
  }

  // Voting API
  async vote(commentId: number, action: 'upvote' | 'downvote' | 'remove') {
    return this.request('/voting', {
      method: 'POST',
      body: JSON.stringify({
        action,
        comment_id: commentId,
      }),
    })
  }

  // Reports API
  async createReport(commentId: number, reason: string, notes?: string) {
    return this.request('/reports', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        comment_id: commentId,
        reason,
        notes,
      }),
    })
  }
}

export const commentumClient = new CommentumClient(
  import.meta.env.VITE_COMMENTUM_API_URL
)
```

## Secure Stores and Composables

### 1. Auth Store with Session Management

```typescript
// src/stores/authStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { commentumClient } from '@/lib/commentum-client'

export interface User {
  id: number
  username: string
  avatar_url?: string
  client_type: 'anilist' | 'mal' | 'simkl'
}

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<User | null>(null)
  const sessionToken = ref<string | null>(null)
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
  const login = async (clientType: string, token: string) => {
    loading.value = true
    error.value = null

    try {
      const data = await commentumClient.resolveIdentity(clientType, token)
      
      if (data.user.banned) {
        throw new Error('Account is banned')
      }

      if (data.user.muted_until && new Date(data.user.muted_until) > new Date()) {
        throw new Error(`Account muted until ${data.user.muted_until}`)
      }

      user.value = data.user
      sessionToken.value = data.session_token
      
      return data
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Authentication failed'
      throw err
    } finally {
      loading.value = false
    }
  }

  const logout = () => {
    user.value = null
    sessionToken.value = null
    commentumClient.clearSessionToken()
  }

  const initializeAuth = () => {
    const token = localStorage.getItem('commentum_session_token')
    if (token) {
      sessionToken.value = token
      commentumClient.setSessionToken(token)
      // You might want to validate the token here
    }
  }

  const clearError = () => {
    error.value = null
  }

  return {
    // State
    user: readonly(user),
    sessionToken: readonly(sessionToken),
    loading: readonly(loading),
    error: readonly(error),
    
    // Getters
    isAuthenticated,
    isModerator,
    isAdmin,
    
    // Actions
    login,
    logout,
    initializeAuth,
    clearError,
  }
})
```

### 2. Comment Store with Session Auth

```typescript
// src/stores/commentStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { commentumClient } from '@/lib/commentum-client'
import type { Comment } from '@/types/comment'

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
      const response = await commentumClient.getComments(mediaId)
      comments.value = response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to fetch comments'
      throw err
    } finally {
      loading.value = false
    }
  }

  const createComment = async (data: {
    media_id?: number
    media_info?: {
      external_id: string
      media_type: 'anime' | 'manga' | 'movie' | 'tv' | 'other'
      title: string
      year?: number
      poster_url?: string
    }
    content: string
    parent_id?: number
  }) => {
    try {
      const response = await commentumClient.createComment(data)
      comments.value.push(response)
      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to create comment'
      throw err
    }
  }

  const updateComment = async (commentId: number, content: string) => {
    try {
      const response = await commentumClient.editComment(commentId, content)

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

  const deleteComment = async (commentId: number) => {
    try {
      const response = await commentumClient.deleteComment(commentId)

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

### 3. Voting Composable with Session Auth

```typescript
// src/composables/useVoting.ts
import { ref, computed } from 'vue'
import { commentumClient } from '@/lib/commentum-client'
import type { VoteData } from '@/types/voting'

export function useVoting(commentId: number, initialVotes: VoteData) {
  // State
  const votes = ref<VoteData>({ ...initialVotes })
  const loading = ref(false)

  // Getters
  const totalVotes = computed(() => votes.value.upvotes - votes.value.downvotes)
  const hasVoted = computed(() => votes.value.userVote !== 0)

  // Actions
  const vote = async (action: 'upvote' | 'downvote' | 'remove') => {
    if (loading.value) return

    loading.value = true

    try {
      await commentumClient.vote(commentId, action)

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

## Secure Components

### 1. Login Component

```vue
<!-- src/components/LoginForm.vue -->
<template>
  <div class="login-form">
    <h3>Login to Comment</h3>
    <form @submit.prevent="handleSubmit">
      <div class="form-group">
        <label>Service:</label>
        <select
          v-model="clientType"
          :disabled="loading"
        >
          <option value="anilist">AniList</option>
          <option value="mal">MyAnimeList</option>
          <option value="simkl">SIMKL</option>
        </select>
      </div>

      <div class="form-group">
        <label>Access Token:</label>
        <input
          v-model="token"
          type="password"
          placeholder="Enter your access token"
          :disabled="loading"
          required
        />
        <small>
          Get your token from your service's developer settings
        </small>
      </div>

      <div v-if="error" class="error">{{ error }}</div>

      <button 
        type="submit" 
        :disabled="loading || !token.trim()"
      >
        {{ loading ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useAuthStore } from '@/stores/authStore'

interface Props {
  onSuccess?: () => void
}

const props = defineProps<Props>()

const authStore = useAuthStore()
const clientType = ref<'anilist' | 'mal' | 'simkl'>('anilist')
const token = ref('')
const loading = ref(false)
const error = ref<string | null>(null)

const handleSubmit = async () => {
  if (!token.value.trim()) return

  loading.value = true
  error.value = null

  try {
    await authStore.login(clientType.value, token.value.trim())
    props.onSuccess?.()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Login failed'
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.login-form {
  max-width: 400px;
  margin: 0 auto;
  padding: 24px;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  background: white;
}

.form-group {
  margin-bottom: 16px;
}

.form-group label {
  display: block;
  margin-bottom: 4px;
  font-weight: 500;
}

.form-group input,
.form-group select {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 14px;
}

.form-group small {
  display: block;
  margin-top: 4px;
  color: #6b7280;
  font-size: 12px;
}

.error {
  color: #ef4444;
  margin-bottom: 16px;
  padding: 8px;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 4px;
}

button {
  width: 100%;
  padding: 10px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 4px;
  font-weight: 500;
  cursor: pointer;
}

button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}
</style>
```

### 2. Comment System Component

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
    <div v-else class="login-prompt">
      <p>Please log in to comment</p>
      <LoginForm @success="handleLoginSuccess" />
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
import LoginForm from './LoginForm.vue'

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

const handleLoginSuccess = () => {
  // Refresh comments after login
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

.login-prompt {
  margin-bottom: 24px;
  padding: 20px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  text-align: center;
}

.login-prompt p {
  margin-bottom: 16px;
  color: #6b7280;
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

### 3. Secure Comment Form Component

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
import { useToast } from '@/composables/useToast'

interface Props {
  parentId?: number
  mediaId?: number
  mediaInfo?: {
    external_id: string
    media_type: 'anime' | 'manga' | 'movie' | 'tv' | 'other'
    title: string
    year?: number
    poster_url?: string
  }
  onSuccess?: () => void
  onCancel?: () => void
}

const props = defineProps<Props>()

const commentStore = useCommentStore()
const { success, error } = useToast()

const formData = ref({
  content: '',
})

const submitting = ref(false)
const errors = ref<Record<string, string>>({})

const placeholder = computed(() => 
  props.parentId ? 'Write a reply...' : 'Share your thoughts...'
)

const buttonText = computed(() => 
  props.parentId ? 'Reply' : 'Post Comment'
)

const characterCount = computed(() => formData.value.content.length)

const validateForm = () => {
  const newErrors: Record<string, string> = {}

  if (!formData.value.content.trim()) {
    newErrors.content = 'Comment cannot be empty'
  } else if (formData.value.content.length > 10000) {
    newErrors.content = 'Comment cannot exceed 10,000 characters'
  }

  errors.value = newErrors
  return Object.keys(newErrors).length === 0
}

const handleSubmit = async () => {
  if (!validateForm()) return

  submitting.value = true

  try {
    const commentData: any = {
      content: formData.value.content.trim(),
      parent_id: props.parentId,
    }

    // Use media_id if provided, otherwise use media_info for auto-creation
    if (props.mediaId) {
      commentData.media_id = props.mediaId
    } else if (props.mediaInfo) {
      commentData.media_info = props.mediaInfo
    }

    await commentStore.createComment(commentData)

    formData.value.content = ''
    success('Comment posted successfully!')
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
  margin-bottom: 16px;
}

.form-group {
  margin-bottom: 16px;
}

.comment-textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  line-height: 1.5;
  resize: vertical;
  min-height: 80px;
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
  margin-top: 4px;
  color: #ef4444;
  font-size: 12px;
}

.form-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.form-left {
  display: flex;
  align-items: center;
}

.character-count {
  font-size: 12px;
  color: #6b7280;
}

.character-count.warning {
  color: #f59e0b;
}

.form-right {
  display: flex;
  gap: 8px;
}

.cancel-button,
.submit-button {
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.cancel-button {
  background: #f3f4f6;
  color: #374151;
  border: 1px solid #d1d5db;
}

.cancel-button:hover:not(:disabled) {
  background: #e5e7eb;
}

.submit-button {
  background: #3b82f6;
  color: white;
  border: 1px solid #3b82f6;
}

.submit-button:hover:not(:disabled) {
  background: #2563eb;
}

.cancel-button:disabled,
.submit-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
```

## Usage Example

### 1. App Integration

```vue
<!-- src/App.vue -->
<template>
  <div id="app">
    <router-view />
  </div>
</template>

<script setup lang="ts">
import { onMounted } from 'vue'
import { useAuthStore } from '@/stores/authStore'

const authStore = useAuthStore()

onMounted(() => {
  authStore.initializeAuth()
})
</script>
```

### 2. Page Component with Comments

```vue
<!-- src/views/AnimeView.vue -->
<template>
  <div class="anime-page">
    <!-- Your anime content -->
    <div class="anime-info">
      <h1>{{ anime.title }}</h1>
      <p>{{ anime.description }}</p>
    </div>

    <!-- Comments section -->
    <div class="comments-section">
      <CommentSystem :media-id="anime.id" media-type="anime" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import CommentSystem from '@/components/CommentSystem.vue'

interface Anime {
  id: number
  title: string
  description: string
}

const anime = ref<Anime>({
  id: 123,
  title: 'Example Anime',
  description: 'This is an example anime description.',
})
</script>

<style scoped>
.anime-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.anime-info {
  margin-bottom: 40px;
}

.comments-section {
  border-top: 1px solid #e2e8f0;
  padding-top: 20px;
}
</style>
```

### 3. With Auto Media Creation

```vue
<!-- src/components/MediaComments.vue -->
<template>
  <div class="media-comments">
    <CommentSystem 
      :media-id="undefined" 
      :media-info="mediaInfo"
      media-type="anime"
    />
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import CommentSystem from '@/components/CommentSystem.vue'

interface Props {
  externalId: string
  title: string
  year?: number
  posterUrl?: string
}

const props = defineProps<Props>()

const mediaInfo = computed(() => ({
  external_id: props.externalId,
  media_type: 'anime' as const,
  title: props.title,
  year: props.year,
  poster_url: props.posterUrl,
}))
</script>
```

## 🔒 Security Features

### Session-Based Authentication
- **No user_id parameters**: User identity is extracted from session tokens
- **Automatic token management**: Tokens are stored and refreshed automatically
- **Session validation**: All API calls validate the session token
- **Automatic logout**: Sessions are cleared on expiration

### Zero-Trust Architecture
- **Server-side validation**: All user data is validated on the server
- **No client trust**: No client-provided user data is trusted
- **Real-time verification**: Provider tokens are verified with real APIs
- **Audit logging**: All actions are logged for security

### Error Handling
- **Session expiration**: Automatic handling of expired sessions
- **Network errors**: Graceful handling of connection issues
- **Permission errors**: Clear error messages for permission issues

## 🚨 Breaking Changes from Old Version

### Before (Vulnerable)
```typescript
// ❌ OLD - Client could send any user_id
await commentumAPI.request('/comments', {
  method: 'POST',
  body: JSON.stringify({
    action: 'create',
    user_id: 123,  // Could be faked!
    media_id: 456,
    content: 'Comment text'
  })
})
```

### After (Secure)
```typescript
// ✅ NEW - User ID extracted from session
await commentumClient.createComment({
  media_id: 456,
  content: 'Comment text'  // No user_id needed!
})
```

This secure Vue.js implementation ensures that comment operations can only be performed by authenticated users while preventing identity spoofing attacks.