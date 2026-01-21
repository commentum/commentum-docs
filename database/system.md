# System Configuration Schema

The system configuration handles global settings, rate limiting, and automation rules.

## Rate Limits Table

Tracks and enforces rate limits for user actions.

```sql
CREATE TABLE rate_limits (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  action_type TEXT NOT NULL CHECK (action_type IN ('comment', 'vote', 'report', 'edit')),
  window_start TIMESTAMPTZ NOT NULL,
  count INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, action_type, window_start)
);
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | SERIAL | Rate limit record identifier |
| `user_id` | INTEGER | User being rate limited |
| `action_type` | TEXT | Type of action being limited |
| `window_start` | TIMESTAMPTZ | Start of the time window |
| `count` | INTEGER | Number of actions in window |
| `created_at` | TIMESTAMPTZ | Record creation time |

### Action Types

| Type | Default Limit | Description |
|------|---------------|-------------|
| `comment` | 30/hour | Comment creation and editing |
| `vote` | 100/hour | Voting on comments |
| `report` | 10/hour | Reporting comments |
| `edit` | 30/hour | Editing own comments |

### Time Windows

- **Window Size**: 1 hour (configurable)
- **Window Alignment**: Starts at the beginning of each hour (HH:00:00)
- **Reset Behavior**: Counts reset at the start of each new window
- **Accumulation**: Multiple actions in same window increment count

## System Config Table (Detailed)

Comprehensive system configuration with validation.

```sql
CREATE TABLE system_config (
  id SERIAL PRIMARY KEY,
  key TEXT NOT NULL UNIQUE,
  value JSONB NOT NULL,
  description TEXT,
  category TEXT NOT NULL DEFAULT 'general',
  data_type TEXT NOT NULL DEFAULT 'string' CHECK (data_type IN ('string', 'number', 'boolean', 'json')),
  min_value NUMERIC,
  max_value NUMERIC,
  allowed_values JSONB,
  is_public BOOLEAN DEFAULT FALSE,
  updated_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Additional Fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | TEXT | Human-readable description |
| `category` | TEXT | Configuration category |
| `data_type` | TEXT | Expected data type |
| `min_value` | NUMERIC | Minimum value for numbers |
| `max_value` | NUMERIC | Maximum value for numbers |
| `allowed_values` | JSONB | List of allowed values |
| `is_public` | BOOLEAN | Whether config is public |

### Configuration Categories

| Category | Description | Example Keys |
|----------|-------------|--------------|
| `general` | General system settings | `site_name`, `maintenance_mode` |
| `limits` | Rate limits and thresholds | `max_comment_length`, `rate_limits` |
| `moderation` | Moderation settings | `auto_moderation`, `warning_thresholds` |
| `features` | Feature flags | `voting_enabled`, `reporting_enabled` |
| `security` | Security settings | `session_timeout`, `max_login_attempts` |

### Default Configuration Entries

```sql
-- General Settings
INSERT INTO system_config (key, value, description, category, data_type, is_public) VALUES
('site_name', '"Commentum"', 'Site name displayed in UI', 'general', 'string', true),
('site_description', '"Advanced comment system"', 'Site description', 'general', 'string', true),
('maintenance_mode', 'false', 'Enable maintenance mode', 'general', 'boolean', true);

-- Rate Limits
INSERT INTO system_config (key, value, description, category, data_type, min_value, max_value) VALUES
('max_comment_length', '10000', 'Maximum comment length in characters', 'limits', 'number', 100, 50000),
('max_nesting_level', '10', 'Maximum comment reply depth', 'limits', 'number', 1, 20),
('rate_limit_comments_per_hour', '30', 'Comments per user per hour', 'limits', 'number', 1, 1000),
('rate_limit_votes_per_hour', '100', 'Votes per user per hour', 'limits', 'number', 1, 1000),
('rate_limit_reports_per_hour', '10', 'Reports per user per hour', 'limits', 'number', 1, 100);

-- Moderation
INSERT INTO system_config (key, value, description, category, data_type, min_value, max_value) VALUES
('auto_warn_threshold', '3', 'Reports before auto-warning', 'moderation', 'number', 1, 10),
('auto_mute_threshold', '5', 'Reports before auto-mute', 'moderation', 'number', 1, 20),
('auto_ban_threshold', '10', 'Reports before auto-ban', 'moderation', 'number', 1, 50),
('auto_moderation_enabled', 'true', 'Enable automated moderation', 'moderation', 'boolean', null, null);

-- Features
INSERT INTO system_config (key, value, description, category, data_type, is_public) VALUES
('voting_system_enabled', 'true', 'Enable voting system', 'features', 'boolean', true),
('reporting_system_enabled', 'true', 'Enable reporting system', 'features', 'boolean', true),
('editing_enabled', 'true', 'Enable comment editing', 'features', 'boolean', true),
('nesting_enabled', 'true', 'Enable comment nesting', 'features', 'boolean', true);
```

## Global Settings Table (Enhanced)

Runtime settings with validation and change tracking.

```sql
CREATE TABLE global_settings (
  id SERIAL PRIMARY KEY,
  setting_key TEXT NOT NULL UNIQUE,
  setting_value JSONB NOT NULL,
  setting_type TEXT NOT NULL DEFAULT 'boolean' CHECK (setting_type IN ('boolean', 'number', 'string', 'json')),
  display_name TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL DEFAULT 'general',
  is_readonly BOOLEAN DEFAULT FALSE,
  requires_restart BOOLEAN DEFAULT FALSE,
  updated_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Enhanced Fields

| Field | Type | Description |
|-------|------|-------------|
| `setting_key` | TEXT | Unique setting identifier |
| `setting_value` | JSONB | Current setting value |
| `setting_type` | TEXT | Data type for validation |
| `display_name` | TEXT | Human-readable name |
| `description` | TEXT | Detailed description |
| `category` | TEXT | Setting category |
| `is_readonly` | BOOLEAN | Cannot be changed via UI |
| `requires_restart` | BOOLEAN | Requires system restart |
| `created_at` | TIMESTAMPTZ | Setting creation time |

### Global Settings Categories

| Category | Settings | Impact |
|----------|----------|--------|
| `system` | Core system behavior | High |
| `features` | Feature toggles | Medium |
| `ui` | User interface settings | Low |
| `performance` | Performance tuning | Medium |
| `security` | Security policies | High |

## Database Functions

### Rate Limit Checking

```sql
CREATE OR REPLACE FUNCTION check_rate_limit(
    p_user_id INTEGER,
    p_action_type TEXT,
    p_window_hours INTEGER DEFAULT 1
)
RETURNS TABLE(
    allowed BOOLEAN,
    current_count INTEGER,
    limit_count INTEGER,
    reset_time TIMESTAMPTZ
)
```

Checks if a user has exceeded their rate limit for an action.

### Configuration Validation

```sql
CREATE OR REPLACE FUNCTION validate_config_value(
    p_config_key TEXT,
    p_new_value JSONB
)
RETURNS TABLE(
    is_valid BOOLEAN,
    error_message TEXT
)
```

Validates a configuration value against its constraints.

### Public Configuration

```sql
CREATE OR REPLACE FUNCTION get_public_config()
RETURNS TABLE(
    key TEXT,
    value JSONB,
    description TEXT
)
```

Returns all public configuration settings.

## Indexes

```sql
-- Rate limiting
CREATE INDEX idx_rate_limits_user_action ON rate_limits(user_id, action_type);
CREATE INDEX idx_rate_limits_window ON rate_limits(window_start);
CREATE INDEX idx_rate_limits_user_window ON rate_limits(user_id, window_start);

-- Configuration
CREATE INDEX idx_system_config_category ON system_config(category);
CREATE INDEX idx_system_config_public ON system_config(is_public);
CREATE INDEX idx_global_settings_category ON global_settings(category);
CREATE INDEX idx_global_settings_key ON global_settings(setting_key);
```

## Row Level Security Policies

### Rate Limits Table
- **View Own Limits**: Users can see their own rate limits
- **Service Role**: Full system access

### System Config Table
- **Public View**: All users can see public configurations
- **Super Admin Access**: Only super admins can modify
- **Service Role**: Full system access

### Global Settings Table
- **Public View**: All users can view settings
- **Super Admin Update**: Only super admins can update
- **Service Role**: Full system access

## Usage Examples

### Rate Limit Management

```sql
-- Check if user can post a comment
SELECT * FROM check_rate_limit(123, 'comment');

-- Increment rate limit counter
INSERT INTO rate_limits (user_id, action_type, window_start, count)
VALUES (123, 'comment', DATE_TRUNC('hour', NOW()), 1)
ON CONFLICT (user_id, action_type, window_start)
DO UPDATE SET count = rate_limits.count + 1;

-- Get user's current rate limits
SELECT 
    action_type,
    window_start,
    count,
    (SELECT value::integer FROM system_config WHERE key = 'rate_limit_' || action_type || '_per_hour') as limit_count
FROM rate_limits 
WHERE user_id = 123 AND window_start >= NOW() - INTERVAL '1 hour';
```

### Configuration Management

```sql
-- Update a configuration value
UPDATE system_config 
SET value = '"50"', updated_by = 1, updated_at = NOW()
WHERE key = 'rate_limit_comments_per_hour';

-- Validate configuration change
SELECT * FROM validate_config_value('rate_limit_comments_per_hour', '"150"');

-- Get all public configurations
SELECT * FROM get_public_config();

-- Get configuration by category
SELECT key, value, description 
FROM system_config 
WHERE category = 'limits' AND is_public = true
ORDER BY key;
```

### System Settings

```sql
-- Toggle a feature
UPDATE global_settings 
SET setting_value = 'false', updated_by = 1
WHERE setting_key = 'voting_system_enabled';

-- Bulk update settings
UPDATE global_settings 
SET setting_value = CASE 
    WHEN setting_key = 'comment_system_enabled' THEN 'false'
    WHEN setting_key = 'emergency_mode' THEN 'true'
    ELSE setting_value
END,
updated_at = NOW()
WHERE setting_key IN ('comment_system_enabled', 'emergency_mode');

-- Get settings by category
SELECT 
    setting_key,
    setting_value,
    display_name,
    description
FROM global_settings 
WHERE category = 'features'
ORDER BY display_name;
```

### Configuration History

```sql
-- Create configuration audit table
CREATE TABLE config_history (
    id SERIAL PRIMARY KEY,
    config_key TEXT NOT NULL,
    old_value JSONB,
    new_value JSONB,
    changed_by INTEGER NOT NULL REFERENCES users(id),
    changed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Track configuration changes (trigger)
CREATE OR REPLACE FUNCTION log_config_change()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO config_history (config_key, old_value, new_value, changed_by)
    VALUES (NEW.key, OLD.value, NEW.value, NEW.updated_by);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_log_config_change
    AFTER UPDATE ON system_config
    FOR EACH ROW
    EXECUTE FUNCTION log_config_change();
```

## Performance Considerations

### Rate Limit Optimization

```sql
-- Efficient rate limit cleanup
DELETE FROM rate_limits 
WHERE window_start < NOW() - INTERVAL '25 hours';

-- Batch rate limit updates
UPDATE rate_limits 
SET count = count + 1
WHERE user_id = 123 
  AND action_type = 'comment' 
  AND window_start = DATE_TRUNC('hour', NOW());
```

### Configuration Caching

- **In-Memory Cache**: Cache frequently accessed configurations
- **Change Notification**: Notify cache of configuration changes
- **Version Control**: Track configuration versions for rollback
- **Validation Cache**: Cache validation results

## Security Considerations

### Configuration Security

```sql
-- Sensitive configurations should be marked private
UPDATE system_config 
SET is_public = false 
WHERE key IN ('database_url', 'secret_key', 'api_keys');

-- Readonly configurations for critical settings
UPDATE global_settings 
SET is_readonly = true 
WHERE setting_key IN ('system_version', 'database_version');
```

### Access Control

- **Role-Based Access**: Different roles can access different configurations
- **Audit Trail**: All configuration changes are logged
- **Validation**: All configuration values are validated before applying
- **Backup**: Critical configurations are backed up before changes

## Integration Points

### With Rate Limiting

- **Dynamic Limits**: Rate limits can be adjusted via configuration
- **User Exemptions**: Certain users can be exempt from limits
- **Action Types**: New action types can be added via configuration

### With Features

- **Feature Flags**: Features can be enabled/disabled via configuration
- **A/B Testing**: Configuration supports experiment parameters
- **Gradual Rollout**: Features can be rolled out gradually

### With Moderation

- **Thresholds**: Moderation thresholds are configurable
- **Automation Rules**: Automated moderation behavior is configurable
- **Escalation Paths**: Escalation rules are configuration-driven