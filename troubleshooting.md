# 🔒 Troubleshooting Guide - Session-Based Authentication

This guide covers common issues, error scenarios, and solutions for implementing and maintaining the Commentum system with **secure session-based authentication**.

## 🚨 **SECURITY UPDATE**

**Previous troubleshooting methods contained vulnerable authentication patterns. This guide uses the new secure session-based authentication system.**

## Table of Contents

- [🔒 Session Authentication Issues](#-session-authentication-issues)
- [Common API Issues](#common-api-issues)
- [Database Problems](#database-problems)
- [Frontend Integration Problems](#frontend-integration-problems)
- [Performance Issues](#performance-issues)
- [Security Issues](#security-issues)
- [Debugging Tools](#debugging-tools)
- [FAQ](#faq)

## 🔒 Session Authentication Issues

### 1. Session Token Problems

**Problem**: Users cannot authenticate or get "Session expired" errors frequently

**Common Causes**:
- Missing session token in localStorage
- Invalid session token format
- Session token expired
- Server-side session validation failures

**Debugging Steps**:
```typescript
// Check session token status
function debugSessionToken() {
  const token = localStorage.getItem('commentum_session_token')
  console.log('Session token exists:', !!token)
  console.log('Session token format:', token ? /^[a-f0-9]{64}$/.test(token) : 'invalid')
  console.log('Session token length:', token?.length || 0)
  
  if (!token) {
    console.error('No session token found - user needs to login')
    return false
  }
  
  return true
}

// Test session validation
async function testSessionValidation() {
  const token = localStorage.getItem('commentum_session_token')
  if (!token) return false
  
  try {
    const response = await fetch('/functions/v1/comments', {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    })
    
    if (response.status === 401) {
      console.error('Session validation failed - token expired or invalid')
      localStorage.removeItem('commentum_session_token')
      return false
    }
    
    console.log('Session validation successful')
    return true
  } catch (error) {
    console.error('Session validation error:', error)
    return false
  }
}
```

**Solutions**:
```typescript
// Implement proper session management
class SessionManager {
  constructor() {
    this.storageKey = 'commentum_session_token'
    this.validateTokenOnLoad = true
  }

  setSessionToken(token: string) {
    // Validate token format before storing
    if (!/^[a-f0-9]{64}$/.test(token)) {
      throw new Error('Invalid session token format')
    }
    
    localStorage.setItem(this.storageKey, token)
    console.log('Session token stored successfully')
  }

  getSessionToken(): string | null {
    const token = localStorage.getItem(this.storageKey)
    
    if (!token) {
      console.log('No session token found')
      return null
    }
    
    // Validate token format
    if (!/^[a-f0-9]{64}$/.test(token)) {
      console.error('Invalid session token format, clearing...')
      this.clearSessionToken()
      return null
    }
    
    return token
  }

  clearSessionToken() {
    localStorage.removeItem(this.storageKey)
    console.log('Session token cleared')
  }

  async validateSession(): Promise<boolean> {
    const token = this.getSessionToken()
    if (!token) return false
    
    try {
      const response = await fetch('/functions/v1/comments', {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })
      
      if (response.status === 401) {
        this.clearSessionToken()
        return false
      }
      
      return true
    } catch (error) {
      console.error('Session validation error:', error)
      this.clearSessionToken()
      return false
    }
  }
}
```

### 2. Identity Resolution Failures

**Problem**: Users cannot authenticate with external providers

**Debugging**:
```typescript
// Debug identity resolution with real provider validation
async function debugIdentityResolution(clientType: string, providerToken: string) {
  console.log('=== Identity Resolution Debug ===')
  console.log('Client Type:', clientType)
  console.log('Provider Token Length:', providerToken.length)
  console.log('Provider Token Prefix:', providerToken.substring(0, 10) + '...')
  
  try {
    const response = await fetch('/functions/v1/identity-resolve', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        client_type: clientType,
        token: providerToken
      })
    })

    console.log('Response Status:', response.status)
    console.log('Response Headers:', Object.fromEntries(response.headers.entries()))

    if (!response.ok) {
      const error = await response.json()
      console.error('Identity Resolution Error:', error)
      return { success: false, error }
    }

    const data = await response.json()
    console.log('Identity Resolution Success:', data)
    
    // Validate response structure
    if (!data.session_token || !data.user) {
      console.error('Invalid response structure:', data)
      return { success: false, error: 'Invalid response structure' }
    }

    return { success: true, data }
  } catch (error) {
    console.error('Identity Resolution Exception:', error)
    return { success: false, error: error.message }
  }
}
```

**Common Issues**:
- Invalid provider token (expired, revoked, incorrect format)
- Network connectivity issues with provider APIs
- Rate limiting from provider APIs
- Incorrect client_type values

### 3. Session Expiration Handling

**Problem**: Users are logged out unexpectedly

**Solutions**:
```typescript
// Implement proactive session refresh
class SessionRefreshManager {
  constructor(sessionManager, authContext) {
    this.sessionManager = sessionManager
    this.authContext = authContext
    this.refreshInterval = null
    this.warningShown = false
  }

  startProactiveRefresh() {
    // Check session every 5 minutes
    this.refreshInterval = setInterval(async () => {
      const isValid = await this.sessionManager.validateSession()
      
      if (!isValid) {
        this.handleSessionExpired()
      }
    }, 5 * 60 * 1000) // 5 minutes
  }

  handleSessionExpired() {
    if (!this.warningShown) {
      this.warningShown = true
      console.warn('Session expired - user will need to log in again')
      
      // Show warning to user (optional)
      this.showSessionExpiredWarning()
    }
    
    // Clear expired session
    this.sessionManager.clearSessionToken()
    this.authContext.logout()
    
    // Stop refresh attempts
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval)
      this.refreshInterval = null
    }
  }

  showSessionExpiredWarning() {
    // Implement user notification
    if (typeof window !== 'undefined') {
      const warning = document.createElement('div')
      warning.className = 'session-expired-warning'
      warning.textContent = 'Your session has expired. Please log in again.'
      warning.style.cssText = `
        position: fixed;
        top: 20px;
        right: 20px;
        background: #f59e0b;
        color: white;
        padding: 12px 16px;
        border-radius: 6px;
        z-index: 1000;
        box-shadow: 0 4px 6px rgba(0,0,0,0.1);
      `
      
      document.body.appendChild(warning)
      
      setTimeout(() => {
        if (warning.parentNode) {
          warning.parentNode.removeChild(warning)
        }
      }, 5000)
    }
  }

  stop() {
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval)
      this.refreshInterval = null
    }
  }
}
```

## Common API Issues

### 1. CORS Errors

**Problem**: 
```
Access to fetch at 'https://your-project.supabase.co/functions/v1/comments' from origin 'http://localhost:3000' has been blocked by CORS policy
```

**Solutions**:
- Ensure your Supabase project has the correct CORS settings
- Add your development domain to allowed origins in Supabase dashboard
- Check that you're using HTTPS in production

**Supabase CORS Configuration**:
```sql
-- Run in Supabase SQL Editor
INSERT INTO cors (
  origin,
  methods,
  headers,
  max_age
) VALUES (
  'http://localhost:3000',
  'GET,POST,PUT,DELETE,OPTIONS',
  'authorization, x-client-info, apikey, content-type',
  3600
);
```

### 2. 401 Unauthorized Errors (Session-Based)

**Problem**: API requests return 401 status with session-related errors

**Common Causes**:
- Missing or invalid session token
- Expired session token
- Session token format validation failure

**Debugging Steps**:
```typescript
// Check session authentication status
async function debugSessionAuth() {
  const sessionToken = localStorage.getItem('commentum_session_token')
  
  console.log('=== Session Auth Debug ===')
  console.log('Session Token exists:', !!sessionToken)
  
  if (!sessionToken) {
    console.error('No session token found - user needs to login')
    return false
  }
  
  console.log('Session Token length:', sessionToken.length)
  console.log('Session Token format valid:', /^[a-f0-9]{64}$/.test(sessionToken))
  
  // Test API call with session token
  try {
    const response = await fetch('/functions/v1/comments', {
      headers: {
        'Authorization': `Bearer ${sessionToken}`,
        'Content-Type': 'application/json'
      }
    })
    
    console.log('API Response Status:', response.status)
    
    if (response.status === 401) {
      const error = await response.json()
      console.error('Session authentication failed:', error)
      
      // Clear invalid session
      localStorage.removeItem('commentum_session_token')
      return false
    }
    
    console.log('Session authentication successful')
    return true
  } catch (error) {
    console.error('Session authentication error:', error)
    return false
  }
}
```

### 3. 403 Forbidden Errors

**Problem**: API requests return 403 with permission-related errors

**Common Causes**:
- User not authenticated (no valid session)
- User banned or muted
- Insufficient permissions for action
- Row Level Security (RLS) policy blocking access

**Debugging Steps**:
```typescript
// Check user authentication and permissions
async function debugUserPermissions() {
  const sessionToken = localStorage.getItem('commentum_session_token')
  if (!sessionToken) {
    console.error('No session token - cannot check permissions')
    return
  }

  try {
    // Test basic API access
    const commentsResponse = await fetch('/functions/v1/comments?media_id=1', {
      headers: {
        'Authorization': `Bearer ${sessionToken}`,
        'Content-Type': 'application/json'
      }
    })
    
    console.log('Comments API Status:', commentsResponse.status)
    
    if (commentsResponse.status === 403) {
      const error = await commentsResponse.json()
      console.error('Comments API forbidden:', error)
      
      // Check if user is banned
      if (error.error?.includes('banned') || error.error?.includes('muted')) {
        console.error('User account restricted:', error.error)
      }
    }
    
    // Test voting permissions
    const voteResponse = await fetch('/functions/v1/voting', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${sessionToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        action: 'upvote',
        comment_id: 1
      })
    })
    
    console.log('Voting API Status:', voteResponse.status)
    
  } catch (error) {
    console.error('Permission check error:', error)
  }
}
```

### 4. Rate Limiting Issues

**Problem**: Getting 429 "Too Many Requests" errors

**Solutions**:
```typescript
// Implement client-side rate limiting with session awareness
class SessionRateLimiter {
  constructor(maxRequests = 30, windowMs = 3600000) {
    this.maxRequests = maxRequests
    this.windowMs = windowMs
    this.requests = new Map() // Track by session token
  }

  canProceed(sessionToken: string) {
    if (!sessionToken) {
      throw new Error('No session token provided')
    }

    const now = Date.now()
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs
    
    if (!this.requests.has(sessionToken)) {
      this.requests.set(sessionToken, [])
    }
    
    const userRequests = this.requests.get(sessionToken)
    
    // Filter requests within current window
    const validRequests = userRequests.filter(time => now - time < this.windowMs)
    
    if (validRequests.length >= this.maxRequests) {
      const oldestRequest = Math.min(...validRequests)
      const resetTime = oldestRequest + this.windowMs
      throw new Error(`Rate limit exceeded. Reset at ${new Date(resetTime)}`)
    }
    
    validRequests.push(now)
    this.requests.set(sessionToken, validRequests)
    return true
  }

  // Clear expired requests for all sessions
  cleanup() {
    const now = Date.now()
    for (const [sessionToken, requests] of this.requests.entries()) {
      const validRequests = requests.filter(time => now - time < this.windowMs)
      if (validRequests.length === 0) {
        this.requests.delete(sessionToken)
      } else {
        this.requests.set(sessionToken, validRequests)
      }
    }
  }
}

const rateLimiter = new SessionRateLimiter()

// Wrap API calls with rate limiting
async function makeAuthenticatedAPIRequest(url: string, options: RequestInit = {}) {
  const sessionToken = localStorage.getItem('commentum_session_token')
  
  if (!sessionToken) {
    throw new Error('No session token - please log in')
  }
  
  // Check rate limit
  rateLimiter.canProceed(sessionToken)
  
  // Add session token to headers
  const headers = new Headers(options.headers || {})
  headers.set('Authorization', `Bearer ${sessionToken}`)
  
  return fetch(url, {
    ...options,
    headers
  })
}
```

## Database Problems

### 1. Connection Issues

**Problem**: Database connection timeouts or failures

**Diagnosis**:
```sql
-- Test database connectivity
SELECT version();
SELECT current_database(), current_user;

-- Check table existence
SELECT table_name, table_type 
FROM information_schema.tables 
WHERE table_schema = 'public';

-- Check RLS status
SELECT tablename, rowsecurity 
FROM pg_tables 
WHERE schemaname = 'public';
```

**Solutions**:
- Verify database connection string
- Check database pool settings
- Ensure database is not in maintenance mode

### 2. RLS Policy Issues

**Problem**: Queries return no data or permission errors

**Common RLS Issues**:
```sql
-- Check if RLS is enabled
SELECT tablename, rowsecurity 
FROM pg_tables 
WHERE tablename IN ('comments', 'users', 'comment_votes');

-- Check existing policies
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual 
FROM pg_policies 
WHERE tablename = 'comments';

-- Test RLS with specific user context
SELECT set_config('app.current_user_id', '123', true);
SELECT set_config('app.current_role', 'authenticated', true);

-- Test query
SELECT * FROM comments LIMIT 1;
```

**Fixing Common RLS Issues**:
```sql
-- Fix missing user context policy
CREATE POLICY "Enable read access for all users" ON comments
    FOR SELECT USING (true);

-- Fix overly restrictive policies
DROP POLICY IF EXISTS "Too restrictive policy" ON comments;

-- Create proper user-based policy
CREATE POLICY "Users can view comments" ON comments
    FOR SELECT USING (
        NOT deleted AND
        user_id NOT IN (
            SELECT id FROM users WHERE shadow_banned = TRUE
        )
    );
```

### 3. Missing Functions

**Problem**: Function does not exist errors

**Solution**: Ensure all database functions are created:
```sql
-- Check if functions exist
SELECT routine_name, routine_type 
FROM information_schema.routines 
WHERE routine_schema = 'public';

-- Recreate missing functions
-- Run the database/functions.sql file from your repository
```

### 4. Index Performance Issues

**Problem**: Slow query performance

**Diagnosis**:
```sql
-- Check query execution plan
EXPLAIN ANALYZE 
SELECT * FROM comments 
WHERE media_id = 123 
ORDER BY created_at DESC 
LIMIT 50;

-- Check missing indexes
SELECT 
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats 
WHERE tablename = 'comments';

-- Create missing indexes if needed
CREATE INDEX CONCURRENTLY IF NOT EXISTS 
idx_comments_media_created ON comments(media_id, created_at DESC);
```

## Authentication Issues

### 1. Identity Resolution Failures

**Problem**: Users cannot authenticate or get created as new users unexpectedly

**Debugging**:
```typescript
// Log identity resolution process
async function debugIdentityResolution(clientType, clientUserId, username) {
  console.log('Resolving identity:', { clientType, clientUserId, username })
  
  try {
    const response = await commentumAPI.request('/identity-resolve', {
      method: 'POST',
      body: JSON.stringify({
        client_type: clientType,
        client_user_id: clientUserId,
        username
      })
    })
    
    console.log('Identity resolution result:', response)
    return response
  } catch (error) {
    console.error('Identity resolution failed:', error)
    throw error
  }
}
```

**Common Issues**:
- Invalid client_type values
- Missing required fields
- Username conflicts
- Database constraint violations

### 2. Session Management Issues

**Problem**: Users lose authentication state

**Solutions**:
```typescript
// Implement proper session persistence
class SessionManager {
  constructor() {
    this.storageKey = 'commentum_session'
  }

  saveSession(user) {
    const session = {
      user,
      expiresAt: Date.now() + (24 * 60 * 60 * 1000), // 24 hours
      timestamp: Date.now()
    }
    
    localStorage.setItem(this.storageKey, JSON.stringify(session))
  }

  loadSession() {
    const stored = localStorage.getItem(this.storageKey)
    if (!stored) return null
    
    try {
      const session = JSON.parse(stored)
      if (session.expiresAt > Date.now()) {
        return session.user
      } else {
        this.clearSession()
        return null
      }
    } catch (error) {
      this.clearSession()
      return null
    }
  }

  clearSession() {
    localStorage.removeItem(this.storageKey)
  }

  isSessionValid() {
    const stored = localStorage.getItem(this.storageKey)
    if (!stored) return false
    
    try {
      const session = JSON.parse(stored)
      return session.expiresAt > Date.now()
    } catch {
      return false
    }
  }
}
```

## Frontend Integration Problems

### 1. React Integration Issues

**Problem**: Components not re-rendering when data changes

**Solutions**:
```typescript
// Ensure proper React Query usage
import { useQuery, useQueryClient } from '@tanstack/react-query'

function CommentSystem({ mediaId }) {
  const queryClient = useQueryClient()
  
  const { data, error, isLoading } = useQuery({
    queryKey: ['comments', mediaId],
    queryFn: () => fetchComments(mediaId),
    staleTime: 5 * 60 * 1000, // 5 minutes
  })

  // Properly invalidate queries after mutations
  const createComment = useMutation({
    mutationFn: createCommentAPI,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['comments', mediaId] })
    }
  })

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
  
  return <CommentList comments={data || []} />
}
```

### 2. Vue.js Integration Issues

**Problem**: Pinia store not updating properly

**Solutions**:
```typescript
// Ensure proper store reactive updates
import { defineStore } from 'pinia'

export const useCommentStore = defineStore('comments', () => {
  const comments = ref([])
  const loading = ref(false)

  const fetchComments = async (mediaId) => {
    loading.value = true
    try {
      const response = await api.getComments(mediaId)
      comments.value = response // Direct assignment for reactivity
    } catch (error) {
      console.error('Failed to fetch comments:', error)
      throw error
    } finally {
      loading.value = false
    }
  }

  return {
    comments: readonly(comments),
    loading: readonly(loading),
    fetchComments
  }
})
```

### 3. Vanilla JS Issues

**Problem**: Event listeners not working properly

**Solutions**:
```javascript
// Proper event delegation for dynamic content
class CommentSystem {
  constructor() {
    this.container = document.getElementById('comment-system')
    this.bindEventDelegation()
  }

  bindEventDelegation() {
    // Use event delegation for dynamic content
    this.container.addEventListener('click', (event) => {
      const target = event.target
      
      if (target.matches('.vote-button')) {
        this.handleVote(target)
      } else if (target.matches('.reply-button')) {
        this.handleReply(target)
      } else if (target.matches('.edit-button')) {
        this.handleEdit(target)
      }
    })

    // Handle form submissions
    this.container.addEventListener('submit', (event) => {
      if (event.target.matches('.comment-form')) {
        event.preventDefault()
        this.handleCommentSubmit(event.target)
      }
    })
  }

  handleVote(button) {
    const commentId = button.closest('.comment-item').dataset.commentId
    const action = button.classList.contains('upvote') ? 'upvote' : 'downvote'
    // Handle vote logic
  }
}
```

## Performance Issues

### 1. Slow Initial Load

**Problem**: Comments take too long to load

**Solutions**:
```typescript
// Implement pagination and lazy loading
class CommentLoader {
  constructor(api, options = {}) {
    this.api = api
    this.pageSize = options.pageSize || 20
    this.cache = new Map()
  }

  async loadComments(mediaId, page = 1) {
    const cacheKey = `${mediaId}-${page}`
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)
    }

    const comments = await this.api.getComments(mediaId, {
      limit: this.pageSize,
      offset: (page - 1) * this.pageSize
    })

    this.cache.set(cacheKey, comments)
    return comments
  }

  // Implement virtual scrolling for large comment lists
  createVirtualScroll(container, comments) {
    const itemHeight = 120 // Estimated height per comment
    const visibleItems = Math.ceil(container.clientHeight / itemHeight) + 2
    
    let startIndex = 0
    let endIndex = Math.min(visibleItems, comments.length)

    const render = () => {
      const visibleComments = comments.slice(startIndex, endIndex)
      this.renderComments(container, visibleComments, startIndex * itemHeight)
    }

    container.addEventListener('scroll', () => {
      const scrollTop = container.scrollTop
      startIndex = Math.floor(scrollTop / itemHeight)
      endIndex = Math.min(startIndex + visibleItems, comments.length)
      render()
    })

    render()
  }
}
```

### 2. Memory Leaks

**Problem**: Memory usage increases over time

**Solutions**:
```typescript
// Proper cleanup in React components
function CommentSystem({ mediaId }) {
  useEffect(() => {
    // Set up subscriptions or intervals
    const interval = setInterval(() => {
      // Refresh comments
    }, 30000)

    // Cleanup function
    return () => {
      clearInterval(interval)
    }
  }, [mediaId])

  // Component logic
}

// Clean up event listeners in vanilla JS
class CommentSystem {
  destroy() {
    // Remove event listeners
    this.eventListeners.forEach(({ element, event, handler }) => {
      element.removeEventListener(event, handler)
    })
    
    // Clear intervals
    this.intervals.forEach(interval => clearInterval(interval))
    
    // Clear cache
    this.cache.clear()
  }
}
```

## Security Issues

### 1. XSS Vulnerabilities

**Problem**: User content contains malicious scripts

**Solutions**:
```typescript
// Implement proper content sanitization
import DOMPurify from 'dompurify'

function sanitizeContent(content) {
  return DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'i', 'b', 'a'],
    ALLOWED_ATTR: ['href', 'title'],
    ALLOW_DATA_ATTR: false
  })
}

// In React, use dangerouslySetInnerHTML with sanitized content
function CommentContent({ content }) {
  const sanitizedContent = sanitizeContent(content)
  
  return (
    <div 
      className="comment-text"
      dangerouslySetInnerHTML={{ __html: sanitizedContent }}
    />
  )
}
```

### 2. CSRF Protection

**Problem**: Missing CSRF protection on API requests

**Solutions**:
```typescript
// Implement CSRF token protection
class CSRFProtection {
  constructor() {
    this.token = this.generateToken()
    this.injectToken()
  }

  generateToken() {
    const array = new Uint8Array(32)
    crypto.getRandomValues(array)
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
  }

  injectToken() {
    const meta = document.createElement('meta')
    meta.name = 'csrf-token'
    meta.content = this.token
    document.head.appendChild(meta)
  }

  addToRequest(options = {}) {
    const headers = new Headers(options.headers || {})
    headers.set('X-CSRF-Token', this.token)
    
    return {
      ...options,
      headers
    }
  }
}

const csrf = new CSRFProtection()

// Use with fetch
fetch('/api/comments', csrf.addToRequest({
  method: 'POST',
  body: JSON.stringify(data)
}))
```

## Debugging Tools

### 1. API Request Logger

```typescript
class APILogger {
  constructor() {
    this.requests = []
    this.maxLogs = 100
  }

  log(request, response, error = null) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      request: {
        url: request.url,
        method: request.method,
        headers: this.sanitizeHeaders(request.headers),
        body: request.body
      },
      response: response ? {
        status: response.status,
        statusText: response.statusText,
        headers: this.sanitizeHeaders(response.headers)
      } : null,
      error: error ? {
        message: error.message,
        stack: error.stack
      } : null
    }

    this.requests.push(logEntry)
    
    if (this.requests.length > this.maxLogs) {
      this.requests.shift()
    }

    this.updateLogDisplay()
  }

  sanitizeHeaders(headers) {
    const sanitized = {}
    for (const [key, value] of headers.entries()) {
      if (key.toLowerCase().includes('authorization') || 
          key.toLowerCase().includes('api-key')) {
        sanitized[key] = '[REDACTED]'
      } else {
        sanitized[key] = value
      }
    }
    return sanitized
  }

  updateLogDisplay() {
    const logContainer = document.getElementById('api-logs')
    if (!logContainer) return

    logContainer.innerHTML = this.requests.map(log => `
      <div class="log-entry ${log.error ? 'error' : 'success'}">
        <div class="log-time">${log.timestamp}</div>
        <div class="log-request">
          <strong>${log.request.method} ${log.request.url}</strong>
        </div>
        ${log.response ? `
          <div class="log-response">
            Status: ${log.response.status} ${log.response.statusText}
          </div>
        ` : ''}
        ${log.error ? `
          <div class="log-error">
            Error: ${log.error.message}
          </div>
        ` : ''}
      </div>
    `).join('')
  }

  exportLogs() {
    const dataStr = JSON.stringify(this.requests, null, 2)
    const dataBlob = new Blob([dataStr], { type: 'application/json' })
    const url = URL.createObjectURL(dataBlob)
    
    const link = document.createElement('a')
    link.href = url
    link.download = `api-logs-${Date.now()}.json`
    link.click()
    
    URL.revokeObjectURL(url)
  }
}

// Wrap fetch to log all requests
const originalFetch = window.fetch
const logger = new APILogger()

window.fetch = async (...args) => {
  const [request, options] = args
  const startTime = Date.now()
  
  try {
    const response = await originalFetch(...args)
    const endTime = Date.now()
    
    logger.log(request, response, null)
    
    return response
  } catch (error) {
    const endTime = Date.now()
    
    logger.log(request, null, error)
    
    throw error
  }
}
```

### 2. Performance Monitor

```typescript
class PerformanceMonitor {
  constructor() {
    this.metrics = {}
    this.observers = []
  }

  startMonitoring() {
    // Monitor API response times
    this.measureAPIPerformance()

    // Monitor render performance
    this.measureRenderPerformance()

    // Monitor memory usage
    this.measureMemoryUsage()
  }

  measureAPIPerformance() {
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (entry.initiatorType === 'fetch') {
          this.recordMetric('api_request', {
            url: entry.name,
            duration: entry.duration,
            size: entry.transferSize
          })
        }
      })
    })

    observer.observe({ entryTypes: ['resource'] })
    this.observers.push(observer)
  }

  measureRenderPerformance() {
    const measureRender = (name, fn) => {
      const start = performance.now()
      const result = fn()
      const end = performance.now()
      
      this.recordMetric('render', {
        name,
        duration: end - start
      })
      
      return result
    }

    // Make available globally
    window.measureRender = measureRender
  }

  measureMemoryUsage() {
    if ('memory' in performance) {
      setInterval(() => {
        this.recordMetric('memory', {
          used: performance.memory.usedJSHeapSize,
          total: performance.memory.totalJSHeapSize,
          limit: performance.memory.jsHeapSizeLimit
        })
      }, 10000) // Every 10 seconds
    }
  }

  recordMetric(type, data) {
    if (!this.metrics[type]) {
      this.metrics[type] = []
    }
    
    this.metrics[type].push({
      timestamp: Date.now(),
      ...data
    })

    // Keep only last 100 entries per type
    if (this.metrics[type].length > 100) {
      this.metrics[type].shift()
    }
  }

  getMetrics() {
    return this.metrics
  }

  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      metrics: this.metrics,
      summary: this.generateSummary()
    }

    return report
  }

  generateSummary() {
    const summary = {}

    Object.entries(this.metrics).forEach(([type, entries]) => {
      if (type === 'api_request') {
        const durations = entries.map(e => e.duration)
        summary[type] = {
          count: entries.length,
          avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
          maxDuration: Math.max(...durations),
          minDuration: Math.min(...durations)
        }
      } else if (type === 'memory') {
        const latest = entries[entries.length - 1]
        summary[type] = {
          used: latest.used,
          total: latest.total,
          utilization: (latest.used / latest.total) * 100
        }
      }
    })

    return summary
  }
}

// Initialize performance monitoring
const perfMonitor = new PerformanceMonitor()
perfMonitor.startMonitoring()

// Make available for debugging
window.perfMonitor = perfMonitor
```

## FAQ

### Q: Why are my comments not appearing after posting?
**A**: Check the following:
1. Verify the API request was successful (check network tab)
2. Ensure you're calling the refresh/reload function after successful creation
3. Check if the comment was marked as deleted or hidden by moderation
4. Verify RLS policies allow viewing the comment

### Q: Why do I get "User not found" errors?
**A**: Common causes:
1. User session expired - try re-authenticating
2. Invalid user ID being passed to API calls
3. Database user record was deleted or modified
4. RLS policies blocking user access

### Q: How do I handle real-time updates?
**A**: Implement WebSocket connections:
```typescript
const ws = new WebSocket('wss://your-project.supabase.co/realtime')
ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  if (data.type === 'comment_created') {
    // Update local state
  }
}
```

### Q: Why are votes not updating immediately?
**A**: Votes may be cached or debounced. Solutions:
1. Implement optimistic updates for better UX
2. Reduce cache duration for vote counts
3. Use real-time subscriptions for instant updates

### Q: How do I handle large numbers of comments?
**A**: Implement pagination and virtual scrolling:
1. Load comments in pages (20-50 per page)
2. Use virtual scrolling for smooth performance
3. Implement "load more" functionality
4. Consider lazy loading of replies

### Q: Why is my moderation panel not working?
**A**: Check permissions and RLS policies:
1. Verify user has moderator/admin role
2. Check RLS policies allow moderation actions
3. Ensure proper user context is set
4. Test with a super admin account

### Q: How do I debug RLS policy issues?
**A**: Use these debugging steps:
```sql
-- Test with specific user context
SELECT set_config('app.current_user_id', '123', true);
SELECT set_config('app.current_role', 'moderator', true);

-- Test the query
SELECT * FROM comments LIMIT 1;

-- Check policy definitions
SELECT * FROM pg_policies WHERE tablename = 'comments';
```

### Q: What should I do if I get database constraint errors?
**A**: Common constraint issues:
1. **Duplicate key errors**: Check for existing records before insertion
2. **Foreign key violations**: Ensure referenced records exist
3. **Check constraint violations**: Validate data before insertion
4. **Not null violations**: Ensure required fields are provided

This troubleshooting guide should help you identify and resolve most common issues with the Commentum system. For additional support, check the logs and use the debugging tools provided.