# Authentication & Authorization Guide

This guide covers the authentication and authorization system in Commentum, including identity resolution, role-based access control, and security best practices.

## Table of Contents

- [Overview](#overview)
- [Identity System](#identity-system)
- [Authentication Flow](#authentication-flow)
- [Role-Based Access Control](#role-based-access-control)
- [Permission System](#permission-system)
- [Security Implementation](#security-implementation)
- [Integration Examples](#integration-examples)

## Overview

Commentum uses a flexible identity-based authentication system that supports multiple external platforms while maintaining a unified user management system.

### Key Concepts

- **Identity Resolution**: Maps external platform identities to internal users
- **Multi-Platform Support**: AniList, MyAnimeList, SIMKL integration
- **Role Hierarchy**: User → Moderator → Admin → Super Admin
- **Row Level Security**: Database-level access control
- **Session Management**: Stateless authentication with context passing

## Identity System

### User vs Identity

The system distinguishes between **Users** (internal accounts) and **Identities** (external platform accounts):

```
User (Internal Account)
├── Identity 1: AniList (user_id: 12345)
├── Identity 2: MyAnimeList (user_id: 67890)
└── Identity 3: SIMKL (user_id: abcdef)
```

### Identity Resolution Process

1. **External Authentication**: User authenticates with external platform
2. **Identity Lookup**: System checks for existing identity record
3. **User Linking**: 
   - If identity exists → Return associated user
   - If user with same username exists → Link identity to existing user
   - If no match → Create new user and identity
4. **Context Setting**: Set user context for subsequent requests

### Supported Platforms

| Platform | Client Type | Data Required | Notes |
|----------|-------------|---------------|-------|
| AniList | `anilist` | User ID, Username, Avatar | Popular anime/manga tracker |
| MyAnimeList | `myanimelist` | User ID, Username, Avatar | Large anime community |
| SIMKL | `simkl` | User ID, Username, Avatar | Universal media tracker |

## Authentication Flow

### 1. Platform Authentication

```typescript
// Example: AniList OAuth Flow
class AniListAuth {
  private clientId: string
  private clientSecret: string
  private redirectUri: string

  async authenticate(): Promise<AniListUser> {
    // 1. Redirect to AniList OAuth
    const authUrl = `https://anilist.co/api/v2/oauth/authorize?client_id=${this.clientId}&redirect_uri=${this.redirectUri}`
    
    // 2. Handle callback and get authorization code
    const code = await this.getAuthorizationCode()
    
    // 3. Exchange code for access token
    const token = await this.exchangeCodeForToken(code)
    
    // 4. Get user profile
    const user = await this.getUserProfile(token.access_token)
    
    return user
  }
  
  private async exchangeCodeForToken(code: string) {
    const response = await fetch('https://anilist.co/api/v2/oauth/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'authorization_code',
        client_id: this.clientId,
        client_secret: this.clientSecret,
        redirect_uri: this.redirectUri,
        code
      })
    })
    
    return response.json()
  }
  
  private async getUserProfile(accessToken: string) {
    const response = await fetch('https://graphql.anilist.co', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        query: `
          query {
            Viewer {
              id
              name
              avatar {
                large
              }
            }
          }
        `
      })
    })
    
    const data = await response.json()
    return data.data.Viewer
  }
}
```

### 2. Identity Resolution

```typescript
// services/identityService.ts
export class IdentityService {
  private apiClient: CommentumAPIClient

  constructor(apiClient: CommentumAPIClient) {
    this.apiClient = apiClient
  }

  async resolveIdentity(
    platform: 'anilist' | 'myanimelist' | 'simkl',
    externalUserId: string,
    username: string,
    avatarUrl?: string
  ): Promise<UserIdentity> {
    try {
      const response = await this.apiClient.resolveIdentity({
        client_type: platform,
        client_user_id: externalUserId,
        username,
        avatar_url: avatarUrl
      })

      // Handle different user states
      if (response.banned) {
        throw new AuthError('Account is banned', 'BANNED')
      }

      if (response.muted_until && new Date(response.muted_until) > new Date()) {
        throw new AuthError(`Account muted until ${response.muted_until}`, 'MUTED')
      }

      return response
    } catch (error) {
      if (error instanceof APIError) {
        throw new AuthError(error.message, error.code)
      }
      throw error
    }
  }

  async refreshIdentity(userId: number): Promise<UserIdentity> {
    // Re-resolve identity to get updated user state
    const response = await this.apiClient.request(`/users/${userId}`)
    return response
  }
}

class AuthError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode?: number
  ) {
    super(message)
    this.name = 'AuthError'
  }
}
```

### 3. Session Management

```typescript
// contexts/AuthContext.tsx
interface AuthState {
  user: UserIdentity | null
  loading: boolean
  error: string | null
  isAuthenticated: boolean
}

interface AuthContextType extends AuthState {
  login: (platform: string, credentials: any) => Promise<void>
  logout: () => void
  refreshUser: () => Promise<void>
}

export const AuthProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [state, setState] = useState<AuthState>({
    user: null,
    loading: false,
    error: null,
    isAuthenticated: false
  })

  const identityService = useMemo(() => new IdentityService(commentumClient), [])

  const login = async (platform: string, credentials: any) => {
    setState(prev => ({ ...prev, loading: true, error: null }))

    try {
      let platformUser: any

      // Authenticate with platform
      switch (platform) {
        case 'anilist':
          const anilistAuth = new AniListAuth()
          platformUser = await anilistAuth.authenticate()
          break
        case 'myanimelist':
          const malAuth = new MyAnimeListAuth()
          platformUser = await malAuth.authenticate(credentials)
          break
        case 'simkl':
          const simklAuth = new SimklAuth()
          platformUser = await simklAuth.authenticate()
          break
        default:
          throw new Error(`Unsupported platform: ${platform}`)
      }

      // Resolve identity with Commentum
      const user = await identityService.resolveIdentity(
        platform as any,
        platformUser.id.toString(),
        platformUser.name,
        platformUser.avatar?.large
      )

      // Store session
      localStorage.setItem('commentum_session', JSON.stringify({
        user,
        platform,
        expiresAt: Date.now() + (24 * 60 * 60 * 1000) // 24 hours
      }))

      setState({
        user,
        loading: false,
        error: null,
        isAuthenticated: true
      })
    } catch (error) {
      setState(prev => ({
        ...prev,
        loading: false,
        error: error instanceof Error ? error.message : 'Authentication failed'
      }))
    }
  }

  const logout = () => {
    localStorage.removeItem('commentum_session')
    setState({
      user: null,
      loading: false,
      error: null,
      isAuthenticated: false
    })
  }

  const refreshUser = async () => {
    if (!state.user) return

    try {
      const updatedUser = await identityService.refreshIdentity(state.user.user_id)
      setState(prev => ({
        ...prev,
        user: updatedUser
      }))
    } catch (error) {
      console.error('Failed to refresh user:', error)
    }
  }

  // Initialize session from storage
  useEffect(() => {
    const storedSession = localStorage.getItem('commentum_session')
    if (storedSession) {
      try {
        const session = JSON.parse(storedSession)
        if (session.expiresAt > Date.now()) {
          setState({
            user: session.user,
            loading: false,
            error: null,
            isAuthenticated: true
          })
        } else {
          logout()
        }
      } catch (error) {
        logout()
      }
    }
  }, [])

  return (
    <AuthContext.Provider value={{
      ...state,
      login,
      logout,
      refreshUser
    }}>
      {children}
    </AuthContext.Provider>
  )
}
```

## Role-Based Access Control

### Role Hierarchy

```typescript
enum UserRole {
  USER = 'user',
  MODERATOR = 'moderator',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

interface RolePermissions {
  level: number
  permissions: Permission[]
}

const ROLE_HIERARCHY: Record<UserRole, RolePermissions> = {
  [UserRole.USER]: {
    level: 0,
    permissions: [
      'comment:create',
      'comment:edit:own',
      'comment:delete:own',
      'vote:create',
      'report:create'
    ]
  },
  [UserRole.MODERATOR]: {
    level: 1,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.USER].permissions,
      'comment:delete:any',
      'comment:pin',
      'comment:lock',
      'comment:tag',
      'report:resolve',
      'user:warn',
      'user:mute'
    ]
  },
  [UserRole.ADMIN]: {
    level: 2,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.MODERATOR].permissions,
      'user:ban',
      'user:shadow_ban',
      'user:promote:moderator',
      'config:view:sensitive'
    ]
  },
  [UserRole.SUPER_ADMIN]: {
    level: 3,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.ADMIN].permissions,
      'user:promote:admin',
      'config:modify:any',
      'system:admin'
    ]
  }
}
```

### Permission Checking

```typescript
// utils/permissions.ts
type Permission = string // e.g., 'comment:create', 'user:ban'

export class PermissionChecker {
  private userRole: UserRole
  private roleHierarchy: Record<UserRole, RolePermissions>

  constructor(userRole: UserRole) {
    this.userRole = userRole
    this.roleHierarchy = ROLE_HIERARCHY
  }

  hasPermission(permission: Permission): boolean {
    const rolePermissions = this.roleHierarchy[this.userRole]
    return rolePermissions.permissions.includes(permission)
  }

  hasAnyPermission(permissions: Permission[]): boolean {
    return permissions.some(permission => this.hasPermission(permission))
  }

  hasAllPermissions(permissions: Permission[]): boolean {
    return permissions.every(permission => this.hasPermission(permission))
  }

  canTargetUser(targetRole: UserRole): boolean {
    const userLevel = this.roleHierarchy[this.userRole].level
    const targetLevel = this.roleHierarchy[targetRole].level
    
    // Super admins cannot be targeted
    if (targetRole === UserRole.SUPER_ADMIN) {
      return false
    }
    
    // Can only target users with lower level (except super admin can target anyone)
    return this.userRole === UserRole.SUPER_ADMIN || userLevel > targetLevel
  }

  getAvailableActions(): string[] {
    const rolePermissions = this.roleHierarchy[this.userRole]
    return rolePermissions.permissions
  }
}

// React Hook for permissions
export const usePermissions = (user?: UserIdentity | null) => {
  const permissionChecker = useMemo(() => {
    if (!user) return null
    return new PermissionChecker(user.role)
  }, [user])

  const hasPermission = useCallback((permission: Permission) => {
    return permissionChecker?.hasPermission(permission) || false
  }, [permissionChecker])

  const canModerate = useCallback(() => {
    return hasPermission('comment:delete:any')
  }, [hasPermission])

  const canAdmin = useCallback(() => {
    return hasPermission('user:ban')
  }, [hasPermission])

  return {
    permissionChecker,
    hasPermission,
    canModerate,
    canAdmin
  }
}
```

### Authorization Components

```typescript
// components/Authorization.tsx
interface AuthorizationProps {
  permission?: Permission
  permissions?: Permission[]
  role?: UserRole
  roles?: UserRole[]
  fallback?: ReactNode
  children: ReactNode
}

export const Authorization: React.FC<AuthorizationProps> = ({
  permission,
  permissions,
  role,
  roles,
  fallback = null,
  children
}) => {
  const { user } = useAuth()

  if (!user) {
    return <>{fallback}</>
  }

  const checker = new PermissionChecker(user.role)

  let isAuthorized = false

  if (permission) {
    isAuthorized = checker.hasPermission(permission)
  } else if (permissions) {
    isAuthorized = checker.hasAnyPermission(permissions)
  } else if (role) {
    isAuthorized = user.role === role
  } else if (roles) {
    isAuthorized = roles.includes(user.role)
  }

  return isAuthorized ? <>{children}</> : <>{fallback}</>
}

// Usage examples
export const ModerationPanel = () => (
  <Authorization permission="comment:delete:any" fallback={<p>Access denied</p>}>
    <ModerationTools />
  </Authorization>
)

export const AdminSettings = () => (
  <Authorization roles={[UserRole.ADMIN, UserRole.SUPER_ADMIN]}>
    <AdminPanel />
  </Authorization>
)
```

## Permission System

### Database-Level Permissions

Row Level Security (RLS) policies enforce permissions at the database level:

```sql
-- Example: Comments table RLS policy
CREATE POLICY "Users can edit own comments" ON comments
    FOR UPDATE USING (
        user_id = current_setting('app.current_user_id', true)::INTEGER AND
        NOT locked AND
        NOT deleted AND
        user_id NOT IN (
            SELECT id FROM users WHERE banned = TRUE OR muted_until > NOW()
        )
    );

-- Moderator policy
CREATE POLICY "Moderators can moderate comments" ON comments
    FOR ALL USING (
        current_setting('app.current_user_id', true)::INTEGER IN (
            SELECT id FROM users WHERE role IN ('moderator', 'admin', 'super_admin')
        )
    );
```

### Context Setting

```typescript
// middleware/authMiddleware.ts
export const setUserContext = async (userId: number, userRole: string) => {
  await commentumAPI.request('/set-context', {
    method: 'POST',
    body: JSON.stringify({
      user_id: userId.toString(),
      user_role: userRole
    })
  })
}

// API request wrapper
export const authenticatedRequest = async (
  userId: number,
  userRole: string,
  endpoint: string,
  options: RequestInit = {}
) => {
  // Set user context for RLS
  await setUserContext(userId, userRole)
  
  // Make the actual request
  return commentumAPI.request(endpoint, options)
}
```

### Permission Validation

```typescript
// services/authorizationService.ts
export class AuthorizationService {
  async validateCommentAction(
    userId: number,
    userRole: string,
    action: string,
    commentId?: number,
    targetUserId?: number
  ): Promise<boolean> {
    // Set user context
    await setUserContext(userId, userRole)

    try {
      // Attempt the action to validate permissions
      switch (action) {
        case 'edit':
          if (!commentId) return false
          await commentumAPI.request(`/comments/${commentId}/validate-edit`, {
            method: 'POST',
            body: JSON.stringify({ user_id: userId })
          })
          break
          
        case 'delete':
          if (!commentId) return false
          await commentumAPI.request(`/comments/${commentId}/validate-delete`, {
            method: 'POST',
            body: JSON.stringify({ user_id: userId })
          })
          break
          
        case 'moderate':
          if (!targetUserId) return false
          await commentumAPI.request('/users/validate-moderation', {
            method: 'POST',
            body: JSON.stringify({
              moderator_id: userId,
              target_user_id: targetUserId
            })
          })
          break
          
        default:
          return false
      }
      
      return true
    } catch (error) {
      return false
    }
  }

  async canPerformAction(
    user: UserIdentity,
    action: string,
    resource?: any
  ): Promise<boolean> {
    const checker = new PermissionChecker(user.role)
    
    // Check basic permission
    if (!checker.hasPermission(action)) {
      return false
    }

    // Check resource-specific permissions
    if (resource) {
      switch (action) {
        case 'comment:edit:own':
          return resource.user_id === user.user_id
          
        case 'user:moderate':
          return checker.canTargetUser(resource.role)
          
        case 'report:resolve':
          return resource.status === 'pending'
      }
    }

    return true
  }
}
```

## Security Implementation

### Token Security

```typescript
// utils/tokenSecurity.ts
export class TokenSecurity {
  private static readonly TOKEN_PREFIX = 'cm_' // Commentum token prefix
  
  static generateSessionToken(user: UserIdentity): string {
    const payload = {
      userId: user.user_id,
      role: user.role,
      timestamp: Date.now(),
      // Add any other non-sensitive data
    }
    
    const encoded = btoa(JSON.stringify(payload))
    return `${this.TOKEN_PREFIX}${encoded}`
  }
  
  static validateSessionToken(token: string): { valid: boolean; payload?: any } {
    if (!token.startsWith(this.TOKEN_PREFIX)) {
      return { valid: false }
    }
    
    try {
      const encoded = token.substring(this.TOKEN_PREFIX.length)
      const payload = JSON.parse(atob(encoded))
      
      // Check token age (24 hours)
      const age = Date.now() - payload.timestamp
      if (age > 24 * 60 * 60 * 1000) {
        return { valid: false }
      }
      
      return { valid: true, payload }
    } catch (error) {
      return { valid: false }
    }
  }
  
  static hashUserId(userId: number): string {
    // Hash user ID for client-side storage
    return btoa(`user_${userId}_${Date.now()}`).substring(0, 16)
  }
}
```

### Request Security

```typescript
// utils/requestSecurity.ts
export class RequestSecurity {
  private static readonly REQUEST_TIMEOUT = 10000 // 10 seconds
  private static readonly MAX_RETRIES = 3
  
  static async secureRequest(
    url: string,
    options: RequestInit & { userId?: number; userRole?: string } = {}
  ): Promise<Response> {
    const { userId, userRole, ...fetchOptions } = options
    
    // Add security headers
    const headers = {
      'X-Requested-With': 'XMLHttpRequest',
      'Cache-Control': 'no-cache',
      'Pragma': 'no-cache',
      ...fetchOptions.headers
    }
    
    // Add user context if available
    if (userId && userRole) {
      headers['X-User-ID'] = userId.toString()
      headers['X-User-Role'] = userRole
    }
    
    let lastError: Error
    
    for (let attempt = 1; attempt <= this.MAX_RETRIES; attempt++) {
      try {
        const controller = new AbortController()
        const timeoutId = setTimeout(() => controller.abort(), this.REQUEST_TIMEOUT)
        
        const response = await fetch(url, {
          ...fetchOptions,
          headers,
          signal: controller.signal
        })
        
        clearTimeout(timeoutId)
        
        // Check for security headers
        const securityHeaders = response.headers.get('X-Security-Check')
        if (securityHeaders) {
          console.warn('Security check failed:', securityHeaders)
        }
        
        return response
      } catch (error) {
        lastError = error as Error
        
        if (attempt === this.MAX_RETRIES) {
          break
        }
        
        // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt) * 1000))
      }
    }
    
    throw lastError!
  }
}
```

### CSRF Protection

```typescript
// utils/csrfProtection.ts
export class CSRFProtection {
  private static readonly TOKEN_KEY = 'csrf_token'
  private static readonly HEADER_NAME = 'X-CSRF-Token'
  
  static generateToken(): string {
    const array = new Uint8Array(32)
    crypto.getRandomValues(array)
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
  }
  
  static getToken(): string {
    let token = localStorage.getItem(this.TOKEN_KEY)
    
    if (!token) {
      token = this.generateToken()
      localStorage.setItem(this.TOKEN_KEY, token)
    }
    
    return token
  }
  
  static validateRequest(request: Request): boolean {
    const token = request.headers.get(this.HEADER_NAME)
    const storedToken = this.getToken()
    
    return token === storedToken
  }
  
  static addTokenToRequest(options: RequestInit): RequestInit {
    return {
      ...options,
      headers: {
        ...options.headers,
        [this.HEADER_NAME]: this.getToken()
      }
    }
  }
}
```

## Integration Examples

### Complete Authentication Flow

```typescript
// components/AuthFlow.tsx
export const AuthFlow: React.FC = () => {
  const { login, loading, error } = useAuth()
  const [selectedPlatform, setSelectedPlatform] = useState<string>('')
  
  const handlePlatformAuth = async (platform: string) => {
    try {
      await login(platform, {})
    } catch (error) {
      console.error('Authentication failed:', error)
    }
  }
  
  return (
    <div className="auth-flow">
      <h2>Sign in with your favorite platform</h2>
      
      <div className="platform-options">
        <button 
          onClick={() => handlePlatformAuth('anilist')}
          disabled={loading}
        >
          <img src="/anilist-logo.png" alt="AniList" />
          Sign in with AniList
        </button>
        
        <button 
          onClick={() => handlePlatformAuth('myanimelist')}
          disabled={loading}
        >
          <img src="/mal-logo.png" alt="MyAnimeList" />
          Sign in with MyAnimeList
        </button>
        
        <button 
          onClick={() => handlePlatformAuth('simkl')}
          disabled={loading}
        >
          <img src="/simkl-logo.png" alt="SIMKL" />
          Sign in with SIMKL
        </button>
      </div>
      
      {loading && <div>Authenticating...</div>}
      {error && <div className="error">{error}</div>}
    </div>
  )
}
```

### Protected Routes

```typescript
// components/ProtectedRoute.tsx
interface ProtectedRouteProps {
  children: ReactNode
  requiredPermission?: Permission
  requiredRole?: UserRole
  fallback?: ReactNode
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({
  children,
  requiredPermission,
  requiredRole,
  fallback = <div>Access denied</div>
}) => {
  const { isAuthenticated, user, loading } = useAuth()
  
  if (loading) {
    return <div>Loading...</div>
  }
  
  if (!isAuthenticated || !user) {
    return <AuthFlow />
  }
  
  if (requiredPermission && !new PermissionChecker(user.role).hasPermission(requiredPermission)) {
    return <>{fallback}</>
  }
  
  if (requiredRole && user.role !== requiredRole) {
    return <>{fallback}</>
  }
  
  return <>{children}</>
}

// Usage in app routing
const AppRoutes = () => (
  <Router>
    <Route path="/" element={<HomePage />} />
    <Route path="/media/:id" element={<MediaPage />} />
    
    <Route 
      path="/moderation" 
      element={
        <ProtectedRoute requiredPermission="comment:delete:any">
          <ModerationPage />
        </ProtectedRoute>
      } 
    />
    
    <Route 
      path="/admin" 
      element={
        <ProtectedRoute requiredRole={UserRole.ADMIN}>
          <AdminPage />
        </ProtectedRoute>
      } 
    />
  </Router>
)
```

### API Client with Auth

```typescript
// lib/authenticatedApiClient.ts
export class AuthenticatedAPIClient extends CommentumAPIClient {
  private getUserId(): number | null {
    const session = localStorage.getItem('commentum_session')
    if (!session) return null
    
    const parsed = JSON.parse(session)
    return parsed.user?.user_id || null
  }
  
  private getUserRole(): string | null {
    const session = localStorage.getItem('commentum_session')
    if (!session) return null
    
    const parsed = JSON.parse(session)
    return parsed.user?.role || null
  }
  
  protected async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
    const userId = this.getUserId()
    const userRole = this.getUserRole()
    
    if (!userId || !userRole) {
      throw new AuthError('Not authenticated', 'NOT_AUTHENTICATED')
    }
    
    // Add user context to request
    const authOptions = {
      ...options,
      headers: {
        ...options.headers,
        'X-User-ID': userId.toString(),
        'X-User-Role': userRole
      }
    }
    
    return super.request(endpoint, authOptions)
  }
  
  // Authenticated methods
  async createComment(params: CreateCommentParams) {
    return this.request('/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...params
      })
    })
  }
  
  async moderateUser(params: ModerateUserParams) {
    return this.request('/moderation', {
      method: 'POST',
      body: JSON.stringify(params)
    })
  }
}

export const authenticatedClient = new AuthenticatedAPIClient()
```

This authentication and authorization guide provides a comprehensive framework for implementing secure user management in Commentum. The system is designed to be flexible, secure, and easy to integrate with various frontend frameworks.