# Moderation System Schema

The moderation system handles reports, warnings, moderation actions, and automated moderation.

## Comment Reports Table

User-generated reports for comment moderation.

```sql
CREATE TABLE comment_reports (
  id SERIAL PRIMARY KEY,
  comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  reporter_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  reason TEXT NOT NULL CHECK (reason IN ('spam', 'offensive', 'harassment', 'spoiler', 'nsfw', 'off_topic', 'other')),
  notes TEXT,
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'reviewed', 'resolved', 'dismissed', 'escalated')),
  reviewed_by INTEGER REFERENCES users(id),
  review_notes TEXT,
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(comment_id, reporter_id)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Report identifier |
| `comment_id` | INTEGER | Comment being reported |
| `reporter_id` | INTEGER | User filing the report |
| `reason` | TEXT | Report reason category |
| `notes` | TEXT | Additional context |
| `status` | TEXT | Current report status |
| `reviewed_by` | INTEGER | Moderator who reviewed |
| `review_notes` | TEXT | Moderator's notes |
| `reviewed_at` | TIMESTAMPTZ | Review completion time |
| `created_at` | TIMESTAMPTZ | Report creation time |

### Report Reasons

| Reason | Priority | Description |
|--------|----------|-------------|
| `spam` | 3 | Unsolicited promotional content |
| `offensive` | 4 | Offensive language or content |
| `harassment` | 5 | Targeted harassment or bullying |
| `spoiler` | 2 | Unmarked spoiler content |
| `nsfw` | 2 | Not safe for work content |
| `off_topic` | 1 | Comment unrelated to discussion |
| `other` | 1 | Other violations (specify in notes) |

### Report Status Flow

1. **pending** → Awaiting moderator review
2. **reviewed** → Currently under review
3. **resolved** → Issue addressed, action taken
4. **dismissed** → No violation found
5. **escalated** → Sent to higher-level moderator

## Report Escalations Table

Tracks report escalations between moderation levels.

```sql
CREATE TABLE report_escalations (
  id SERIAL PRIMARY KEY,
  report_id INTEGER NOT NULL REFERENCES comment_reports(id) ON DELETE CASCADE,
  escalated_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  escalated_to INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  reason TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Escalation identifier |
| `report_id` | INTEGER | Report being escalated |
| `escalated_by` | INTEGER | Who escalated the report |
| `escalated_to` | INTEGER | Who received the escalation |
| `reason` | TEXT | Reason for escalation |
| `created_at` | TIMESTAMPTZ | Escalation time |

## Moderation Actions Table

Audit log of all moderation actions performed.

```sql
CREATE TABLE moderation_actions (
  id SERIAL PRIMARY KEY,
  moderator_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  target_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  target_comment_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  action_type TEXT NOT NULL CHECK (action_type IN (
    'delete_comment', 'restore_comment', 'lock_thread', 'unlock_thread', 
    'pin_comment', 'unpin_comment', 'tag_comment', 'untag_comment',
    'warn_user', 'mute_user', 'ban_user', 'shadow_ban_user',
    'unban_user', 'unshadow_ban_user', 'promote_user', 'demote_user'
  )),
  action_details JSONB,
  reason TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Action identifier |
| `moderator_id` | INTEGER | Moderator performing action |
| `target_user_id` | INTEGER | User being acted upon |
| `target_comment_id` | INTEGER | Comment being acted upon |
| `action_type` | TEXT | Type of moderation action |
| `action_details` | JSONB | Additional action context |
| `reason` | TEXT | Reason for the action |
| `created_at` | TIMESTAMPTZ | When action was performed |

### Action Types

| Category | Actions | Description |
|----------|---------|-------------|
| **Comment** | `delete_comment`, `restore_comment`, `lock_thread`, `unlock_thread`, `pin_comment`, `unpin_comment`, `tag_comment`, `untag_comment` | Direct comment moderation |
| **User** | `warn_user`, `mute_user`, `ban_user`, `shadow_ban_user`, `unban_user`, `unshadow_ban_user` | User account moderation |
| **Role** | `promote_user`, `demote_user` | Role management |

## System Configuration Table

System-wide configuration settings (super admin only).

```sql
CREATE TABLE system_config (
  id SERIAL PRIMARY KEY,
  key TEXT NOT NULL UNIQUE,
  value JSONB NOT NULL,
  updated_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Config identifier |
| `key` | TEXT | Configuration key |
| `value` | JSONB | Configuration value |
| `updated_by` | INTEGER | Who last updated |
| `updated_at` | TIMESTAMPTZ | Last update time |

### Default Configuration

| Key | Default Value | Description |
|-----|---------------|-------------|
| `max_comment_length` | `"10000"` | Maximum comment characters |
| `max_nesting_level` | `"10"` | Maximum reply depth |
| `rate_limit_comments_per_hour` | `"30"` | Comments per user per hour |
| `rate_limit_votes_per_hour` | `"100"` | Votes per user per hour |
| `rate_limit_reports_per_hour` | `"10"` | Reports per user per hour |
| `auto_warn_threshold` | `"3"` | Reports before auto-warning |
| `auto_mute_threshold` | `"5"` | Reports before auto-mute |
| `auto_ban_threshold` | `"10"` | Reports before auto-ban |

## Global Settings Table

Runtime system settings and feature flags.

```sql
CREATE TABLE global_settings (
  id SERIAL PRIMARY KEY,
  comment_system_enabled BOOLEAN DEFAULT TRUE,
  voting_system_enabled BOOLEAN DEFAULT TRUE,
  reporting_system_enabled BOOLEAN DEFAULT TRUE,
  emergency_mode BOOLEAN DEFAULT FALSE,
  global_lock_enabled BOOLEAN DEFAULT FALSE,
  auto_moderation_enabled BOOLEAN DEFAULT TRUE,
  updated_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Settings identifier |
| `comment_system_enabled` | BOOLEAN | Enable/disable comments |
| `voting_system_enabled` | BOOLEAN | Enable/disable voting |
| `reporting_system_enabled` | BOOLEAN | Enable/disable reporting |
| `emergency_mode` | BOOLEAN | Emergency shutdown mode |
| `global_lock_enabled` | BOOLEAN | Prevent all interactions |
| `auto_moderation_enabled` | BOOLEAN | Enable automated moderation |
| `updated_by` | INTEGER | Who last updated |
| `updated_at` | TIMESTAMPTZ | Last update time |

## Automation Rules Table

Configurable automated moderation rules.

```sql
CREATE TABLE automation_rules (
  id SERIAL PRIMARY KEY,
  rule_name TEXT NOT NULL UNIQUE,
  conditions JSONB NOT NULL,
  actions JSONB NOT NULL,
  enabled BOOLEAN DEFAULT TRUE,
  created_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Rule identifier |
| `rule_name` | TEXT | Human-readable rule name |
| `conditions` | JSONB | Rule trigger conditions |
| `actions` | JSONB | Actions to perform |
| `enabled` | BOOLEAN | Whether rule is active |
| `created_by` | INTEGER | Who created the rule |
| `created_at` | TIMESTAMPTZ | Rule creation time |
| `updated_at` | TIMESTAMPTZ | Last update time |

## Banned Keywords Table

Keyword-based automated moderation.

```sql
CREATE TABLE banned_keywords (
  id SERIAL PRIMARY KEY,
  keyword TEXT NOT NULL UNIQUE,
  severity INTEGER DEFAULT 1 CHECK (severity BETWEEN 1 AND 5),
  action_type TEXT NOT NULL DEFAULT 'flag' CHECK (action_type IN ('flag', 'hide', 'delete', 'warn')),
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Keyword identifier |
| `keyword` | TEXT | Banned keyword/phrase |
| `severity` | INTEGER | Severity level (1-5) |
| `action_type` | TEXT | Automated action |
| `enabled` | BOOLEAN | Whether keyword is active |
| `created_at` | TIMESTAMPTZ | When keyword was added |

### Severity Levels

| Severity | Action | Description |
|----------|--------|-------------|
| 1 | `flag` | Mark for review |
| 2 | `flag` + `warn` | Flag and warn user |
| 3 | `hide` | Hide comment, warn user |
| 4 | `hide` + `warn` | Hide, warn, escalate |
| 5 | `delete` + `warn` | Delete, warn, escalate |

## Database Functions

### Moderation Queue

```sql
CREATE OR REPLACE FUNCTION get_moderation_queue(moderator_role TEXT DEFAULT 'moderator')
RETURNS TABLE(
    report_id INTEGER,
    comment_id INTEGER,
    comment_content TEXT,
    reporter_id INTEGER,
    reporter_username TEXT,
    reason TEXT,
    notes TEXT,
    status TEXT,
    created_at TIMESTAMPTZ,
    comment_author_id INTEGER,
    comment_author_username TEXT,
    priority INTEGER
)
```

Returns prioritized moderation queue based on report severity and age.

### Automated Moderation Check

```sql
CREATE OR REPLACE FUNCTION check_automated_moderation(comment_content TEXT, user_id INTEGER)
RETURNS TABLE(
    should_flag BOOLEAN,
    should_hide BOOLEAN,
    should_delete BOOLEAN,
    should_warn BOOLEAN,
    severity INTEGER,
    matched_keywords TEXT[]
)
```

Analyzes comment content against automated moderation rules.

## Indexes

```sql
-- Report management
CREATE INDEX idx_comment_reports_comment_id ON comment_reports(comment_id);
CREATE INDEX idx_comment_reports_status ON comment_reports(status);
CREATE INDEX idx_comment_reports_created_at ON comment_reports(created_at);
CREATE INDEX idx_comment_reports_reporter_id ON comment_reports(reporter_id);

-- Moderation actions
CREATE INDEX idx_moderation_actions_moderator_id ON moderation_actions(moderator_id);
CREATE INDEX idx_moderation_actions_target_user_id ON moderation_actions(target_user_id);
CREATE INDEX idx_moderation_actions_created_at ON moderation_actions(created_at);

-- Escalations
CREATE INDEX idx_report_escalations_report_id ON report_escalations(report_id);
CREATE INDEX idx_report_escalations_escalated_to ON report_escalations(escalated_to);

-- Automation
CREATE INDEX idx_automation_rules_enabled ON automation_rules(enabled);
CREATE INDEX idx_banned_keywords_enabled ON banned_keywords(enabled);
CREATE INDEX idx_banned_keywords_severity ON banned_keywords(severity);
```

## Row Level Security Policies

### Comment Reports Table
- **View Own Reports**: Users can see reports they filed
- **Create Reports**: Users can create reports if not banned
- **Moderator Access**: Moderators can view all reports
- **Service Role**: Full system access

### Report Escalations Table
- **View Own Escalations**: Users can see escalations they created/received
- **Moderator Access**: Moderators can view all escalations
- **Service Role**: Full system access

### Moderation Actions Table
- **View Own Actions**: Users can see actions they performed
- **Moderator Access**: Moderators can view all actions
- **Service Role**: Full system access

### System Configuration Table
- **Super Admin Access**: Only super admins can view/manage
- **Service Role**: Full system access

### Global Settings Table
- **Public View**: All users can view settings
- **Super Admin Update**: Only super admins can update
- **Service Role**: Full system access

## Usage Examples

### Report Management

```sql
-- Create a new report
INSERT INTO comment_reports (comment_id, reporter_id, reason, notes)
VALUES (1, 123, 'offensive', 'Contains inappropriate language');

-- Get moderation queue
SELECT * FROM get_moderation_queue('moderator')
ORDER BY priority DESC, created_at ASC;

-- Resolve a report
UPDATE comment_reports 
SET status = 'resolved', reviewed_by = 456, review_notes = 'Removed offensive content', reviewed_at = NOW()
WHERE id = 1;
```

### Automated Moderation

```sql
-- Check content for violations
SELECT * FROM check_automated_moderation('This comment contains bad words', 123);

-- Add banned keyword
INSERT INTO banned_keywords (keyword, severity, action_type)
VALUES ('spam', 3, 'hide');

-- Create automation rule
INSERT INTO automation_rules (rule_name, conditions, actions, created_by)
VALUES (
  'New User Spam Protection',
  '{"user_age_hours": 24, "comment_count": 5}',
  '{"action": "flag", "auto_mute": true}',
  1
);
```

### System Configuration

```sql
-- Update system setting
UPDATE system_config 
SET value = '50', updated_by = 1
WHERE key = 'rate_limit_comments_per_hour';

-- Toggle feature flag
UPDATE global_settings 
SET auto_moderation_enabled = false, updated_by = 1
WHERE id = 1;

-- Get current settings
SELECT * FROM global_settings;
```

### Audit and Escalation

```sql
-- Log moderation action
INSERT INTO moderation_actions (moderator_id, target_comment_id, action_type, reason)
VALUES (456, 1, 'delete_comment', 'Violation of community guidelines');

-- Escalate report
INSERT INTO report_escalations (report_id, escalated_by, escalated_to, reason)
VALUES (1, 456, 789, 'Requires admin review due to severity');

-- Get moderation history
SELECT * FROM moderation_actions 
WHERE moderator_id = 456 
ORDER BY created_at DESC 
LIMIT 50;
```

## Security Considerations

- **Audit Trail**: All moderation actions are logged permanently
- **Access Control**: Strict role-based access to moderation tools
- **Escalation Paths**: Clear escalation procedures for serious issues
- **Automated Safeguards**: Automated rules have built-in safeguards
- **Transparency**: Users can see their own reports and warnings