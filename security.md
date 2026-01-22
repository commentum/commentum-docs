# 🔒 Security Best Practices Guide

This guide covers comprehensive security best practices for implementing and maintaining the Commentum system with **session-based authentication**.

## 🚨 **CRITICAL SECURITY OVERVIEW**

Commentum implements a **zero-trust architecture** that prevents identity spoofing attacks through secure session-based authentication. This guide ensures your implementation remains secure.

## Table of Contents

- [🔒 Authentication Security](#-authentication-security)
- [API Security](#api-security)
- [Frontend Security](#frontend-security)
- [Database Security](#database-security)
- [Session Management](#session-management)
- [Rate Limiting](#rate-limiting)
- [Input Validation](#input-validation)
- [Monitoring & Logging](#monitoring--logging)
- [Common Vulnerabilities](#common-vulnerabilities)
- [Security Checklist](#security-checklist)

## 🔒 Authentication Security

### 1. Session Token Management

**✅ Best Practices:**
```typescript
// ✅ Secure session token generation and storage
class SecureSessionManager {
  private readonly storageKey = 'commentum_session_token'
  private readonly tokenPattern = /^[a-f0-9]{64}$/

  setSessionToken(token: string): void {
    // Validate token format before storing
    if (!this.tokenPattern.test(token)) {
      throw new Error('Invalid session token format')
    }
    
    // Use secure storage
    if (typeof window !== 'undefined') {
      localStorage.setItem(this.storageKey, token)
    }
  }

  getSessionToken(): string | null {
    if (typeof window === 'undefined') return null
    
    const token = localStorage.getItem(this.storageKey)
    return this.tokenPattern.test(token || '') ? token : null
  }

  clearSessionToken(): void {
    if (typeof window !== 'undefined') {
      localStorage.removeItem(this.storageKey)
    }
  }

  // Validate session with server
  async validateSession(token: string): Promise<boolean> {
    try {
      const response = await fetch('/functions/v1/comments', {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })
      
      return response.status !== 401
    } catch {
      return false
    }
  }
}
```

**❌ Anti-Patterns:**
```typescript
// ❌ NEVER store user_id or sensitive data in localStorage
localStorage.setItem('user_id', '123') // VULNERABLE

// ❌ NEVER trust client-provided user data
const userId = req.body.user_id // VULNERABLE

// ❌ NEVER use predictable session tokens
const sessionToken = 'user_' + userId // VULNERABLE
```

### 2. Provider Token Security

**✅ Best Practices:**
```typescript
// ✅ Secure provider token handling
class ProviderTokenHandler {
  async validateProviderToken(
    clientType: 'anilist' | 'mal' | 'simkl',
    token: string
  ): Promise<boolean> {
    try {
      // Validate with real provider API
      const response = await fetch(this.getProviderValidationURL(clientType), {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        }
      })
      
      return response.ok
    } catch {
      return false
    }
  }

  private getProviderValidationURL(clientType: string): string {
    const urls = {
      'anilist': 'https://graphql.anilist.co',
      'mal': 'https://api.myanimelist.net/v2',
      'simkl': 'https://api.simkl.com'
    }
    return urls[clientType]
  }

  // Never log or store provider tokens
  sanitizeTokenForLogging(token: string): string {
    return token.substring(0, 8) + '...'
  }
}
```

## API Security

### 1. Request Security Headers

**✅ Best Practices:**
```typescript
// ✅ Secure API client with proper headers
class SecureAPIClient {
  private baseURL: string
  private sessionToken: string | null = null

  private getSecureHeaders(): Record<string, string> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      'X-Requested-With': 'XMLHttpRequest'
    }

    // Add session token for authenticated requests
    if (this.sessionToken) {
      headers['Authorization'] = `Bearer ${this.sessionToken}`
    }

    return headers
  }

  async makeAuthenticatedRequest(endpoint: string, options: RequestInit = {}): Promise<Response> {
    if (!this.sessionToken) {
      throw new Error('No session token - authentication required')
    }

    const url = `${this.baseURL}${endpoint}`
    const headers = this.getSecureHeaders()

    const response = await fetch(url, {
      ...options,
      headers
    })

    // Handle session expiration
    if (response.status === 401) {
      this.clearSessionToken()
      throw new Error('Session expired - please log in again')
    }

    return response
  }
}
```

### 2. Request Validation

**✅ Best Practices:**
```typescript
// ✅ Input validation for API requests
interface CommentRequest {
  action: 'create' | 'edit' | 'delete' | 'vote' | 'report'
  media_id?: number
  media_info?: MediaInfo
  content?: string
  comment_id?: number
  parent_id?: number
  reason?: string
  notes?: string
}

// ✅ Request validation
function validateCommentRequest(request: CommentRequest): string[] {
  const errors: string[] = []

  // Validate action
  const validActions = ['create', 'edit', 'delete', 'vote', 'report']
  if (!validActions.includes(request.action)) {
    errors.push('Invalid action')
  }

  // Validate content length
  if (request.content && request.content.length > 10000) {
    errors.push('Content exceeds maximum length')
  }

  // Validate required fields for each action
  switch (request.action) {
    case 'create':
      if (!request.media_id && !request.media_info) {
        errors.push('Either media_id or media_info is required for create action')
      }
      if (!request.content?.trim()) {
        errors.push('Content is required for create action')
      }
      break
    case 'edit':
      if (!request.comment_id) {
        errors.push('comment_id is required for edit action')
      }
      if (!request.content?.trim()) {
        errors.push('Content is required for edit action')
      }
      break
    case 'vote':
      if (!request.comment_id) {
        errors.push('comment_id is required for vote action')
      }
      break
    case 'report':
      if (!request.comment_id) {
        errors.push('comment_id is required for report action')
      }
      if (!request.reason) {
        errors.push('reason is required for report action')
      }
      break
  }

  return errors
}
```

## Frontend Security

### 1. DOM Security

**✅ Best Practices:**
```typescript
// ✅ Secure DOM manipulation
import DOMPurify from 'dompurify'

class SecureDOMRenderer {
  static renderCommentContent(content: string): string {
    // Sanitize HTML content
    return DOMPureify.sanitize(content, {
      ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'i', 'a'],
      ALLOWED_ATTR: ['href', 'title', 'target'],
      ALLOW_DATA_ATTR: false
    })
  }

  static createSecureElement(tag: string, content: string, attributes: Record<string, string> = {}): HTMLElement {
    const element = document.createElement(tag)
    
    // Set secure attributes
    if (tag === 'a' && attributes.href) {
      // Validate URL
      try {
        new URL(attributes.href)
        element.href = attributes.href
      } catch {
        throw new Error('Invalid URL provided')
      }
    }

    // Set sanitized content
    if (content) {
      if (tag === 'div' || tag === 'span') {
        element.innerHTML = this.renderCommentContent(content)
      } else {
        element.textContent = content
      }
    }

    return element
  }
}
```

### 2. Event Security

**✅ Best Practices:**
```typescript
// ✅ Secure event handling
class SecureEventHandler {
  private static allowedActions = ['vote', 'reply', 'edit', 'delete', 'report']

  static handleCommentAction(event: Event, sessionToken: string): void {
    const target = event.target as HTMLElement
    const action = target.dataset.action
    
    // Validate action
    if (!action || !this.allowedActions.includes(action)) {
      console.warn('Invalid or missing action')
      return
    }

    // Validate session
    if (!sessionToken) {
      console.error('No session token for action')
      return
    }

    // Get comment ID from closest comment element
    const commentElement = target.closest('[data-comment-id]')
    const commentId = commentElement?.dataset.commentId

    if (!commentId) {
      console.error('No comment ID found')
      return
    }

    // Execute action with proper validation
    this.executeSecureAction(action, commentId, sessionToken)
  }

  private static async executeSecureAction(
    action: string, 
    commentId: string, 
    sessionToken: string
  ): Promise<void> {
    const endpoint = `/functions/v1/${this.getActionEndpoint(action)}`
    
    const requestBody = this.buildRequestBody(action, commentId)
    
    try {
      const response = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${sessionToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(requestBody)
      })

      if (!response.ok) {
        const error = await response.json()
        throw new Error(error.error || 'Action failed')
      }

      // Handle successful response
      console.log(`Action ${action} completed successfully`)
    } catch (error) {
      console.error(`Action ${action} failed:`, error)
      throw error
    }
  }

  private getActionEndpoint(action: string): string {
    const endpoints = {
      'vote': 'voting',
      'reply': 'comments',
      'edit': 'comments',
      'delete': 'comments',
      'report': 'reports'
    }
    return endpoints[action] || 'comments'
  }

  private buildRequestBody(action: string, commentId: string): any {
    const baseBody = { action }

    switch (action) {
      case 'vote':
        return {
          ...baseBody,
          comment_id: parseInt(commentId)
        }
      case 'reply':
      case 'edit':
        return {
          ...baseBody,
          comment_id: parseInt(commentId),
          content: 'Content from form' // Should come from form
        }
      case 'delete':
        return {
          ...baseBody,
          comment_id: parseInt(commentId)
        }
      case 'report':
        return {
          ...baseBody,
          comment_id: parseInt(commentId),
          reason: 'spam' // Should come from form
        }
      default:
        return baseBody
    }
  }
}
```

## Database Security

### 1. Row Level Security (RLS)

**✅ Best Practices:**
```sql
-- ✅ Secure RLS policies for comments table
CREATE POLICY "Users can view non-deleted comments" ON comments
  FOR SELECT
  USING (
    -- Users can see comments unless they're deleted or from shadow-banned users
    NOT deleted AND
    user_id NOT IN (
      SELECT id FROM users WHERE shadow_banned = TRUE
    )
  );

-- ✅ Secure RLS policies for comment creation
CREATE POLICY "Authenticated users can create comments" ON comments
  FOR INSERT WITH CHECK (
    -- Users must be authenticated and not banned/muted
    (
      SELECT 
        CASE 
          WHEN banned = TRUE THEN false
          WHEN muted_until > NOW() THEN false
          ELSE true
        END
      FROM users 
      WHERE id = auth.uid()
    )
  );

-- ✅ Secure RLS policies for comment editing
CREATE POLICY "Users can edit their own comments" ON comments
  FOR UPDATE
  USING (
    -- Users can only edit their own non-locked comments
    user_id = auth.uid() AND
    NOT locked AND
    NOT deleted
  );
```

### 2. Session Table Security

**✅ Best Practices:**
```sql
-- ✅ Secure user_sessions table structure
CREATE TABLE user_sessions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_token TEXT NOT NULL UNIQUE,
  client_type TEXT NOT NULL CHECK (client_type IN ('anilist', 'myanimelist', 'simkl')),
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ DEFAULT NOW()
);

-- ✅ Indexes for performance and security
CREATE INDEX idx_user_sessions_token ON user_sessions(session_token);
CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);

-- ✅ Function to validate session tokens
CREATE OR REPLACE FUNCTION validate_session_token(session_token TEXT)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM user_sessions 
    WHERE session_token = validate_session_token.session_token
      AND expires_at > NOW()
  );
END;
```

## Session Management

### 1. Session Lifecycle

**✅ Best Practices:**
```typescript
// ✅ Secure session lifecycle management
class SessionLifecycleManager {
  private readonly sessionDuration = 30 * 24 * 60 * 60 * 1000 // 30 days in milliseconds
  private readonly cleanupInterval = 60 * 60 * 1000 // 1 hour cleanup

  constructor() {
    this.startAutomaticCleanup()
  }

  createSession(userId: number, clientType: string): string {
    // Generate cryptographically secure token
    const token = this.generateSecureToken()
    const expiresAt = new Date(Date.now() + this.sessionDuration)
    
    // Store session in database
    // This would be handled by the identity-resolve function
    return token
  }

  private generateSecureToken(): string {
    const array = new Uint8Array(32)
    crypto.getRandomValues(array)
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
  }

  isSessionValid(token: string): boolean {
    // Check token format
    if (!/^[a-f0-9]{64}$/.test(token)) {
      return false
    }

    // Check expiration (would validate with database in production)
    return true // Placeholder - actual validation would check database
  }

  startAutomaticCleanup(): void {
    setInterval(() => {
      this.cleanupExpiredSessions()
    }, this.cleanupInterval)
  }

  private async cleanupExpiredSessions(): Promise<void> {
    // This would call the database cleanup function
    // const cleanedCount = await db.query('SELECT cleanup_expired_sessions()')
    // console.log(`Cleaned ${cleanedCount} expired sessions`)
  }
}
```

### 2. Session Storage Security

**✅ Best Practices:**
```typescript
// ✅ Secure session storage with validation
class SecureSessionStorage {
  private readonly storageKey = 'commentum_session_token'
  private readonly tokenPattern = /^[a-f0-9]{64}$/

  storeSession(token: string): void {
    // Validate token format
    if (!this.tokenPattern.test(token)) {
      throw new Error('Invalid session token format')
    }

    // Store securely
    localStorage.setItem(this.storageKey, token)
    
    // Set expiration reminder
    this.setExpirationReminder(token)
  }

  retrieveSession(): string | null {
    const token = localStorage.getItem(this.storageKey)
    
    if (!token) {
      return null
    }

    // Validate token format
    if (!this.tokenPattern.test(token)) {
      this.clearSession()
      return null
    }

    return token
  }

  clearSession(): void {
    localStorage.removeItem(this.storageKey)
    localStorage.removeItem('commentum_session_expires')
  }

  private setExpirationReminder(token: string): void {
    const expiresAt = new Date(Date.now() + (30 * 24 * 60 * 60 * 1000))
    localStorage.setItem('commentum_expires', expiresAt.toISOString())
  }

  checkExpiration(): boolean {
    const expiresAt = localStorage.getItem('commentum_expires')
    if (!expiresAt) return true
    
    const expirationTime = new Date(expiresAt)
    return expirationTime > new Date()
  }
}
```

## Rate Limiting

### 1. Session-Aware Rate Limiting

**✅ Best Practices:**
```typescript
// ✅ Session-aware rate limiting
class SessionRateLimiter {
  private readonly requests = new Map<string, number[]>()
  private readonly maxRequests = 30
  private readonly windowMs = 60 * 60 * 1000 // 1 hour

  canProceed(sessionToken: string): boolean {
    if (!sessionToken) {
      throw new Error('No session token provided')
    }

    const now = Date.now()
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs
    
    if (!this.requests.has(sessionToken)) {
      this.requests.set(sessionToken, [])
    }

    const userRequests = this.requests.get(sessionToken)
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

  getRemainingRequests(sessionToken: string): number {
    if (!sessionToken) return 0
    
    const now = Date.now()
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs
    
    if (!this.requests.has(sessionToken)) {
      return this.maxRequests
    }
    
    const userRequests = this.requests.get(sessionToken)
    const validRequests = userRequests.filter(time => now - time < this.windowMs)
    
    return Math.max(0, this.maxRequests - validRequests.length)
  }

  cleanupExpiredSessions(): void {
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
```

### 2. Differentiated Rate Limits

**✅ Best Practices:**
```typescript
// ✅ Differentiated rate limiting by action type
class DifferentiatedRateLimiter {
  private readonly limitConfig = {
    comment: { maxRequests: 30, windowMs: 60 * 60 * 1000 },
    vote: { maxRequests: 100, windowMs: 60 * 60 * 1000 },
    report: { maxRequests: 10, windowMs: 60 * 60 * 1000 },
    edit: { maxRequests: 30, windowMs: 60 * 60 * 1000 }
  }

  private readonly limiters = new Map<string, SessionRateLimiter>()

  getLimiter(action: string): SessionRateLimiter {
    if (!this.limiters.has(action)) {
      const config = this.limitConfig[action] || this.limitConfig.comment
      this.limiters.set(action, new SessionRateLimiter(config.maxRequests, config.windowMs))
    }
    return this.limiters.get(action)!
  }

  canProceed(sessionToken: string, action: string): boolean {
    const limiter = this.getLimiter(action)
    return limiter.canProceed(sessionToken)
  }

  getRemainingRequests(sessionToken: string, action: string): number {
    const limiter = this.getLimiter(action)
    return limiter.getRemainingRequests(sessionToken)
  }

  // Cleanup all expired requests
  cleanupAllExpiredSessions(): void {
    for (const limiter of this.limiters.values()) {
      limiter.cleanupExpiredSessions()
    }
  }
}
```

## Input Validation

### 1. Content Validation

**✅ Best Practices:**
```typescript
// ✅ Comprehensive content validation
class ContentValidator {
  private readonly maxCommentLength = 10000
  private readonly allowedTags = ['p', 'br', 'strong', 'em', 'u', 'i', 'a']
  private readonly allowedAttributes = ['href', 'title', 'target', 'rel']
  private readonly urlPattern = /^https?:\/\/.+/i

  validateCommentContent(content: string): ValidationResult {
    const errors: string[] = []

    // Length validation
    if (content.length === 0) {
      errors.push('Comment cannot be empty')
    } else if (content.length > this.maxCommentLength) {
      errors.push(`Comment cannot exceed ${this.maxCommentLength} characters`)
    }

    // Content validation
    if (this.containsForbiddenWords(content)) {
      errors.push('Comment contains forbidden words')
    }

    // Structure validation
    if (this.hasExcessiveRepetition(content)) {
      errors.push('Comment contains excessive repetition')
    }

    return {
      isValid: errors.length === 0,
      errors,
      sanitizedContent: this.sanitizeContent(content)
    }
  }

  private containsForbiddenWords(content: string): boolean {
    const forbiddenWords = [
      'spam', 'scam', 'phishing', 'malware', 'virus'
    ]
    
    const lowerContent = content.toLowerCase()
    return forbiddenWords.some(word => lowerContent.includes(word))
  }

  hasExcessiveRepetition(content: string): boolean {
    const words = content.toLowerCase().split(/\s+/)
    const wordCount = new Map<string, number>()
    
    for (const word of words) {
      wordCount.set(word, (wordCount.get(word) || 0) + 1)
      if (wordCount.get(word) > 5) {
        return true
      }
    }
    
    return false
  }

  sanitizeContent(content: string): string {
    // Remove potentially harmful content
    return content
      .replace(/<script[^>]*>/gi, '')
      .replace(/javascript:/gi, '')
      .replace(/on\w+\s*=/gi, '')
      .trim()
  }
}

interface ValidationResult {
  isValid: boolean
  errors: string[]
  sanitizedContent?: string
}
```

### 2. Parameter Validation

**✅ Best Practices:**
```typescript
// ✅ Comprehensive parameter validation
class ParameterValidator {
  static validateCommentId(commentId: string): number {
    const id = parseInt(commentId, 10)
    
    if (isNaN(id) || id <= 0) {
      throw new Error('Invalid comment ID: must be a positive integer')
    }
    
    return id
  }

  static validateMediaId(mediaId: string): number {
    const id = parseInt(mediaId, 10)
    
    if (isNaN(id) || id <= 0) {
      throw new Error('Invalid media ID: must be a positive integer')
    }
    
    return id
  }

  static validateVoteAction(action: string): 'upvote' | 'downvote' | 'remove' {
    const validActions = ['upvote', 'downvote', 'remove']
    
    if (!validActions.includes(action)) {
      throw new Error(`Invalid vote action: ${action}. Valid actions: ${validActions.join(', ')}`)
    }
    
    return action as 'upvote' | 'downvote' | 'remove'
  }

  static validateReportReason(reason: string): string {
    const validReasons = [
      'spam', 'offensive', 'harassment', 'spoiler', 'nsfw', 'off_topic', 'other'
    ]
    
    if (!validReasons.includes(reason)) {
      throw new Error(`Invalid report reason: ${reason}. Valid reasons: ${validReasons.join(', ')}`)
    }
    
    return reason
  }

  static validateCommentLength(content: string): void {
    if (content.length > 10000) {
      throw new Error('Comment cannot exceed 10,000 characters')
    }
    
    if (content.trim().length === 0) {
      throw new Error('Comment cannot be empty')
    }
  }
}
```

## Monitoring & Logging

### 1. Security Event Logging

**✅ Best Practices:**
```typescript
// ✅ Security event logging
class SecurityLogger {
  private readonly logEndpoint = '/functions/v1/security-logs'
  private sessionToken: string | null = null

  setSessionToken(token: string): void {
    this.sessionToken = token
  }

  async logSecurityEvent(event: SecurityEvent): Promise<void> {
    if (!this.sessionToken) {
      console.error('No session token for security logging')
      return
    }

    const logEntry = {
      timestamp: new Date().toISOString(),
      event_type: event.type,
      user_id: event.userId,
      ip_address: event.ipAddress,
      user_agent: event.userAgent,
      details: event.details,
      severity: event.severity,
      session_token: this.sessionToken
    }

    try {
      await fetch(this.logEndpoint, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.sessionToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(logEntry)
      })
    } catch (error) {
      console.error('Failed to log security event:', error)
    }
  }

  // Security event types
  static readonly EventTypes = {
    LOGIN_SUCCESS: 'login_success',
    LOGIN_FAILURE: 'login_failure',
    SESSION_EXPIRED: 'session_expired',
    INVALID_TOKEN: 'invalid_token',
    PERMISSION_DENIED: 'permission_denied',
    SUSPICIOUS_ACTIVITY: 'suspicious_activity',
    RATE_LIMIT_EXCEEDED: 'rate_limit_exceeded'
  }

  // Security severity levels
  static readonly SeverityLevels = {
    LOW: 'low',
    MEDIUM: 'medium',
    HIGH: 'high',
    CRITICAL: 'critical'
  }
}

interface SecurityEvent {
  type: string
  userId?: number
  ipAddress?: string
  userAgent?: string
  details: any
  severity: string
}
```

### 2. Error Monitoring

**✅ Best Practices:**
```typescript
// ✅ Comprehensive error monitoring
class ErrorMonitor {
  private readonly errorThreshold = 10 // errors per minute
  private readonly windowMs = 60 * 1000 // 1 minute
  private errors: Array<{ timestamp: number; message: string; count: number }> = []

  recordError(error: Error, context?: string): void {
    const errorMessage = error.message
    const timestamp = Date.now()
    
    // Find existing error for this message
    const existingError = this.errors.find(
      e => e.message === errorMessage && 
      timestamp - e.timestamp < this.windowMs
    )
    )

    if (existingError) {
      existingError.count++
    } else {
      this.errors.push({
        timestamp,
        message: errorMessage,
        count: 1,
        context
      })
    }

    // Check if threshold exceeded
    const recentErrors = this.errors.filter(
      e => timestamp - e.timestamp < this.windowMs
    )

    if (recentErrors.length >= this.errorThreshold) {
      this.triggerErrorAlert(recentErrors)
    }
  }

  private triggerErrorAlert(errors: Array<{ timestamp: number; message: string; count: number }>): void {
    const errorCounts = errors.reduce((acc, error) => {
      acc[error.message] = (acc[error.message] || 0) + error.count
      return acc
    }, {})

    const alertMessage = Object.entries(errorCounts)
      .map(([message, count]) => `${message}: ${count} times`)
      .join(', ')

    console.error('🚨 Error threshold exceeded:', alertMessage)
    
    // Send to monitoring service
    this.sendAlertToMonitoring(alertMessage, errors)
  }

  private sendAlertToMonitoring(message: string, errors: any[]): void {
    // Implementation depends on your monitoring service
    console.error('🚨 Security Alert:', message, errors)
  }

  clearOldErrors(): void {
    const cutoff = Date.now() - this.windowMs
    this.errors = this.errors.filter(e => e.timestamp > cutoff)
  }
}
```

## Common Vulnerabilities

### 1. Identity Spoofing Prevention

**✅ Prevention:**
```typescript
// ✅ Identity spoofing prevention through session validation
class IdentitySpoofingPrevention {
  static validateUserIdentity(
    providedUserId: string | null,
    sessionToken: string | null
  ): boolean {
    // ❌ NEVER trust client-provided user_id
    if (providedUserId) {
      console.warn('Client-provided user_id detected - potential spoofing attempt')
      return false
    }

    // ✅ ALWAYS validate session token
    if (!sessionToken) {
      console.error('No session token provided')
      return false
    }

    // ✅ Validate session with server
    return this.validateSessionWithServer(sessionToken)
  }

  private static async validateSessionWithServer(sessionToken: string): Promise<boolean> {
    try {
      const response = await fetch('/functions/v1/comments', {
        headers: {
          'Authorization': `Bearer ${sessionToken}`,
          'Content-Type': 'application/json'
        }
      })
      
      return response.status !== 401
    } catch {
      return false
    }
  }
}
```

### 2. XSS Prevention

**✅ Prevention:**
```typescript
// ✅ XSS prevention with content sanitization
class XSSPrevention {
  static sanitizeHTML(content: string): string {
    return DOMPurify.sanitize(content, {
      ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'i', 'a'],
      ALLOWED_ATTR: ['href', 'title', 'target', 'rel'],
      ALLOW_DATA_ATTR: false,
      FORBID_TAGS: ['script', 'iframe', 'object', 'embed', 'form', 'input', 'textarea'],
      SANITIZE_DOM: true,
      KEEP_CONTENT: true
    })
  }

  static sanitizeUserInput(input: string): string {
    // Remove potentially dangerous characters
    return input
      .replace(/[<>]/g, '')
      .replace(/javascript:/gi, '')
      .replace(/on\w+\s*=/gi, '')
      .trim()
  }

  static validateURL(url: string): boolean {
    try {
      const urlObj = new URL(url)
      return urlObj.protocol === 'https:' || urlObj.protocol === 'http:'
    } catch {
      return false
    }
  }
}
```

### 3. CSRF Prevention

**✅ Prevention:**
```typescript
// ✅ CSRF prevention with token validation
class CSRFProtection {
  private static readonly tokenName = 'commentum-csrf-token'
  private token: string | null = null

  initialize(): void {
    this.token = this.generateToken()
    this.injectToken()
    this.setupFormValidation()
  }

  private generateToken(): string {
    const array = new Uint8Array(32)
    crypto.getRandomValues(array)
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
  }

  private injectToken(): void {
    if (typeof document !== 'undefined') {
      const meta = document.createElement('meta')
      meta.name = this.tokenName
      meta.content = this.token
      document.head.appendChild(meta)
    }
  }

  private setupFormValidation(): void {
    if (typeof document !== 'undefined') {
      document.addEventListener('submit', (event) => {
        const form = event.target as HTMLFormElement
        const tokenInput = form.querySelector(`input[name="${this.tokenName}"]`)
        
        if (tokenInput) {
          tokenInput.value = this.token || ''
        }
      })
    }
  }

  addToRequest(options: RequestInit = {}): RequestInit {
    const headers = new Headers(options.headers || {})
    
    // Add CSRF token to all requests
    if (this.token) {
      headers.set('X-CSRF-Token', this.token)
    }
    
    return {
      ...options,
      headers
    }
  }

  validateRequest(request: Request): boolean {
    const token = request.headers.get('X-CSRF-Token')
    return token === this.token
  }
}
```

## Security Checklist

### 🔒 Authentication Security

- [ ] Session tokens are cryptographically generated
- [ ] Session tokens are stored securely in localStorage
- [ ] Session tokens have proper format validation
- [ ] Session tokens expire after 30 days
- [ ] Expired sessions are automatically cleared
- [ ] Provider tokens are validated with real APIs
- [ ] No client-provided user data is trusted
- [ ] Session validation happens on every API request

### 🔒 API Security

- [ ] All API requests use session-based authentication
- [ ] Authorization headers are properly formatted
- [ ] Request parameters are validated server-side
- [ ] Error responses don't leak sensitive information
- [ ] CORS is properly configured
- - Rate limiting is session-aware
- Input validation is comprehensive
- XSS prevention is implemented
- CSRF protection is active

### 🔒 Frontend Security

- [ ] DOM content is sanitized
- - User input is validated
- - URLs are validated
- - Event handlers are secure
- - No sensitive data in client-side storage
- - Session tokens are handled securely
- - Error messages don't leak information

### 🔒 Database Security

- [ ] RLS policies are properly configured
- - Session table has proper constraints
- - User data is protected
- - Audit trails are maintained
- - Expired sessions are cleaned up
- - Foreign key constraints ensure integrity

### 🔒 Monitoring & Logging

- [ ] Security events are logged
- - Error thresholds are configured
- - Suspicious activity is detected
- - Rate limit violations are tracked
- - Session lifecycle is monitored
- - Authentication failures are logged

### 🔒 Common Vulnerabilities

- [ ] Identity spoofing is prevented
- [ ] XSS attacks are prevented
- [ ] CSRF attacks are prevented
- [ ] SQL injection is prevented
- [ ] Session hijacking is prevented
- [ ] Rate limiting abuse is prevented

---

**🔒 Security Reminder**: This system implements enterprise-grade security through session-based authentication. Always validate sessions server-side and never trust client-provided user data.