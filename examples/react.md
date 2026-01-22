# 🔒 React Integration Example - Session-Based Authentication

This example demonstrates how to integrate Commentum into a React application using TypeScript and modern React patterns with **secure session-based authentication**.

## 🚨 Security Notice

This example uses **session-based authentication** to prevent identity spoofing attacks. The `user_id` is never passed from the client - it's extracted from the session token on the server.

## Project Setup

### 1. Install Dependencies

```bash
npm install @supabase/supabase-js @tanstack/react-query zustand react-hook-form @hookform/resolvers zod date-fns
# or
yarn add @supabase/supabase-js @tanstack/react-query zustand react-hook-form @hookform/resolvers zod date-fns
```

### 2. Environment Configuration

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
NEXT_PUBLIC_COMMENTUM_API_URL=https://your-project.supabase.co/functions/v1
```

### 3. Secure API Client with Session Management

```typescript
// lib/commentum-client.ts
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
        'Authorization': `Bearer ${process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY}`,
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
  process.env.NEXT_PUBLIC_COMMENTUM_API_URL!
)
```

## Authentication Context

### 1. Authentication Provider

```typescript
// contexts/AuthProvider.tsx
import React, { createContext, useContext, ReactNode, useState, useEffect } from 'react'
import { commentumClient } from '../lib/commentum-client'

interface User {
  id: number
  username: string
  avatar_url?: string
  client_type: 'anilist' | 'mal' | 'simkl'
}

interface AuthContextType {
  user: User | null
  sessionToken: string | null
  isAuthenticated: boolean
  login: (clientType: string, token: string) => Promise<void>
  logout: () => void
  loading: boolean
}

const AuthContext = createContext<AuthContextType | null>(null)

export const useAuth = () => {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}

interface AuthProviderProps {
  children: ReactNode
}

export const AuthProvider: React.FC<AuthProviderProps> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null)
  const [sessionToken, setSessionToken] = useState<string | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Check for existing session on mount
    const token = localStorage.getItem('commentum_session_token')
    if (token) {
      setSessionToken(token)
      commentumClient.setSessionToken(token)
      // You might want to validate the token here
    }
    setLoading(false)
  }, [])

  const login = async (clientType: string, token: string) => {
    setLoading(true)
    try {
      const data = await commentumClient.resolveIdentity(clientType, token)
      setUser(data.user)
      setSessionToken(data.session_token)
    } catch (error) {
      throw error
    } finally {
      setLoading(false)
    }
  }

  const logout = () => {
    setUser(null)
    setSessionToken(null)
    commentumClient.clearSessionToken()
  }

  const value: AuthContextType = {
    user,
    sessionToken,
    isAuthenticated: !!user,
    login,
    logout,
    loading,
  }

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  )
}
```

### 2. Login Component

```typescript
// components/LoginForm.tsx
import React, { useState } from 'react'
import { useAuth } from '../contexts/AuthProvider'

interface LoginFormProps {
  onSuccess?: () => void
}

export const LoginForm: React.FC<LoginFormProps> = ({ onSuccess }) => {
  const { login } = useAuth()
  const [clientType, setClientType] = useState<'anilist' | 'mal' | 'simkl'>('anilist')
  const [token, setToken] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!token.trim()) return

    setLoading(true)
    setError(null)

    try {
      await login(clientType, token.trim())
      onSuccess?.()
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="login-form">
      <h3>Login to Comment</h3>
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Service:</label>
          <select
            value={clientType}
            onChange={(e) => setClientType(e.target.value as any)}
            disabled={loading}
          >
            <option value="anilist">AniList</option>
            <option value="mal">MyAnimeList</option>
            <option value="simkl">SIMKL</option>
          </select>
        </div>

        <div className="form-group">
          <label>Access Token:</label>
          <input
            type="password"
            value={token}
            onChange={(e) => setToken(e.target.value)}
            placeholder="Enter your access token"
            disabled={loading}
            required
          />
          <small>
            Get your token from your service's developer settings
          </small>
        </div>

        {error && <div className="error">{error}</div>}

        <button type="submit" disabled={loading || !token.trim()}>
          {loading ? 'Logging in...' : 'Login'}
        </button>
      </form>
    </div>
  )
}
```

## Core Components

### 1. Comment System Provider

```typescript
// contexts/CommentSystemProvider.tsx
import React, { createContext, useContext, ReactNode } from 'react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { CommentSystem } from './CommentSystem'

interface CommentSystemContextType {
  mediaId: number
  mediaType: 'anime' | 'manga' | 'movie' | 'tv'
}

const CommentSystemContext = createContext<CommentSystemContextType | null>(null)

export const useCommentSystem = () => {
  const context = useContext(CommentSystemContext)
  if (!context) {
    throw new Error('useCommentSystem must be used within CommentSystemProvider')
  }
  return context
}

interface CommentSystemProviderProps {
  children: ReactNode
  mediaId: number
  mediaType: 'anime' | 'manga' | 'movie' | 'tv'
}

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      retry: 1,
    },
  },
})

export const CommentSystemProvider: React.FC<CommentSystemProviderProps> = ({
  children,
  mediaId,
  mediaType,
}) => {
  const contextValue: CommentSystemContextType = {
    mediaId,
    mediaType,
  }

  return (
    <CommentSystemContext.Provider value={contextValue}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </CommentSystemContext.Provider>
  )
}
```

### 2. Main Comment System Component

```typescript
// components/CommentSystem.tsx
import React from 'react'
import { useComments } from '../hooks/useComments'
import { CommentForm } from './CommentForm'
import { CommentList } from './CommentList'
import { CommentSort } from './CommentSort'
import { useAuth } from '../contexts/AuthProvider'
import { useCommentSystem } from '../contexts/CommentSystemProvider'

export const CommentSystem: React.FC = () => {
  const { mediaId } = useCommentSystem()
  const { isAuthenticated } = useAuth()
  const { 
    data: comments, 
    isLoading, 
    error, 
    refetch 
  } = useComments(mediaId)

  if (isLoading) {
    return (
      <div className="comment-system loading">
        <div className="loading-spinner">Loading comments...</div>
      </div>
    )
  }

  if (error) {
    return (
      <div className="comment-system error">
        <div className="error-message">
          Failed to load comments: {error.message}
        </div>
        <button onClick={() => refetch()} className="retry-button">
          Retry
        </button>
      </div>
    )
  }

  return (
    <div className="comment-system">
      <div className="comment-system-header">
        <h3>Comments ({comments?.length || 0})</h3>
        <CommentSort />
      </div>

      {isAuthenticated ? (
        <div className="comment-form-section">
          <CommentForm onSuccess={refetch} />
        </div>
      ) : (
        <div className="login-prompt">
          <p>Please log in to comment</p>
        </div>
      )}

      <div className="comments-section">
        <CommentList comments={comments || []} onUpdate={refetch} />
      </div>
    </div>
  )
}
```

### 3. Secure Comment Form Component

```typescript
// components/CommentForm.tsx
import React from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { useCreateComment } from '../hooks/useComments'
import { useCommentSystem } from '../contexts/CommentSystemProvider'
import { useToast } from '../hooks/useToast'

const commentSchema = z.object({
  content: z
    .string()
    .min(1, 'Comment cannot be empty')
    .max(10000, 'Comment cannot exceed 10,000 characters'),
})

type CommentFormData = z.infer<typeof commentSchema>

interface CommentFormProps {
  parentId?: number
  onSuccess?: () => void
  onCancel?: () => void
}

export const CommentForm: React.FC<CommentFormProps> = ({
  parentId,
  onSuccess,
  onCancel,
}) => {
  const { mediaId } = useCommentSystem()
  const { success, error } = useToast()
  const createComment = useCreateComment()

  const {
    register,
    handleSubmit,
    reset,
    formState: { errors, isSubmitting },
    watch,
  } = useForm<CommentFormData>({
    resolver: zodResolver(commentSchema),
  })

  const content = watch('content', '')

  const onSubmit = async (data: CommentFormData) => {
    try {
      await createComment.mutateAsync({
        media_id: mediaId,
        content: data.content.trim(),
        parent_id: parentId,
        // No user_id needed - extracted from session!
      })

      reset()
      success('Comment posted successfully!')
      onSuccess?.()
    } catch (err) {
      error(err instanceof Error ? err.message : 'Failed to post comment')
    }
  }

  const characterCount = content.length
  const maxCharacters = 10000

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="comment-form">
      <div className="form-group">
        <textarea
          {...register('content')}
          placeholder={parentId ? 'Write a reply...' : 'Share your thoughts...'}
          className={`comment-textarea ${errors.content ? 'error' : ''}`}
          rows={4}
          disabled={isSubmitting}
        />
        {errors.content && (
          <span className="error-message">{errors.content.message}</span>
        )}
      </div>

      <div className="form-actions">
        <div className="form-left">
          <span className={`character-count ${characterCount > maxCharacters * 0.9 ? 'warning' : ''}`}>
            {characterCount}/{maxCharacters}
          </span>
        </div>

        <div className="form-right">
          {onCancel && (
            <button
              type="button"
              onClick={onCancel}
              disabled={isSubmitting}
              className="cancel-button"
            >
              Cancel
            </button>
          )}
          
          <button
            type="submit"
            disabled={isSubmitting || !content.trim()}
            className="submit-button"
          >
            {isSubmitting ? 'Posting...' : parentId ? 'Reply' : 'Post Comment'}
          </button>
        </div>
      </div>
    </form>
  )
}
```

## Secure Hooks

### 1. Comments Hook with Session Auth

```typescript
// hooks/useComments.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumClient } from '../lib/commentum-client'
import { Comment } from '../types/comment'

export const useComments = (mediaId: number) => {
  return useQuery({
    queryKey: ['comments', mediaId],
    queryFn: () => commentumClient.getComments(mediaId),
    enabled: !!mediaId,
  })
}

export const useCreateComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: {
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
    }) => commentumClient.createComment(data),
    onSuccess: (_, variables) => {
      // Invalidate and refetch comments
      if (variables.media_id) {
        queryClient.invalidateQueries({ queryKey: ['comments', variables.media_id] })
      } else {
        // If using media_info, we don't know the media_id yet
        queryClient.invalidateQueries({ queryKey: ['comments'] })
      }
    },
  })
}

export const useEditComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ commentId, content }: {
      commentId: number
      content: string
    }) => commentumClient.editComment(commentId, content),
    onSuccess: () => {
      // Invalidate all comments queries to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['comments'] })
    },
  })
}

export const useDeleteComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (commentId: number) => commentumClient.deleteComment(commentId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['comments'] })
    },
  })
}
```

### 2. Voting Hook with Session Auth

```typescript
// hooks/useVoting.ts
import { useState, useCallback } from 'react'
import { commentumClient } from '../lib/commentum-client'

interface UseVotingResult {
  upvote: () => Promise<void>
  downvote: () => Promise<void>
  removeVote: () => Promise<void>
  votes: { upvotes: number; downvotes: number; userVote: number }
  loading: boolean
  error: string | null
}

export const useVoting = (
  commentId: number,
  initialVotes: { upvotes: number; downvotes: number; userVote: number }
): UseVotingResult => {
  const [votes, setVotes] = useState(initialVotes)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const vote = useCallback(async (action: 'upvote' | 'downvote' | 'remove') => {
    setLoading(true)
    setError(null)

    try {
      await commentumClient.vote(commentId, action)

      // Update local state based on the action
      setVotes(prev => {
        const newVotes = { ...prev }
        
        if (action === 'remove') {
          if (prev.userVote === 1) newVotes.upvotes--
          if (prev.userVote === -1) newVotes.downvotes--
          newVotes.userVote = 0
        } else if (action === 'upvote') {
          if (prev.userVote === -1) newVotes.downvotes--
          if (prev.userVote !== 1) newVotes.upvotes++
          newVotes.userVote = 1
        } else if (action === 'downvote') {
          if (prev.userVote === 1) newVotes.upvotes--
          if (prev.userVote !== -1) newVotes.downvotes++
          newVotes.userVote = -1
        }
        
        return newVotes
      })
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Voting failed')
    } finally {
      setLoading(false)
    }
  }, [commentId])

  return {
    upvote: () => vote('upvote'),
    downvote: () => vote('downvote'),
    removeVote: () => vote('remove'),
    votes,
    loading,
    error
  }
}
```

## Usage Example

### 1. App Integration

```typescript
// pages/anime/[id].tsx
import React from 'react'
import { AuthProvider } from '../contexts/AuthProvider'
import { CommentSystemProvider } from '../contexts/CommentSystemProvider'
import { CommentSystem } from '../components/CommentSystem'

export default function AnimePage({ anime }) {
  return (
    <AuthProvider>
      <CommentSystemProvider 
        mediaId={anime.id} 
        mediaType="anime"
      >
        <div className="anime-page">
          {/* Your anime content */}
          
          <div className="comments-section">
            <CommentSystem />
          </div>
        </div>
      </CommentSystemProvider>
    </AuthProvider>
  )
}
```

### 2. With Auto Media Creation

```typescript
// components/MediaComments.tsx
import React from 'react'
import { AuthProvider } from '../contexts/AuthProvider'
import { CommentSystemProvider } from '../contexts/CommentSystemProvider'
import { CommentSystem } from './CommentSystem'

interface MediaCommentsProps {
  mediaInfo: {
    external_id: string
    media_type: 'anime' | 'manga' | 'movie' | 'tv' | 'other'
    title: string
    year?: number
    poster_url?: string
  }
}

export const MediaComments: React.FC<MediaCommentsProps> = ({ mediaInfo }) => {
  return (
    <AuthProvider>
      <CommentSystemProvider 
        mediaId={undefined} // No media_id - will be created automatically
        mediaType={mediaInfo.media_type}
      >
        <CommentSystem />
      </CommentSystemProvider>
    </AuthProvider>
  )
}
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

This secure implementation ensures that comment operations can only be performed by authenticated users while preventing identity spoofing attacks.