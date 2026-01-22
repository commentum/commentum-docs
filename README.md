# Commentum Documentation

A **comprehensive, enterprise-grade secure** comment system with advanced moderation, voting, and user management capabilities.

## Overview

Commentum is a **secure** comment system built on Supabase with TypeScript/Edge Functions. It provides a complete solution for adding comments to any media platform with **zero-trust security**:

- **Session-Based Authentication**: Prevents identity spoofing attacks
- **Nested Comments**: Support for threaded discussions with configurable nesting levels
- **Voting System**: Upvote/downvote functionality with abuse detection
- **Advanced Moderation**: Automated moderation, manual moderation, and escalation workflows
- **User Management**: Role-based permissions, warnings, bans, and shadow bans
- **Multi-Platform Identity**: **Real token verification** for AniList, MyAnimeList, and SIMKL
- **Rate Limiting**: Comprehensive rate limiting to prevent abuse
- **Real-time Updates**: WebSocket support for live comment updates
- **Zero-Trust Architecture**: No client-provided user data is trusted

## 📋 Table of Contents

- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Integration Guide](#integration-guide)
- [Authentication](#-authentication)
- [Security](#️-security)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)

## Quick Start

### Prerequisites

- Supabase project with PostgreSQL database
- Edge Functions enabled
- Database functions and RLS policies deployed
- **Session-based authentication system deployed**

### Basic Secure Integration

```typescript
// 1. Authenticate with session-based resolution
const authResponse = await fetch('/functions/v1/identity-resolve', {
  method: 'POST',
  headers: { 
    'Authorization': `Bearer ${SUPABASE_ANON_KEY}`,
    'Content-Type': 'application/json' 
  },
  body: JSON.stringify({
    client_type: 'anilist',
    token: 'real_provider_oauth_token'
  })
});

const { session_token, user } = await authResponse.json();
localStorage.setItem('commentum_session_token', session_token);

// 2. Create a comment with session authentication
const commentResponse = await fetch('/functions/v1/comments', {
  method: 'POST',
  headers: { 
    'Authorization': `Bearer ${session_token}`,
    'Content-Type': 'application/json' 
  },
  body: JSON.stringify({
    action: 'create',
    media_id: 1,
    content: 'This is a great comment!'
    // ❌ NO user_id needed - extracted from session!
  })
});

const comment = await commentResponse.json();
```

## API Reference

### Core Endpoints (All Session-Based)

- **[Identity Resolution](./api/identity-resolve.md)** - Secure session creation and token verification
- **[Comments](./api/comments.md)** - Session-based comment CRUD operations
- **[Voting](./api/voting.md)** - Session-based upvote/downvote functionality
- **[Reports](./api/reports.md)** - Session-based comment reporting and moderation
- **[Moderation](./api/moderation.md)** - Session-based user moderation and admin actions

## Database Schema

The Commentum system uses a **secure** PostgreSQL schema with the following key components:

- **[Users & Sessions](./database/users.md)** - User management and **secure session storage**
- **[Comments System](./database/comments.md)** - Comments, threads, and content management
- **[Moderation System](./database/moderation.md)** - Reports, warnings, and moderation actions
- **[Voting System](./database/voting.md)** - Votes and abuse tracking
- **[System Configuration](./database/system.md)** - Settings and configuration

## Integration Guide

For detailed **secure** integration instructions, see the [Integration Guide](./integration.md).

## Authentication

Commentum uses a **secure session-based authentication system** that prevents identity spoofing. See the [Authentication Guide](./authentication.md) for complete details.

## Security

**Enterprise-grade security features**:
- **Session-Based Authentication**: Cryptographically secure session tokens
- **Zero-Trust Architecture**: No client-provided user data trusted
- **Real Token Verification**: All provider tokens verified with respective APIs
- Row Level Security (RLS) on all tables
- Rate limiting on all actions
- Automated abuse detection
- Role-based permissions
- Input validation and sanitization
- **Identity Spoofing Prevention**: Impossible to fake user identities

See the [Security Guide](./security.md) for comprehensive security documentation.

## Examples

Check out the [Examples](./examples/) directory for **secure** integration examples:

- [React Integration](./examples/react.md) - Session-based authentication
- [Vue.js Integration](./examples/vue.md) - Secure session management
- [Vanilla JavaScript](./examples/vanilla.md) - Session-based API calls

## Troubleshooting

Common issues and solutions can be found in the [Troubleshooting Guide](./troubleshooting.md).

## License

This documentation is part of the Commentum project. See the main repository [main repository](https://github.com/commentum/commentum) for license information.

## 🤝 Contributing

For contributions to the documentation, please refer to the contributing guidelines in the [main repository](https://github.com/commentum/commentum).

---

**Security Notice**: This system implements enterprise-grade session-based authentication that prevents identity spoofing attacks. All API calls require valid session tokens.
