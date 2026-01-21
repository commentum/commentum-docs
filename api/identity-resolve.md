# Identity Resolution API

The Identity Resolution API handles user authentication and identity management across multiple platforms (AniList, MyAnimeList, SIMKL).

## Endpoint

```
POST /identity-resolve
```

## Request Body

```typescript
interface IdentityRequest {
  client_type: 'anilist' | 'myanimelist' | 'simkl'
  client_user_id: string
  username: string
  avatar_url?: string
  verification_token?: string
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_type` | string | Yes | The platform identifier ('anilist', 'myanimelist', 'simkl') |
| `client_user_id` | string | Yes | User ID from the external platform |
| `username` | string | Yes | Username from the external platform |
| `avatar_url` | string | No | Profile image URL |
| `verification_token` | string | No | Token for identity verification |

## Response

### Success Response (200 OK)

```typescript
interface IdentityResponse {
  user_id: number
  username: string
  role: 'user' | 'moderator' | 'admin' | 'super_admin'
  banned: boolean
  shadow_banned: boolean
  muted_until: string | null
  existing_user: boolean
}
```

### Error Responses

#### 400 Bad Request
```json
{
  "error": "Missing required fields: client_type, client_user_id, username"
}
```

#### 403 Forbidden
```json
{
  "error": "User banned or muted",
  "banned": true,
  "muted_until": "2024-01-01T00:00:00Z"
}
```

#### 500 Internal Server Error
```json
{
  "error": "Internal server error"
}
```

## Behavior

### User Resolution Flow

1. **Check Existing Identity**: The system first checks if an identity already exists for the given `client_type` and `client_user_id`.

2. **User Status Validation**: If the user exists, the system checks:
   - If the user is banned or muted
   - Updates identity information if username/avatar has changed

3. **User Linking**: If no identity exists but a user with the same username exists, the new identity is linked to the existing user.

4. **New User Creation**: If no matching user is found, a new user account is created with the 'user' role.

### Identity Management

- **Multiple Identities**: A single user can have multiple identities across different platforms
- **Username Updates**: Identity information is automatically updated when changes are detected
- **Cross-Platform Linking**: Users are linked by username across platforms

## Usage Examples

### Basic Identity Resolution

```javascript
const resolveIdentity = async (platform, userId, username, avatar) => {
  const response = await fetch('/identity-resolve', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      client_type: platform,
      client_user_id: userId,
      username: username,
      avatar_url: avatar
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error);
  }

  return await response.json();
};

// Example usage
const userInfo = await resolveIdentity(
  'anilist',
  '5724017',
  'ASheby',
  'https://example.com/avatar.jpg'
);

console.log(`User ID: ${userInfo.user_id}, Role: ${userInfo.role}`);
```

### Error Handling

```javascript
try {
  const userInfo = await resolveIdentity('anilist', '5724017', 'ASheby');
  
  if (userInfo.banned) {
    console.error('User is banned');
    return;
  }
  
  if (userInfo.muted_until && new Date(userInfo.muted_until) > new Date()) {
    console.error(`User is muted until ${userInfo.muted_until}`);
    return;
  }
  
  // Proceed with user session
  console.log('Identity resolved successfully:', userInfo);
  
} catch (error) {
  console.error('Identity resolution failed:', error.message);
}
```

### React Hook Example

```typescript
import { useState, useCallback } from 'react';

interface UseIdentityResult {
  resolveIdentity: (platform: string, userId: string, username: string, avatar?: string) => Promise<void>;
  user: IdentityResponse | null;
  loading: boolean;
  error: string | null;
}

export const useIdentity = (): UseIdentityResult => {
  const [user, setUser] = useState<IdentityResponse | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const resolveIdentity = useCallback(async (
    platform: string,
    userId: string,
    username: string,
    avatar?: string
  ) => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch('/identity-resolve', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          client_type: platform,
          client_user_id: userId,
          username,
          avatar_url: avatar
        })
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error);
      }

      setUser(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  }, []);

  return { resolveIdentity, user, loading, error };
};
```

## Security Considerations

- **Input Validation**: All inputs are validated for type and format
- **Rate Limiting**: Identity resolution is subject to rate limiting
- **No Sensitive Data**: The API never returns sensitive information like passwords or tokens

## Integration Notes

- This endpoint should be called once per user session to establish identity
- Store the returned `user_id` for subsequent API calls
- Handle banned/muted states appropriately in your UI
- Consider implementing caching to reduce API calls for the same user