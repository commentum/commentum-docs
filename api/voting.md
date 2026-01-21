# Voting API

The Voting API handles upvote/downvote functionality for comments with built-in abuse detection and rate limiting.

## Endpoint

```
POST /voting
```

## Request Body

```typescript
interface VoteRequest {
  action: 'upvote' | 'downvote' | 'remove'
  user_id: number
  comment_id: number
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Voting action to perform |
| `user_id` | number | Yes | ID of the user voting |
| `comment_id` | number | Yes | ID of the comment being voted on |

## Actions

### 1. Upvote Comment

**Required Parameters:** `action: 'upvote'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "upvote",
  "user_id": 123,
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

**Required Parameters:** `action: 'downvote'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "downvote",
  "user_id": 123,
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

**Required Parameters:** `action: 'remove'`, `user_id`, `comment_id`

#### Request
```json
{
  "action": "remove",
  "user_id": 123,
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
  "user_id": 123,
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
  "error": "User not found"
}
```

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

### React Voting Component

```typescript
import { useState, useEffect } from 'react';

interface VotingComponentProps {
  commentId: number;
  userId: number;
  initialVotes?: { upvotes: number; downvotes: number; userVote: number };
}

export const VotingComponent: React.FC<VotingComponentProps> = ({
  commentId,
  userId,
  initialVotes = { upvotes: 0, downvotes: 0, userVote: 0 }
}) => {
  const [votes, setVotes] = useState(initialVotes);
  const [voting, setVoting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleVote = async (action: 'upvote' | 'downvote' | 'remove') => {
    setVoting(true);
    setError(null);

    try {
      const response = await fetch('/voting', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action,
          user_id: userId,
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

### Vote Management Hook

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
  userId: number,
  initialVotes: { upvotes: number; downvotes: number; userVote: number }
): UseVotingResult => {
  const [votes, setVotes] = useState(initialVotes);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const vote = useCallback(async (action: 'upvote' | 'downvote' | 'remove') => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch('/voting', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action,
          user_id: userId,
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
  }, [commentId, userId]);

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

## Security Considerations

- **Self-Vote Prevention**: Users cannot vote on their own comments
- **Abuse Tracking**: Suspicious voting patterns are logged and tracked
- **Rate Limiting**: Prevents vote spamming and manipulation
- **Audit Trail**: All voting actions are logged for moderation review

## Performance Considerations

- **Vote Counting**: Use database functions for efficient vote aggregation
- **Caching**: Consider caching vote counts for popular comments
- **Batch Operations**: Load vote data for multiple comments efficiently
- **Real-time Updates**: Consider WebSocket integration for live vote updates