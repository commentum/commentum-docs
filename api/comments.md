# Comments API

The Comments API handles all comment-related operations including creation, editing, deletion, and moderation actions.

## Endpoint

```
POST /comments
```

## Request Body

```typescript
interface CommentRequest {
  action: 'create' | 'edit' | 'delete' | 'restore' | 'pin' | 'unpin' | 'lock' | 'unlock' | 'tag' | 'untag'
  user_id: number
  media_id?: number
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
| `user_id` | number | Yes | All | ID of the user performing the action |
| `media_id` | number | create | Create | ID of the media item being commented on |
| `parent_id` | number | create | Create | ID of parent comment (for replies) |
| `comment_id` | number | edit/delete/restore/pin/unpin/lock/unlock/tag/untag | Moderation | ID of the target comment |
| `content` | string | create/edit | Content | Comment content (max length configured in system) |
| `tag_type` | string | tag/untag | Tagging | Type of tag to apply/remove |
| `reason` | string | moderation actions | Moderation | Reason for the action |

## Actions

### 1. Create Comment

**Required Parameters:** `action: 'create'`, `user_id`, `media_id`, `content`

**Optional Parameters:** `parent_id`

#### Request
```json
{
  "action": "create",
  "user_id": 123,
  "media_id": 456,
  "content": "This is a great anime!",
  "parent_id": 789
}
```

#### Response (201 Created)
```json
{
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
}
```

### 2. Edit Comment

**Required Parameters:** `action: 'edit'`, `user_id`, `comment_id`, `content`

#### Request
```json
{
  "action": "edit",
  "user_id": 123,
  "comment_id": 1001,
  "content": "This is an amazing anime! The animation is stunning."
}
```

#### Response (200 OK)
```json
{
  "id": 1001,
  "content": "This is an amazing anime! The animation is stunning.",
  "content_html": "This is an amazing anime! The animation is stunning.",
  "edited": true,
  "edited_at": "2024-01-15T11:00:00Z",
  "edit_count": 1,
  "updated_at": "2024-01-15T11:00:00Z"
}
```

### 3. Delete Comment

**Required Parameters:** `action: 'delete'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "delete",
  "user_id": 123,
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 1001,
  "deleted": true,
  "deleted_at": "2024-01-15T11:30:00Z",
  "deleted_by": 123
}
```

### 4. Restore Comment

**Required Parameters:** `action: 'restore'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "restore",
  "user_id": 123,
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 1001,
  "deleted": false,
  "deleted_at": null,
  "deleted_by": null
}
```

### 5. Pin Comment

**Required Parameters:** `action: 'pin'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "pin",
  "user_id": 123,
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 1001,
  "pinned": true,
  "pinned_at": "2024-01-15T12:00:00Z",
  "pinned_by": 123
}
```

### 6. Lock Comment Thread

**Required Parameters:** `action: 'lock'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "lock",
  "user_id": 123,
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 1001,
  "locked": true,
  "locked_at": "2024-01-15T12:30:00Z",
  "locked_by": 123
}
```

### 7. Tag Comment

**Required Parameters:** `action: 'tag'`, `user_id`, `comment_id`, `tag_type`

#### Request
```json
{
  "action": "tag",
  "user_id": 123,
  "comment_id": 1001,
  "tag_type": "spoiler"
}
```

#### Response (201 Created)
```json
{
  "id": 501,
  "comment_id": 1001,
  "tag_type": "spoiler",
  "tagged_by": 123,
  "created_at": "2024-01-15T13:00:00Z"
}
```

## Error Responses

### 400 Bad Request
```json
{
  "error": "Comment too long"
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
  "error": "User not found"
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

## Usage Examples

### React Component for Comment Creation

```typescript
import { useState } from 'react';

interface CommentFormProps {
  mediaId: number;
  userId: number;
  parentId?: number;
  onCommentCreated: (comment: any) => void;
}

export const CommentForm: React.FC<CommentFormProps> = ({
  mediaId,
  userId,
  parentId,
  onCommentCreated
}) => {
  const [content, setContent] = useState('');
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!content.trim()) return;

    setSubmitting(true);
    setError(null);

    try {
      const response = await fetch('/comments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action: 'create',
          user_id: userId,
          media_id: mediaId,
          content: content.trim(),
          parent_id: parentId
        })
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error);
      }

      onCommentCreated(data);
      setContent('');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to post comment');
    } finally {
      setSubmitting(false);
    }
  };

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
  );
};
```

### Comment Moderation Panel

```typescript
interface ModerationActionsProps {
  comment: any;
  userId: number;
  userRole: string;
  onAction: (action: string, result: any) => void;
}

export const ModerationActions: React.FC<ModerationActionsProps> = ({
  comment,
  userId,
  userRole,
  onAction
}) => {
  const handleAction = async (action: string, extraData = {}) => {
    try {
      const response = await fetch('/comments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action,
          user_id: userId,
          comment_id: comment.id,
          ...extraData
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      onAction(action, result);
    } catch (error) {
      console.error(`Failed to ${action} comment:`, error);
    }
  };

  return (
    <div className="moderation-actions">
      {!comment.deleted && (
        <button onClick={() => handleAction('delete')}>
          Delete
        </button>
      )}
      
      {comment.deleted && (
        <button onClick={() => handleAction('restore')}>
          Restore
        </button>
      )}
      
      <button onClick={() => handleAction(comment.pinned ? 'unpin' : 'pin')}>
        {comment.pinned ? 'Unpin' : 'Pin'}
      </button>
      
      <button onClick={() => handleAction(comment.locked ? 'unlock' : 'lock')}>
        {comment.locked ? 'Unlock' : 'Lock'}
      </button>
      
      <select onChange={(e) => {
        if (e.target.value) {
          handleAction('tag', { tag_type: e.target.value });
          e.target.value = '';
        }
      }}>
        <option value="">Add Tag...</option>
        <option value="spoiler">Spoiler</option>
        <option value="nsfw">NSFW</option>
        <option value="warning">Warning</option>
        <option value="offensive">Offensive</option>
        <option value="spam">Spam</option>
      </select>
    </div>
  );
};
```

## Rate Limiting

- **Comments**: 30 per hour per user (configurable)
- **Edits**: Included in comment rate limit
- **Moderation Actions**: Exempt from rate limits for moderators

## Security Considerations

- **Input Sanitization**: All content is sanitized before storage
- **Permission Checks**: All actions are validated against user permissions
- **Audit Logging**: All actions are logged for moderation review
- **Rate Limiting**: Prevents spam and abuse