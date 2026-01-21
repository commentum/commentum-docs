# Vanilla JavaScript Integration Example

This example demonstrates how to integrate Commentum into a vanilla JavaScript application without any frameworks.

## Project Setup

### 1. HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Commentum Integration - Vanilla JS</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div id="app">
        <header>
            <h1>Media Title</h1>
            <p>Type: anime</p>
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

    <script src="commentum.js"></script>
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
    margin-right: 8px;
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

### 3. JavaScript Implementation

```javascript
// commentum.js
class CommentumAPI {
    constructor(baseURL) {
        this.baseURL = baseURL
    }

    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`
        
        const response = await fetch(url, {
            headers: {
                'Content-Type': 'application/json',
                ...options.headers,
            },
            ...options,
        })
        
        if (!response.ok) {
            const error = await response.json()
            throw new Error(error.error || 'API request failed')
        }
        
        return response.json()
    }

    // Identity Resolution
    async resolveIdentity(clientType, clientUserId, username, avatarUrl) {
        return this.request('/identity-resolve', {
            method: 'POST',
            body: JSON.stringify({
                client_type: clientType,
                client_user_id: clientUserId,
                username,
                avatar_url: avatarUrl
            })
        })
    }

    // Comments
    async getComments(mediaId) {
        return this.request(`/comments?media_id=${mediaId}`)
    }

    async createComment(userId, mediaId, content, parentId = null) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'create',
                user_id: userId,
                media_id: mediaId,
                content,
                parent_id: parentId
            })
        })
    }

    async editComment(userId, commentId, content) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'edit',
                user_id: userId,
                comment_id: commentId,
                content
            })
        })
    }

    async deleteComment(userId, commentId) {
        return this.request('/comments', {
            method: 'POST',
            body: JSON.stringify({
                action: 'delete',
                user_id: userId,
                comment_id: commentId
            })
        })
    }

    // Voting
    async vote(userId, commentId, action) {
        return this.request('/voting', {
            method: 'POST',
            body: JSON.stringify({
                action,
                user_id: userId,
                comment_id: commentId
            })
        })
    }

    // Reports
    async reportComment(userId, commentId, reason, notes = '') {
        return this.request('/reports', {
            method: 'POST',
            body: JSON.stringify({
                action: 'create',
                user_id: userId,
                comment_id: commentId,
                reason,
                notes
            })
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
        this.api = new CommentumAPI(options.apiURL || 'https://your-project.supabase.co/functions/v1')
        this.mediaId = options.mediaId
        this.mediaType = options.mediaType || 'anime'
        this.currentUser = options.currentUser || null
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
        this.render()
        await this.loadComments()
    }

    setCurrentUser(user) {
        this.currentUser = user
        this.render()
    }

    async loadComments() {
        this.loading = true
        this.error = null
        this.render()

        try {
            this.comments = await this.api.getComments(this.mediaId)
            this.error = null
        } catch (error) {
            this.error = error.message
            this.toasts.error(`Failed to load comments: ${error.message}`)
        } finally {
            this.loading = false
            this.render()
        }
    }

    async createComment(content, parentId = null) {
        if (!this.currentUser) {
            this.toasts.error('You must be logged in to comment')
            return
        }

        try {
            const comment = await this.api.createComment(
                this.currentUser.user_id,
                this.mediaId,
                content,
                parentId
            )
            
            this.comments.push(comment)
            this.toasts.success('Comment posted successfully!')
            this.render()
            
            return comment
        } catch (error) {
            this.toasts.error(`Failed to post comment: ${error.message}`)
            throw error
        }
    }

    async editComment(commentId, content) {
        if (!this.currentUser) {
            this.toasts.error('You must be logged in to edit comments')
            return
        }

        try {
            const updatedComment = await this.api.editComment(
                this.currentUser.user_id,
                commentId,
                content
            )
            
            const index = this.comments.findIndex(c => c.id === commentId)
            if (index !== -1) {
                this.comments[index] = updatedComment
            }
            
            this.toasts.success('Comment updated successfully!')
            this.render()
            
            return updatedComment
        } catch (error) {
            this.toasts.error(`Failed to update comment: ${error.message}`)
            throw error
        }
    }

    async deleteComment(commentId) {
        if (!this.currentUser) {
            this.toasts.error('You must be logged in to delete comments')
            return
        }

        try {
            const deletedComment = await this.api.deleteComment(
                this.currentUser.user_id,
                commentId
            )
            
            const index = this.comments.findIndex(c => c.id === commentId)
            if (index !== -1) {
                this.comments[index] = deletedComment
            }
            
            this.toasts.success('Comment deleted successfully!')
            this.render()
            
            return deletedComment
        } catch (error) {
            this.toasts.error(`Failed to delete comment: ${error.message}`)
            throw error
        }
    }

    async vote(commentId, action) {
        if (!this.currentUser) {
            this.toasts.error('You must be logged in to vote')
            return
        }

        try {
            await this.api.vote(this.currentUser.user_id, commentId, action)
            await this.loadComments() // Reload to get updated vote counts
        } catch (error) {
            this.toasts.error(`Failed to vote: ${error.message}`)
            throw error
        }
    }

    async reportComment(commentId, reason, notes = '') {
        if (!this.currentUser) {
            this.toasts.error('You must be logged in to report comments')
            return
        }

        try {
            await this.api.reportComment(
                this.currentUser.user_id,
                commentId,
                reason,
                notes
            )
            
            this.toasts.success('Comment reported successfully!')
        } catch (error) {
            this.toasts.error(`Failed to report comment: ${error.message}`)
            throw error
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

        // Comment form
        if (this.currentUser) {
            const formContainer = document.createElement('div')
            formContainer.className = 'comment-form-section'
            formContainer.appendChild(new CommentForm(this).element)
            this.container.appendChild(formContainer)
        }

        // Comments list
        const commentsSection = document.createElement('div')
        commentsSection.className = 'comments-section'

        if (this.loading) {
            commentsSection.innerHTML = '<div class="loading">Loading comments...</div>'
        } else if (this.error) {
            commentsSection.innerHTML = `
                <div class="error">
                    <div>Failed to load comments: ${this.error}</div>
                    <button class="retry-button" onclick="window.commentSystem.loadComments()">Retry</button>
                </div>
            `
        } else if (this.comments.length === 0) {
            commentsSection.innerHTML = '<div class="no-comments">No comments yet. Be the first to share your thoughts!</div>'
        } else {
            const topLevelComments = this.comments.filter(c => !c.parent_id)
            const repliesByParent = {}
            
            this.comments.forEach(comment => {
                if (comment.parent_id) {
                    if (!repliesByParent[comment.parent_id]) {
                        repliesByParent[comment.parent_id] = []
                    }
                    repliesByParent[comment.parent_id].push(comment)
                }
            })

            const commentsList = document.createElement('div')
            commentsList.className = 'comments-tree'

            topLevelComments.forEach(comment => {
                const commentElement = new CommentItem(
                    comment,
                    repliesByParent[comment.id] || [],
                    this
                )
                commentsList.appendChild(commentElement.element)
            })

            commentsSection.appendChild(commentsList)
        }

        this.container.appendChild(commentsSection)
    }
}

class CommentForm {
    constructor(commentSystem) {
        this.commentSystem = commentSystem
        this.element = this.createElement()
        this.bindEvents()
    }

    createElement() {
        const template = document.getElementById('comment-form-template')
        const form = template.content.cloneNode(true).querySelector('.comment-form')
        
        return form
    }

    bindEvents() {
        const textarea = this.element.querySelector('.comment-textarea')
        const characterCount = this.element.querySelector('.character-count')
        const submitButton = this.element.querySelector('.submit-button')

        // Character count
        textarea.addEventListener('input', () => {
            const count = textarea.value.length
            characterCount.textContent = `${count}/10000`
            characterCount.classList.toggle('warning', count > 9000)
            
            submitButton.disabled = !textarea.value.trim() || count > 10000
        })

        // Form submission
        this.element.addEventListener('submit', async (e) => {
            e.preventDefault()
            
            const content = textarea.value.trim()
            if (!content) return

            submitButton.disabled = true
            submitButton.textContent = 'Posting...'

            try {
                await this.commentSystem.createComment(content)
                textarea.value = ''
                characterCount.textContent = '0/10000'
                characterCount.classList.remove('warning')
            } catch (error) {
                // Error is already handled by CommentSystem
            } finally {
                submitButton.disabled = false
                submitButton.textContent = 'Post Comment'
            }
        })
    }
}

class CommentItem {
    constructor(comment, replies, commentSystem) {
        this.comment = comment
        this.replies = replies
        this.commentSystem = commentSystem
        this.isExpanded = false
        this.isReplying = false
        this.isEditing = false
        this.element = this.createElement()
        this.bindEvents()
    }

    createElement() {
        const template = document.getElementById('comment-item-template')
        const item = template.content.cloneNode(true).querySelector('.comment-item')

        // Set comment data
        const avatar = item.querySelector('.author-avatar')
        const authorName = item.querySelector('.author-name')
        const commentTime = item.querySelector('.comment-time')
        const commentText = item.querySelector('.comment-text')
        const upvoteCount = item.querySelector('.upvote-count')
        const downvoteCount = item.querySelector('.downvote-count')

        if (this.comment.avatar_url) {
            avatar.src = this.comment.avatar_url
        }
        authorName.textContent = this.comment.username
        commentTime.textContent = this.formatTime(this.comment.created_at)
        commentText.textContent = this.comment.content
        upvoteCount.textContent = this.comment.upvotes
        downvoteCount.textContent = this.comment.downvotes

        // Set meta information
        const editedIndicator = item.querySelector('.edited-indicator')
        const pinnedIndicator = item.querySelector('.pinned-indicator')
        const lockedIndicator = item.querySelector('.locked-indicator')

        if (this.comment.edited) {
            editedIndicator.style.display = 'inline'
        }
        if (this.comment.pinned) {
            pinnedIndicator.style.display = 'inline'
        }
        if (this.comment.locked) {
            lockedIndicator.style.display = 'inline'
        }

        // Set voting state
        const upvoteButton = item.querySelector('.upvote')
        const downvoteButton = item.querySelector('.downvote')
        
        if (this.comment.user_vote === 1) {
            upvoteButton.classList.add('voted')
        } else if (this.comment.user_vote === -1) {
            downvoteButton.classList.add('voted')
        }

        // Show/hide action buttons based on user permissions
        this.updateActionButtons(item)

        // Handle replies
        if (this.replies.length > 0) {
            const repliesSection = item.querySelector('.comment-replies')
            const toggleButton = item.querySelector('.toggle-replies-button')
            const repliesList = item.querySelector('.replies-list')

            toggleButton.textContent = `Show ${this.replies.length} ${this.replies.length === 1 ? 'reply' : 'replies'}`

            toggleButton.addEventListener('click', () => {
                this.isExpanded = !this.isExpanded
                repliesSection.style.display = this.isExpanded ? 'block' : 'none'
                toggleButton.textContent = this.isExpanded 
                    ? `Hide ${this.replies.length} ${this.replies.length === 1 ? 'reply' : 'replies'}`
                    : `Show ${this.replies.length} ${this.replies.length === 1 ? 'reply' : 'replies'}`

                if (this.isExpanded && repliesList.children.length === 0) {
                    this.renderReplies(repliesList)
                }
            })
        }

        return item
    }

    updateActionButtons(item) {
        const replyButton = item.querySelector('.reply-button')
        const editButton = item.querySelector('.edit-button')
        const deleteButton = item.querySelector('.delete-button')
        const reportButton = item.querySelector('.report-button')
        const votingButtons = item.querySelectorAll('.vote-button')

        const user = this.commentSystem.currentUser
        const canEdit = user && user.user_id === this.comment.user_id && !this.comment.deleted
        const canVote = user && !this.comment.deleted
        const canReport = user && !this.comment.deleted

        replyButton.disabled = !user
        editButton.style.display = canEdit ? 'inline-block' : 'none'
        deleteButton.style.display = canEdit ? 'inline-block' : 'none'
        reportButton.disabled = !canReport

        votingButtons.forEach(button => {
            button.disabled = !canVote
        })
    }

    renderReplies(container) {
        this.replies.forEach(reply => {
            const replyElement = new CommentItem(reply, [], this.commentSystem)
            replyElement.element.classList.add('level-1')
            container.appendChild(replyElement.element)
        })
    }

    bindEvents() {
        const upvoteButton = this.element.querySelector('.upvote')
        const downvoteButton = this.element.querySelector('.downvote')
        const replyButton = this.element.querySelector('.reply-button')
        const editButton = this.element.querySelector('.edit-button')
        const deleteButton = this.element.querySelector('.delete-button')
        const reportButton = this.element.querySelector('.report-button')

        // Voting
        upvoteButton.addEventListener('click', async () => {
            const action = this.comment.user_vote === 1 ? 'remove' : 'upvote'
            try {
                await this.commentSystem.vote(this.comment.id, action)
            } catch (error) {
                // Error handled by CommentSystem
            }
        })

        downvoteButton.addEventListener('click', async () => {
            const action = this.comment.user_vote === -1 ? 'remove' : 'downvote'
            try {
                await this.commentSystem.vote(this.comment.id, action)
            } catch (error) {
                // Error handled by CommentSystem
            }
        })

        // Reply
        replyButton.addEventListener('click', () => {
            this.toggleReplyForm()
        })

        // Edit
        if (editButton.style.display !== 'none') {
            editButton.addEventListener('click', () => {
                this.startEdit()
            })
        }

        // Delete
        if (deleteButton.style.display !== 'none') {
            deleteButton.addEventListener('click', async () => {
                if (confirm('Are you sure you want to delete this comment?')) {
                    try {
                        await this.commentSystem.deleteComment(this.comment.id)
                    } catch (error) {
                        // Error handled by CommentSystem
                    }
                }
            })
        }

        // Report
        reportButton.addEventListener('click', () => {
            this.showReportDialog()
        })
    }

    toggleReplyForm() {
        const repliesSection = this.element.querySelector('.comment-replies')
        let replyForm = repliesSection.querySelector('.reply-form')

        if (replyForm) {
            replyForm.remove()
            this.isReplying = false
        } else {
            replyForm = document.createElement('div')
            replyForm.className = 'reply-form'
            
            const form = new CommentForm(this.commentSystem)
            form.element.querySelector('.comment-textarea').placeholder = 'Write a reply...'
            form.element.querySelector('.submit-button').textContent = 'Reply'
            
            // Override form submission to handle replies
            const originalSubmit = form.element.onsubmit
            form.element.onsubmit = async (e) => {
                e.preventDefault()
                const textarea = form.element.querySelector('.comment-textarea')
                const content = textarea.value.trim()
                
                if (!content) return

                const submitButton = form.element.querySelector('.submit-button')
                submitButton.disabled = true
                submitButton.textContent = 'Posting...'

                try {
                    await this.commentSystem.createComment(content, this.comment.id)
                    replyForm.remove()
                    this.isReplying = false
                } catch (error) {
                    // Error handled by CommentSystem
                } finally {
                    submitButton.disabled = false
                    submitButton.textContent = 'Reply'
                }
            }

            replyForm.appendChild(form.element)
            repliesSection.appendChild(replyForm)
            repliesSection.style.display = 'block'
            this.isReplying = true

            // Focus on textarea
            form.element.querySelector('.comment-textarea').focus()
        }
    }

    startEdit() {
        const commentBody = this.element.querySelector('.comment-body')
        const originalContent = this.comment.content

        const editForm = document.createElement('div')
        editForm.className = 'edit-form'
        editForm.innerHTML = `
            <textarea class="comment-textarea" maxlength="10000">${originalContent}</textarea>
            <div class="form-actions">
                <button type="button" class="cancel-button">Cancel</button>
                <button type="button" class="submit-button">Save</button>
            </div>
        `

        const textarea = editForm.querySelector('.comment-textarea')
        const cancelButton = editForm.querySelector('.cancel-button')
        const saveButton = editForm.querySelector('.submit-button')

        const cancelEdit = () => {
            commentBody.innerHTML = `<p class="comment-text">${originalContent}</p>`
            this.isEditing = false
        }

        const saveEdit = async () => {
            const content = textarea.value.trim()
            if (!content || content === originalContent) {
                cancelEdit()
                return
            }

            saveButton.disabled = true
            saveButton.textContent = 'Saving...'

            try {
                await this.commentSystem.editComment(this.comment.id, content)
                this.isEditing = false
            } catch (error) {
                // Error handled by CommentSystem
            } finally {
                saveButton.disabled = false
                saveButton.textContent = 'Save'
            }
        }

        cancelButton.addEventListener('click', cancelEdit)
        saveButton.addEventListener('click', saveEdit)

        commentBody.innerHTML = ''
        commentBody.appendChild(editForm)
        this.isEditing = true

        // Focus on textarea
        textarea.focus()
    }

    showReportDialog() {
        const reason = prompt('Please select a reason for reporting:\n1. spam\n2. offensive\n3. harassment\n4. spoiler\n5. nsfw\n6. off_topic\n7. other\n\nEnter the number or reason:')
        
        if (!reason) return

        const validReasons = ['spam', 'offensive', 'harassment', 'spoiler', 'nsfw', 'off_topic', 'other']
        const selectedReason = validReasons.includes(reason) ? reason : 
                               validReasons[parseInt(reason) - 1] || null

        if (!selectedReason) {
            this.commentSystem.toasts.error('Invalid reason selected')
            return
        }

        const notes = prompt('Additional notes (optional):') || ''

        this.reportComment(selectedReason, notes)
    }

    async reportComment(reason, notes) {
        try {
            await this.commentSystem.reportComment(this.comment.id, reason, notes)
        } catch (error) {
            // Error handled by CommentSystem
        }
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
        if (days < 30) return `${days} day${days === 1 ? '' : 's'} ago`
        
        return date.toLocaleDateString()
    }
}

// Initialize the comment system
document.addEventListener('DOMContentLoaded', () => {
    // Configuration
    const config = {
        mediaId: 123, // Replace with your media ID
        mediaType: 'anime', // Replace with your media type
        apiURL: 'https://your-project.supabase.co/functions/v1' // Replace with your API URL
    }

    // Create comment system
    window.commentSystem = new CommentSystem(config)

    // Example: Set current user (you would get this from your auth system)
    // window.commentSystem.setCurrentUser({
    //     user_id: 123,
    //     username: 'testuser',
    //     role: 'user'
    // })

    // Example authentication function
    window.loginWithAniList = async () => {
        try {
            // This is a simplified example - implement proper OAuth flow
            const response = await window.commentSystem.api.resolveIdentity(
                'anilist',
                '12345',
                'TestUser',
                'https://example.com/avatar.jpg'
            )
            
            window.commentSystem.setCurrentUser(response)
            window.commentSystem.toasts.success(`Logged in as ${response.username}`)
        } catch (error) {
            window.commentSystem.toasts.error(`Login failed: ${error.message}`)
        }
    }

    // Example logout function
    window.logout = () => {
        window.commentSystem.setCurrentUser(null)
        window.commentSystem.toasts.info('Logged out')
    }
})
```

## Usage Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Commentum Demo - Vanilla JS</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div id="app">
        <header>
            <h1>Sample Anime Page</h1>
            <p>Type: anime</p>
            <div class="auth-section">
                <button onclick="loginWithAniList()">Login with AniList</button>
                <button onclick="logout()" style="display: none;" id="logout-btn">Logout</button>
            </div>
        </header>

        <main>
            <section class="media-content">
                <h2>About this Anime</h2>
                <p>This is a sample anime page where users can discuss and share their thoughts about the show.</p>
            </section>

            <section class="comments-section">
                <div id="comment-system">
                    <!-- Comment system will be rendered here -->
                </div>
            </section>
        </main>
    </div>

    <!-- Include the templates and script from above -->
    <!-- ... templates here ... -->
    <script src="commentum.js"></script>
    
    <script>
        // Update auth UI based on login state
        document.addEventListener('DOMContentLoaded', () => {
            const updateAuthUI = () => {
                const logoutBtn = document.getElementById('logout-btn')
                const loginBtn = document.querySelector('button[onclick="loginWithAniList()"]')
                
                if (window.commentSystem?.currentUser) {
                    loginBtn.style.display = 'none'
                    logoutBtn.style.display = 'inline-block'
                } else {
                    loginBtn.style.display = 'inline-block'
                    logoutBtn.style.display = 'none'
                }
            }

            // Override the login function to update UI
            const originalLogin = window.loginWithAniList
            window.loginWithAniList = async () => {
                await originalLogin()
                updateAuthUI()
            }

            // Override the logout function to update UI
            const originalLogout = window.logout
            window.logout = () => {
                originalLogout()
                updateAuthUI()
            }

            // Initial UI update
            setTimeout(updateAuthUI, 100)
        })
    </script>
</body>
</html>
```

This vanilla JavaScript implementation provides a complete, framework-agnostic comment system that can be easily integrated into any web application.