# Reports API

The Reports API handles comment reporting and moderation workflow management.

## Endpoint

```
POST /reports
```

## Request Body

```typescript
interface ReportRequest {
  action: 'create' | 'resolve' | 'dismiss' | 'escalate'
  user_id: number
  comment_id?: number
  report_id?: number
  reason?: 'spam' | 'offensive' | 'harassment' | 'spoiler' | 'nsfw' | 'off_topic' | 'other'
  notes?: string
  escalate_to?: number
}
```

### Parameters

| Parameter | Type | Required | Actions | Description |
|-----------|------|----------|---------|-------------|
| `action` | string | Yes | All | The action to perform |
| `user_id` | number | Yes | All | ID of the user performing the action |
| `comment_id` | number | create | Create | ID of the comment being reported |
| `report_id` | number | resolve/dismiss/escalate | Moderation | ID of the report being acted upon |
| `reason` | string | create | Create | Reason for the report |
| `notes` | string | All | Optional | Additional notes or explanation |
| `escalate_to` | number | escalate | Escalate | ID of user to escalate to |

## Actions

### 1. Create Report

**Required Parameters:** `action: 'create'`, `user_id`, `comment_id`, `reason`

#### Request
```json
{
  "action": "create",
  "user_id": 123,
  "comment_id": 1001,
  "reason": "offensive",
  "notes": "This comment contains inappropriate language"
}
```

#### Response (201 Created)
```json
{
  "id": 3001,
  "comment_id": 1001,
  "reporter_id": 123,
  "reason": "offensive",
  "notes": "This comment contains inappropriate language",
  "status": "pending",
  "created_at": "2024-01-15T15:00:00Z"
}
```

### 2. Resolve Report

**Required Parameters:** `action: 'resolve'`, `user_id`, `report_id`

**Permissions:** Moderator, Admin, Super Admin

#### Request
```json
{
  "action": "resolve",
  "user_id": 456,
  "report_id": 3001,
  "notes": "Removed offensive content, warned user"
}
```

#### Response (200 OK)
```json
{
  "id": 3001,
  "status": "resolved",
  "reviewed_by": 456,
  "review_notes": "Removed offensive content, warned user",
  "reviewed_at": "2024-01-15T15:30:00Z"
}
```

### 3. Dismiss Report

**Required Parameters:** `action: 'dismiss'`, `user_id`, `report_id`

**Permissions:** Moderator, Admin, Super Admin

#### Request
```json
{
  "action": "dismiss",
  "user_id": 456,
  "report_id": 3001,
  "notes": "No violation found, comment is appropriate"
}
```

#### Response (200 OK)
```json
{
  "id": 3001,
  "status": "dismissed",
  "reviewed_by": 456,
  "review_notes": "No violation found, comment is appropriate",
  "reviewed_at": "2024-01-15T16:00:00Z"
}
```

### 4. Escalate Report

**Required Parameters:** `action: 'escalate'`, `user_id`, `report_id`, `escalate_to`

**Permissions:** Moderator, Admin, Super Admin

#### Request
```json
{
  "action": "escalate",
  "user_id": 456,
  "report_id": 3001,
  "escalate_to": 789,
  "notes": "Requires admin review due to severity"
}
```

#### Response (200 OK)
```json
{
  "id": 3001,
  "status": "escalated",
  "reviewed_by": 456,
  "review_notes": "Requires admin review due to severity",
  "reviewed_at": "2024-01-15T16:30:00Z"
}
```

## Report Reasons

| Reason | Description | Priority |
|--------|-------------|----------|
| `spam` | Unsolicited promotional content | 3 |
| `offensive` | Offensive language or content | 4 |
| `harassment` | Targeted harassment or bullying | 5 |
| `spoiler` | Unmarked spoiler content | 2 |
| `nsfw` | Not safe for work content | 2 |
| `off_topic` | Comment unrelated to discussion | 1 |
| `other` | Other violations (specify in notes) | 1 |

## Error Responses

### 400 Bad Request
```json
{
  "error": "You have already reported this comment"
}
```

### 403 Forbidden
```json
{
  "error": "Insufficient permissions"
}
```

```json
{
  "error": "User banned"
}
```

### 404 Not Found
```json
{
  "error": "Report not found"
}
```

### 429 Too Many Requests
```json
{
  "error": "Rate limit exceeded"
}
```

## Validation Rules

### Report Creation
- **Unique Reports**: Users can only report a specific comment once
- **Rate Limiting**: 10 reports per hour per user (configurable)
- **Banned Users**: Cannot create reports
- **Comment Status**: Can only report non-deleted comments

### Moderation Actions
- **Permission Levels**: Different actions require different permission levels
- **Escalation Rules**: Can only escalate to users with equal or higher permissions
- **Status Transitions**: Reports follow a strict status flow

## Usage Examples

### Report Comment Component

```typescript
import { useState } from 'react';

interface ReportButtonProps {
  commentId: number;
  userId: number;
  onReported: (report: any) => void;
}

export const ReportButton: React.FC<ReportButtonProps> = ({
  commentId,
  userId,
  onReported
}) => {
  const [showForm, setShowForm] = useState(false);
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (reason: string, notes: string) => {
    setSubmitting(true);
    setError(null);

    try {
      const response = await fetch('/reports', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action: 'create',
          user_id: userId,
          comment_id: commentId,
          reason,
          notes
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      onReported(result);
      setShowForm(false);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to report comment');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="report-button">
      <button onClick={() => setShowForm(!showForm)}>
        🚩 Report
      </button>
      
      {showForm && (
        <ReportForm
          onSubmit={handleSubmit}
          onCancel={() => setShowForm(false)}
          submitting={submitting}
          error={error}
        />
      )}
    </div>
  );
};

interface ReportFormProps {
  onSubmit: (reason: string, notes: string) => void;
  onCancel: () => void;
  submitting: boolean;
  error: string | null;
}

const ReportForm: React.FC<ReportFormProps> = ({
  onSubmit,
  onCancel,
  submitting,
  error
}) => {
  const [reason, setReason] = useState('');
  const [notes, setNotes] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!reason) return;
    onSubmit(reason, notes);
  };

  return (
    <form onSubmit={handleSubmit} className="report-form">
      <h3>Report Comment</h3>
      
      <div className="form-group">
        <label>Reason:</label>
        <select value={reason} onChange={(e) => setReason(e.target.value)} required>
          <option value="">Select a reason...</option>
          <option value="spam">Spam</option>
          <option value="offensive">Offensive Content</option>
          <option value="harassment">Harassment</option>
          <option value="spoiler">Spoiler</option>
          <option value="nsfw">NSFW</option>
          <option value="off_topic">Off Topic</option>
          <option value="other">Other</option>
        </select>
      </div>

      <div className="form-group">
        <label>Additional Notes:</label>
        <textarea
          value={notes}
          onChange={(e) => setNotes(e.target.value)}
          placeholder="Provide any additional context..."
          rows={3}
        />
      </div>

      {error && <div className="error">{error}</div>}

      <div className="form-actions">
        <button type="submit" disabled={submitting || !reason}>
          {submitting ? 'Reporting...' : 'Submit Report'}
        </button>
        <button type="button" onClick={onCancel} disabled={submitting}>
          Cancel
        </button>
      </div>
    </form>
  );
};
```

### Moderation Queue Component

```typescript
import { useState, useEffect } from 'react';

interface ModerationQueueProps {
  userId: number;
  userRole: string;
}

export const ModerationQueue: React.FC<ModerationQueueProps> = ({
  userId,
  userRole
}) => {
  const [reports, setReports] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    loadModerationQueue();
  }, []);

  const loadModerationQueue = async () => {
    try {
      const response = await fetch('/moderation-queue');
      const data = await response.json();
      
      if (!response.ok) {
        throw new Error(data.error);
      }
      
      setReports(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load queue');
    } finally {
      setLoading(false);
    }
  };

  const handleAction = async (reportId: number, action: string, notes?: string) => {
    try {
      const response = await fetch('/reports', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          action,
          user_id: userId,
          report_id: reportId,
          notes
        })
      });

      const result = await response.json();

      if (!response.ok) {
        throw new Error(result.error);
      }

      // Update local state
      setReports(prev => prev.map(report => 
        report.id === reportId ? { ...report, ...result } : report
      ));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Action failed');
    }
  };

  if (loading) return <div>Loading moderation queue...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="moderation-queue">
      <h2>Moderation Queue</h2>
      
      {reports.length === 0 ? (
        <p>No pending reports.</p>
      ) : (
        <div className="reports-list">
          {reports.map(report => (
            <ReportCard
              key={report.id}
              report={report}
              onAction={handleAction}
              userRole={userRole}
            />
          ))}
        </div>
      )}
    </div>
  );
};

interface ReportCardProps {
  report: any;
  onAction: (reportId: number, action: string, notes?: string) => void;
  userRole: string;
}

const ReportCard: React.FC<ReportCardProps> = ({
  report,
  onAction,
  userRole
}) => {
  const [showNotes, setShowNotes] = useState(false);
  const [notes, setNotes] = useState('');

  const handleAction = (action: string) => {
    onAction(report.id, action, notes || undefined);
    setNotes('');
    setShowNotes(false);
  };

  return (
    <div className="report-card">
      <div className="report-header">
        <span className="report-reason">{report.reason}</span>
        <span className="report-priority">Priority: {report.priority}</span>
        <span className="report-date">
          {new Date(report.created_at).toLocaleDateString()}
        </span>
      </div>

      <div className="report-content">
        <p><strong>Comment:</strong> {report.comment_content}</p>
        <p><strong>Reporter:</strong> {report.reporter_username}</p>
        <p><strong>Author:</strong> {report.comment_author_username}</p>
        
        {report.notes && (
          <p><strong>Notes:</strong> {report.notes}</p>
        )}
      </div>

      <div className="report-actions">
        <button onClick={() => handleAction('resolve')}>
          ✅ Resolve
        </button>
        <button onClick={() => handleAction('dismiss')}>
          ❌ Dismiss
        </button>
        
        {['admin', 'super_admin'].includes(userRole) && (
          <button onClick={() => setShowNotes(!showNotes)}>
            ⬆️ Escalate
          </button>
        )}
      </div>

      {showNotes && (
        <div className="escalation-form">
          <textarea
            value={notes}
            onChange={(e) => setNotes(e.target.value)}
            placeholder="Reason for escalation..."
            rows={2}
          />
          <button onClick={() => handleAction('escalate')}>
            Submit Escalation
          </button>
        </div>
      )}
    </div>
  );
};
```

## Rate Limiting

- **Report Creation**: 10 per hour per user (configurable)
- **Moderation Actions**: No rate limits for authorized moderators
- **Escalation**: No rate limits to prevent moderation bottlenecks

## Security Considerations

- **Permission Validation**: All actions are validated against user roles
- **Audit Trail**: All report actions are logged for accountability
- **Duplicate Prevention**: Users cannot report the same comment multiple times
- **Rate Limiting**: Prevents report spam and abuse

## Workflow Integration

Reports integrate with the broader moderation system:
- **Automated Moderation**: Can trigger automated actions based on report patterns
- **User Warnings**: Multiple reports can trigger automatic warnings
- **Comment Actions**: Resolved reports can automatically delete/hide comments
- **Escalation Paths**: Clear escalation workflow for serious issues