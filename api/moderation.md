# Moderation API

The Moderation API handles user moderation actions including warnings, mutes, bans, and role management.

## Endpoint

```
POST /moderation
```

## Request Body

```typescript
interface ModerationRequest {
  action: 'warn' | 'mute' | 'ban' | 'shadow_ban' | 'unban' | 'unshadow_ban' | 'promote' | 'demote'
  user_id: number
  target_user_id: number
  reason: string
  duration?: number // in hours, for mute
  severity?: 'warning' | 'mute' | 'ban' // for warn action
}
```

### Parameters

| Parameter | Type | Required | Actions | Description |
|-----------|------|----------|---------|-------------|
| `action` | string | Yes | All | The moderation action to perform |
| `user_id` | number | Yes | All | ID of the moderator performing the action |
| `target_user_id` | number | Yes | All | ID of the user being moderated |
| `reason` | string | Yes | All | Reason for the moderation action |
| `duration` | number | mute | Mute | Duration in hours (default: 24) |
| `severity` | string | warn | Warning | Warning severity level |

## Actions

### 1. Warn User

**Required Parameters:** `action: 'warn'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Moderator, Admin, Super Admin

#### Request
```json
{
  "action": "warn",
  "user_id": 456,
  "target_user_id": 123,
  "reason": "Spamming comments with promotional content",
  "severity": "warning"
}
```

#### Response (201 Created)
```json
{
  "id": 4001,
  "user_id": 123,
  "issued_by": 456,
  "reason": "Spamming comments with promotional content",
  "severity": "warning",
  "expires_at": "2024-02-14T15:00:00Z",
  "active": true,
  "created_at": "2024-01-15T15:00:00Z"
}
```

### 2. Mute User

**Required Parameters:** `action: 'mute'`, `user_id`, `target_user_id`, `reason`

**Optional Parameters:** `duration` (default: 24 hours)

**Permissions:** Moderator, Admin, Super Admin

#### Request
```json
{
  "action": "mute",
  "user_id": 456,
  "target_user_id": 123,
  "reason": "Continued harassment after warning",
  "duration": 72
}
```

#### Response (200 OK)
```json
{
  "id": 123,
  "username": "problem_user",
  "role": "user",
  "banned": false,
  "shadow_banned": false,
  "muted_until": "2024-01-18T15:00:00Z"
}
```

### 3. Ban User

**Required Parameters:** `action: 'ban'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Admin, Super Admin

#### Request
```json
{
  "action": "ban",
  "user_id": 789,
  "target_user_id": 123,
  "reason": "Severe harassment and threats"
}
```

#### Response (200 OK)
```json
{
  "id": 123,
  "username": "banned_user",
  "role": "user",
  "banned": true,
  "shadow_banned": false,
  "muted_until": null
}
```

### 4. Shadow Ban User

**Required Parameters:** `action: 'shadow_ban'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Admin, Super Admin

#### Request
```json
{
  "action": "shadow_ban",
  "user_id": 789,
  "target_user_id": 123,
  "reason": "Subtle rule violations, borderline content"
}
```

#### Response (200 OK)
```json
{
  "id": 123,
  "username": "shadow_banned_user",
  "role": "user",
  "banned": false,
  "shadow_banned": true,
  "muted_until": null
}
```

### 5. Unban User

**Required Parameters:** `action: 'unban'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Admin, Super Admin

#### Request
```json
{
  "action": "unban",
  "user_id": 789,
  "target_user_id": 123,
  "reason": "Appeal approved, user shows reform"
}
```

#### Response (200 OK)
```json
{
  "id": 123,
  "username": "reformed_user",
  "role": "user",
  "banned": false,
  "shadow_banned": false,
  "muted_until": null
}
```

### 6. Promote User

**Required Parameters:** `action: 'promote'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Super Admin only

#### Request
```json
{
  "action": "promote",
  "user_id": 1,
  "target_user_id": 456,
  "reason": "Excellent moderation performance, trusted community member"
}
```

#### Response (200 OK)
```json
{
  "id": 456,
  "username": "new_moderator",
  "role": "moderator",
  "banned": false,
  "shadow_banned": false,
  "muted_until": null
}
```

### 7. Demote User

**Required Parameters:** `action: 'demote'`, `user_id`, `target_user_id`, `reason`

**Permissions:** Super Admin only

#### Request
```json
{
  "action": "demote",
  "user_id": 1,
  "target_user_id": 456,
  "reason": "Abuse of moderation privileges"
}
```

#### Response (200 OK)
```json
{
  "id": 456,
  "username": "demoted_moderator",
  "role": "user",
  "banned": false,
  "shadow_banned": false,
  "muted_until": null
}
```

## Role Hierarchy

| Role | Level | Can Target | Description |
|------|-------|------------|-------------|
| `user` | 0 | None | Regular user |
| `moderator` | 1 | Users | Can warn, mute, pin, lock comments |
| `admin` | 2 | Users, Moderators | Can ban, shadow ban, promote to moderator |
| `super_admin` | 3 | All | Full system access, can manage all roles |

## Warning Severity Levels

| Severity | Duration | Description |
|----------|----------|-------------|
| `warning` | 30 days | Standard warning, expires after 30 days |
| `mute` | Variable | User is muted for specified duration |
| `ban` | Permanent | Permanent ban until manually lifted |

## Error Responses

### 403 Forbidden
```json
{
  "error": "Insufficient permissions"
}
```

```json
{
  "error": "Cannot target super admin"
}
```

```json
{
  "error": "Cannot target user with equal or higher role"
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
  "error": "Target user not found"
}
```

## Validation Rules

### Permission Validation
- **Role Hierarchy**: Users can only target lower-ranked users
- **Super Admin Protection**: Super admins cannot be targeted by any action
- **Self Targeting**: Users cannot perform actions on themselves

### Action Validation
- **Ban Protection**: Only admins and super admins can ban users
- **Role Management**: Only super admins can promote/demote users
- **Duration Limits**: Mute duration must be reasonable (1-168 hours)

## Usage Examples

### Moderation Panel Component

```typescript
import { useState } from 'react';

interface ModerationPanelProps {
  targetUser: any;
  moderatorId: number;
  moderatorRole: string;
  onAction: (action: string, result: any) => void;
}

export const ModerationPanel: React.FC<ModerationPanelProps> = ({
  targetUser,
  moderatorId,
  moderatorRole,
  onAction
}) => {
  const [showForm, setShowForm] = useState(false);
  const [action, setAction] = useState('');
  const [reason, setReason] = useState('');
  const [duration, setDuration] = useState(24);
  const [severity, setSeverity] = useState('warning');
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const canPerformAction = (requiredAction: string): boolean => {
    const roleHierarchy = { 'user': 0, 'moderator': 1, 'admin': 2, 'super_admin': 3 };
    const moderatorLevel = roleHierarchy[moderatorRole as keyof typeof roleHierarchy];
    const targetLevel = roleHierarchy[targetUser.role as keyof typeof roleHierarchy];

    switch (requiredAction) {
      case 'warn':
      case 'mute':
        return moderatorLevel >= 1 && moderatorLevel > targetLevel;
      case 'ban':
      case 'shadow_ban':
      case 'unban':
      case 'unshadow_ban':
        return moderatorLevel >= 2 && moderatorLevel > targetLevel;
      case 'promote':
      case 'demote':
        return moderatorLevel >= 3 && moderatorLevel > targetLevel;
      default:
        return false;
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!action || !reason.trim()) return;

    setSubmitting(true);
    setError(null);

    try {
      const requestBody: any = {
        action,
        user_id: moderatorId,
        target_user_id: targetUser.id,
        reason: reason.trim()
      };

      if (action === 'mute') {
        requestBody.duration = duration;
      } else if (action === 'warn') {
        requestBody.severity = severity;
      }

      const response = await fetch('/moderation', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(requestBody)
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      onAction(action, result);
      setShowForm(false);
      setAction('');
      setReason('');
      setDuration(24);
      setSeverity('warning');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Action failed');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="moderation-panel">
      <div className="user-info">
        <h3>{targetUser.username}</h3>
        <p>Role: {targetUser.role}</p>
        <p>Status: {targetUser.banned ? 'Banned' : targetUser.shadow_banned ? 'Shadow Banned' : 'Active'}</p>
        {targetUser.muted_until && (
          <p>Muted until: {new Date(targetUser.muted_until).toLocaleString()}</p>
        )}
      </div>

      <div className="moderation-actions">
        {canPerformAction('warn') && (
          <button onClick={() => { setAction('warn'); setShowForm(true); }}>
            ⚠️ Warn
          </button>
        )}
        
        {canPerformAction('mute') && (
          <button onClick={() => { setAction('mute'); setShowForm(true); }}>
            🔇 Mute
          </button>
        )}
        
        {canPerformAction('ban') && (
          <button onClick={() => { setAction('ban'); setShowForm(true); }}>
            🚫 Ban
          </button>
        )}
        
        {canPerformAction('shadow_ban') && (
          <button onClick={() => { setAction('shadow_ban'); setShowForm(true); }}>
            👻 Shadow Ban
          </button>
        )}
        
        {canPerformAction('unban') && targetUser.banned && (
          <button onClick={() => { setAction('unban'); setShowForm(true); }}>
            ✅ Unban
          </button>
        )}
        
        {canPerformAction('unshadow_ban') && targetUser.shadow_banned && (
          <button onClick={() => { setAction('unshadow_ban'); setShowForm(true); }}>
            👁️ Unshadow Ban
          </button>
        )}
        
        {canPerformAction('promote') && (
          <button onClick={() => { setAction('promote'); setShowForm(true); }}>
            ⬆️ Promote
          </button>
        )}
        
        {canPerformAction('demote') && targetUser.role !== 'user' && (
          <button onClick={() => { setAction('demote'); setShowForm(true); }}>
            ⬇️ Demote
          </button>
        )}
      </div>

      {showForm && (
        <form onSubmit={handleSubmit} className="moderation-form">
          <h3>Perform Action: {action}</h3>
          
          <div className="form-group">
            <label>Reason:</label>
            <textarea
              value={reason}
              onChange={(e) => setReason(e.target.value)}
              placeholder="Reason for this action..."
              required
              rows={3}
            />
          </div>

          {action === 'mute' && (
            <div className="form-group">
              <label>Duration (hours):</label>
              <input
                type="number"
                value={duration}
                onChange={(e) => setDuration(parseInt(e.target.value))}
                min="1"
                max="168"
              />
            </div>
          )}

          {action === 'warn' && (
            <div className="form-group">
              <label>Severity:</label>
              <select value={severity} onChange={(e) => setSeverity(e.target.value)}>
                <option value="warning">Warning</option>
                <option value="mute">Mute</option>
                <option value="ban">Ban</option>
              </select>
            </div>
          )}

          {error && <div className="error">{error}</div>}

          <div className="form-actions">
            <button type="submit" disabled={submitting}>
              {submitting ? 'Processing...' : `Confirm ${action}`}
            </button>
            <button type="button" onClick={() => setShowForm(false)} disabled={submitting}>
              Cancel
            </button>
          </div>
        </form>
      )}
    </div>
  );
};
```

### User Management Hook

```typescript
import { useState, useCallback } from 'react';

interface UseModerationResult {
  warnUser: (targetUserId: number, reason: string, severity?: string) => Promise<void>;
  muteUser: (targetUserId: number, reason: string, duration?: number) => Promise<void>;
  banUser: (targetUserId: number, reason: string) => Promise<void>;
  shadowBanUser: (targetUserId: number, reason: string) => Promise<void>;
  unbanUser: (targetUserId: number, reason: string) => Promise<void>;
  unshadowBanUser: (targetUserId: number, reason: string) => Promise<void>;
  promoteUser: (targetUserId: number, reason: string) => Promise<void>;
  demoteUser: (targetUserId: number, reason: string) => Promise<void>;
  loading: boolean;
  error: string | null;
}

export const useModeration = (moderatorId: number): UseModerationResult => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const performAction = useCallback(async (
    action: string,
    targetUserId: number,
    reason: string,
    extraData = {}
  ) => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch('/moderation', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action,
          user_id: moderatorId,
          target_user_id: targetUserId,
          reason,
          ...extraData
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      return result;
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Action failed';
      setError(errorMessage);
      throw new Error(errorMessage);
    } finally {
      setLoading(false);
    }
  }, [moderatorId]);

  return {
    warnUser: (targetUserId, reason, severity = 'warning') =>
      performAction('warn', targetUserId, reason, { severity }),
    
    muteUser: (targetUserId, reason, duration = 24) =>
      performAction('mute', targetUserId, reason, { duration }),
    
    banUser: (targetUserId, reason) =>
      performAction('ban', targetUserId, reason),
    
    shadowBanUser: (targetUserId, reason) =>
      performAction('shadow_ban', targetUserId, reason),
    
    unbanUser: (targetUserId, reason) =>
      performAction('unban', targetUserId, reason),
    
    unshadowBanUser: (targetUserId, reason) =>
      performAction('unshadow_ban', targetUserId, reason),
    
    promoteUser: (targetUserId, reason) =>
      performAction('promote', targetUserId, reason),
    
    demoteUser: (targetUserId, reason) =>
      performAction('demote', targetUserId, reason),
    
    loading,
    error
  };
};
```

## Security Considerations

- **Permission Validation**: All actions are validated against role hierarchy
- **Audit Trail**: All moderation actions are logged with full context
- **Self Protection**: Users cannot target themselves with moderation actions
- **Super Admin Protection**: Super admins have immunity from moderation actions

## Impact on User Experience

### Banned Users
- Cannot create, edit, or vote on comments
- Cannot report other comments
- See existing comments but cannot interact

### Shadow Banned Users
- Can still post comments and interact
- Only they can see their own comments
- No indication they are shadow banned

### Muted Users
- Cannot create new comments
- Can still view and vote on existing comments
- Mute status is visible to the user

### Warnings
- Recorded in user history
- Can contribute to automated moderation decisions
- Visible to moderators but not to other users