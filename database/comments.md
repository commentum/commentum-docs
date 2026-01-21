# Comments System Schema

The comments system handles threaded discussions, content management, and comment metadata.

## Media Items Table

Represents media content that can be commented on (anime, manga, movies, etc.).

```sql
CREATE TABLE media_items (
  id SERIAL PRIMARY KEY,
  external_id TEXT NOT NULL,
  media_type TEXT NOT NULL CHECK (media_type IN ('anime', 'manga', 'movie', 'tv', 'other')),
  title TEXT NOT NULL,
  year INTEGER,
  poster_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(external_id, media_type)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Internal media identifier |
| `external_id` | TEXT | External platform ID (e.g., AniList ID) |
| `media_type` | TEXT | Type of media content |
| `title` | TEXT | Media title |
| `year` | INTEGER | Release year |
| `poster_url` | TEXT | Poster/thumbnail image URL |
| `created_at` | TIMESTAMPTZ | When media was added to system |

### Media Types

| Type | Description | Example |
|------|-------------|---------|
| `anime` | Japanese animation | "Attack on Titan" |
| `manga` | Japanese comics | "One Piece" |
| `movie` | Feature films | "Inception" |
| `tv` | Television series | "Breaking Bad" |
| `other` | Other media types | Podcasts, games, etc. |

## Comments Table

Core comments table with full threading support.

```sql
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  media_id INTEGER NOT NULL REFERENCES media_items(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  parent_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  content_html TEXT,
  deleted BOOLEAN DEFAULT FALSE,
  deleted_at TIMESTAMPTZ,
  deleted_by INTEGER REFERENCES users(id),
  pinned BOOLEAN DEFAULT FALSE,
  pinned_at TIMESTAMPTZ,
  pinned_by INTEGER REFERENCES users(id),
  locked BOOLEAN DEFAULT FALSE,
  locked_at TIMESTAMPTZ,
  locked_by INTEGER REFERENCES users(id),
  edited BOOLEAN DEFAULT FALSE,
  edited_at TIMESTAMPTZ,
  edit_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Unique comment identifier |
| `media_id` | INTEGER | Media item being commented on |
| `user_id` | INTEGER | Author of the comment |
| `parent_id` | INTEGER | Parent comment for threading |
| `content` | TEXT | Plain text comment content |
| `content_html` | TEXT | Processed HTML content |
| `deleted` | BOOLEAN | Soft delete status |
| `deleted_at` | TIMESTAMPTZ | When comment was deleted |
| `deleted_by` | INTEGER | Who deleted the comment |
| `pinned` | BOOLEAN | Pinned status (shows first) |
| `pinned_at` | TIMESTAMPTZ | When comment was pinned |
| `pinned_by` | INTEGER | Who pinned the comment |
| `locked` | BOOLEAN | Thread locked status |
| `locked_at` | TIMESTAMPTZ | When thread was locked |
| `locked_by` | INTEGER | Who locked the thread |
| `edited` | BOOLEAN | Edit status |
| `edited_at` | TIMESTAMPTZ | Last edit time |
| `edit_count` | INTEGER | Number of times edited |
| `created_at` | TIMESTAMPTZ | Comment creation time |
| `updated_at` | TIMESTAMPTZ | Last update time |

### Comment States

| State | Description | User Experience |
|-------|-------------|-----------------|
| `deleted = false` | Active comment | Visible to all users |
| `deleted = true` | Deleted comment | Hidden from normal view |
| `pinned = true` | Pinned comment | Shows at top of thread |
| `locked = true` | Locked thread | No replies allowed |
| `edited = true` | Edited comment | Shows edit indicator |

## Comment Edits Table

Tracks edit history for comments.

```sql
CREATE TABLE comment_edits (
  id SERIAL PRIMARY KEY,
  comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  old_content TEXT NOT NULL,
  new_content TEXT NOT NULL,
  edited_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Edit record identifier |
| `comment_id` | INTEGER | Comment being edited |
| `user_id` | INTEGER | User who made the edit |
| `old_content` | TEXT | Content before edit |
| `new_content` | TEXT | Content after edit |
| `edited_at` | TIMESTAMPTZ | When the edit occurred |

## Comment Tags Table

Content warnings and metadata tags for comments.

```sql
CREATE TABLE comment_tags (
  id SERIAL PRIMARY KEY,
  comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  tag_type TEXT NOT NULL CHECK (tag_type IN ('spoiler', 'nsfw', 'warning', 'offensive', 'spam')),
  tagged_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(comment_id, tag_type)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Tag identifier |
| `comment_id` | INTEGER | Comment being tagged |
| `tag_type` | TEXT | Type of tag |
| `tagged_by` | INTEGER | Who applied the tag |
| `created_at` | TIMESTAMPTZ | When tag was applied |

### Tag Types

| Tag | Purpose | Display |
|-----|---------|---------|
| `spoiler` | Contains spoilers | Hidden behind click |
| `nsfw` | Not safe for work | Warning before showing |
| `warning` | Content warning | General warning |
| `offensive` | Offensive content | Strong warning |
| `spam` | Spam content | May be auto-hidden |

## Database Functions

### Comment Thread Retrieval

```sql
CREATE OR REPLACE FUNCTION get_comment_thread(media_id INTEGER, p_parent_id INTEGER DEFAULT NULL)
RETURNS TABLE(
    id INTEGER,
    content TEXT,
    content_html TEXT,
    user_id INTEGER,
    username TEXT,
    avatar_url TEXT,
    parent_id INTEGER,
    created_at TIMESTAMPTZ,
    edited BOOLEAN,
    edited_at TIMESTAMPTZ,
    edit_count INTEGER,
    deleted BOOLEAN,
    pinned BOOLEAN,
    locked BOOLEAN,
    upvotes INTEGER,
    downvotes INTEGER,
    total_votes INTEGER,
    user_vote SMALLINT,
    reply_count INTEGER,
    tags TEXT[]
)
```

Retrieves a complete comment thread with all metadata, votes, and tags.

### Media Item Creation

```sql
CREATE OR REPLACE FUNCTION create_media_item_if_not_exists(
    p_external_id TEXT,
    p_media_type TEXT,
    p_title TEXT,
    p_year INTEGER DEFAULT NULL,
    p_poster_url TEXT DEFAULT NULL
)
RETURNS INTEGER
```

Creates a media item if it doesn't exist, returns the media ID.

## Indexes

```sql
-- Comment queries
CREATE INDEX idx_comments_media_id ON comments(media_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);
CREATE INDEX idx_comments_created_at ON comments(created_at);
CREATE INDEX idx_comments_deleted ON comments(deleted);
CREATE INDEX idx_comments_pinned ON comments(pinned);

-- Edit history
CREATE INDEX idx_comment_edits_comment_id ON comment_edits(comment_id);
CREATE INDEX idx_comment_edits_user_id ON comment_edits(user_id);

-- Tags
CREATE INDEX idx_comment_tags_comment_id ON comment_tags(comment_id);
CREATE INDEX idx_comment_tags_type ON comment_tags(tag_type);
```

## Triggers

### Edit Tracking

```sql
CREATE OR REPLACE FUNCTION track_comment_edit()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.content IS DISTINCT FROM NEW.content THEN
        INSERT INTO comment_edits (comment_id, user_id, old_content, new_content)
        VALUES (NEW.id, NEW.user_id, OLD.content, NEW.content);
        
        NEW.edited = TRUE;
        NEW.edited_at = NOW();
        NEW.edit_count = OLD.edit_count + 1;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Automatically tracks comment edits and updates metadata.

## Row Level Security Policies

### Comments Table
- **View Comments**: Users can see non-deleted comments (except from shadow-banned users)
- **Shadow-banned View**: Shadow-banned users can only see their own comments
- **Create Comments**: Users can create comments if not banned/muted
- **Edit Comments**: Users can edit their own non-locked comments
- **Delete Comments**: Users can soft-delete their own non-locked comments
- **Moderator Access**: Moderators can perform all actions
- **Service Role**: Full system access

### Comment Edits Table
- **View Own Edits**: Users can see edit history of their comments
- **Moderator Access**: Moderators can see all edit history
- **Service Role**: Full system access

### Comment Tags Table
- **View Tags**: All users can see comment tags
- **Manage Tags**: Moderators can add/remove tags
- **Service Role**: Full system access

### Media Items Table
- **View Media**: All users can view media items
- **Service Role**: Full system access

## Usage Examples

### Creating a Comment Thread

```sql
-- Create media item (if not exists)
SELECT create_media_item_if_not_exists('12345', 'anime', 'Attack on Titan', 2013);

-- Create top-level comment
INSERT INTO comments (media_id, user_id, content)
VALUES (1, 123, 'This is an amazing anime!');

-- Create reply
INSERT INTO comments (media_id, user_id, parent_id, content)
VALUES (1, 456, 1, 'I agree, the animation is incredible!');
```

### Querying Comment Threads

```sql
-- Get top-level comments for a media item
SELECT * FROM get_comment_thread(1, NULL)
ORDER BY pinned DESC, created_at ASC;

-- Get replies to a specific comment
SELECT * FROM get_comment_thread(1, 1)
ORDER BY created_at ASC;
```

### Managing Comment States

```sql
-- Pin a comment
UPDATE comments 
SET pinned = true, pinned_at = NOW(), pinned_by = 789
WHERE id = 1;

-- Lock a thread
UPDATE comments 
SET locked = true, locked_at = NOW(), locked_by = 789
WHERE id = 1;

-- Soft delete a comment
UPDATE comments 
SET deleted = true, deleted_at = NOW(), deleted_by = 123
WHERE id = 2;

-- Add a spoiler tag
INSERT INTO comment_tags (comment_id, tag_type, tagged_by)
VALUES (1, 'spoiler', 789);
```

### Comment Moderation

```sql
-- Get edit history for a comment
SELECT * FROM comment_edits 
WHERE comment_id = 1 
ORDER BY edited_at DESC;

-- Get all tags for a comment
SELECT * FROM comment_tags 
WHERE comment_id = 1;

-- Find comments by a user
SELECT * FROM comments 
WHERE user_id = 123 AND deleted = false
ORDER BY created_at DESC;
```

## Performance Considerations

- **Threaded Queries**: Use the `get_comment_thread` function for optimal performance
- **Index Strategy**: Indexes support common query patterns (media, user, threading)
- **Soft Deletes**: Deleted comments are filtered out at the database level
- **Vote Aggregation**: Vote counts are calculated efficiently in the thread function

## Security Considerations

- **Content Sanitization**: Content should be sanitized before storage
- **Edit Tracking**: All edits are logged for accountability
- **Access Control**: RLS policies ensure users can only see appropriate content
- **Rate Limiting**: Comment creation is rate limited to prevent spam