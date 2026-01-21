# Users & Identities Schema

The user management system handles user accounts, multi-platform identities, and user state management.

## Users Table

Core user accounts that represent individual people regardless of how many platform identities they have.

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username TEXT NOT NULL,
  avatar_url TEXT,
  role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'moderator', 'admin', 'super_admin')),
  banned BOOLEAN DEFAULT FALSE,
  shadow_banned BOOLEAN DEFAULT FALSE,
  muted_until TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Unique identifier for the user |
| `username` | TEXT | Display username (can be changed) |
| `avatar_url` | TEXT | Profile image URL |
| `role` | TEXT | User permission level |
| `banned` | BOOLEAN | Hard ban status |
| `shadow_banned` | BOOLEAN | Soft ban status |
| `muted_until` | TIMESTAMPTZ | Temporary mute expiration |
| `created_at` | TIMESTAMPTZ | Account creation time |
| `updated_at` | TIMESTAMPTZ | Last profile update |

### Role Hierarchy

| Role | Level | Permissions |
|------|-------|-------------|
| `user` | 0 | Basic commenting and voting |
| `moderator` | 1 | Can warn, mute, pin, lock comments |
| `admin` | 2 | Can ban, shadow ban, manage moderators |
| `super_admin` | 3 | Full system control |

### User States

| State | Description | User Experience |
|-------|-------------|-----------------|
| `banned = false` | Normal user | Full access to all features |
| `banned = true` | Hard banned | Cannot interact with the system |
| `shadow_banned = true` | Shadow banned | Can interact but only they see their content |
| `muted_until > NOW()` | Temporarily muted | Cannot post comments but can view/vote |

## User Identities Table

External platform identities linked to user accounts.

```sql
CREATE TABLE user_identities (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  client_type TEXT NOT NULL CHECK (client_type IN ('anilist', 'myanimelist', 'simkl')),
  client_user_id TEXT NOT NULL,
  username TEXT NOT NULL,
  avatar_url TEXT,
  verification_token TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(client_type, client_user_id)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Unique identity identifier |
| `user_id` | INTEGER | Reference to the user account |
| `client_type` | TEXT | Platform type (anilist, myanimelist, simkl) |
| `client_user_id` | TEXT | User ID from the external platform |
| `username` | TEXT | Username from the external platform |
| `avatar_url` | TEXT | Avatar from the external platform |
| `verification_token` | TEXT | Token for identity verification |
| `created_at` | TIMESTAMPTZ | When this identity was linked |

### Supported Platforms

| Platform | Client Type | Description |
|----------|-------------|-------------|
| AniList | `anilist` | Anime and manga tracking platform |
| MyAnimeList | `myanimelist` | Large anime/manga community |
| SIMKL | `simkl` | Universal tracking platform |

### Identity Resolution Flow

1. **Check Existing**: Look for existing identity by `(client_type, client_user_id)`
2. **User Linking**: If no identity exists, check for existing user by username
3. **New User**: If no matching user, create new user account
4. **Link Identity**: Connect the platform identity to the user account

## User Warnings Table

Tracks warnings and penalties issued to users.

```sql
CREATE TABLE user_warnings (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  issued_by INTEGER REFERENCES users(id),
  reason TEXT NOT NULL,
  severity TEXT NOT NULL CHECK (severity IN ('warning', 'mute', 'ban')),
  expires_at TIMESTAMPTZ,
  active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Warning identifier |
| `user_id` | INTEGER | User who received the warning |
| `issued_by` | INTEGER | Moderator who issued the warning |
| `reason` | TEXT | Reason for the warning |
| `severity` | TEXT | Warning severity level |
| `expires_at` | TIMESTAMPTZ | When the warning expires |
| `active` | BOOLEAN | Whether the warning is currently active |
| `created_at` | TIMESTAMPTZ | When the warning was issued |

### Warning Severity

| Severity | Duration | Effect |
|----------|----------|--------|
| `warning` | 30 days | Recorded in user history |
| `mute` | Variable | User is muted for specified duration |
| `ban` | Permanent | User is banned until manually lifted |

## Appeals Table

User appeals for warnings and moderation actions.

```sql
CREATE TABLE appeals (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  warning_id INTEGER REFERENCES user_warnings(id),
  reason TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'reviewed', 'approved', 'denied')),
  reviewed_by INTEGER REFERENCES users(id),
  review_notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Appeal identifier |
| `user_id` | INTEGER | User filing the appeal |
| `warning_id` | INTEGER | Warning being appealed (optional) |
| `reason` | TEXT | User's appeal reason |
| `status` | TEXT | Current appeal status |
| `reviewed_by` | INTEGER | Admin who reviewed the appeal |
| `review_notes` | TEXT | Admin's review notes |
| `created_at` | TIMESTAMPTZ | Appeal submission time |
| `updated_at` | TIMESTAMPTZ | Last update time |

### Appeal Status Flow

1. **pending** → Awaiting review
2. **reviewed** → Under review by staff
3. **approved** → Appeal granted, action reversed
4. **denied** → Appeal denied, action stands

## Database Functions

### User Statistics

```sql
CREATE OR REPLACE FUNCTION get_user_statistics(target_user_id INTEGER)
RETURNS TABLE(
    total_comments INTEGER,
    total_votes_received INTEGER,
    total_upvotes_received INTEGER,
    total_downvotes_received INTEGER,
    total_votes_cast INTEGER,
    total_reports_created INTEGER,
    total_warnings_received INTEGER,
    active_warnings INTEGER,
    account_age_days INTEGER
)
```

Returns comprehensive statistics about a user's activity and history.

### User Context Setting

```sql
CREATE OR REPLACE FUNCTION set_user_context(user_id TEXT, user_role TEXT)
RETURNS void
```

Sets the current user context for Row Level Security (RLS) policies.

## Indexes

```sql
-- User identities lookups
CREATE INDEX idx_user_identities_user_id ON user_identities(user_id);
CREATE INDEX idx_user_identities_client ON user_identities(client_type, client_user_id);

-- Warning queries
CREATE INDEX idx_user_warnings_user_id ON user_warnings(user_id);
CREATE INDEX idx_user_warnings_active ON user_warnings(active);

-- Appeal management
CREATE INDEX idx_appeals_user_id ON appeals(user_id);
CREATE INDEX idx_appeals_status ON appeals(status);
```

## Row Level Security Policies

### Users Table
- **View Own Profile**: Users can see their full profile
- **View Public Profiles**: Users can see limited info about other non-banned users
- **Update Own Profile**: Users can update their own profile (except role/ban status)
- **Service Role**: Full access for system operations

### User Identities Table
- **View Own Identities**: Users can see their linked identities
- **View Public Identities**: Users can see identities of non-shadow-banned users
- **Service Role**: Full access for system operations

### User Warnings Table
- **View Own Warnings**: Users can see warnings issued to them
- **Moderator Access**: Moderators can see all warnings
- **Service Role**: Full access for system operations

### Appeals Table
- **View Own Appeals**: Users can see their appeals
- **Create Appeals**: Users can create appeals
- **Moderator Access**: Moderators can view and manage appeals
- **Service Role**: Full access for system operations

## Usage Examples

### Creating a New User with Identity

```sql
-- This is typically handled by the identity resolution function
-- but here's the manual process:

-- 1. Create user account
INSERT INTO users (username, role) VALUES ('new_user', 'user')
RETURNING id;

-- 2. Link platform identity
INSERT INTO user_identities (user_id, client_type, client_user_id, username, avatar_url)
VALUES (1, 'anilist', '12345', 'new_user', 'https://example.com/avatar.jpg');
```

### Querying User Activity

```sql
-- Get comprehensive user statistics
SELECT * FROM get_user_statistics(123);

-- Get user's active warnings
SELECT * FROM user_warnings 
WHERE user_id = 123 AND active = true 
ORDER BY created_at DESC;

-- Get user's appeal history
SELECT * FROM appeals 
WHERE user_id = 123 
ORDER BY created_at DESC;
```

### Managing User States

```sql
-- Mute a user for 24 hours
UPDATE users 
SET muted_until = NOW() + INTERVAL '24 hours'
WHERE id = 123;

-- Ban a user
UPDATE users 
SET banned = true 
WHERE id = 123;

-- Shadow ban a user
UPDATE users 
SET shadow_banned = true 
WHERE id = 123;

-- Issue a warning
INSERT INTO user_warnings (user_id, issued_by, reason, severity)
VALUES (123, 456, 'Spam behavior', 'warning');
```

## Security Considerations

- **Identity Verification**: Users can only link identities they own
- **Rate Limiting**: Identity resolution is rate limited to prevent abuse
- **Audit Trail**: All user state changes are logged
- **Privacy**: User information is protected by RLS policies
- **Data Integrity**: Foreign key constraints ensure data consistency