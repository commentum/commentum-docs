# 🔒 Voting API - Session-Based Authentication

The Voting API handles upvote/downvote functionality for comments with built-in abuse detection and rate limiting using secure session-based authentication.

## 🔒 Security Overview

**CRITICAL**: This API uses session-based authentication. The `user_id` parameter has been removed to prevent identity spoofing. User identity is extracted from the session token.

## Endpoint

```
POST /functions/v1/voting
```

## Request Headers

```http
Authorization: Bearer <session_token>
Content-Type: application/json
```

## Request Body

```typescript
interface VoteRequest {
  action: 'upvote' | 'downvote' | 'remove'
  comment_id: number
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Voting action to perform |
| `comment_id` | number | Yes | ID of the comment being voted on |

## Actions

### 1. Upvote Comment

**Required Parameters:** `action: 'upvote'`, `comment_id`

#### Request
```json
{
  "action": "upvote",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 2001,
  "comment_id": 1001,
  "user_id": 123,
  "vote_type": 1,
  "created_at": "2024-01-15T14:00:00Z",
  "updated_at": "2024-01-15T14:00:00Z",
  "action": "created"
}
```

### 2. Downvote Comment

**Required Parameters:** `action: 'downvote'`, `comment_id`

#### Request
```json
{
  "action": "downvote",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 2002,
  "comment_id": 1001,
  "user_id": 123,
  "vote_type": -1,
  "created_at": "2024-01-15T14:05:00Z",
  "updated_at": "2024-01-15T14:05:00Z",
  "action": "created"
}
```

### 3. Remove Vote

**Required Parameters:** `action: 'remove'`, `comment_id`

#### Request
```json
{
  "action": "remove",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "action": "removed"
}
```

### 4. Change Vote

If a user votes on a comment they've already voted on, the vote is updated:

#### Request (changing from upvote to downvote)
```json
{
  "action": "downvote",
  "comment_id": 1001
}
```

#### Response (200 OK)
```json
{
  "id": 2001,
  "comment_id": 1001,
  "user_id": 123,
  "vote_type": -1,
  "created_at": "2024-01-15T14:00:00Z",
  "updated_at": "2024-01-15T14:10:00Z",
  "action": "updated"
}
```

## Error Responses

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
  "error": "User banned"
}
```

```json
{
  "error": "Cannot vote on own comment"
}
```

### 404 Not Found
```json
{
  "error": "Comment not found"
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
const response = await fetch('/functions/v1/voting', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    action: 'upvote',
    user_id: 123,  // Could be faked!
    comment_id: 1001
  })
})

// ✅ NEW (Secure)
const sessionToken = localStorage.getItem('commentum_session_token')
const response = await fetch('/functions/v1/voting', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${sessionToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    action: 'upvote',
    comment_id: 1001  // No user_id needed!
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

## Voting Rules

### Permission Validation
- **Banned Users**: Cannot vote on any comments
- **Self-Voting**: Users cannot vote on their own comments
- **Comment Status**: Can only vote on non-deleted comments

### Vote Types
- **Upvote**: `vote_type = 1`
- **Downvote**: `vote_type = -1`

### Abuse Detection
The system automatically tracks and prevents:
- **Self-Vote Manipulation**: Attempts to vote on own comments
- **Rapid Voting**: Excessive voting in short time periods
- **Brigading**: Coordinated voting patterns
- **Vote Reversal Abuse**: Frequent vote changing

## Rate Limiting

- **Votes**: 100 per hour per user (configurable)
- **Super Admin**: Exempt from rate limits
- **Vote Removal**: Not counted towards rate limit

## Usage Examples

### React Voting Component with Session Management

```typescript
import { useState, useEffect } from 'react';

interface VotingComponentProps {
  commentId: number;
  initialVotes?: { upvotes: number; downvotes: number; userVote: number };
}

export const VotingComponent: React.FC<VotingComponentProps> = ({
  commentId,
  initialVotes = { upvotes: 0, downvotes: 0, userVote: 0 }
}) => {
  const [votes, setVotes] = useState(initialVotes);
  const [voting, setVoting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleVote = async (action: 'upvote' | 'downvote' | 'remove') => {
    setVoting(true);
    setError(null);

    try {
      const sessionToken = localStorage.getItem('commentum_session_token');
      if (!sessionToken) {
        throw new Error('Not authenticated');
      }

      const response = await fetch('/functions/v1/voting', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${sessionToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          action,
          comment_id: commentId
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      // Update local state based on the action
      setVotes(prev => {
        const newVotes = { ...prev };
        
        if (action === 'remove') {
          if (prev.userVote === 1) newVotes.upvotes--;
          if (prev.userVote === -1) newVotes.downvotes--;
          newVotes.userVote = 0;
        } else if (action === 'upvote') {
          if (prev.userVote === -1) newVotes.downvotes--;
          if (prev.userVote !== 1) newVotes.upvotes++;
          newVotes.userVote = 1;
        } else if (action === 'downvote') {
          if (prev.userVote === 1) newVotes.upvotes--;
          if (prev.userVote !== -1) newVotes.downvotes++;
          newVotes.userVote = -1;
        }
        
        return newVotes;
      });
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Voting failed');
    } finally {
      setVoting(false);
    }
  };

  return (
    <div className="voting-component">
      <button
        onClick={() => handleVote(votes.userVote === 1 ? 'remove' : 'upvote')}
        disabled={voting}
        className={`vote-button ${votes.userVote === 1 ? 'voted' : ''}`}
      >
        ▲ {votes.upvotes}
      </button>
      
      <button
        onClick={() => handleVote(votes.userVote === -1 ? 'remove' : 'downvote')}
        disabled={voting}
        className={`vote-button ${votes.userVote === -1 ? 'voted' : ''}`}
      >
        ▼ {votes.downvotes}
      </button>
      
      {error && <div className="error">{error}</div>}
    </div>
  );
};
```

### Vote Management Hook with Session Auth

```typescript
import { useState, useCallback } from 'react';

interface UseVotingResult {
  upvote: () => Promise<void>;
  downvote: () => Promise<void>;
  removeVote: () => Promise<void>;
  votes: { upvotes: number; downvotes: number; userVote: number };
  loading: boolean;
  error: string | null;
}

export const useVoting = (
  commentId: number,
  initialVotes: { upvotes: number; downvotes: number; userVote: number }
): UseVotingResult => {
  const [votes, setVotes] = useState(initialVotes);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const vote = useCallback(async (action: 'upvote' | 'downvote' | 'remove') => {
    setLoading(true);
    setError(null);

    try {
      const sessionToken = localStorage.getItem('commentum_session_token');
      if (!sessionToken) {
        throw new Error('Not authenticated');
      }

      const response = await fetch('/functions/v1/voting', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${sessionToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          action,
          comment_id: commentId
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      // Update votes based on response
      if (action === 'remove') {
        setVotes(prev => ({
          ...prev,
          upvotes: prev.userVote === 1 ? prev.upvotes - 1 : prev.upvotes,
          downvotes: prev.userVote === -1 ? prev.downvotes - 1 : prev.downvotes,
          userVote: 0
        }));
      } else {
        const voteType = action === 'upvote' ? 1 : -1;
        setVotes(prev => {
          const newVotes = { ...prev };
          
          // Remove previous vote if different
          if (prev.userVote === 1 && voteType === -1) {
            newVotes.upvotes--;
          } else if (prev.userVote === -1 && voteType === 1) {
            newVotes.downvotes--;
          }
          
          // Add new vote if different from previous
          if (prev.userVote !== voteType) {
            if (voteType === 1) newVotes.upvotes++;
            else newVotes.downvotes++;
          }
          
          newVotes.userVote = prev.userVote === voteType ? 0 : voteType;
          return newVotes;
        });
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Voting failed');
    } finally {
      setLoading(false);
    }
  }, [commentId]);

  return {
    upvote: () => vote('upvote'),
    downvote: () => vote('downvote'),
    removeVote: () => vote('remove'),
    votes,
    loading,
    error
  };
};
```

### Vanilla JavaScript API Client with Session Auth

```javascript
class CommentumAPI {
  constructor(sessionToken) {
    this.sessionToken = sessionToken
  }

  async vote(commentId, action) {
    return this.makeRequest('/functions/v1/voting', {
      method: 'POST',
      body: JSON.stringify({
        action,
        comment_id: commentId
      })
    })
  }

  async upvote(commentId) {
    return this.vote(commentId, 'upvote')
  }

  async downvote(commentId) {
    return this.vote(commentId, 'downvote')
  }

  async removeVote(commentId) {
    return this.vote(commentId, 'remove')
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

// Vote on a comment
try {
  const result = await api.upvote(1001)
  console.log('Vote successful:', result)
} catch (error) {
  console.error('Voting error:', error.message)
}
```

### Batch Vote Loading

```typescript
// Load vote counts for multiple comments at once
export const loadVoteCounts = async (commentIds: number[]) => {
  const response = await fetch('/vote-counts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ comment_ids: commentIds })
  });

  if (!response.ok) {
    throw new Error('Failed to load vote counts');
  }

  return await response.json();
};

// Usage in a comment list component
const CommentList = ({ comments, userId }) => {
  const [voteData, setVoteData] = useState({});
  
  useEffect(() => {
    const commentIds = comments.map(c => c.id);
    loadVoteCounts(commentIds).then(setVoteData);
  }, [comments]);

  return comments.map(comment => (
    <div key={comment.id}>
      <p>{comment.content}</p>
      <VotingComponent
        commentId={comment.id}
        userId={userId}
        initialVotes={voteData[comment.id] || { upvotes: 0, downvotes: 0, userVote: 0 }}
      />
    </div>
  ));
};
```

## 🔒 Security Considerations

- **Session Authentication**: All requests require valid session tokens
- **Zero-Trust Architecture**: No client-provided user data is trusted
- **Self-Vote Prevention**: Users cannot vote on their own comments
- **Abuse Tracking**: Suspicious voting patterns are logged and tracked
- **Rate Limiting**: Prevents vote spamming and manipulation
- **Audit Trail**: All voting actions are logged for moderation review

## 🚨 Breaking Changes from Previous Version

### Before (Vulnerable)
```javascript
// ❌ Client could send any user_id
fetch('/functions/v1/voting', {
  body: JSON.stringify({
    action: 'upvote',
    user_id: 123,  // Could be faked!
    comment_id: 1001
  })
})
```

### After (Secure)
```javascript
// ✅ User ID extracted from session
fetch('/functions/v1/voting', {
  headers: { 'Authorization': `Bearer ${sessionToken}` },
  body: JSON.stringify({
    action: 'upvote',
    comment_id: 1001  // No user_id needed!
  })
})
```

This secure API ensures that voting operations can only be performed by authenticated users while maintaining robust abuse detection and prevention mechanisms.

## Performance Considerations

- **Vote Counting**: Use database functions for efficient vote aggregation
- **Caching**: Consider caching vote counts for popular comments
- **Batch Operations**: Load vote data for multiple comments efficiently
- **Real-time Updates**: Consider WebSocket integration for live vote updates