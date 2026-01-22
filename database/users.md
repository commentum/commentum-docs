# 🔒 Users & Secure Session Management Schema

The user management system handles user accounts, **secure session management**, multi-platform identities, and user state management with **zero-trust architecture**.

## 🚨 **SECURITY UPDATE**

**Previous versions had a critical identity spoofing vulnerability. This has been completely fixed with session-based authentication.**

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

## 🔒 User Sessions Table (NEW - CRITICAL FOR SECURITY)

**Secure session management** that prevents identity spoofing attacks.

```sql
CREATE TABLE user_sessions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_token TEXT NOT NULL UNIQUE,
  client_type TEXT NOT NULL CHECK (client_type IN ('anilist', 'myanimelist', 'simkl')),
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Unique session identifier |
| `user_id` | INTEGER | Reference to the user account |
| `session_token` | TEXT | **Cryptographically secure session token** |
| `client_type` | TEXT | Platform used for authentication |
| `expires_at` | TIMESTAMPTZ | **Session expiration** (30 days) |
| `created_at` | TIMESTAMPTZ | Session creation time |
| `last_used_at` | TIMESTAMPTZ | Last activity timestamp |

### 🔒 Security Features

- **Cryptographic tokens** generated with `crypto.getRandomValues()`
- **Automatic expiration** with 30-day lifecycle
- **Server-side management** - no client control
- **Activity tracking** for security monitoring
- **Prevents identity spoofing** - user ID extracted from verified session

### Session Management Flow

1. **Token Verification**: Provider token verified with respective API
2. **Session Creation**: Cryptographically secure session token generated
3. **Session Storage**: Token stored with user association and expiry
4. **Session Validation**: Token verified on each API request
5. **Automatic Cleanup**: Expired sessions automatically removed

## User Identities Table (UPDATED)

External platform identities linked to user accounts.

```sql
CREATE TABLE user_identities (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  client_type TEXT NOT NULL CHECK (client_type IN ('anilist', 'myanimelist', 'simkl')),
  client_user_id TEXT NOT NULL,
  username TEXT NOT NULL,
  avatar_url TEXT,
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
| `created_at` | TIMESTAMPTZ | When this identity was linked |

### 🚨 **SECURITY UPDATE**

**REMOVED**: `verification_token` field - no longer needed  
**NEW**: Real-time token verification with provider APIs

### Supported Platforms

| Platform | Client Type | Verification Method |
|----------|-------------|-------------------|
| AniList | `anilist` | GraphQL API token verification |
| MyAnimeList | `myanimelist` | REST API token verification |
| SIMKL | `simkl` | User settings API verification |

### 🔒 Secure Identity Resolution Flow

1. **Provider Token Verification**: Token verified with real provider API
2. **User Lookup**: Find existing user by platform identity
3. **Session Creation**: Create secure session token
4. **User Association**: Link session to verified user
5. **Session Return**: Return session token to client

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

### 🔒 User Statistics (Session-Aware)

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

### 🔒 User Context Setting (Session-Based)

```sql
CREATE OR REPLACE FUNCTION set_user_context(user_id TEXT, user_role TEXT)
RETURNS void
```

Sets the current user context for Row Level Security (RLS) policies from verified session.

### 🔒 Session Cleanup Function

```sql
CREATE OR REPLACE FUNCTION cleanup_expired_sessions()
RETURNS INTEGER
```

Automatically removes expired sessions and returns count of cleaned sessions.

## Indexes (Updated for Security)

```sql
-- 🔒 Session management indexes
CREATE INDEX idx_user_sessions_token ON user_sessions(session_token);
CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);
CREATE INDEX idx_user_sessions_client_type ON user_sessions(client_type);

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

## 🔒 Row Level Security Policies (Enhanced)

### Users Table
- **View Own Profile**: Users can see their full profile
- **View Public Profiles**: Users can see limited info about other non-banned users
- **Update Own Profile**: Users can update their own profile (except role/ban status)
- **Service Role**: Full access for system operations

### 🔒 User Sessions Table (NEW)
- **Own Sessions Only**: Users can only see their own sessions
- **Session Validation**: Sessions validated against user status
- **Automatic Cleanup**: Expired sessions automatically filtered
- **Service Role**: Full access for session management

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

## 🔒 Usage Examples (Secure)

### Creating a New User with Secure Session

```sql
-- This is handled by the identity resolution function with session creation:
-- 1. Verify provider token with real API
-- 2. Create or find user account
-- 3. Create secure session token
-- 4. Return session token to client

-- Manual session creation (for debugging only):
INSERT INTO user_sessions (user_id, session_token, client_type, expires_at)
VALUES (1, 'cryptographically_secure_token', 'anilist', NOW() + INTERVAL '30 days');
```

### 🔒 Querying User Activity (Session-Aware)

```sql
-- Get comprehensive user statistics
SELECT * FROM get_user_statistics(123);

-- Get user's active warnings
SELECT * FROM user_warnings 
WHERE user_id = 123 AND active = true 
ORDER BY created_at DESC;

-- Get user's active sessions
SELECT * FROM user_sessions 
WHERE user_id = 123 AND expires_at > NOW()
ORDER BY last_used_at DESC;

-- Get user's appeal history
SELECT * FROM appeals 
WHERE user_id = 123 
ORDER BY created_at DESC;
```

### 🔒 Managing User States (Session-Verified)

```sql
-- Mute a user for 24 hours
UPDATE users 
SET muted_until = NOW() + INTERVAL '24 hours'
WHERE id = 123;

-- Ban a user (also invalidates all sessions)
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

## 🔒 Security Considerations (Enhanced)

- **🔒 Session-Based Authentication**: Prevents identity spoofing completely
- **Real Token Verification**: All provider tokens verified with respective APIs
- **Cryptographic Security**: Session tokens generated with secure random values
- **Automatic Session Expiry**: 30-day lifecycle with automatic cleanup
- **Zero-Trust Architecture**: No client-provided user data trusted
- **Rate Limiting**: Identity resolution is rate limited to prevent abuse
- **Audit Trail**: All user state changes are logged
- **Privacy**: User information is protected by RLS policies
- **Data Integrity**: Foreign key constraints ensure data consistency
- **Session Isolation**: Users can only access their own sessions
- **Activity Monitoring**: Last used time tracks session activity