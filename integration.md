# 🔒 Integration Guide - Session-Based Authentication

This guide provides comprehensive instructions for integrating the Commentum system into your frontend application using **secure session-based authentication**.

## 🚨 **SECURITY UPDATE**

**Previous integration methods had a critical identity spoofing vulnerability. This guide uses the new secure session-based authentication system.**

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start (Secure)](#quick-start-secure)
- [🔒 Session Authentication Flow](#-session-authentication-flow)
- [API Integration (Secure)](#api-integration-secure)
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

### 🔒 Secure API Client with Session Management

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
    if (!this.sessionToken && endpoint !== '/identity-resolve') {
      throw new Error('Not authenticated - Please log in first')
    }

    const url = `${this.baseURL}${endpoint}`
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers,
    }

    // Add auth header for all requests except identity resolution
    if (endpoint !== '/identity-resolve') {
      headers['Authorization'] = `Bearer ${this.sessionToken}`
    }

    const response = await fetch(url, {
      headers,
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

  // Comments API (No user_id needed - extracted from session)
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
        // No user_id needed - extracted from session!
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
        // No user_id needed - extracted from session!
      }),
    })
  }

  async deleteComment(commentId: number) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'delete',
        comment_id: commentId,
        // No user_id needed - extracted from session!
      }),
    })
  }

  // Voting API (No user_id needed - extracted from session)
  async vote(commentId: number, action: 'upvote' | 'downvote' | 'remove') {
    return this.request('/voting', {
      method: 'POST',
      body: JSON.stringify({
        action,
        comment_id: commentId,
        // No user_id needed - extracted from session!
      }),
    })
  }

  // Reports API (No user_id needed - extracted from session)
  async createReport(commentId: number, reason: string, notes?: string) {
    return this.request('/reports', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        comment_id: commentId,
        reason,
        notes,
        // No user_id needed - extracted from session!
      }),
    })
  }
}

export const commentumClient = new CommentumClient(
  process.env.NEXT_PUBLIC_COMMENTUM_API_URL!
)
```

## Quick Start (Secure)

### 1. Authentication Provider

```typescript
// contexts/AuthProvider.tsx
import React, { createContext, useContext, ReactNode, useState, useEffect } from 'react'
import { commentumClient } from '@/lib/commentum-client'

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
  error: string | null
}

const AuthContext = createContext<AuthContextType | null>(null)

export const useAuth = () => {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider')
  }
  return context
}

interface AuthProviderProps {
  children: ReactNode
}

export const AuthProvider: React.FC<AuthProviderProps> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null)
  const [sessionToken, setSessionToken] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

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
    setError(null)

    try {
      const data = await commentumClient.resolveIdentity(clientType, token)
      
      if (data.user.banned) {
        throw new Error('Account is banned')
      }

      if (data.user.muted_until && new Date(data.user.muted_until) > new Date()) {
        throw new Error(`Account muted until ${data.user.muted_until}`)
      }

      setUser(data.user)
      setSessionToken(data.session_token)
      
      return data
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Authentication failed')
      throw err
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
    error,
  }

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  )
}
```

### 2. Basic Comment Component (Secure)

```typescript
// components/CommentSystem.tsx
import React, { useState, useEffect } from 'react'
import { commentumClient } from '@/lib/commentum-client'
import { useAuth } from '@/contexts/AuthProvider'

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
}

export const CommentSystem: React.FC<CommentSystemProps> = ({
  mediaId,
  mediaType,
}) => {
  const { isAuthenticated } = useAuth()
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
      const response = await commentumClient.getComments(mediaId)
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
        {isAuthenticated ? (
          <CommentForm mediaId={mediaId} onCommentPosted={loadComments} />
        ) : (
          <div className="login-prompt">
            <p>Please log in to comment</p>
            <LoginForm onLoginSuccess={loadComments} />
          </div>
        )}
      </div>
      
      <div className="comments-list">
        {comments.map(comment => (
          <CommentItem 
            key={comment.id} 
            comment={comment} 
            onUpdate={loadComments}
          />
        ))}
      </div>
    </div>
  )
}
```

### 3. Secure Comment Form Component

```typescript
// components/CommentForm.tsx
import React, { useState } from 'react'
import { commentumClient } from '@/lib/commentum-client'

interface CommentFormProps {
  mediaId: number
  onCommentPosted: () => void
  parentId?: number
}

export const CommentForm: React.FC<CommentFormProps> = ({
  mediaId,
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
      // ✅ SECURE: No user_id needed - extracted from session!
      await commentumClient.createComment({
        media_id: mediaId,
        content: content.trim(),
        parent_id: parentId
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

### 4. Login Component

```typescript
// components/LoginForm.tsx
import React, { useState } from 'react'
import { useAuth } from '@/contexts/AuthProvider'

interface LoginFormProps {
  onLoginSuccess: () => void
}

export const LoginForm: React.FC<LoginFormProps> = ({ onLoginSuccess }) => {
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
      setToken('')
      onLoginSuccess()
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed')
    } finally {
      setLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="login-form">
      <h3>Login to Comment</h3>
      
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
        <small>Get your token from your service's developer settings</small>
      </div>

      {error && <div className="error">{error}</div>}

      <button 
        type="submit" 
        disabled={loading || !token.trim()}
      >
        {loading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  )
}
```

## 🔒 Session Authentication Flow

### 1. Secure Authentication Hook

```typescript
// hooks/useAuth.ts
import { useState, useCallback } from 'react'
import { commentumClient } from '@/lib/commentum-client'

interface User {
  id: number
  username: string
  avatar_url?: string
  client_type: 'anilist' | 'mal' | 'simkl'
  banned: boolean
  shadow_banned: boolean
  muted_until: string | null
}

export const useAuth = () => {
  const [user, setUser] = useState<User | null>(null)
  const [sessionToken, setSessionToken] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const login = useCallback(async (clientType: string, token: string) => {
    setLoading(true)
    setError(null)

    try {
      const data = await commentumClient.resolveIdentity(clientType, token)
      
      if (data.user.banned) {
        throw new Error('Account is banned')
      }

      if (data.user.muted_until && new Date(data.user.muted_until) > new Date()) {
        throw new Error(`Account muted until ${data.user.muted_until}`)
      }

      setUser(data.user)
      setSessionToken(data.session_token)
      
      return data
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Authentication failed'
      setError(errorMessage)
      throw new Error(errorMessage)
    } finally {
      setLoading(false)
    }
  }, [])

  const logout = useCallback(() => {
    setUser(null)
    setSessionToken(null)
    commentumClient.clearSessionToken()
    setError(null)
  }, [])

  const initializeAuth = useCallback(() => {
    const token = localStorage.getItem('commentum_session_token')
    if (token) {
      setSessionToken(token)
      commentumClient.setSessionToken(token)
      // You might want to validate the token here
    }
  }, [])

  return {
    user,
    sessionToken,
    isAuthenticated: !!user,
    loading,
    error,
    login,
    logout,
    initializeAuth
  }
}
```

## API Integration (Secure)

### 1. Secure API Client Class

```typescript
// lib/apiClient.ts
import { commentumClient } from '@/lib/commentum-client'

class CommentumAPIClient {
  // ✅ All methods are now secure - no user_id parameters needed
  
  // Comments
  async createComment(params: {
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
    return commentumClient.createComment(params)
  }

  async editComment(commentId: number, content: string) {
    return commentumClient.editComment(commentId, content)
  }

  async deleteComment(commentId: number) {
    return commentumClient.deleteComment(commentId)
  }

  // Voting
  async vote(commentId: number, action: 'upvote' | 'downvote' | 'remove') {
    return commentumClient.vote(commentId, action)
  }

  // Reports
  async reportComment(commentId: number, reason: string, notes?: string) {
    return commentumClient.createReport(commentId, reason, notes)
  }

  // Identity Resolution
  async resolveIdentity(clientType: string, token: string) {
    return commentumClient.resolveIdentity(clientType, token)
  }
}

export const commentumAPIClient = new CommentumAPIClient()
```

### 2. React Query Integration (Secure)

```typescript
// hooks/useComments.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumClient } from '@/lib/commentum-client'

export const useComments = (mediaId: number) => {
  return useQuery({
    queryKey: ['comments', mediaId],
    queryFn: () => commentumClient.getComments(mediaId),
    staleTime: 5 * 60 * 1000, // 5 minutes
  })
}

export const useCreateComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: commentumClient.createComment,
    onSuccess: (_, variables) => {
      // Invalidate comments query for this media
      if (variables.media_id) {
        queryClient.invalidateQueries({ queryKey: ['comments', variables.media_id] })
      } else {
        // If using media_info, invalidate all comments queries
        queryClient.invalidateQueries({ queryKey: ['comments'] })
      }
    },
  })
}

export const useVote = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ commentId, action }: { commentId: number; action: 'upvote' | 'downvote' | 'remove' }) =>
      commentumClient.vote(commentId, action),
    onSuccess: (_, variables) => {
      // Invalidate relevant queries
      queryClient.invalidateQueries({ queryKey: ['comments'] })
    },
  })
}
```

## Real-time Updates

### 1. WebSocket Integration (Session-Aware)

```typescript
// hooks/useRealtimeComments.ts
import { useEffect, useRef } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import { useAuth } from '@/contexts/AuthProvider'

interface RealtimeComment {
  type: 'created' | 'updated' | 'deleted'
  comment_id: number
  media_id: number
  data: any
}

export const useRealtimeComments = (mediaId: number) => {
  const queryClient = useQueryClient()
  const { sessionToken } = useAuth()
  const wsRef = useRef<WebSocket | null>(null)

  useEffect(() => {
    if (!sessionToken) return

    const wsUrl = `${process.env.NEXT_PUBLIC_COMMENTUM_API_URL?.replace('http', 'ws')}/comments?XTransformPort=3003`
    
    wsRef.current = new WebSocket(wsUrl)

    wsRef.current.onopen = () => {
      console.log('Connected to comment updates')
      // Subscribe to media-specific updates
      wsRef.current?.send(JSON.stringify({
        action: 'subscribe',
        media_id: mediaId,
        session_token: sessionToken // Include session token for authentication
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
  }, [mediaId, sessionToken, queryClient])

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

## Error Handling

### 1. Session Error Handling

```typescript
// hooks/useSessionErrorHandling.ts
import { useCallback } from 'react'
import { useAuth } from '@/contexts/AuthProvider'

export const useSessionErrorHandling = () => {
  const { logout } = useAuth()

  const handleSessionError = useCallback((error: any) => {
    if (error.message?.includes('Session expired') || 
        error.message?.includes('Invalid or expired session token') ||
        error.message?.includes('Not authenticated')) {
      logout()
      // You might want to show a login modal or redirect to login page
      return 'session_expired'
    }
    return 'other_error'
  }, [logout])

  const safeAPICall = useCallback(async <T>(
    apiCall: () => Promise<T>,
    onError?: (error: string) => void
  ): Promise<T | null> => {
    try {
      return await apiCall()
    } catch (error) {
      const errorType = handleSessionError(error)
      if (errorType === 'session_expired') {
        onError?.('Your session has expired. Please log in again.')
      } else {
        onError?.(error instanceof Error ? error.message : 'An error occurred')
      }
      return null
    }
  }, [handleSessionError])

  return {
    handleSessionError,
    safeAPICall
  }
}
```

## Performance Optimization

### 1. Session Management Optimization

```typescript
// hooks/useOptimisticSession.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumClient } from '@/lib/commentum-client'

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
      const optimisticComment = {
        id: Date.now(), // Temporary ID
        ...newComment,
        created_at: new Date().toISOString(),
        user_vote: 0,
        upvotes: 0,
        downvotes: 0,
        replies: []
      }

      queryClient.setQueryData(['comments', mediaId], (old: any) => {
        if (!old) return [optimisticComment]
        return [...old, optimisticComment]
      })

      return { previousComments }
    },
    onError: (err, newComment, context) => {
      // If the mutation fails, use the context returned from onMutate to roll back
      queryClient.setQueryData(['comments', mediaId], context?.previousComments)
    },
    onSettled: () => {
      // Always refetch after error or success to make sure the server state is synchronized
      queryClient.invalidateQueries({ queryKey: ['comments', mediaId] })
    },
  })
}
```

## Security Best Practices

### 1. Session Security

```typescript
// lib/sessionSecurity.ts
export const sessionSecurity = {
  // Validate session token format
  validateSessionToken(token: string): boolean {
    // Session tokens should be 64-character hex strings
    return /^[a-f0-9]{64}$/.test(token)
  },

  // Check if session is expired
  isSessionExpired(expiresAt: string): boolean {
    return new Date(expiresAt) < new Date()
  },

  // Securely store session token
  storeSessionToken(token: string): void {
    if (typeof window !== 'undefined' && this.validateSessionToken(token)) {
      localStorage.setItem('commentum_session_token', token)
    }
  },

  // Securely clear session token
  clearSessionToken(): void {
    if (typeof window !== 'undefined') {
      localStorage.removeItem('commentum_session_token')
    }
  },

  // Get session token
  getSessionToken(): string | null {
    if (typeof window !== 'undefined') {
      const token = localStorage.getItem('commentum_session_token')
      return this.validateSessionToken(token || '') ? token : null
    }
    return null
  }
}
```

### 2. API Security Headers

```typescript
// lib/apiSecurity.ts
export const apiSecurity = {
  // Get secure headers for API requests
  getSecureHeaders(sessionToken: string): Record<string, string> {
    return {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${sessionToken}`,
      'X-Requested-With': 'XMLHttpRequest'
    }
  },

  // Validate API response
  validateResponse(response: Response): boolean {
    return response.ok && response.headers.get('content-type')?.includes('application/json')
  },

  // Handle security errors
  handleSecurityError(error: any): string {
    if (error.status === 401) {
      return 'Session expired - Please log in again'
    }
    if (error.status === 403) {
      return 'Access denied - You do not have permission to perform this action'
    }
    return error.message || 'An error occurred'
  }
}
```

## 🚨 Breaking Changes from Previous Version

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

This secure integration guide ensures that all comment operations are performed with proper session-based authentication, preventing identity spoofing attacks while maintaining a smooth developer experience.