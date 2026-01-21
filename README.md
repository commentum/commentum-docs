# Commentum Documentation

A comprehensive, production-ready comment system with advanced moderation, voting, and user management capabilities.

## 🚀 Overview

Commentum is a powerful comment system built on Supabase with TypeScript/Edge Functions. It provides a complete solution for adding comments to any media platform with features like:

- **Nested Comments**: Support for threaded discussions with configurable nesting levels
- **Voting System**: Upvote/downvote functionality with abuse detection
- **Advanced Moderation**: Automated moderation, manual moderation, and escalation workflows
- **User Management**: Role-based permissions, warnings, bans, and shadow bans
- **Multi-Platform Identity**: Support for AniList, MyAnimeList, and SIMKL integration
- **Rate Limiting**: Comprehensive rate limiting to prevent abuse
- **Real-time Updates**: WebSocket support for live comment updates

## 📋 Table of Contents

- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Integration Guide](#integration-guide)
- [Authentication](#authentication)
- [Security](#security)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)

## 🚀 Quick Start

### Prerequisites

- Supabase project with PostgreSQL database
- Edge Functions enabled
- Database functions and RLS policies deployed

### Basic Integration

```typescript
// 1. Resolve user identity
const userResponse = await fetch('/api/identity-resolve', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    client_type: 'anilist',
    client_user_id: 'anilist userid',
    username: 'anilist username',
    avatar_url: 'https://example.com/avatar.jpg'
  })
});

const { user_id } = await userResponse.json();

// 2. Create a comment
const commentResponse = await fetch('/api/comments', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    action: 'create',
    user_id,
    media_id: 1,
    content: 'This is a great comment!'
  })
});

const comment = await commentResponse.json();
```

## 📚 API Reference

### Core Endpoints

- **[Identity Resolution](./docs/api/identity-resolve.md)** - User authentication and identity management
- **[Comments](./docs/api/comments.md)** - Comment CRUD operations and management
- **[Voting](./docs/api/voting.md)** - Upvote/downvote functionality
- **[Reports](./docs/api/reports.md)** - Comment reporting and moderation
- **[Moderation](./docs/api/moderation.md)** - User moderation and admin actions

## 🗄️ Database Schema

The Commentum system uses a comprehensive PostgreSQL schema with the following key components:

- **[Users & Identities](./docs/database/users.md)** - User management and multi-platform identities
- **[Comments System](./docs/database/comments.md)** - Comments, threads, and content management
- **[Moderation System](./docs/database/moderation.md)** - Reports, warnings, and moderation actions
- **[Voting System](./docs/database/voting.md)** - Votes and abuse tracking
- **[System Configuration](./docs/database/system.md)** - Settings and configuration

## 🔧 Integration Guide

For detailed integration instructions, see the [Integration Guide](./docs/integration.md).

## 🔐 Authentication

Commentum uses a flexible identity system that supports multiple authentication providers. See the [Authentication Guide](./docs/authentication.md) for details.

## 🛡️ Security

Security features include:
- Row Level Security (RLS) on all tables
- Rate limiting on all actions
- Automated abuse detection
- Role-based permissions
- Input validation and sanitization

See the [Security Guide](./docs/security.md) for comprehensive security documentation.

## 💡 Examples

Check out the [Examples](./docs/examples/) directory for complete integration examples:

- [React Integration](./docs/examples/react.md)
- [Vue.js Integration](./docs/examples/vue.md)
- [Vanilla JavaScript](./docs/examples/vanilla.md)

## ❓ Troubleshooting

Common issues and solutions can be found in the [Troubleshooting Guide](./docs/troubleshooting.md).

## 📝 License

This documentation is part of the Commentum project. See the main repository [main repository](https://github.com/commentum/commentum) for license information.

## 🤝 Contributing

For contributions to the documentation, please refer to the contributing guidelines in the [main repository](https://github.com/commentum/commentum).

---

**Note**: This documentation contains only public API information and integration guidance. No sensitive data like API keys, admin credentials, or internal configuration is included.