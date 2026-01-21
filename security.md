# Security & Rate Limiting Guide

This guide covers the security features, rate limiting mechanisms, and best practices for implementing a secure Commentum integration.

## Table of Contents

- [Security Overview](#security-overview)
- [Rate Limiting System](#rate-limiting-system)
- [Input Validation & Sanitization](#input-validation--sanitization)
- [Abuse Detection](#abuse-detection)
- [Row Level Security (RLS)](row-level-security-rls)
- [API Security](#api-security)
- [Client-Side Security](#client-side-security)
- [Security Monitoring](#security-monitoring)

## Security Overview

Commentum implements multiple layers of security to protect against common web application vulnerabilities and abuse patterns.

### Security Layers

1. **Network Layer**: HTTPS, CORS, request validation
2. **Application Layer**: Authentication, authorization, input validation
3. **Database Layer**: Row Level Security, parameterized queries
4. **Business Logic Layer**: Rate limiting, abuse detection
5. **Client Layer**: XSS protection, CSRF protection

### Threat Model

| Threat Category | Protection Mechanism |
|-----------------|---------------------|
| **Authentication Bypass** | Multi-platform identity verification, session management |
| **Authorization Escalation** | Role-based access control, permission validation |
| **Injection Attacks** | Parameterized queries, input sanitization |
| **XSS Attacks** | Content sanitization, CSP headers |
| **CSRF Attacks** | Token validation, same-origin checks |
| **Rate Limiting Bypass** | Multi-level rate limiting, IP tracking |
| **Data Exposure** | RLS policies, data filtering |
| **Abuse & Spam** | Automated moderation, pattern detection |

## Rate Limiting System

### Rate Limiting Architecture

```
Client Request → Rate Limit Check → API Processing → Rate Limit Update → Response
```

### Rate Limit Levels

#### 1. Global Rate Limits
Applied to all users regardless of authentication status:

| Action | Limit | Window | Description |
|--------|-------|--------|-------------|
| API Requests | 1000/hour | 1 hour | Total API requests per IP |
| Identity Resolution | 10/hour | 1 hour | Identity lookups per IP |
| Failed Auth | 5/hour | 1 hour | Failed authentication attempts |

#### 2. User Rate Limits
Applied to authenticated users:

| Action | Default Limit | Configurable | Description |
|--------|---------------|--------------|-------------|
| Comments | 30/hour | Yes | New comments and edits |
| Votes | 100/hour | Yes | Upvotes/downvotes |
| Reports | 10/hour | Yes | Comment reports |
| Edits | 30/hour | Yes | Comment edits |

#### 3. Action-Specific Limits
Special limits for sensitive actions:

| Action | Limit | Window | Additional Rules |
|--------|-------|--------|------------------|
| User Moderation | 50/hour | 1 hour | Moderators only |
| Role Changes | 10/hour | 1 hour | Admins only |
| Config Changes | 5/hour | 1 hour | Super admins only |

### Rate Limit Implementation

#### Database Schema

```sql
CREATE TABLE rate_limits (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  ip_address INET,
  action_type TEXT NOT NULL,
  window_start TIMESTAMPTZ NOT NULL,
  count INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, action_type, window_start),
  UNIQUE(ip_address, action_type, window_start)
);

-- Indexes for performance
CREATE INDEX idx_rate_limits_user_action ON rate_limits(user_id, action_type);
CREATE INDEX idx_rate_limits_ip_action ON rate_limits(ip_address, action_type);
CREATE INDEX idx_rate_limits_window ON rate_limits(window_start);
```

#### Rate Limit Checking Function

```sql
CREATE OR REPLACE FUNCTION check_rate_limit(
    p_user_id INTEGER DEFAULT NULL,
    p_ip_address INET DEFAULT NULL,
    p_action_type TEXT,
    p_window_hours INTEGER DEFAULT 1,
    p_max_requests INTEGER DEFAULT 100
)
RETURNS TABLE(
    allowed BOOLEAN,
    current_count INTEGER,
    remaining_requests INTEGER,
    reset_time TIMESTAMPTZ
)
AS $$
DECLARE
    v_window_start TIMESTAMPTZ;
    v_current_count INTEGER;
    v_remaining INTEGER;
    v_reset_time TIMESTAMPTZ;
BEGIN
    -- Calculate window start time
    v_window_start := date_trunc('hour', NOW()) - 
                     INTERVAL '1 hour' * (EXTRACT(HOUR FROM NOW())::INTEGER % p_window_hours);
    
    -- Get current count for user or IP
    IF p_user_id IS NOT NULL THEN
        SELECT COALESCE(COUNT, 0) INTO v_current_count
        FROM rate_limits
        WHERE user_id = p_user_id 
          AND action_type = p_action_type 
          AND window_start = v_window_start;
    ELSIF p_ip_address IS NOT NULL THEN
        SELECT COALESCE(COUNT, 0) INTO v_current_count
        FROM rate_limits
        WHERE ip_address = p_ip_address 
          AND action_type = p_action_type 
          AND window_start = v_window_start;
    ELSE
        v_current_count := 0;
    END IF;
    
    -- Calculate remaining requests
    v_remaining := GREATEST(0, p_max_requests - v_current_count);
    
    -- Calculate reset time
    v_reset_time := v_window_start + INTERVAL '1 hour' * p_window_hours;
    
    -- Return result
    RETURN QUERY SELECT 
        v_current_count < p_max_requests as allowed,
        v_current_count as current_count,
        v_remaining as remaining_requests,
        v_reset_time as reset_time;
END;
$$ LANGUAGE plpgsql;
```

#### Client-Side Rate Limiting

```typescript
// utils/rateLimiter.ts
interface RateLimitState {
  count: number
  windowStart: number
  windowSize: number
  maxRequests: number
}

export class ClientRateLimiter {
  private limits: Map<string, RateLimitState> = new Map()
  
  constructor(
    private defaultLimits: Record<string, { maxRequests: number; windowSize: number }>
  ) {}
  
  canProceed(action: string, identifier: string = 'default'): boolean {
    const key = `${action}:${identifier}`
    const limit = this.limits.get(key) || this.createLimitState(action)
    
    const now = Date.now()
    const windowElapsed = now - limit.windowStart
    
    // Reset window if expired
    if (windowElapsed >= limit.windowSize) {
      limit.count = 0
      limit.windowStart = now
    }
    
    return limit.count < limit.maxRequests
  }
  
  recordRequest(action: string, identifier: string = 'default'): void {
    const key = `${action}:${identifier}`
    const limit = this.limits.get(key) || this.createLimitState(action)
    
    const now = Date.now()
    const windowElapsed = now - limit.windowStart
    
    // Reset window if expired
    if (windowElapsed >= limit.windowSize) {
      limit.count = 0
      limit.windowStart = now
    }
    
    limit.count++
    this.limits.set(key, limit)
  }
  
  getStatus(action: string, identifier: string = 'default') {
    const key = `${action}:${identifier}`
    const limit = this.limits.get(key)
    
    if (!limit) {
      return {
        canProceed: true,
        remaining: this.defaultLimits[action]?.maxRequests || 100,
        resetTime: Date.now() + (this.defaultLimits[action]?.windowSize || 3600000)
      }
    }
    
    const now = Date.now()
    const windowElapsed = now - limit.windowStart
    const resetTime = limit.windowStart + limit.windowSize
    
    if (windowElapsed >= limit.windowSize) {
      return {
        canProceed: true,
        remaining: limit.maxRequests,
        resetTime: now + limit.windowSize
      }
    }
    
    return {
      canProceed: limit.count < limit.maxRequests,
      remaining: Math.max(0, limit.maxRequests - limit.count),
      resetTime
    }
  }
  
  private createLimitState(action: string): RateLimitState {
    const config = this.defaultLimits[action] || { maxRequests: 100, windowSize: 3600000 }
    const state: RateLimitState = {
      count: 0,
      windowStart: Date.now(),
      windowSize: config.windowSize,
      maxRequests: config.maxRequests
    }
    
    this.limits.set(`${action}:default`, state)
    return state
  }
}

// Usage
const rateLimiter = new ClientRateLimiter({
  'comment': { maxRequests: 30, windowSize: 3600000 }, // 30/hour
  'vote': { maxRequests: 100, windowSize: 3600000 },   // 100/hour
  'report': { maxRequests: 10, windowSize: 3600000 }   // 10/hour
})

// Before making an API call
if (rateLimiter.canProceed('comment', userId)) {
  rateLimiter.recordRequest('comment', userId)
  // Make API call
} else {
  // Show rate limit message
  const status = rateLimiter.getStatus('comment', userId)
  showRateLimitError(status.resetTime)
}
```

## Input Validation & Sanitization

### Server-Side Validation

```typescript
// utils/validation.ts
export class InputValidator {
  private static readonly COMMENT_MAX_LENGTH = 10000
  private static readonly USERNAME_MAX_LENGTH = 50
  private static readonly REASON_MAX_LENGTH = 500
  
  static validateComment(content: string): ValidationResult {
    const errors: string[] = []
    
    if (!content || content.trim().length === 0) {
      errors.push('Comment content cannot be empty')
    }
    
    if (content.length > this.COMMENT_MAX_LENGTH) {
      errors.push(`Comment cannot exceed ${this.COMMENT_MAX_LENGTH} characters`)
    }
    
    // Check for suspicious patterns
    const suspiciousPatterns = [
      /<script[^>]*>.*?<\/script>/gi,
      /javascript:/gi,
      /on\w+\s*=/gi,
      /data:text\/html/gi
    ]
    
    for (const pattern of suspiciousPatterns) {
      if (pattern.test(content)) {
        errors.push('Comment contains potentially dangerous content')
        break
      }
    }
    
    return {
      isValid: errors.length === 0,
      errors,
      sanitized: this.sanitizeComment(content)
    }
  }
  
  static sanitizeComment(content: string): string {
    return content
      .trim()
      // Remove potentially dangerous HTML
      .replace(/<script[^>]*>.*?<\/script>/gi, '')
      .replace(/javascript:/gi, '')
      .replace(/on\w+\s*=/gi, '')
      // Limit consecutive newlines
      .replace(/\n{3,}/g, '\n\n')
      // Remove excessive whitespace
      .replace(/\s{2,}/g, ' ')
  }
  
  static validateUsername(username: string): ValidationResult {
    const errors: string[] = []
    
    if (!username || username.trim().length === 0) {
      errors.push('Username cannot be empty')
    }
    
    if (username.length > this.USERNAME_MAX_LENGTH) {
      errors.push(`Username cannot exceed ${this.USERNAME_MAX_LENGTH} characters`)
    }
    
    if (!/^[a-zA-Z0-9_-]+$/.test(username)) {
      errors.push('Username can only contain letters, numbers, underscores, and hyphens')
    }
    
    return {
      isValid: errors.length === 0,
      errors,
      sanitized: username.trim()
    }
  }
  
  static validateReportReason(reason: string, notes?: string): ValidationResult {
    const errors: string[] = []
    const validReasons = ['spam', 'offensive', 'harassment', 'spoiler', 'nsfw', 'off_topic', 'other']
    
    if (!validReasons.includes(reason)) {
      errors.push('Invalid report reason')
    }
    
    if (notes && notes.length > this.REASON_MAX_LENGTH) {
      errors.push(`Notes cannot exceed ${this.REASON_MAX_LENGTH} characters`)
    }
    
    return {
      isValid: errors.length === 0,
      errors,
      sanitized: {
        reason,
        notes: notes?.trim()
      }
    }
  }
}

interface ValidationResult {
  isValid: boolean
  errors: string[]
  sanitized?: any
}
```

### Content Sanitization

```typescript
// utils/sanitizer.ts
import DOMPurify from 'dompurify'

export class ContentSanitizer {
  private static readonly ALLOWED_TAGS = [
    'p', 'br', 'strong', 'em', 'u', 'i', 'b',
    'a', 'code', 'pre', 'blockquote',
    'ul', 'ol', 'li'
  ]
  
  private static readonly ALLOWED_ATTRIBUTES = {
    'a': ['href', 'title'],
    '*': ['class']
  }
  
  static sanitizeHTML(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: this.ALLOWED_TAGS,
      ALLOWED_ATTR: this.ALLOWED_ATTRIBUTES,
      ALLOW_DATA_ATTR: false,
      FORBID_TAGS: ['script', 'object', 'embed', 'iframe'],
      FORBID_ATTR: ['onclick', 'onload', 'onerror', 'onmouseover']
    })
  }
  
  static sanitizePlainText(text: string): string {
    return text
      .replace(/[<>]/g, '') // Remove HTML brackets
      .replace(/javascript:/gi, '') // Remove JavaScript protocols
      .replace(/on\w+\s*=/gi, '') // Remove event handlers
      .trim()
  }
  
  static extractLinks(text: string): { text: string; links: string[] } {
    const urlRegex = /https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)/g
    const links = text.match(urlRegex) || []
    
    const sanitizedText = text.replace(urlRegex, '[link]')
    
    return {
      text: sanitizedText,
      links
    }
  }
}
```

## Abuse Detection

### Automated Moderation

```typescript
// services/abuseDetection.ts
export class AbuseDetectionService {
  private static readonly SUSPICIOUS_PATTERNS = [
    /\b(spam|advert|buy now|click here)\b/gi,
    /\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b/g, // IP addresses
    /\b[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}\b/gi, // Email addresses
    /(?:https?:\/\/)?(?:www\.)?[a-z0-9-]+\.(?:com|org|net|gov|edu)\b/gi // Domains
  ]
  
  static async analyzeComment(
    content: string,
    userId: number,
    userHistory: UserHistory
  ): Promise<AbuseAnalysisResult> {
    const result: AbuseAnalysisResult = {
      score: 0,
      flags: [],
      shouldAutoModerate: false,
      recommendedAction: 'none'
    }
    
    // 1. Content analysis
    const contentFlags = this.analyzeContent(content)
    result.flags.push(...contentFlags)
    result.score += contentFlags.length * 10
    
    // 2. User behavior analysis
    const behaviorFlags = this.analyzeUserBehavior(userHistory)
    result.flags.push(...behaviorFlags)
    result.score += behaviorFlags.length * 15
    
    // 3. Pattern analysis
    const patternFlags = this.analyzePatterns(content, userHistory)
    result.flags.push(...patternFlags)
    result.score += patternFlags.length * 20
    
    // 4. Determine action
    if (result.score >= 80) {
      result.shouldAutoModerate = true
      result.recommendedAction = 'delete'
    } else if (result.score >= 50) {
      result.shouldAutoModerate = true
      result.recommendedAction = 'flag'
    } else if (result.score >= 30) {
      result.recommendedAction = 'review'
    }
    
    return result
  }
  
  private static analyzeContent(content: string): string[] {
    const flags: string[] = []
    
    // Check for suspicious patterns
    for (const pattern of this.SUSPICIOUS_PATTERNS) {
      if (pattern.test(content)) {
        flags.push('suspicious_pattern')
        break
      }
    }
    
    // Check for excessive repetition
    const words = content.toLowerCase().split(/\s+/)
    const wordFreq = new Map<string, number>()
    
    for (const word of words) {
      wordFreq.set(word, (wordFreq.get(word) || 0) + 1)
    }
    
    const maxRepetition = Math.max(...wordFreq.values())
    if (maxRepetition > words.length * 0.3) {
      flags.push('excessive_repetition')
    }
    
    // Check for excessive caps
    const capsRatio = (content.match(/[A-Z]/g) || []).length / content.length
    if (capsRatio > 0.5) {
      flags.push('excessive_caps')
    }
    
    // Check for special characters
    const specialCharRatio = (content.match(/[^a-zA-Z0-9\s]/g) || []).length / content.length
    if (specialCharRatio > 0.3) {
      flags.push('excessive_special_chars')
    }
    
    return flags
  }
  
  private static analyzeUserBehavior(history: UserHistory): string[] {
    const flags: string[] = []
    
    // Check comment frequency
    const recentComments = history.comments.filter(
      c => Date.now() - new Date(c.created_at).getTime() < 3600000 // 1 hour
    )
    
    if (recentComments.length > 20) {
      flags.push('high_frequency_posting')
    }
    
    // Check report ratio
    const reportRatio = history.reportsReceived / Math.max(history.commentsPosted, 1)
    if (reportRatio > 0.1) {
      flags.push('high_report_ratio')
    }
    
    // Check account age
    const accountAge = Date.now() - new Date(history.accountCreated).getTime()
    if (accountAge < 86400000 && history.commentsPosted > 10) { // 1 day
      flags.push('new_account_spam')
    }
    
    return flags
  }
  
  private static analyzePatterns(content: string, history: UserHistory): string[] {
    const flags: string[] = []
    
    // Check for similar content to previous comments
    const similarComments = history.comments.filter(comment => 
      this.calculateSimilarity(content, comment.content) > 0.8
    )
    
    if (similarComments.length > 3) {
      flags.push('repetitive_content')
    }
    
    // Check for rapid posting to same media
    const sameMediaComments = history.comments.filter(comment =>
      comment.media_id === history.lastMediaId &&
      Date.now() - new Date(comment.created_at).getTime() < 300000 // 5 minutes
    )
    
    if (sameMediaComments.length > 5) {
      flags.push('media_spam')
    }
    
    return flags
  }
  
  private static calculateSimilarity(str1: string, str2: string): number {
    const longer = str1.length > str2.length ? str1 : str2
    const shorter = str1.length > str2.length ? str2 : str1
    
    if (longer.length === 0) return 1.0
    
    const editDistance = this.levenshteinDistance(longer, shorter)
    return (longer.length - editDistance) / longer.length
  }
  
  private static levenshteinDistance(str1: string, str2: string): number {
    const matrix = Array(str2.length + 1).fill(null).map(() => Array(str1.length + 1).fill(null))
    
    for (let i = 0; i <= str1.length; i++) matrix[0][i] = i
    for (let j = 0; j <= str2.length; j++) matrix[j][0] = j
    
    for (let j = 1; j <= str2.length; j++) {
      for (let i = 1; i <= str1.length; i++) {
        const indicator = str1[i - 1] === str2[j - 1] ? 0 : 1
        matrix[j][i] = Math.min(
          matrix[j][i - 1] + 1, // deletion
          matrix[j - 1][i] + 1, // insertion
          matrix[j - 1][i - 1] + indicator // substitution
        )
      }
    }
    
    return matrix[str2.length][str1.length]
  }
}

interface AbuseAnalysisResult {
  score: number
  flags: string[]
  shouldAutoModerate: boolean
  recommendedAction: 'none' | 'review' | 'flag' | 'hide' | 'delete'
}

interface UserHistory {
  commentsPosted: number
  reportsReceived: number
  accountCreated: string
  lastMediaId?: number
  comments: Array<{
    content: string
    created_at: string
    media_id: number
  }>
}
```

## Row Level Security (RLS)

### RLS Policy Examples

```sql
-- Enable RLS on tables
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE comment_votes ENABLE ROW LEVEL SECURITY;
ALTER TABLE comment_reports ENABLE ROW LEVEL SECURITY;

-- Users can only see non-deleted comments from non-shadow-banned users
CREATE POLICY "Users can view appropriate comments" ON comments
    FOR SELECT USING (
        NOT deleted AND
        user_id NOT IN (
            SELECT id FROM users WHERE shadow_banned = TRUE
        )
    );

-- Users can only edit their own non-locked comments
CREATE POLICY "Users can edit own comments" ON comments
    FOR UPDATE USING (
        user_id = current_setting('app.current_user_id', true)::INTEGER AND
        NOT locked AND
        NOT deleted
    );

-- Users can only vote on comments they don't own
CREATE POLICY "Users cannot vote on own comments" ON comment_votes
    FOR INSERT WITH CHECK (
        user_id != (
            SELECT user_id FROM comments 
            WHERE id = comment_id
        )
    );

-- Users can only see reports they filed or reports they can moderate
CREATE POLICY "Users can view appropriate reports" ON comment_reports
    FOR SELECT USING (
        reporter_id = current_setting('app.current_user_id', true)::INTEGER
        OR
        current_setting('app.current_user_id', true)::INTEGER IN (
            SELECT id FROM users WHERE role IN ('moderator', 'admin', 'super_admin')
        )
    );
```

### Context Setting Function

```sql
CREATE OR REPLACE FUNCTION set_user_context(user_id TEXT, user_role TEXT)
RETURNS void AS $$
BEGIN
    PERFORM set_config('app.current_user_id', user_id, true);
    PERFORM set_config('app.current_role', user_role, true);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## API Security

### Request Validation Middleware

```typescript
// middleware/apiSecurity.ts
export class APISecurityMiddleware {
  private static readonly RATE_LIMITS = {
    'default': { requests: 1000, window: 3600000 }, // 1000/hour
    'auth': { requests: 10, window: 3600000 },      // 10/hour
    'comment': { requests: 30, window: 3600000 },   // 30/hour
    'vote': { requests: 100, window: 3600000 }      // 100/hour
  }
  
  static validateRequest(req: Request): SecurityValidationResult {
    const result: SecurityValidationResult = {
      allowed: true,
      reason: '',
      headers: {}
    }
    
    // 1. Check Content-Type
    const contentType = req.headers.get('content-type')
    if (req.method === 'POST' && !contentType?.includes('application/json')) {
      result.allowed = false
      result.reason = 'Invalid Content-Type'
      return result
    }
    
    // 2. Check Content-Length
    const contentLength = parseInt(req.headers.get('content-length') || '0')
    if (contentLength > 1048576) { // 1MB
      result.allowed = false
      result.reason = 'Request too large'
      return result
    }
    
    // 3. Add security headers
    result.headers = {
      'X-Content-Type-Options': 'nosniff',
      'X-Frame-Options': 'DENY',
      'X-XSS-Protection': '1; mode=block',
      'Referrer-Policy': 'strict-origin-when-cross-origin'
    }
    
    return result
  }
  
  static async checkRateLimit(
    userId: number | null,
    ipAddress: string,
    action: string
  ): Promise<RateLimitResult> {
    const config = this.RATE_LIMITS[action] || this.RATE_LIMITS['default']
    
    // Check rate limit (implementation depends on your rate limiting store)
    const rateLimitStatus = await this.checkRateLimitStore(
      userId,
      ipAddress,
      action,
      config.requests,
      config.window
    )
    
    return rateLimitStatus
  }
  
  private static async checkRateLimitStore(
    userId: number | null,
    ipAddress: string,
    action: string,
    maxRequests: number,
    windowMs: number
  ): Promise<RateLimitResult> {
    // This would integrate with your rate limiting storage
    // Could be Redis, database, or in-memory store
    
    return {
      allowed: true,
      remaining: maxRequests - 1,
      resetTime: Date.now() + windowMs
    }
  }
}

interface SecurityValidationResult {
  allowed: boolean
  reason: string
  headers: Record<string, string>
}

interface RateLimitResult {
  allowed: boolean
  remaining: number
  resetTime: number
}
```

### API Response Security

```typescript
// utils/responseSecurity.ts
export class ResponseSecurity {
  static sanitizeResponse(data: any, userRole: string): any {
    if (Array.isArray(data)) {
      return data.map(item => this.sanitizeItem(item, userRole))
    }
    
    return this.sanitizeItem(data, userRole)
  }
  
  private static sanitizeItem(item: any, userRole: string): any {
    if (!item || typeof item !== 'object') {
      return item
    }
    
    const sanitized = { ...item }
    
    // Remove sensitive fields based on user role
    const sensitiveFields = ['email', 'ip_address', 'internal_notes']
    
    if (userRole !== 'admin' && userRole !== 'super_admin') {
      for (const field of sensitiveFields) {
        delete sanitized[field]
      }
    }
    
    // Remove internal fields for all users
    const internalFields = ['internal_id', 'cache_version']
    for (const field of internalFields) {
      delete sanitized[field]
    }
    
    return sanitized
  }
  
  static addSecurityHeaders(response: Response): Response {
    const headers = new Headers(response.headers)
    
    headers.set('X-Content-Type-Options', 'nosniff')
    headers.set('X-Frame-Options', 'DENY')
    headers.set('X-XSS-Protection', '1; mode=block')
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
    headers.set('Cache-Control', 'no-cache, no-store, must-revalidate')
    headers.set('Pragma', 'no-cache')
    headers.set('Expires', '0')
    
    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers
    })
  }
}
```

## Client-Side Security

### XSS Protection

```typescript
// utils/xssProtection.ts
export class XSSProtection {
  static escapeHtml(text: string): string {
    const div = document.createElement('div')
    div.textContent = text
    return div.innerHTML
  }
  
  static sanitizeUrl(url: string): string {
    try {
      const parsed = new URL(url)
      
      // Only allow http and https protocols
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        return '#'
      }
      
      // Prevent javascript: protocol
      if (parsed.protocol === 'javascript:') {
        return '#'
      }
      
      return url
    } catch (error) {
      return '#'
    }
  }
  
  static validateInput(input: string, type: 'text' | 'url' | 'email'): boolean {
    switch (type) {
      case 'text':
        return !/<script|javascript:|on\w+=/i.test(input)
      case 'url':
        try {
          const url = new URL(input)
          return ['http:', 'https:'].includes(url.protocol)
        } catch {
          return false
        }
      case 'email':
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input)
      default:
        return false
    }
  }
}
```

### CSP Implementation

```typescript
// utils/csp.ts
export class ContentSecurityPolicy {
  static generateCSP(): string {
    const directives = [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' 'unsafe-eval'", // Adjust based on needs
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self' data:",
      "connect-src 'self' https://api.anilist.co https://api.myanimelist.net",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'"
    ]
    
    return directives.join('; ')
  }
  
  static injectCSP(): void {
    const csp = this.generateCSP()
    const meta = document.createElement('meta')
    meta.httpEquiv = 'Content-Security-Policy'
    meta.content = csp
    document.head.appendChild(meta)
  }
}
```

## Security Monitoring

### Security Event Logging

```typescript
// services/securityMonitoring.ts
export class SecurityMonitoring {
  private static readonly SECURITY_EVENTS = [
    'auth_failure',
    'rate_limit_exceeded',
    'permission_denied',
    'suspicious_content',
    'abuse_detected',
    'sql_injection_attempt',
    'xss_attempt'
  ]
  
  static logSecurityEvent(
    eventType: string,
    details: Record<string, any>,
    userId?: number,
    ipAddress?: string
  ): void {
    const event = {
      timestamp: new Date().toISOString(),
      eventType,
      userId,
      ipAddress,
      details,
      severity: this.getEventSeverity(eventType)
    }
    
    // Send to logging service
    this.sendToSecurityLog(event)
    
    // Check for automatic responses
    this.handleSecurityEvent(event)
  }
  
  private static getEventSeverity(eventType: string): 'low' | 'medium' | 'high' | 'critical' {
    const severityMap: Record<string, string> = {
      'auth_failure': 'medium',
      'rate_limit_exceeded': 'medium',
      'permission_denied': 'low',
      'suspicious_content': 'high',
      'abuse_detected': 'high',
      'sql_injection_attempt': 'critical',
      'xss_attempt': 'critical'
    }
    
    return (severityMap[eventType] || 'low') as any
  }
  
  private static async sendToSecurityLog(event: SecurityEvent): Promise<void> {
    try {
      await fetch('/api/security/log', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(event)
      })
    } catch (error) {
      console.error('Failed to log security event:', error)
    }
  }
  
  private static handleSecurityEvent(event: SecurityEvent): void {
    switch (event.eventType) {
      case 'rate_limit_exceeded':
        if (event.severity === 'high') {
          // Could implement temporary IP blocking
          this.temporaryIPBlock(event.ipAddress!)
        }
        break
        
      case 'sql_injection_attempt':
      case 'xss_attempt':
        // Immediate temporary ban
        this.temporaryUserBan(event.userId!, 24) // 24 hours
        break
        
      case 'abuse_detected':
        if (event.details.score > 80) {
          this.flagForReview(event.userId!)
        }
        break
    }
  }
  
  private static temporaryIPBlock(ipAddress: string, durationHours: number = 1): void {
    // Implement IP blocking logic
    console.log(`Temporarily blocking IP ${ipAddress} for ${durationHours} hours`)
  }
  
  private static temporaryUserBan(userId: number, durationHours: number): void {
    // Implement user banning logic
    console.log(`Temporarily banning user ${userId} for ${durationHours} hours`)
  }
  
  private static flagForReview(userId: number): void {
    // Flag user for moderator review
    console.log(`Flagging user ${userId} for moderator review`)
  }
}

interface SecurityEvent {
  timestamp: string
  eventType: string
  userId?: number
  ipAddress?: string
  details: Record<string, any>
  severity: 'low' | 'medium' | 'high' | 'critical'
}
```

This comprehensive security and rate limiting guide provides multiple layers of protection for the Commentum system. Implement these measures to ensure a secure and reliable comment system.