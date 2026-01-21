# Comments API

The Comments API handles all comment-related operations including creation, editing, deletion, and moderation actions using secure session-based authentication.

## 🔒 Security Overview

**CRITICAL**: This API uses session-based authentication. The `user_id` parameter has been removed to prevent identity spoofing. User identity is extracted from the session token.

## Endpoint

```
POST /functions/v1/comments
```

## Request Headers

```http
Authorization: Bearer <session_token>
Content-Type: application/json
```

## Request Body

```typescript
interface CommentRequest {
  action: 'create' | 'edit' | 'delete' | 'restore' | 'pin' | 'unpin' | 'lock' | 'unlock' | 'tag' | 'untag'
  media_id?: number
  media_info?: {
    external_id: string
    media_type: 'anime' | 'manga' | 'movie' | 'tv' | 'other'
    title: string
    year?: number
    poster_url?: string
  }
  parent_id?: number
  comment_id?: number
  content?: string
  tag_type?: 'spoiler' | 'nsfw' | 'warning' | 'offensive' | 'spam'
  reason?: string
}
```

### Parameters

| Parameter | Type | Required | Actions | Description |
|-----------|------|----------|---------|-------------|
| `action` | string | Yes | All | The action to perform |
| `media_id` | number | create | Create | ID of the media item being commented on |
| `media_info` | object | create | Create | Media information for auto-creation |
| `parent_id` | number | create | Create | ID of parent comment (for replies) |
| `comment_id` | number | edit/delete/restore/pin/unpin/lock/unlock/tag/untag | Moderation | ID of the target comment |
| `content` | string | create/edit | Content | Comment content (max length configured in system) |
| `tag_type` | string | tag/untag | Tagging | Type of tag to apply/remove |
| `reason` | string | moderation actions | Moderation | Reason for the action |

## 🆕 Auto Media Creation

### Option 1: Use Existing Media ID
```json
{
  "action": "create",
  "media_id": 456,
  "content": "This is a great anime!"
}
```

### Option 2: Auto-Create Media (NEW)
```json
{
  "action": "create",
  "media_info": {
    "external_id": "12345",
    "media_type": "anime",
    "title": "Attack on Titan",
    "year": 2013,
    "poster_url": "https://example.com/poster.jpg"
  },
  "content": "This is a great anime!"
}
```

**Response for Auto-Creation:**
```json
{
  "comment": { ... },
  "media_id": 456
}
```

## Actions

### 1. Create Comment

**Required Parameters:** `action: 'create'`, `content`

**Either `media_id` OR `media_info` is required**

#### Request (Existing Media)
```json
{
  "action": "create",
  "media_id": 456,
  "content": "This is a great anime!",
  "parent_id": 789
}
```

#### Request (Auto-Create Media)
```json
{
  "action": "create",
  "media_info": {
    "external_id": "12345",
    "media_type": "anime",
    "title": "Attack on Titan",
    "year": 2013,
    "poster_url": "https://example.com/poster.jpg"
  },
  "content": "This is a great anime!",
  "parent_id": 789
}
```

#### Response (201 Created)
```json
{
  "comment": {
    "id": 1001,
    "media_id": 456,
    "user_id": 123,
    "parent_id": 789,
    "content": "This is a great anime!",
    "content_html": "This is a great anime!",
    "deleted": false,
    "pinned": false,
    "locked": false,
    "edited": false,
    "edit_count": 0,
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  },
  "media_id": 456
}
```

### 2. Edit Comment

**Required Parameters:** `action: 'edit'`, `comment_id`, `content`

#### Request
```json
{
  "action": "edit",
  "comment_id": 1001,
  "content": "This is an amazing anime! The animation is stunning."
}
```

#### Response (200 OK)
```json
{
  "comment": {
    "id": 1001,
    "content": "This is an amazing anime! The animation is stunning.",
    "content_html": "This is an amazing anime! The animation is stunning.",
    "edited": true,
    "edited_at": "2024-01-15T11:00:00Z",
    "edit_count": 1,
    "updated_at": "2024-01-15T11:00:00Z"
  }
}
```

### 3. Delete Comment

**Required Parameters:** `action: 'delete'`, `comment_id`

#### Request
```json
{
  "action": "delete",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "comment": {
    "id": 1001,
    "deleted": true,
    "deleted_at": "2024-01-15T11:30:00Z",
    "deleted_by": 123
  }
}
```

### 4. Restore Comment

**Required Parameters:** `action: 'restore'`, `comment_id`

#### Request
```json
{
  "action": "restore",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "comment": {
    "id": 1001,
    "deleted": false,
    "deleted_at": null,
    "deleted_by": null
  }
}
```

### 5. Pin Comment

**Required Parameters:** `action: 'pin'`, `comment_id`

#### Request
```json
{
  "action": "pin",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "comment": {
    "id": 1001,
    "pinned": true,
    "pinned_at": "2024-01-15T12:00:00Z",
    "pinned_by": 123
  }
}
```

### 6. Lock Comment Thread

**Required Parameters:** `action: 'lock'`, `comment_id`

#### Request
```json
{
  "action": "lock",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "comment": {
    "id": 1001,
    "locked": true,
    "locked_at": "2024-01-15T12:30:00Z",
    "locked_by": 123
  }
}
```

### 7. Tag Comment

**Required Parameters:** `action: 'tag'`, `comment_id`, `tag_type`

#### Request
```json
{
  "action": "tag",
  "comment_id": 1001,
  "tag_type": "spoiler"
}
```

#### Response (201 Created)
```json
{
  "tag": {
    "id": 501,
    "comment_id": 1001,
    "tag_type": "spoiler",
    "tagged_by": 123,
    "created_at": "2024-01-15T13:00:00Z"
  }
}
```

## Error Responses

### 400 Bad Request
```json
{
  "error": "Comment too long"
}
```

### 401 Unauthorized
```json
{
  "error": "Missing session token"
}
```

### 401 Unauthorized (Session Expired)
```json
{
  "error": "Invalid or expired session token"
}
```

### 403 Forbidden
```json
{
  "error": "User banned or muted"
}
```

### 404 Not Found
```json
{
  "error": "Media ID or media info is required"
}
```

### 429 Too Many Requests
```json
{
  "error": "Rate limit exceeded"
}
```

### 500 Internal Server Error
```json
{
  "error": "Internal server error"
}
```

## 🔐 Authentication

### Session-Based Authentication

All requests must include a valid session token:

```javascript
// ❌ OLD (Vulnerable)
const response = await fetch('/functions/v1/comments', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    action: 'create',
    user_id: 123,  // Could be faked!
    media_id: 456,
    content: 'Comment text'
  })
})

// ✅ NEW (Secure)
const sessionToken = localStorage.getItem('commentum_session_token')
const response = await fetch('/functions/v1/comments', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${sessionToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    action: 'create',
    media_id: 456,
    content: 'Comment text'  // No user_id needed!
  })
})
```

### Session Validation

The API automatically:
1. Extracts session token from Authorization header
2. Validates session against database
3. Checks user status (banned/muted)
4. Updates last used time
5. Extracts user ID from session

## Validation Rules

### Content Validation
- **Maximum Length**: Configurable (default: 10,000 characters)
- **Required Fields**: Content cannot be empty
- **HTML Sanitization**: Content is stored as plain text and HTML

### Nesting Validation
- **Maximum Nesting Level**: Configurable (default: 10 levels)
- **Parent Validation**: Parent comment must exist and not be deleted
- **Self-Reply Prevention**: Users cannot reply to their own comments

### Permission Validation
- **Banned Users**: Cannot create or edit comments
- **Muted Users**: Cannot comment while muted
- **Locked Comments**: Cannot reply to locked comments
- **Deleted Comments**: Cannot edit deleted comments

## 🚀 Usage Examples

### React Hook with Session Management

```typescript
import { useState, useCallback } from 'react'

interface CommentFormProps {
  mediaId?: number
  mediaInfo?: MediaInfo
  parentId?: number
  onCommentCreated: (comment: any) => void
}

export const CommentForm: React.FC<CommentFormProps> = ({
  mediaId,
  mediaInfo,
  parentId,
  onCommentCreated
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
      const sessionToken = localStorage.getItem('commentum_session_token')
      if (!sessionToken) {
        throw new Error('Not authenticated')
      }

      const requestBody: any = {
        action: 'create',
        content: content.trim(),
        parent_id: parentId
      }

      // Use existing media_id or provide media_info for auto-creation
      if (mediaId) {
        requestBody.media_id = mediaId
      } else if (mediaInfo) {
        requestBody.media_info = mediaInfo
      }

      const response = await fetch('/functions/v1/comments', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${sessionToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(requestBody)
      })

      const data = await response.json()

      if (!response.ok) {
        throw new Error(data.error)
      }

      onCommentCreated(data.comment)
      setContent('')
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
        placeholder="Write a comment..."
        disabled={submitting}
        maxLength={10000}
      />
      {error && <div className="error">{error}</div>}
      <button type="submit" disabled={submitting || !content.trim()}>
        {submitting ? 'Posting...' : 'Post Comment'}
      </button>
    </form>
  )
}
```

### Vue.js with Auto Media Creation

```typescript
import { ref, computed } from 'vue'

export default {
  props: {
    mediaId: Number,
    parentId: Number
  },
  emits: ['comment-created'],
  setup(props, { emit }) {
    const content = ref('')
    const submitting = ref(false)
    const error = ref(null)
    const sessionToken = localStorage.getItem('commentum_session_token')

    const createComment = async () => {
      if (!content.value.trim()) return
      if (!sessionToken) {
        error.value = 'Not authenticated'
        return
      }

      submitting.value = true
      error.value = null

      try {
        const requestBody = {
          action: 'create',
          content: content.value.trim(),
          parent_id: props.parentId
        }

        // If no mediaId, create media automatically
        if (!props.mediaId) {
          requestBody.media_info = {
            external_id: '12345',
            media_type: 'anime',
            title: 'Auto-created Media',
            year: 2024
          }
        } else {
          requestBody.media_id = props.mediaId
        }

        const response = await fetch('/functions/v1/comments', {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${sessionToken}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(requestBody)
        })

        const data = await response.json()

        if (!response.ok) {
          throw new Error(data.error)
        }

        emit('comment-created', data.comment)
        content.value = ''
      } catch (err) {
        error.value = err instanceof Error ? err.message : 'Failed to post comment'
      } finally {
        submitting.value = false
      }
    }

    return {
      content,
      submitting,
      error,
      createComment
    }
  }
}
```

### Vanilla JavaScript API Client

```javascript
class CommentumAPI {
  constructor(sessionToken) {
    this.sessionToken = sessionToken
  }

  async createComment(options) {
    const requestBody = {
      action: 'create',
      ...options
    }

    return this.makeRequest('/functions/v1/comments', {
      method: 'POST',
      body: JSON.stringify(requestBody)
    })
  }

  async createCommentWithMedia(mediaInfo, content, parentId = null) {
    return this.createComment({
      media_info: mediaInfo,
      content,
      parent_id: parentId
    })
  }

  async editComment(commentId, content) {
    return this.makeRequest('/functions/v1/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'edit',
        comment_id: commentId,
        content
      })
    })
  }

  async deleteComment(commentId) {
    return this.makeRequest('/functions/v1/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'delete',
        comment_id: commentId
      })
    })
  }

  async makeRequest(endpoint, options = {}) {
    if (!this.sessionToken) {
      throw new Error('Not authenticated')
    }

    const response = await fetch(endpoint, {
      ...options,
      headers: {
        'Authorization': `Bearer ${this.sessionToken}`,
        'Content-Type': 'application/json',
        ...options.headers
      }
    })

    if (response.status === 401) {
      localStorage.removeItem('commentum_session_token')
      throw new Error('Session expired. Please log in again.')
    }

    return response.json()
  }
}

// Usage
const api = new CommentumAPI(localStorage.getItem('commentum_session_token'))

// Create comment with auto media creation
try {
  const comment = await api.createCommentWithMedia(
    {
      external_id: '12345',
      media_type: 'anime',
      title: 'Attack on Titan',
      year: 2013,
      poster_url: 'https://example.com/poster.jpg'
    },
    'This is a great anime!'
  )
  
  console.log('Comment created:', comment)
  console.log('Media ID:', comment.media_id)
} catch (error) {
  console.error('Error:', error.message)
}
```

## Rate Limiting

- **Comments**: 30 per hour per user (configurable)
- **Edits**: Included in comment rate limit
- **Moderation Actions**: Exempt from rate limits for moderators

## 🔒 Security Considerations

- **Session Authentication**: All requests require valid session tokens
- **Zero-Trust Architecture**: No client-provided user data is trusted
- **Input Sanitization**: All content is sanitized before storage
- **Permission Checks**: All actions are validated against user permissions
- **Audit Logging**: All actions are logged for moderation review
- **Rate Limiting**: Prevents spam and abuse
- **Auto Media Creation**: Prevents media ID spoofing by creating media server-side

## 🚨 Breaking Changes from Previous Version

### Before (Vulnerable)
```javascript
// ❌ Client could send any user_id
fetch('/functions/v1/comments', {
  body: JSON.stringify({
    action: 'create',
    user_id: 123,  // Could be faked!
    media_id: 456,
    content: 'Comment text'
  })
})
```

### After (Secure)
```javascript
// ✅ User ID extracted from session
fetch('/functions/v1/comments', {
  headers: { 'Authorization': `Bearer ${sessionToken}` },
  body: JSON.stringify({
    action: 'create',
    media_id: 456,
    content: 'Comment text'  // No user_id needed!
  })
})
```

This secure API ensures that comment operations can only be performed by authenticated users while providing powerful features like automatic media creation.