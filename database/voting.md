# Voting System Schema

The voting system handles comment upvotes/downvotes with abuse detection and vote tracking.

## Comment Votes Table

Tracks individual user votes on comments.

```sql
CREATE TABLE comment_votes (
  id SERIAL PRIMARY KEY,
  comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  vote_type SMALLINT NOT NULL CHECK (vote_type IN (-1, 1)),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(comment_id, user_id)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Vote identifier |
| `comment_id` | INTEGER | Comment being voted on |
| `user_id` | INTEGER | User casting the vote |
| `vote_type` | SMALLINT | Vote type: 1 (upvote) or -1 (downvote) |
| `created_at` | TIMESTAMPTZ | When vote was cast |
| `updated_at` | TIMESTAMPTZ | When vote was last changed |

### Vote Types

| Type | Value | Description |
|------|-------|-------------|
| Upvote | 1 | Positive vote on comment |
| Downvote | -1 | Negative vote on comment |

### Voting Behavior

- **One Vote Per Comment**: Users can only have one active vote per comment
- **Vote Changing**: Users can change their vote type (upvote ↔ downvote)
- **Vote Removal**: Users can remove their vote entirely
- **Self-Voting**: Users cannot vote on their own comments

## Vote Abuse Tracking Table

Detects and tracks suspicious voting patterns.

```sql
CREATE TABLE vote_abuse_tracking (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  target_comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  abuse_type TEXT NOT NULL CHECK (abuse_type IN ('rapid_voting', 'brigading', 'self_vote_manipulation')),
  detected_at TIMESTAMPTZ DEFAULT NOW(),
  severity INTEGER DEFAULT 1 CHECK (severity BETWEEN 1 AND 5)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Abuse record identifier |
| `user_id` | INTEGER | User exhibiting abusive behavior |
| `target_comment_id` | INTEGER | Comment being abused |
| `abuse_type` | TEXT | Type of abuse detected |
| `detected_at` | TIMESTAMPTZ | When abuse was detected |
| `severity` | INTEGER | Abuse severity level (1-5) |

### Abuse Types

| Type | Description | Detection Method |
|------|-------------|------------------|
| `rapid_voting` | Excessive voting in short time | Rate limit analysis |
| `brigading` | Coordinated voting patterns | IP/user agent analysis |
| `self_vote_manipulation` | Attempts to vote on own comments | Direct prevention |

### Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| 1 | Minor suspicious activity | Log only |
| 2 | Moderate concern | Flag for review |
| 3 | High concern | Temporary voting restriction |
| 4 | Severe abuse | Voting suspension |
| 5 | Critical abuse | Account review |

## Database Functions

### Vote Count Aggregation

```sql
CREATE OR REPLACE FUNCTION get_comment_vote_counts(comment_id INTEGER)
RETURNS TABLE(upvotes INTEGER, downvotes INTEGER, total INTEGER)
```

Calculates vote counts for a specific comment.

### User Vote Retrieval

```sql
CREATE OR REPLACE FUNCTION get_user_comment_vote(user_id INTEGER, comment_id INTEGER)
RETURNS SMALLINT
```

Gets a specific user's vote on a comment (returns 0 if no vote).

## Indexes

```sql
-- Vote lookups and aggregation
CREATE INDEX idx_comment_votes_comment_id ON comment_votes(comment_id);
CREATE INDEX idx_comment_votes_user_id ON comment_votes(user_id);
CREATE INDEX idx_comment_votes_type ON comment_votes(vote_type);

-- Abuse tracking
CREATE INDEX idx_vote_abuse_user_id ON vote_abuse_tracking(user_id);
CREATE INDEX idx_vote_abuse_type ON vote_abuse_tracking(abuse_type);
CREATE INDEX idx_vote_abuse_detected_at ON vote_abuse_tracking(detected_at);
```

## Row Level Security Policies

### Comment Votes Table
- **View Vote Counts**: Users can see vote totals (but not who voted)
- **Vote Creation**: Users can vote if not banned
- **Vote Modification**: Users can change their own votes
- **Vote Deletion**: Users can remove their own votes
- **Service Role**: Full system access

### Vote Abuse Tracking Table
- **Service Role Only**: Internal system use only
- **No User Access**: Users cannot see abuse tracking records

## Usage Examples

### Basic Voting Operations

```sql
-- Upvote a comment
INSERT INTO comment_votes (comment_id, user_id, vote_type)
VALUES (1, 123, 1)
ON CONFLICT (comment_id, user_id) 
DO UPDATE SET vote_type = 1, updated_at = NOW();

-- Downvote a comment
INSERT INTO comment_votes (comment_id, user_id, vote_type)
VALUES (1, 123, -1)
ON CONFLICT (comment_id, user_id) 
DO UPDATE SET vote_type = -1, updated_at = NOW();

-- Remove vote
DELETE FROM comment_votes 
WHERE comment_id = 1 AND user_id = 123;
```

### Vote Analytics

```sql
-- Get vote counts for a comment
SELECT * FROM get_comment_vote_counts(1);

-- Get user's vote on a comment
SELECT get_user_comment_vote(123, 1);

-- Get most upvoted comments
SELECT 
    c.id,
    c.content,
    COALESCE(SUM(cv.vote_type), 0) as total_votes,
    COUNT(CASE WHEN cv.vote_type = 1 THEN 1 END) as upvotes,
    COUNT(CASE WHEN cv.vote_type = -1 THEN 1 END) as downvotes
FROM comments c
LEFT JOIN comment_votes cv ON c.id = cv.comment_id
WHERE c.media_id = 1 AND c.deleted = false
GROUP BY c.id, c.content
ORDER BY total_votes DESC
LIMIT 10;
```

### Abuse Detection

```sql
-- Track rapid voting
INSERT INTO vote_abuse_tracking (user_id, target_comment_id, abuse_type, severity)
VALUES (123, 1, 'rapid_voting', 2);

-- Find users with suspicious voting patterns
SELECT 
    user_id,
    COUNT(*) as vote_count,
    COUNT(DISTINCT comment_id) as unique_comments,
    MIN(created_at) as first_vote,
    MAX(created_at) as last_vote
FROM comment_votes 
WHERE created_at > NOW() - INTERVAL '1 hour'
GROUP BY user_id
HAVING COUNT(*) > 50  -- More than 50 votes in 1 hour
ORDER BY vote_count DESC;
```

### Vote Statistics

```sql
-- User voting activity
SELECT 
    u.username,
    COUNT(CASE WHEN cv.vote_type = 1 THEN 1 END) as upvotes_cast,
    COUNT(CASE WHEN cv.vote_type = -1 THEN 1 END) as downvotes_cast,
    COUNT(*) as total_votes,
    COUNT(DISTINCT cv.comment_id) as unique_comments_voted
FROM users u
LEFT JOIN comment_votes cv ON u.id = cv.user_id
WHERE u.id = 123
GROUP BY u.id, u.username;

-- Comment vote distribution
SELECT 
    c.id,
    c.content,
    COUNT(CASE WHEN cv.vote_type = 1 THEN 1 END) as upvotes,
    COUNT(CASE WHEN cv.vote_type = -1 THEN 1 END) as downvotes,
    COUNT(cv.id) as total_votes,
    ROUND(
        COUNT(CASE WHEN cv.vote_type = 1 THEN 1 END)::numeric / 
        NULLIF(COUNT(cv.id), 0) * 100, 2
    ) as upvote_percentage
FROM comments c
LEFT JOIN comment_votes cv ON c.id = cv.comment_id
WHERE c.media_id = 1
GROUP BY c.id, c.content
ORDER BY total_votes DESC;
```

## Performance Considerations

### Query Optimization

```sql
-- Efficient vote count calculation
WITH vote_counts AS (
    SELECT 
        comment_id,
        COUNT(CASE WHEN vote_type = 1 THEN 1 END) as upvotes,
        COUNT(CASE WHEN vote_type = -1 THEN 1 END) as downvotes,
        SUM(vote_type) as total
    FROM comment_votes
    GROUP BY comment_id
)
SELECT 
    c.*,
    COALESCE(vc.upvotes, 0) as upvotes,
    COALESCE(vc.downvotes, 0) as downvotes,
    COALESCE(vc.total, 0) as total_votes
FROM comments c
LEFT JOIN vote_counts vc ON c.id = vc.comment_id
WHERE c.media_id = 1;
```

### Caching Strategy

- **Vote Counts**: Cache vote counts for popular comments
- **User Votes**: Cache user's vote on comments they've viewed
- **Trending**: Maintain cached lists of trending comments
- **Abuse Metrics**: Cache abuse detection results

## Security Considerations

### Vote Manipulation Prevention

```sql
-- Prevent self-voting (handled in application layer)
-- This is a pseudo-constraint shown for documentation
ALTER TABLE comment_votes 
ADD CONSTRAINT no_self_voting 
CHECK (
    user_id NOT IN (
        SELECT user_id FROM comments WHERE id = comment_id
    )
);
```

### Rate Limiting Integration

```sql
-- Vote rate limiting query
SELECT 
    user_id,
    COUNT(*) as vote_count,
    COUNT(DISTINCT DATE_TRUNC('hour', created_at)) as active_hours
FROM comment_votes 
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY user_id
HAVING COUNT(*) > 100;  -- Exceeds rate limit
```

### Privacy Protection

- **Vote Anonymity**: Users cannot see who voted on comments
- **Aggregate Data**: Only aggregate vote statistics are public
- **Abuse Privacy**: Abuse tracking is internal-only
- **Audit Trail**: All voting actions are logged for security

## Integration Points

### With Comment System

- **Comment Score**: Vote counts affect comment sorting and visibility
- **Comment Quality**: High-voted comments may be highlighted
- **User Reputation**: Voting patterns contribute to user trust scores

### With Moderation System

- **Abuse Detection**: Suspicious voting triggers moderation review
- **Vote Brigading**: Coordinated voting patterns are flagged
- **User Penalties**: Vote abuse can result in voting restrictions

### With User System

- **User Statistics**: Voting activity contributes to user profiles
- **Trust Scores**: Consistent voting behavior builds trust
- **Permissions**: Abusive voting can affect user permissions