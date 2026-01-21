# Integration Guide

This guide provides comprehensive instructions for integrating the Commentum system into your frontend application.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Authentication Flow](#authentication-flow)
- [API Integration](#api-integration)
- [Real-time Updates](#real-time-updates)
- [Error Handling](#error-handling)
- [Performance Optimization](#performance-optimization)
- [Security Best Practices](#security-best-practices)

## Prerequisites

### Required Dependencies

```bash
# For JavaScript/TypeScript projects
npm install @supabase/supabase-js
# or
yarn add @supabase/supabase-js

# For React projects
npm install @supabase/supabase-js react-query zustand
# or
yarn add @supabase/supabase-js react-query zustand
```

### Environment Configuration

Create a `.env.local` file in your project root:

```env
# Supabase Configuration
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key

# Commentum Configuration
NEXT_PUBLIC_COMMENTUM_API_URL=https://your-project.supabase.co/functions/v1
```

### Supabase Client Setup

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)

// Commentum API client
export const commentumAPI = {
  baseURL: process.env.NEXT_PUBLIC_COMMENTUM_API_URL,
  
  async request(endpoint: string, options: RequestInit = {}) {
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

## Quick Start

### 1. Basic Comment Component

```typescript
// components/CommentSystem.tsx
import { useState, useEffect } from 'react'
import { commentumAPI } from '@/lib/supabase'

interface Comment {
  id: number
  content: string
  user_id: number
  username: string
  avatar_url?: string
  created_at: string
  upvotes: number
  downvotes: number
  user_vote: number
  replies: Comment[]
}

interface CommentSystemProps {
  mediaId: number
  mediaType: 'anime' | 'manga' | 'movie' | 'tv'
  userId?: number
}

export const CommentSystem: React.FC<CommentSystemProps> = ({
  mediaId,
  mediaType,
  userId
}) => {
  const [comments, setComments] = useState<Comment[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  // Load comments
  useEffect(() => {
    loadComments()
  }, [mediaId])

  const loadComments = async () => {
    try {
      setLoading(true)
      const response = await commentumAPI.request(`/comments?media_id=${mediaId}`)
      setComments(response)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load comments')
    } finally {
      setLoading(false)
    }
  }

  if (loading) return <div>Loading comments...</div>
  if (error) return <div>Error: {error}</div>

  return (
    <div className="comment-system">
      <div className="comment-form">
        {userId ? (
          <CommentForm mediaId={mediaId} userId={userId} onCommentPosted={loadComments} />
        ) : (
          <p>Please log in to comment</p>
        )}
      </div>
      
      <div className="comments-list">
        {comments.map(comment => (
          <CommentItem 
            key={comment.id} 
            comment={comment} 
            userId={userId}
            onUpdate={loadComments}
          />
        ))}
      </div>
    </div>
  )
}
```

### 2. Comment Form Component

```typescript
// components/CommentForm.tsx
import { useState } from 'react'
import { commentumAPI } from '@/lib/supabase'

interface CommentFormProps {
  mediaId: number
  userId: number
  onCommentPosted: () => void
  parentId?: number
}

export const CommentForm: React.FC<CommentFormProps> = ({
  mediaId,
  userId,
  onCommentPosted,
  parentId
}) => {
  const [content, setContent] = useState('')
  const [submitting, setSubmitting] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!content.trim()) return

    setSubmitting(true)
    setError(null)

    try {
      await commentumAPI.request('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'create',
          user_id: userId,
          media_id: mediaId,
          content: content.trim(),
          parent_id: parentId
        })
      })

      setContent('')
      onCommentPosted()
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to post comment')
    } finally {
      setSubmitting(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="comment-form">
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        placeholder={parentId ? "Write a reply..." : "Write a comment..."}
        disabled={submitting}
        rows={3}
        maxLength={10000}
      />
      
      {error && <div className="error">{error}</div>}
      
      <div className="form-actions">
        <button 
          type="submit" 
          disabled={submitting || !content.trim()}
        >
          {submitting ? 'Posting...' : 'Post'}
        </button>
        
        {content.length > 1000 && (
          <span className="character-count">
            {content.length}/10000
          </span>
        )}
      </div>
    </form>
  )
}
```

## Authentication Flow

### 1. Identity Resolution Hook

```typescript
// hooks/useIdentity.ts
import { useState, useCallback } from 'react'
import { commentumAPI } from '@/lib/supabase'

interface UserIdentity {
  user_id: number
  username: string
  role: 'user' | 'moderator' | 'admin' | 'super_admin'
  banned: boolean
  shadow_banned: boolean
  muted_until: string | null
  existing_user: boolean
}

export const useIdentity = () => {
  const [user, setUser] = useState<UserIdentity | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const resolveIdentity = useCallback(async (
    clientType: 'anilist' | 'myanimelist' | 'simkl',
    clientUserId: string,
    username: string,
    avatarUrl?: string
  ) => {
    setLoading(true)
    setError(null)

    try {
      const response = await commentumAPI.request('/identity-resolve', {
        method: 'POST',
        body: JSON.stringify({
          client_type: clientType,
          client_user_id: clientUserId,
          username,
          avatar_url: avatarUrl
        })
      })

      setUser(response)
      return response
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Identity resolution failed'
      setError(errorMessage)
      throw new Error(errorMessage)
    } finally {
      setLoading(false)
    }
  }, [])

  const clearIdentity = useCallback(() => {
    setUser(null)
    setError(null)
  }, [])

  return {
    user,
    loading,
    error,
    resolveIdentity,
    clearIdentity
  }
}
```

### 2. Authentication Provider

```typescript
// contexts/AuthContext.tsx
import { createContext, useContext, ReactNode } from 'react'
import { useIdentity } from '@/hooks/useIdentity'

interface AuthContextType {
  user: UserIdentity | null
  login: (clientType: string, clientUserId: string, username: string, avatarUrl?: string) => Promise<void>
  logout: () => void
  loading: boolean
  error: string | null
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export const AuthProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const { user, loading, error, resolveIdentity, clearIdentity } = useIdentity()

  const login = async (
    clientType: string,
    clientUserId: string,
    username: string,
    avatarUrl?: string
  ) => {
    await resolveIdentity(
      clientType as 'anilist' | 'myanimelist' | 'simkl',
      clientUserId,
      username,
      avatarUrl
    )
  }

  const logout = () => {
    clearIdentity()
    // Clear any stored tokens/session data
    localStorage.removeItem('commentum_user')
  }

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      loading,
      error
    }}>
      {children}
    </AuthContext.Provider>
  )
}

export const useAuth = () => {
  const context = useContext(AuthContext)
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider')
  }
  return context
}
```

## API Integration

### 1. API Client with Error Handling

```typescript
// lib/apiClient.ts
import { commentumAPI } from '@/lib/supabase'

class CommentumAPIClient {
  private baseURL: string

  constructor() {
    this.baseURL = process.env.NEXT_PUBLIC_COMMENTUM_API_URL!
  }

  private async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`
    
    try {
      const response = await fetch(url, {
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
        ...options,
      })

      const data = await response.json()

      if (!response.ok) {
        throw new APIError(data.error || 'API request failed', response.status)
      }

      return data
    } catch (error) {
      if (error instanceof APIError) {
        throw error
      }
      throw new APIError('Network error occurred', 0)
    }
  }

  // Identity Resolution
  async resolveIdentity(params: {
    client_type: 'anilist' | 'myanimelist' | 'simkl'
    client_user_id: string
    username: string
    avatar_url?: string
  }) {
    return this.request('/identity-resolve', {
      method: 'POST',
      body: JSON.stringify(params),
    })
  }

  // Comments
  async createComment(params: {
    user_id: number
    media_id: number
    content: string
    parent_id?: number
  }) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...params
      }),
    })
  }

  async editComment(params: {
    user_id: number
    comment_id: number
    content: string
  }) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'edit',
        ...params
      }),
    })
  }

  async deleteComment(params: {
    user_id: number
    comment_id: number
  }) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'delete',
        ...params
      }),
    })
  }

  // Voting
  async vote(params: {
    user_id: number
    comment_id: number
    action: 'upvote' | 'downvote' | 'remove'
  }) {
    return this.request('/voting', {
      method: 'POST',
      body: JSON.stringify(params),
    })
  }

  // Reports
  async reportComment(params: {
    user_id: number
    comment_id: number
    reason: 'spam' | 'offensive' | 'harassment' | 'spoiler' | 'nsfw' | 'off_topic' | 'other'
    notes?: string
  }) {
    return this.request('/reports', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...params
      }),
    })
  }
}

class APIError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string
  ) {
    super(message)
    this.name = 'APIError'
  }
}

export const commentumClient = new CommentumAPIClient()
export { APIError }
```

### 2. React Query Integration

```typescript
// hooks/useComments.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumClient } from '@/lib/apiClient'

export const useComments = (mediaId: number) => {
  return useQuery({
    queryKey: ['comments', mediaId],
    queryFn: () => commentumClient.request(`/comments?media_id=${mediaId}`),
    staleTime: 5 * 60 * 1000, // 5 minutes
  })
}

export const useCreateComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: commentumClient.createComment,
    onSuccess: (_, variables) => {
      // Invalidate comments query for this media
      queryClient.invalidateQueries({ queryKey: ['comments', variables.media_id] })
    },
  })
}

export const useVote = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: commentumClient.vote,
    onSuccess: (_, variables) => {
      // Invalidate relevant queries
      queryClient.invalidateQueries({ queryKey: ['comments'] })
      queryClient.invalidateQueries({ queryKey: ['vote-counts', variables.comment_id] })
    },
  })
}
```

## Real-time Updates

### 1. WebSocket Integration

```typescript
// hooks/useRealtimeComments.ts
import { useEffect, useRef } from 'react'
import { useQueryClient } from '@tanstack/react-query'

interface RealtimeComment {
  type: 'created' | 'updated' | 'deleted'
  comment_id: number
  media_id: number
  data: any
}

export const useRealtimeComments = (mediaId: number) => {
  const queryClient = useQueryClient()
  const wsRef = useRef<WebSocket | null>(null)

  useEffect(() => {
    const wsUrl = `${process.env.NEXT_PUBLIC_COMMENTUM_API_URL?.replace('http', 'ws')}/comments?XTransformPort=3003`
    
    wsRef.current = new WebSocket(wsUrl)

    wsRef.current.onopen = () => {
      console.log('Connected to comment updates')
      // Subscribe to media-specific updates
      wsRef.current?.send(JSON.stringify({
        action: 'subscribe',
        media_id: mediaId
      }))
    }

    wsRef.current.onmessage = (event) => {
      try {
        const message: RealtimeComment = JSON.parse(event.data)
        
        if (message.media_id === mediaId) {
          switch (message.type) {
            case 'created':
              handleCommentCreated(message.data)
              break
            case 'updated':
              handleCommentUpdated(message.data)
              break
            case 'deleted':
              handleCommentDeleted(message.data)
              break
          }
        }
      } catch (error) {
        console.error('Failed to parse websocket message:', error)
      }
    }

    wsRef.current.onclose = () => {
      console.log('Disconnected from comment updates')
    }

    wsRef.current.onerror = (error) => {
      console.error('WebSocket error:', error)
    }

    return () => {
      wsRef.current?.close()
    }
  }, [mediaId, queryClient])

  const handleCommentCreated = (comment: any) => {
    queryClient.setQueryData(['comments', mediaId], (old: any) => {
      if (!old) return [comment]
      return [...old, comment]
    })
  }

  const handleCommentUpdated = (updatedComment: any) => {
    queryClient.setQueryData(['comments', mediaId], (old: any) => {
      if (!old) return old
      return old.map((comment: any) => 
        comment.id === updatedComment.id ? updatedComment : comment
      )
    })
  }

  const handleCommentDeleted = (deletedComment: any) => {
    queryClient.setQueryData(['comments', mediaId], (old: any) => {
      if (!old) return old
      return old.filter((comment: any) => comment.id !== deletedComment.id)
    })
  }
}
```

### 2. Optimistic Updates

```typescript
// hooks/useOptimisticComment.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumClient } from '@/lib/apiClient'

export const useOptimisticComment = (mediaId: number) => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: commentumClient.createComment,
    onMutate: async (newComment) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['comments', mediaId] })

      // Snapshot the previous value
      const previousComments = queryClient.getQueryData(['comments', mediaId])

      // Optimistically update to the new value
      queryClient.setQueryData(['comments', mediaId], (old: any) => {
        if (!old) return old
        const optimisticComment = {
          id: Date.now(), // Temporary ID
          ...newComment,
          created_at: new Date().toISOString(),
          upvotes: 0,
          downvotes: 0,
          user_vote: 0,
          replies: []
        }
        return [...old, optimisticComment]
      })

      return { previousComments }
    },
    onError: (err, newComment, context) => {
      // If the mutation fails, use the context returned from onMutate to roll back
      if (context?.previousComments) {
        queryClient.setQueryData(['comments', mediaId], context.previousComments)
      }
    },
    onSettled: () => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['comments', mediaId] })
    },
  })
}
```

## Error Handling

### 1. Error Boundary Component

```typescript
// components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
}

interface State {
  hasError: boolean
  error?: Error
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false }
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('CommentSystem Error:', error, errorInfo)
    // Send error to reporting service
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-fallback">
          <h3>Something went wrong</h3>
          <p>Please refresh the page and try again.</p>
        </div>
      )
    }

    return this.props.children
  }
}
```

### 2. Toast Notifications

```typescript
// hooks/useToast.ts
import { useState } from 'react'

interface Toast {
  id: string
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
  duration?: number
}

export const useToast = () => {
  const [toasts, setToasts] = useState<Toast[]>([])

  const addToast = (message: string, type: Toast['type'] = 'info', duration = 5000) => {
    const id = Date.now().toString()
    const toast: Toast = { id, message, type, duration }
    
    setToasts(prev => [...prev, toast])

    if (duration > 0) {
      setTimeout(() => {
        removeToast(id)
      }, duration)
    }

    return id
  }

  const removeToast = (id: string) => {
    setToasts(prev => prev.filter(toast => toast.id !== id))
  }

  const success = (message: string) => addToast(message, 'success')
  const error = (message: string) => addToast(message, 'error')
  const warning = (message: string) => addToast(message, 'warning')
  const info = (message: string) => addToast(message, 'info')

  return {
    toasts,
    addToast,
    removeToast,
    success,
    error,
    warning,
    info
  }
}
```

## Performance Optimization

### 1. Virtual Scrolling

```typescript
// components/VirtualCommentList.tsx
import { FixedSizeList as List } from 'react-window'

interface VirtualCommentListProps {
  comments: Comment[]
  userId?: number
  onCommentUpdate: () => void
}

const CommentRow = ({ index, style, data }: any) => {
  const { comments, userId, onCommentUpdate } = data
  const comment = comments[index]

  return (
    <div style={style}>
      <CommentItem 
        comment={comment} 
        userId={userId}
        onUpdate={onCommentUpdate}
      />
    </div>
  )
}

export const VirtualCommentList: React.FC<VirtualCommentListProps> = ({
  comments,
  userId,
  onCommentUpdate
}) => {
  return (
    <List
      height={600}
      itemCount={comments.length}
      itemSize={200}
      itemData={{ comments, userId, onCommentUpdate }}
    >
      {CommentRow}
    </List>
  )
}
```

### 2. Memoization

```typescript
// components/CommentItem.tsx
import { memo, useMemo } from 'react'

interface CommentItemProps {
  comment: Comment
  userId?: number
  onUpdate: () => void
}

export const CommentItem = memo<CommentItemProps>(({ comment, userId, onUpdate }) => {
  const formattedDate = useMemo(() => {
    return new Date(comment.created_at).toLocaleDateString()
  }, [comment.created_at])

  const canEdit = useMemo(() => {
    return userId && userId === comment.user_id && !comment.deleted
  }, [userId, comment.user_id, comment.deleted])

  return (
    <div className="comment-item">
      <div className="comment-header">
        <img src={comment.avatar_url || '/default-avatar.png'} alt={comment.username} />
        <div>
          <h4>{comment.username}</h4>
          <time>{formattedDate}</time>
        </div>
      </div>
      
      <div className="comment-content">
        {comment.deleted ? (
          <em>This comment has been deleted</em>
        ) : (
          <p>{comment.content}</p>
        )}
      </div>

      <div className="comment-actions">
        <VotingButtons commentId={comment.id} userId={userId} />
        
        {canEdit && (
          <button onClick={() => {/* Edit logic */}}>
            Edit
          </button>
        )}
        
        <ReportButton commentId={comment.id} userId={userId} />
      </div>
    </div>
  )
})

CommentItem.displayName = 'CommentItem'
```

## Security Best Practices

### 1. Content Sanitization

```typescript
// utils/sanitize.ts
import DOMPurify from 'dompurify'

export const sanitizeContent = (content: string): string => {
  return DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'code', 'pre'],
    ALLOWED_ATTR: ['href', 'title'],
    ALLOW_DATA_ATTR: false
  })
}

export const truncateContent = (content: string, maxLength: number): string => {
  if (content.length <= maxLength) return content
  return content.substring(0, maxLength) + '...'
}
```

### 2. Rate Limiting Client-Side

```typescript
// hooks/useRateLimit.ts
import { useState, useCallback } from 'react'

interface RateLimitState {
  count: number
  resetTime: number
}

export const useRateLimit = (maxRequests: number, windowMs: number) => {
  const [rateLimit, setRateLimit] = useState<RateLimitState>({
    count: 0,
    resetTime: Date.now() + windowMs
  })

  const checkRateLimit = useCallback(() => {
    const now = Date.now()
    
    if (now > rateLimit.resetTime) {
      // Reset window
      setRateLimit({
        count: 1,
        resetTime: now + windowMs
      })
      return true
    }

    if (rateLimit.count >= maxRequests) {
      return false
    }

    setRateLimit(prev => ({
      ...prev,
      count: prev.count + 1
    }))

    return true
  }, [rateLimit, maxRequests, windowMs])

  const timeUntilReset = Math.max(0, rateLimit.resetTime - Date.now())

  return {
    canProceed: checkRateLimit(),
    remainingRequests: Math.max(0, maxRequests - rateLimit.count),
    timeUntilReset
  }
}
```

### 3. XSS Prevention

```typescript
// components/SafeContent.tsx
import { sanitizeContent } from '@/utils/sanitize'

interface SafeContentProps {
  content: string
  className?: string
}

export const SafeContent: React.FC<SafeContentProps> = ({ content, className }) => {
  const sanitizedContent = sanitizeContent(content)
  
  return (
    <div 
      className={className}
      dangerouslySetInnerHTML={{ __html: sanitizedContent }}
    />
  )
}
```

## Complete Integration Example

```typescript
// pages/media/[id].tsx
import { GetServerSideProps } from 'next'
import { useState } from 'react'
import { CommentSystem } from '@/components/CommentSystem'
import { AuthProvider } from '@/contexts/AuthContext'
import { ErrorBoundary } from '@/components/ErrorBoundary'

interface MediaPageProps {
  media: {
    id: number
    title: string
    type: 'anime' | 'manga' | 'movie' | 'tv'
  }
}

export default function MediaPage({ media }: MediaPageProps) {
  const [currentUser, setCurrentUser] = useState<any>(null)

  return (
    <AuthProvider>
      <ErrorBoundary>
        <div className="media-page">
          <header>
            <h1>{media.title}</h1>
            <p>Type: {media.type}</p>
          </header>
          
          <main>
            <section className="media-info">
              {/* Media content here */}
            </section>
            
            <section className="comments-section">
              <h2>Comments</h2>
              <CommentSystem 
                mediaId={media.id}
                mediaType={media.type}
                userId={currentUser?.user_id}
              />
            </section>
          </main>
        </div>
      </ErrorBoundary>
    </AuthProvider>
  )
}

export const getServerSideProps: GetServerSideProps = async (context) => {
  const { id } = context.params
  
  // Fetch media data
  const media = await fetchMediaById(id as string)
  
  if (!media) {
    return { notFound: true }
  }

  return {
    props: {
      media
    }
  }
}

async function fetchMediaById(id: string) {
  // Implement media fetching logic
  // This would typically call your media API
  return {
    id: parseInt(id),
    title: 'Sample Media',
    type: 'anime'
  }
}
```

This integration guide provides a comprehensive foundation for implementing the Commentum system in your frontend application. The examples use React with TypeScript, but the concepts can be adapted to other frameworks and languages.