# React Integration Example

This example demonstrates how to integrate Commentum into a React application using TypeScript and modern React patterns.

## Project Setup

### 1. Install Dependencies

```bash
npm install @supabase/supabase-js @tanstack/react-query zustand react-hook-form @hookform/resolvers zod
# or
yarn add @supabase/supabase-js @tanstack/react-query zustand react-hook-form @hookform/resolvers zod
```

### 2. Environment Configuration

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
NEXT_PUBLIC_COMMENTUM_API_URL=https://your-project.supabase.co/functions/v1
```

### 3. Supabase Client Configuration

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)

// Commentum API client
export const commentumAPI = {
  baseURL: process.env.NEXT_PUBLIC_COMMENTUM_API_URL!,
  
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
  userId?: number
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
  userId?: number
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
  userId,
}) => {
  const contextValue: CommentSystemContextType = {
    mediaId,
    mediaType,
    userId,
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
import { useCommentSystem } from '../contexts/CommentSystemProvider'

export const CommentSystem: React.FC = () => {
  const { mediaId, userId } = useCommentSystem()
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

      {userId && (
        <div className="comment-form-section">
          <CommentForm onSuccess={refetch} />
        </div>
      )}

      <div className="comments-section">
        <CommentList comments={comments || []} onUpdate={refetch} />
      </div>
    </div>
  )
}
```

### 3. Comment Form Component

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
  const { mediaId, userId } = useCommentSystem()
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
    if (!userId) return

    try {
      await createComment.mutateAsync({
        user_id: userId,
        media_id: mediaId,
        content: data.content.trim(),
        parent_id: parentId,
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

### 4. Comment List Component

```typescript
// components/CommentList.tsx
import React, { useState } from 'react'
import { CommentItem } from './CommentItem'
import { Comment } from '../types/comment'

interface CommentListProps {
  comments: Comment[]
  onUpdate?: () => void
}

export const CommentList: React.FC<CommentListProps> = ({
  comments,
  onUpdate,
}) => {
  const [expandedReplies, setExpandedReplies] = useState<Set<number>>(new Set())

  const toggleReplies = (commentId: number) => {
    setExpandedReplies(prev => {
      const newSet = new Set(prev)
      if (newSet.has(commentId)) {
        newSet.delete(commentId)
      } else {
        newSet.add(commentId)
      }
      return newSet
    })
  }

  // Organize comments into threads
  const topLevelComments = comments.filter(comment => !comment.parent_id)
  const repliesByParent = comments.reduce((acc, comment) => {
    if (comment.parent_id) {
      if (!acc[comment.parent_id]) {
        acc[comment.parent_id] = []
      }
      acc[comment.parent_id].push(comment)
    }
    return acc
  }, {} as Record<number, Comment[]>)

  return (
    <div className="comment-list">
      {topLevelComments.length === 0 ? (
        <div className="no-comments">
          <p>No comments yet. Be the first to share your thoughts!</p>
        </div>
      ) : (
        <div className="comments-tree">
          {topLevelComments.map(comment => (
            <CommentItem
              key={comment.id}
              comment={comment}
              replies={repliesByParent[comment.id] || []}
              isExpanded={expandedReplies.has(comment.id)}
              onToggleReplies={() => toggleReplies(comment.id)}
              onUpdate={onUpdate}
            />
          ))}
        </div>
      )}
    </div>
  )
}
```

### 5. Individual Comment Item

```typescript
// components/CommentItem.tsx
import React, { useState } from 'react'
import { Comment } from '../types/comment'
import { VotingButtons } from './VotingButtons'
import { CommentActions } from './CommentActions'
import { CommentForm } from './CommentForm'
import { useAuth } from '../contexts/AuthContext'
import { formatDistanceToNow } from 'date-fns'

interface CommentItemProps {
  comment: Comment
  replies?: Comment[]
  isExpanded?: boolean
  onToggleReplies?: () => void
  onUpdate?: () => void
  level?: number
}

export const CommentItem: React.FC<CommentItemProps> = ({
  comment,
  replies = [],
  isExpanded = false,
  onToggleReplies,
  onUpdate,
  level = 0,
}) => {
  const { user } = useAuth()
  const [isReplying, setIsReplying] = useState(false)
  const [isEditing, setIsEditing] = useState(false)

  const canEdit = user?.user_id === comment.user_id && !comment.deleted
  const hasReplies = replies.length > 0

  return (
    <div className={`comment-item level-${level}`}>
      <div className="comment-content">
        <div className="comment-header">
          <div className="comment-author">
            <img
              src={comment.avatar_url || '/default-avatar.png'}
              alt={comment.username}
              className="author-avatar"
            />
            <div className="author-info">
              <span className="author-name">{comment.username}</span>
              <span className="comment-time">
                {formatDistanceToNow(new Date(comment.created_at), { addSuffix: true })}
              </span>
              {comment.edited && (
                <span className="edited-indicator">(edited)</span>
              )}
            </div>
          </div>

          <div className="comment-meta">
            {comment.pinned && (
              <span className="pinned-indicator">📌 Pinned</span>
            )}
            {comment.locked && (
              <span className="locked-indicator">🔒 Locked</span>
            )}
          </div>
        </div>

        <div className="comment-body">
          {comment.deleted ? (
            <p className="deleted-message">This comment has been deleted</p>
          ) : isEditing ? (
            <CommentEditForm
              comment={comment}
              onSave={() => {
                setIsEditing(false)
                onUpdate?.()
              }}
              onCancel={() => setIsEditing(false)}
            />
          ) : (
            <div className="comment-text">
              {comment.content}
            </div>
          )}
        </div>

        {!comment.deleted && !isEditing && (
          <div className="comment-actions">
            <VotingButtons
              commentId={comment.id}
              initialVotes={{
                upvotes: comment.upvotes,
                downvotes: comment.downvotes,
                userVote: comment.user_vote,
              }}
              onUpdate={onUpdate}
            />

            <div className="action-buttons">
              {user && (
                <button
                  onClick={() => setIsReplying(!isReplying)}
                  className="reply-button"
                >
                  Reply
                </button>
              )}

              {canEdit && (
                <button
                  onClick={() => setIsEditing(true)}
                  className="edit-button"
                >
                  Edit
                </button>
              )}

              <CommentActions
                comment={comment}
                onUpdate={onUpdate}
              />
            </div>
          </div>
        )}
      </div>

      {/* Replies Section */}
      {hasReplies && (
        <div className="comment-replies">
          <button
            onClick={onToggleReplies}
            className="toggle-replies-button"
          >
            {isExpanded ? 'Hide' : 'Show'} {replies.length} {replies.length === 1 ? 'reply' : 'replies'}
          </button>

          {isExpanded && (
            <div className="replies-list">
              {replies.map(reply => (
                <CommentItem
                  key={reply.id}
                  comment={reply}
                  replies={[]} // Nested replies would need additional logic
                  onUpdate={onUpdate}
                  level={level + 1}
                />
              ))}
            </div>
          )}
        </div>
      )}

      {/* Reply Form */}
      {isReplying && user && (
        <div className="reply-form">
          <CommentForm
            parentId={comment.id}
            onSuccess={() => {
              setIsReplying(false)
              onUpdate?.()
            }}
            onCancel={() => setIsReplying(false)}
          />
        </div>
      )}
    </div>
  )
}
```

## Custom Hooks

### 1. Comments Hook

```typescript
// hooks/useComments.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { commentumAPI } from '../lib/supabase'
import { Comment } from '../types/comment'

export const useComments = (mediaId: number) => {
  return useQuery({
    queryKey: ['comments', mediaId],
    queryFn: () => commentumAPI.request<Comment[]>(`/comments?media_id=${mediaId}`),
    enabled: !!mediaId,
  })
}

export const useCreateComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: {
      user_id: number
      media_id: number
      content: string
      parent_id?: number
    }) =>
      commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'create',
          ...data,
        }),
      }),
    onSuccess: (_, variables) => {
      // Invalidate and refetch comments
      queryClient.invalidateQueries({ queryKey: ['comments', variables.media_id] })
    },
  })
}

export const useEditComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: {
      user_id: number
      comment_id: number
      content: string
    }) =>
      commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'edit',
          ...data,
        }),
      }),
    onSuccess: () => {
      // Invalidate all comments queries to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['comments'] })
    },
  })
}

export const useDeleteComment = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: { user_id: number; comment_id: number }) =>
      commentumAPI.request<Comment>('/comments', {
        method: 'POST',
        body: JSON.stringify({
          action: 'delete',
          ...data,
        }),
      }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['comments'] })
    },
  })
}
```

### 2. Voting Hook

```typescript
// hooks/useVoting.ts
import { useState, useCallback } from 'react'
import { commentumAPI } from '../lib/supabase'

interface VoteData {
  upvotes: number
  downvotes: number
  userVote: number
}

export const useVoting = (
  commentId: number,
  initialVotes: VoteData,
  userId?: number
) => {
  const [votes, setVotes] = useState<VoteData>(initialVotes)
  const [loading, setLoading] = useState(false)

  const vote = useCallback(async (action: 'upvote' | 'downvote' | 'remove') => {
    if (!userId || loading) return

    setLoading(true)

    try {
      const response = await commentumAPI.request('/voting', {
        method: 'POST',
        body: JSON.stringify({
          action,
          user_id: userId,
          comment_id: commentId,
        }),
      })

      // Update local state based on response
      if (action === 'remove') {
        setVotes(prev => ({
          ...prev,
          upvotes: prev.userVote === 1 ? prev.upvotes - 1 : prev.upvotes,
          downvotes: prev.userVote === -1 ? prev.downvotes - 1 : prev.downvotes,
          userVote: 0,
        }))
      } else {
        const voteType = action === 'upvote' ? 1 : -1
        setVotes(prev => {
          const newVotes = { ...prev }
          
          // Remove previous vote if different
          if (prev.userVote === 1 && voteType === -1) {
            newVotes.upvotes--
          } else if (prev.userVote === -1 && voteType === 1) {
            newVotes.downvotes--
          }
          
          // Add new vote if different from previous
          if (prev.userVote !== voteType) {
            if (voteType === 1) newVotes.upvotes++
            else newVotes.downvotes++
          }
          
          newVotes.userVote = prev.userVote === voteType ? 0 : voteType
          return newVotes
        })
      }
    } catch (error) {
      console.error('Vote failed:', error)
      throw error
    } finally {
      setLoading(false)
    }
  }, [commentId, userId, loading])

  return {
    votes,
    loading,
    upvote: () => vote('upvote'),
    downvote: () => vote('downvote'),
    removeVote: () => vote('remove'),
  }
}
```

### 3. Toast Hook

```typescript
// hooks/useToast.ts
import { useState, useCallback } from 'react'

interface Toast {
  id: string
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
  duration?: number
}

export const useToast = () => {
  const [toasts, setToasts] = useState<Toast[]>([])

  const addToast = useCallback(
    (message: string, type: Toast['type'] = 'info', duration = 5000) => {
      const id = Date.now().toString()
      const toast: Toast = { id, message, type, duration }

      setToasts(prev => [...prev, toast])

      if (duration > 0) {
        setTimeout(() => {
          removeToast(id)
        }, duration)
      }

      return id
    },
    []
  )

  const removeToast = useCallback((id: string) => {
    setToasts(prev => prev.filter(toast => toast.id !== id))
  }, [])

  const success = useCallback((message: string) => addToast(message, 'success'), [addToast])
  const error = useCallback((message: string) => addToast(message, 'error'), [addToast])
  const warning = useCallback((message: string) => addToast(message, 'warning'), [addToast])
  const info = useCallback((message: string) => addToast(message, 'info'), [addToast])

  return {
    toasts,
    addToast,
    removeToast,
    success,
    error,
    warning,
    info,
  }
}
```

## Usage Example

```typescript
// pages/media/[id].tsx
import { GetServerSideProps } from 'next'
import { CommentSystemProvider } from '../contexts/CommentSystemProvider'
import { CommentSystem } from '../components/CommentSystem'
import { AuthProvider } from '../contexts/AuthContext'

interface MediaPageProps {
  media: {
    id: number
    title: string
    type: 'anime' | 'manga' | 'movie' | 'tv'
  }
}

export default function MediaPage({ media }: MediaPageProps) {
  // This would come from your auth system
  const [currentUser, setCurrentUser] = useState<any>(null)

  return (
    <AuthProvider>
      <CommentSystemProvider
        mediaId={media.id}
        mediaType={media.type}
        userId={currentUser?.user_id}
      >
        <div className="media-page">
          <header>
            <h1>{media.title}</h1>
            <p>Type: {media.type}</p>
          </header>

          <main>
            <section className="media-content">
              {/* Your media content here */}
            </section>

            <section className="comments-section">
              <CommentSystem />
            </section>
          </main>
        </div>
      </CommentSystemProvider>
    </AuthProvider>
  )
}

export const getServerSideProps: GetServerSideProps = async (context) => {
  const { id } = context.params

  // Fetch your media data
  const media = await fetchMediaById(id as string)

  if (!media) {
    return { notFound: true }
  }

  return {
    props: {
      media,
    },
  }
}

async function fetchMediaById(id: string) {
  // Implement your media fetching logic
  return {
    id: parseInt(id),
    title: 'Sample Media Title',
    type: 'anime' as const,
  }
}
```

## Styling

```css
/* styles/CommentSystem.css */
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

.comment-form {
  background: #f8fafc;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 24px;
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

.form-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12px;
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

.reply-form {
  margin-top: 12px;
  padding-left: 16px;
}

.loading, .error {
  display: flex;
  flex-direction: column;
  align-items: center;
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
```

This React integration provides a complete, production-ready comment system with modern React patterns, TypeScript support, and comprehensive error handling.