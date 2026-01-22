# 🔒 Vanilla JavaScript Integration Example - Session-Based Authentication

This example demonstrates how to integrate Commentum into a vanilla JavaScript application without any frameworks using **secure session-based authentication**.

## 🚨 Security Notice

This example uses **session-based authentication** to prevent identity spoofing attacks. The `user_id` is never passed from the client - it's extracted from the session token on the server.

## Project Setup

### 1. HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Commentum Integration - Vanilla JS (Secure)</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div id="app">
        <header>
            <h1>Media Title</h1>
            <p>Type: anime</p>
            <div id="auth-status">
                <!-- Auth status will be shown here -->
            </div>
        </header>

        <main>
            <section class="media-content">
                <!-- Your media content here -->
            </section>

            <section class="comments-section">
                <div id="comment-system">
                    <!-- Comment system will be rendered here -->
                </div>
            </section>
        </main>
    </div>

    <!-- Toast container -->
    <div id="toast-container" class="toast-container"></div>

    <!-- Templates -->
    <template id="login-form-template">
        <form class="login-form">
            <h3>Login to Comment</h3>
            <div class="form-group">
                <label>Service:</label>
                <select name="clientType" required>
                    <option value="anilist">AniList</option>
                    <option value="mal">MyAnimeList</option>
                    <option value="simkl">SIMKL</option>
                </select>
            </div>
            <div class="form-group">
                <label>Access Token:</label>
                <input 
                    type="password" 
                    name="token" 
                    placeholder="Enter your access token"
                    required
                />
                <small>Get your token from your service's developer settings</small>
            </div>
            <button type="submit" class="login-button">Login</button>
        </form>
    </template>

    <template id="comment-form-template">
        <form class="comment-form">
            <div class="form-group">
                <textarea 
                    class="comment-textarea" 
                    placeholder="Share your thoughts..."
                    rows="4"
                    maxlength="10000"
                ></textarea>
                <div class="character-count">0/10000</div>
            </div>
            <div class="form-actions">
                <button type="button" class="cancel-button" style="display: none;">Cancel</button>
                <button type="submit" class="submit-button" disabled>Post Comment</button>
            </div>
        </form>
    </template>

    <template id="comment-item-template">
        <div class="comment-item">
            <div class="comment-content">
                <div class="comment-header">
                    <div class="comment-author">
                        <img class="author-avatar" src="/default-avatar.png" alt="">
                        <div class="author-info">
                            <span class="author-name"></span>
                            <span class="comment-time"></span>
                            <span class="edited-indicator" style="display: none;">(edited)</span>
                        </div>
                    </div>
                    <div class="comment-meta">
                        <span class="pinned-indicator" style="display: none;">📌 Pinned</span>
                        <span class="locked-indicator" style="display: none;">🔒 Locked</span>
                    </div>
                </div>
                <div class="comment-body">
                    <p class="comment-text"></p>
                </div>
                <div class="comment-actions">
                    <div class="voting-buttons">
                        <button class="vote-button upvote" disabled>
                            ▲ <span class="upvote-count">0</span>
                        </button>
                        <button class="vote-button downvote" disabled>
                            ▼ <span class="downvote-count">0</span>
                        </button>
                    </div>
                    <div class="action-buttons">
                        <button class="reply-button" disabled>Reply</button>
                        <button class="edit-button" style="display: none;">Edit</button>
                        <button class="delete-button" style="display: none;">Delete</button>
                        <button class="report-button" disabled>Report</button>
                    </div>
                </div>
            </div>
            <div class="comment-replies" style="display: none;">
                <button class="toggle-replies-button">Show replies</button>
                <div class="replies-list"></div>
            </div>
        </div>
    </template>

    <script src="commentum-secure.js"></script>
</body>
</html>
```

### 2. CSS Styles

```css
/* styles.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    line-height: 1.6;
    color: #333;
    background-color: #f8fafc;
}

#app {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

header {
    margin-bottom: 40px;
    padding-bottom: 20px;
    border-bottom: 1px solid #e5e7eb;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

header h1 {
    font-size: 2.5rem;
    color: #1f2937;
    margin-bottom: 8px;
}

header p {
    color: #6b7280;
    font-size: 1.1rem;
}

#auth-status {
    display: flex;
    align-items: center;
    gap: 12px;
}

.user-info {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    background: #f3f4f6;
    border-radius: 6px;
}

.user-avatar {
    width: 24px;
    height: 24px;
    border-radius: 50%;
    object-fit: cover;
}

.user-name {
    font-weight: 500;
    color: #374151;
}

.logout-button {
    background: #ef4444;
    color: white;
    border: none;
    padding: 6px 12px;
    border-radius: 4px;
    font-size: 12px;
    cursor: pointer;
}

.logout-button:hover {
    background: #dc2626;
}

/* Comment System Styles */
#comment-system {
    max-width: 800px;
    margin: 0 auto;
}

.comment-system-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    padding-bottom: 10px;
    border-bottom: 1px solid #e2e8f0;
}

.comment-system-header h3 {
    font-size: 1.5rem;
    color: #1f2937;
}

/* Login Form Styles */
.login-form {
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    padding: 24px;
    margin-bottom: 24px;
    max-width: 400px;
    margin-left: auto;
    margin-right: auto;
}

.login-form h3 {
    margin-bottom: 16px;
    text-align: center;
    color: #1f2937;
}

.login-form .form-group {
    margin-bottom: 16px;
}

.login-form label {
    display: block;
    margin-bottom: 4px;
    font-weight: 500;
    color: #374151;
}

.login-form input,
.login-form select {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    font-size: 14px;
}

.login-form small {
    display: block;
    margin-top: 4px;
    color: #6b7280;
    font-size: 12px;
}

.login-button {
    width: 100%;
    padding: 10px;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
}

.login-button:hover:not(:disabled) {
    background: #2563eb;
}

.login-button:disabled {
    background: #9ca3af;
    cursor: not-allowed;
}

.comment-form {
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 24px;
}

.form-group {
    margin-bottom: 12px;
    position: relative;
}

.comment-textarea {
    width: 100%;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    padding: 12px;
    font-size: 14px;
    line-height: 1.5;
    resize: vertical;
    transition: border-color 0.2s;
    font-family: inherit;
}

.comment-textarea:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.character-count {
    position: absolute;
    bottom: 8px;
    right: 12px;
    font-size: 12px;
    color: #6b7280;
    background: rgba(255, 255, 255, 0.9);
    padding: 2px 4px;
    border-radius: 4px;
}

.character-count.warning {
    color: #f59e0b;
}

.form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
}

.submit-button, .cancel-button {
    padding: 8px 16px;
    border-radius: 6px;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
    border: none;
}

.submit-button {
    background: #3b82f6;
    color: white;
}

.submit-button:hover:not(:disabled) {
    background: #2563eb;
}

.submit-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.cancel-button {
    background: transparent;
    color: #6b7280;
    border: 1px solid #d1d5db;
}

.cancel-button:hover:not(:disabled) {
    background: #f3f4f6;
}

/* Comment Item Styles */
.comment-item {
    margin-bottom: 16px;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    overflow: hidden;
    background: #ffffff;
}

.comment-item.level-1 {
    margin-left: 24px;
    border-color: #f3f4f6;
}

.comment-item.level-2 {
    margin-left: 48px;
    border-color: #f9fafb;
}

.comment-content {
    padding: 16px;
}

.comment-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    margin-bottom: 8px;
}

.comment-author {
    display: flex;
    align-items: center;
    gap: 8px;
}

.author-avatar {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    object-fit: cover;
}

.author-name {
    font-weight: 600;
    color: #1f2937;
}

.comment-time {
    font-size: 12px;
    color: #6b7280;
    margin-left: 8px;
}

.edited-indicator {
    font-size: 12px;
    color: #9ca3af;
    font-style: italic;
}

.pinned-indicator {
    font-size: 12px;
    color: #10b981;
}

.locked-indicator {
    font-size: 12px;
    color: #f59e0b;
}

.comment-text {
    line-height: 1.6;
    color: #374151;
    white-space: pre-wrap;
    margin-bottom: 12px;
}

.deleted-message {
    font-style: italic;
    color: #6b7280;
}

.comment-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-top: 12px;
    border-top: 1px solid #f3f4f6;
}

.voting-buttons {
    display: flex;
    gap: 8px;
}

.vote-button {
    background: transparent;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    padding: 4px 8px;
    font-size: 12px;
    cursor: pointer;
    transition: all 0.2s;
    display: flex;
    align-items: center;
    gap: 4px;
}

.vote-button:hover:not(:disabled) {
    background: #f3f4f6;
}

.vote-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.vote-button.voted {
    background: #3b82f6;
    color: white;
    border-color: #3b82f6;
}

.vote-button.upvote.voted {
    background: #10b981;
    border-color: #10b981;
}

.vote-button.downvote.voted {
    background: #ef4444;
    border-color: #ef4444;
}

.action-buttons {
    display: flex;
    gap: 8px;
}

.reply-button, .edit-button, .delete-button, .report-button {
    background: transparent;
    border: none;
    color: #6b7280;
    font-size: 12px;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 4px;
    transition: all 0.2s;
}

.reply-button:hover:not(:disabled),
.edit-button:hover:not(:disabled),
.delete-button:hover:not(:disabled),
.report-button:hover:not(:disabled) {
    background: #f3f4f6;
    color: #374151;
}

.delete-button:hover {
    color: #ef4444;
}

.report-button:hover {
    color: #f59e0b;
}

/* Replies */
.comment-replies {
    margin-top: 12px;
    padding-left: 16px;
    border-left: 2px solid #e5e7eb;
}

.toggle-replies-button {
    background: transparent;
    border: none;
    color: #3b82f6;
    font-size: 12px;
    cursor: pointer;
    padding: 4px 0;
    margin-bottom: 8px;
}

.toggle-replies-button:hover {
    text-decoration: underline;
}

.replies-list {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.reply-form {
    margin-top: 12px;
    padding-left: 16px;
}

/* Loading and Error States */
.loading, .error, .no-comments {
    text-align: center;
    padding: 40px;
    color: #6b7280;
}

.error {
    color: #ef4444;
}

.retry-button {
    background: #3b82f6;
    color: white;
    border: none;
    padding: 8px 16px;
    border-radius: 6px;
    cursor: pointer;
    margin-top: 16px;
}

/* Toast Notifications */
.toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 1000;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.toast {
    padding: 12px 16px;
    border-radius: 6px;
    color: white;
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 250px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    animation: slideIn 0.3s ease-out;
}

.toast.success {
    background: #10b981;
}

.toast.error {
    background: #ef4444;
}

.toast.warning {
    background: #f59e0b;
}

.toast.info {
    background: #3b82f6;
}

.toast-close {
    background: transparent;
    border: none;
    color: white;
    font-size: 18px;
    cursor: pointer;
    margin-left: auto;
    padding: 0;
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
}

@keyframes slideIn {
    from {
        transform: translateX(100%);
        opacity: 0;
    }
    to {
        transform: translateX(0);
        opacity: 1;
    }
}

/* Responsive Design */
@media (max-width: 768px) {
    #app {
        padding: 16px;
    }
    
    header {
        flex-direction: column;
        align-items: flex-start;
        gap: 16px;
    }
    
    .comment-item.level-1 {
        margin-left: 16px;
    }
    
    .comment-item.level-2 {
        margin-left: 32px;
    }
    
    .comment-actions {
        flex-direction: column;
        gap: 12px;
        align-items: flex-start;
    }
    
    .action-buttons {
        flex-wrap: wrap;
    }
}
```

### 3. Secure JavaScript Implementation

```javascript
// commentum-secure.js
class CommentumClient {
    constructor(baseURL) {
        this.baseURL = baseURL
        this.sessionToken = this.loadSessionToken()
    }

    loadSessionToken() {
        if (typeof window !== 'undefined') {
            return localStorage.getItem('commentum_session_token')
        }
        return null
    }

    setSessionToken(token) {
        this.sessionToken = token
        if (typeof window !== 'undefined') {
            localStorage.setItem('commentum_session_token', token)
        }
    }

    clearSessionToken() {
        this.sessionToken = null
        if (typeof window !== 'undefined') {
            localStorage.removeItem('commentum_session_token')
        }
    }

    async request(endpoint, options = {}) {
        if (!this.sessionToken && endpoint !== '/identity-resolve') {
            throw new Error('Not authenticated - Please log in first')
        }

        const url = `${this.baseURL}${endpoint}`
        const headers = {
            'Content-Type': 'application/json',
            ...options.headers,
        }

        // Add auth header for all requests except identity resolution
        if (endpoint !== '/identity-resolve') {
            headers['Authorization'] = `Bearer ${this.sessionToken}`
        }

        const response = await fetch(url, {
            headers,
            ...options,
        })

        if (response.status === 401) {
            this.clearSessionToken()
            throw new Error('Session expired - Please log in again')
        }

        if (!response.ok) {
            const error = await response.json()
            throw new Error(error.error || 'API request failed')
        }

        return response.json()
    }

    // Identity Resolution
    async resolveIdentity(clientType, token) {
        const response = await fetch(`${this.baseURL}/identity-resolve`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.supabaseAnonKey}`,
            },
            body: JSON.stringify({
                client_type: clientType,
                token: token,
            }),
        })

        if (!response.ok) {
            const error = await response.json()
            throw new Error(error.error || 'Identity resolution failed')
        }

        const data = await response.json()
        this.setSessionToken(data.session_token)
        return data
    }

    // Comments API (No user_id needed - extracted from session)
    async getComments(mediaId) {
        return this.request(`/comments?media_id=${mediaId}`)
    }

    async createComment(data) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'create',
                ...data,
                // No user_id needed - extracted from session!
            }),
        })
    }

    async editComment(commentId, content) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'edit',
                comment_id: commentId,
                content,
                // No user_id needed - extracted from session!
            }),
        })
    }

    async deleteComment(commentId) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'delete',
                comment_id: commentId,
                // No user_id needed - extracted from session!
            }),
        })
    }

    // Voting API (No user_id needed - extracted from session)
    async vote(commentId, action) {
        return this.request('/voting', {
            method: 'POST',
            body: JSON.stringify({
                action,
                comment_id: commentId,
                // No user_id needed - extracted from session!
            }),
        })
    }

    // Reports API (No user_id needed - extracted from session)
    async createReport(commentId, reason, notes = '') {
        return this.request('/reports', {
            method: 'POST',
            body: JSON.stringify({
                action: 'create',
                comment_id: commentId,
                reason,
                notes,
                // No user_id needed - extracted from session!
            }),
        })
    }
}

class ToastManager {
    constructor() {
        this.container = document.getElementById('toast-container')
    }

    show(message, type = 'info', duration = 5000) {
        const toast = document.createElement('div')
        toast.className = `toast ${type}`
        
        const messageSpan = document.createElement('span')
        messageSpan.textContent = message
        
        const closeButton = document.createElement('button')
        closeButton.className = 'toast-close'
        closeButton.textContent = '×'
        closeButton.onclick = () => this.remove(toast)
        
        toast.appendChild(messageSpan)
        toast.appendChild(closeButton)
        
        this.container.appendChild(toast)
        
        if (duration > 0) {
            setTimeout(() => this.remove(toast), duration)
        }
        
        return toast
    }

    remove(toast) {
        if (toast && toast.parentNode) {
            toast.parentNode.removeChild(toast)
        }
    }

    success(message) {
        return this.show(message, 'success')
    }

    error(message) {
        return this.show(message, 'error')
    }

    warning(message) {
        return this.show(message, 'warning')
    }

    info(message) {
        return this.show(message, 'info')
    }
}

class CommentSystem {
    constructor(options = {}) {
        this.api = new CommentumClient(options.apiURL || 'https://your-project.supabase.co/functions/v1')
        this.api.supabaseAnonKey = options.supabaseAnonKey || 'your-anon-key'
        this.mediaId = options.mediaId
        this.mediaInfo = options.mediaInfo || null
        this.mediaType = options.mediaType || 'anime'
        this.currentUser = null
        this.isAuthenticated = false
        this.toasts = new ToastManager()
        
        this.comments = []
        this.loading = false
        this.error = null
        
        this.container = document.getElementById('comment-system')
        if (!this.container) {
            throw new Error('Comment system container not found')
        }

        this.init()
    }

    async init() {
        this.setupEventListeners()
        await this.checkAuthStatus()
        await this.loadComments()
        this.render()
    }

    async checkAuthStatus() {
        const token = localStorage.getItem('commentum_session_token')
        if (token) {
            this.api.setSessionToken(token)
            this.isAuthenticated = true
            // You might want to validate the token here
            this.updateAuthStatus()
        }
    }

    async login(clientType, token) {
        try {
            const data = await this.api.resolveIdentity(clientType, token)
            
            if (data.user.banned) {
                throw new Error('Account is banned')
            }

            if (data.user.muted_until && new Date(data.user.muted_until) > new Date()) {
                throw new Error(`Account muted until ${data.user.muted_until}`)
            }

            this.currentUser = data.user
            this.isAuthenticated = true
            this.updateAuthStatus()
            this.render()
            
            this.toasts.success('Logged in successfully!')
            return data
        } catch (error) {
            this.toasts.error(error.message)
            throw error
        }
    }

    logout() {
        this.currentUser = null
        this.isAuthenticated = false
        this.api.clearSessionToken()
        this.updateAuthStatus()
        this.render()
        this.toasts.info('Logged out')
    }

    updateAuthStatus() {
        const authStatus = document.getElementById('auth-status')
        if (!authStatus) return

        if (this.isAuthenticated && this.currentUser) {
            authStatus.innerHTML = `
                <div class="user-info">
                    <img class="user-avatar" src="${this.currentUser.avatar_url || '/default-avatar.png'}" alt="${this.currentUser.username}">
                    <span class="user-name">${this.currentUser.username}</span>
                </div>
                <button class="logout-button" onclick="commentSystem.logout()">Logout</button>
            `
        } else {
            authStatus.innerHTML = `
                <span style="color: #6b7280;">Not logged in</span>
            `
        }
    }

    setupEventListeners() {
        // Event listeners will be set up in render method
    }

    async loadComments() {
        if (!this.mediaId) return

        this.loading = true
        this.error = null
        this.render()

        try {
            this.comments = await this.api.getComments(this.mediaId)
        } catch (error) {
            this.error = error.message
            this.toasts.error('Failed to load comments')
        } finally {
            this.loading = false
            this.render()
        }
    }

    async createComment(content, parentId = null) {
        try {
            const commentData = {
                content: content.trim(),
                parent_id: parentId,
            }

            // Use media_id if available, otherwise use media_info for auto-creation
            if (this.mediaId) {
                commentData.media_id = this.mediaId
            } else if (this.mediaInfo) {
                commentData.media_info = this.mediaInfo
            }

            const comment = await this.api.createComment(commentData)
            this.comments.push(comment)
            this.render()
            this.toasts.success('Comment posted successfully!')
            return comment
        } catch (error) {
            this.toasts.error(error.message)
            throw error
        }
    }

    async vote(commentId, action) {
        try {
            await this.api.vote(commentId, action)
            
            // Update local vote counts
            const comment = this.comments.find(c => c.id === commentId)
            if (comment) {
                // This would need to be implemented based on the response
                // For now, just reload comments
                await this.loadComments()
            }
        } catch (error) {
            this.toasts.error(error.message)
        }
    }

    async reportComment(commentId, reason, notes = '') {
        try {
            await this.api.createReport(commentId, reason, notes)
            this.toasts.success('Comment reported successfully!')
        } catch (error) {
            this.toasts.error(error.message)
        }
    }

    render() {
        this.container.innerHTML = ''

        // Header
        const header = document.createElement('div')
        header.className = 'comment-system-header'
        header.innerHTML = `
            <h3>Comments (${this.comments.length})</h3>
        `
        this.container.appendChild(header)

        // Login form or comment form
        if (!this.isAuthenticated) {
            const loginForm = this.createLoginForm()
            this.container.appendChild(loginForm)
        } else {
            const commentForm = this.createCommentForm()
            this.container.appendChild(commentForm)
        }

        // Comments list
        const commentsSection = document.createElement('div')
        commentsSection.className = 'comments-section'

        if (this.loading) {
            commentsSection.innerHTML = '<div class="loading">Loading comments...</div>'
        } else if (this.error) {
            commentsSection.innerHTML = `
                <div class="error">
                    Failed to load comments: ${this.error}
                    <button class="retry-button" onclick="commentSystem.loadComments()">Retry</button>
                </div>
            `
        } else if (this.comments.length === 0) {
            commentsSection.innerHTML = '<div class="no-comments">No comments yet. Be the first to share your thoughts!</div>'
        } else {
            const commentsList = this.createCommentsList(this.comments.filter(c => !c.parent_id))
            commentsSection.appendChild(commentsList)
        }

        this.container.appendChild(commentsSection)
    }

    createLoginForm() {
        const template = document.getElementById('login-form-template')
        const form = template.content.cloneNode(true).querySelector('form')

        form.addEventListener('submit', async (e) => {
            e.preventDefault()
            const formData = new FormData(form)
            const clientType = formData.get('clientType')
            const token = formData.get('token')

            if (!clientType || !token) return

            const submitButton = form.querySelector('.login-button')
            submitButton.disabled = true
            submitButton.textContent = 'Logging in...'

            try {
                await this.login(clientType, token)
                form.reset()
            } catch (error) {
                // Error already handled in login method
            } finally {
                submitButton.disabled = false
                submitButton.textContent = 'Login'
            }
        })

        return form
    }

    createCommentForm(parentId = null) {
        const template = document.getElementById('comment-form-template')
        const form = template.content.cloneNode(true).querySelector('form')

        const textarea = form.querySelector('.comment-textarea')
        const characterCount = form.querySelector('.character-count')
        const submitButton = form.querySelector('.submit-button')
        const cancelButton = form.querySelector('.cancel-button')

        if (parentId) {
            textarea.placeholder = 'Write a reply...'
            submitButton.textContent = 'Reply'
            cancelButton.style.display = 'inline-block'
        }

        const updateCharacterCount = () => {
            const count = textarea.value.length
            characterCount.textContent = `${count}/10000`
            characterCount.className = count > 9000 ? 'character-count warning' : 'character-count'
            submitButton.disabled = !textarea.value.trim() || count > 10000
        }

        textarea.addEventListener('input', updateCharacterCount)

        form.addEventListener('submit', async (e) => {
            e.preventDefault()
            if (!textarea.value.trim()) return

            submitButton.disabled = true
            submitButton.textContent = parentId ? 'Replying...' : 'Posting...'

            try {
                await this.createComment(textarea.value, parentId)
                form.reset()
                updateCharacterCount()
                
                if (cancelButton.onclick) {
                    cancelButton.onclick()
                }
            } catch (error) {
                // Error already handled in createComment
            } finally {
                submitButton.disabled = false
                submitButton.textContent = parentId ? 'Reply' : 'Post Comment'
            }
        })

        if (cancelButton && parentId) {
            cancelButton.onclick = () => {
                form.remove()
            }
        }

        return form
    }

    createCommentsList(comments, level = 0) {
        const container = document.createElement('div')
        container.className = 'comments-tree'

        comments.forEach(comment => {
            const commentElement = this.createCommentElement(comment, level)
            container.appendChild(commentElement)
        })

        return container
    }

    createCommentElement(comment, level = 0) {
        const template = document.getElementById('comment-item-template')
        const element = template.content.cloneNode(true).querySelector('.comment-item')

        element.className = `comment-item level-${level}`

        // Set comment data
        const authorName = element.querySelector('.author-name')
        const authorAvatar = element.querySelector('.author-avatar')
        const commentTime = element.querySelector('.comment-time')
        const commentText = element.querySelector('.comment-text')
        const upvoteCount = element.querySelector('.upvote-count')
        const downvoteCount = element.querySelector('.downvote-count')

        authorName.textContent = comment.username
        authorAvatar.src = comment.avatar_url || '/default-avatar.png'
        authorAvatar.alt = comment.username
        commentTime.textContent = this.formatTime(comment.created_at)
        commentText.textContent = comment.content
        upvoteCount.textContent = comment.upvotes || 0
        downvoteCount.textContent = comment.downvotes || 0

        // Set up voting buttons
        const upvoteButton = element.querySelector('.upvote')
        const downvoteButton = element.querySelector('.downvote')

        if (this.isAuthenticated) {
            upvoteButton.disabled = false
            downvoteButton.disabled = false

            upvoteButton.onclick = () => this.vote(comment.id, 'upvote')
            downvoteButton.onclick = () => this.vote(comment.id, 'downvote')

            if (comment.user_vote === 1) {
                upvoteButton.classList.add('voted')
            } else if (comment.user_vote === -1) {
                downvoteButton.classList.add('voted')
            }
        }

        // Set up action buttons
        const replyButton = element.querySelector('.reply-button')
        const reportButton = element.querySelector('.report-button')

        if (this.isAuthenticated) {
            replyButton.disabled = false
            reportButton.disabled = false

            replyButton.onclick = () => {
                const existingReplyForm = element.querySelector('.reply-form')
                if (existingReplyForm) {
                    existingReplyForm.remove()
                } else {
                    const replyForm = this.createCommentForm(comment.id)
                    replyForm.className = 'reply-form'
                    element.appendChild(replyForm)
                }
            }

            reportButton.onclick = () => {
                const reason = prompt('Reason for reporting:')
                if (reason) {
                    const notes = prompt('Additional notes (optional):')
                    this.reportComment(comment.id, reason, notes || '')
                }
            }
        }

        // Handle replies
        const replies = this.comments.filter(c => c.parent_id === comment.id)
        if (replies.length > 0) {
            const repliesSection = element.querySelector('.comment-replies')
            const toggleButton = element.querySelector('.toggle-replies-button')
            const repliesList = element.querySelector('.replies-list')

            toggleButton.style.display = 'inline-block'
            toggleButton.textContent = `Show ${replies.length} ${replies.length === 1 ? 'reply' : 'replies'}`

            let isExpanded = false
            toggleButton.onclick = () => {
                isExpanded = !isExpanded
                toggleButton.textContent = isExpanded ? 'Hide replies' : `Show ${replies.length} ${replies.length === 1 ? 'reply' : 'replies'}`
                
                if (isExpanded) {
                    const repliesContainer = this.createCommentsList(replies, level + 1)
                    repliesList.appendChild(repliesContainer)
                } else {
                    repliesList.innerHTML = ''
                }
            }
        }

        return element
    }

    formatTime(dateString) {
        const date = new Date(dateString)
        const now = new Date()
        const diff = now - date

        const minutes = Math.floor(diff / 60000)
        const hours = Math.floor(diff / 3600000)
        const days = Math.floor(diff / 86400000)

        if (minutes < 1) return 'just now'
        if (minutes < 60) return `${minutes} minute${minutes === 1 ? '' : 's'} ago`
        if (hours < 24) return `${hours} hour${hours === 1 ? '' : 's'} ago`
        if (days < 7) return `${days} day${days === 1 ? '' : 's'} ago`

        return date.toLocaleDateString()
    }
}

// Initialize the comment system
document.addEventListener('DOMContentLoaded', () => {
    window.commentSystem = new CommentSystem({
        apiURL: 'https://your-project.supabase.co/functions/v1',
        supabaseAnonKey: 'your-supabase-anon-key',
        mediaId: 123, // Your media ID
        mediaType: 'anime',
        // Alternatively, use mediaInfo for auto-creation:
        // mediaInfo: {
        //     external_id: '12345',
        //     media_type: 'anime',
        //     title: 'Attack on Titan',
        //     year: 2013,
        //     poster_url: 'https://example.com/poster.jpg'
        // }
    })
})
```

## Usage Example

### 1. Basic Integration

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Anime Site</title>
    <link rel="stylesheet" href="commentum-styles.css">
</head>
<body>
    <div id="app">
        <h1>Attack on Titan</h1>
        <p>Anime description...</p>
        
        <div id="comment-system"></div>
    </div>

    <script src="commentum-secure.js"></script>
    <script>
        document.addEventListener('DOMContentLoaded', () => {
            window.commentSystem = new CommentSystem({
                apiURL: 'https://your-project.supabase.co/functions/v1',
                supabaseAnonKey: 'your-supabase-anon-key',
                mediaId: 123,
                mediaType: 'anime'
            })
        })
    </script>
</body>
</html>
```

### 2. With Auto Media Creation

```javascript
window.commentSystem = new CommentSystem({
    apiURL: 'https://your-project.supabase.co/functions/v1',
    supabaseAnonKey: 'your-supabase-anon-key',
    mediaType: 'anime',
    mediaInfo: {
        external_id: '12345',
        media_type: 'anime',
        title: 'Attack on Titan',
        year: 2013,
        poster_url: 'https://example.com/poster.jpg'
    }
    // No mediaId needed - will be created automatically!
})
```

## 🔒 Security Features

### Session-Based Authentication
- **No user_id parameters**: User identity is extracted from session tokens
- **Automatic token management**: Tokens are stored and refreshed automatically
- **Session validation**: All API calls validate the session token
- **Automatic logout**: Sessions are cleared on expiration

### Zero-Trust Architecture
- **Server-side validation**: All user data is validated on the server
- **No client trust**: No client-provided user data is trusted
- **Real-time verification**: Provider tokens are verified with real APIs
- **Audit logging**: All actions are logged for security

### Error Handling
- **Session expiration**: Automatic handling of expired sessions
- **Network errors**: Graceful handling of connection issues
- **Permission errors**: Clear error messages for permission issues

## 🚨 Breaking Changes from Old Version

### Before (Vulnerable)
```javascript
// ❌ OLD - Client could send any user_id
await api.createComment(userId, mediaId, content)

// API call included user_id parameter
fetch('/functions/v1/comments', {
  body: JSON.stringify({
    action: 'create',
    user_id: 123,  // Could be faked!
    media_id: 456,
    content: 'Comment text'
  })
})
```

### After (Secure)
```javascript
// ✅ NEW - User ID extracted from session
await api.createComment({ media_id: mediaId, content })

// No user_id needed - extracted from session token!
fetch('/functions/v1/comments', {
  headers: { 'Authorization': `Bearer ${sessionToken}` },
  body: JSON.stringify({
    action: 'create',
    media_id: 456,
    content: 'Comment text'
  })
})
```

This secure vanilla JavaScript implementation ensures that comment operations can only be performed by authenticated users while preventing identity spoofing attacks.