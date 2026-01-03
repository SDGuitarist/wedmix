# CLAUDE.md - WedMix AI Assistant Guide

This document provides guidance for AI assistants working with the WedMix codebase.

## Project Overview

**WedMix** is an AI-powered wedding music setlist generator that integrates with Spotify and GPT. The application allows users to create personalized wedding playlists using AI recommendations based on their preferences.

- **Repository**: wedmix
- **Package Name**: wedmix-api
- **License**: MIT
- **Current Status**: Early-stage development (scaffolding complete, no source code yet)

## Technology Stack

### Core Technologies
- **Runtime**: Node.js
- **Language**: TypeScript 5.0.4
- **Framework**: Express.js 4.18.2
- **API Integration**: spotify-web-api-node 5.0.2

### Supporting Libraries
- **cors**: Cross-origin request handling
- **dotenv**: Environment variable management

### Development Tools
- **ts-node-dev**: Hot-reloading TypeScript development server
- **TypeScript**: Static typing and compilation

## Project Structure

### Current Structure
```
wedmix/
├── .env.example          # Environment variables template
├── .gitignore            # Git ignore rules (node_modules, .env, dist/)
├── LICENSE               # MIT License
├── README.md             # Project description
├── CLAUDE.md             # This file - AI assistant guide
├── package.json          # NPM configuration and dependencies
├── package-lock.json     # Dependency lock file
├── index.html            # Placeholder landing page
└── wedmix/               # Empty directory (placeholder)
```

### Expected Structure (To Be Implemented)
```
wedmix/
├── src/
│   ├── index.ts          # Express server entry point
│   ├── routes/           # API route definitions
│   │   ├── spotify.ts    # Spotify OAuth and API routes
│   │   └── playlist.ts   # Playlist generation routes
│   ├── controllers/      # Business logic handlers
│   ├── services/         # External API integrations
│   │   ├── spotify.ts    # Spotify API service
│   │   └── gpt.ts        # GPT integration service
│   ├── middleware/       # Express middleware
│   ├── types/            # TypeScript interfaces and types
│   └── utils/            # Utility functions
├── dist/                 # Compiled JavaScript (generated, gitignored)
├── tests/                # Test files
└── [config files]
```

## Development Commands

```bash
# Install dependencies
npm install

# Start development server with hot-reload
npm run dev

# Build TypeScript to JavaScript
npm run build

# Run production server (requires build first)
npm start

# Run tests (Jest - not yet configured)
npm test
```

## Environment Configuration

### Required Environment Variables

Copy `.env.example` to `.env` and configure:

```bash
PORT=3000                                                    # Server port
SPOTIFY_CLIENT_ID=your_spotify_client_id                     # Spotify OAuth client ID
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret             # Spotify OAuth client secret
SPOTIFY_REDIRECT_URI=http://localhost:3000/api/spotify/callback  # OAuth callback URL
```

### Getting Spotify Credentials
1. Go to [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)
2. Create a new application
3. Note the Client ID and Client Secret
4. Add the redirect URI to your app settings

## Code Conventions

### TypeScript Guidelines
- Use strict TypeScript typing - avoid `any` types
- Define interfaces for all data structures in `src/types/`
- Use async/await for asynchronous operations
- Export types and interfaces that may be reused

### Express Patterns
- Use route files to organize endpoints by feature
- Implement controller pattern for business logic separation
- Use middleware for cross-cutting concerns (auth, logging, error handling)
- Return consistent JSON response structures

### Naming Conventions
- **Files**: kebab-case (`spotify-service.ts`)
- **Classes**: PascalCase (`SpotifyService`)
- **Functions/Variables**: camelCase (`getUserPlaylists`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_TRACKS_PER_REQUEST`)
- **Interfaces**: PascalCase with `I` prefix optional (`User` or `IUser`)

### Error Handling
- Use try-catch blocks for async operations
- Create custom error classes for domain-specific errors
- Return appropriate HTTP status codes
- Log errors with sufficient context for debugging

## API Design

### Expected Endpoints

```
GET  /api/spotify/auth          # Initiate Spotify OAuth
GET  /api/spotify/callback      # OAuth callback handler
GET  /api/spotify/me            # Get authenticated user info

POST /api/playlist/generate     # Generate AI wedding playlist
GET  /api/playlist/:id          # Get playlist details
POST /api/playlist/:id/save     # Save playlist to Spotify
```

### Response Format
```typescript
// Success response
{
  "success": true,
  "data": { ... }
}

// Error response
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

## Testing Guidelines

### Setup (To Be Configured)
- Testing framework: Jest
- Add Jest to devDependencies: `npm install -D jest @types/jest ts-jest`
- Create `jest.config.js` for TypeScript support

### Test Organization
```
tests/
├── unit/                 # Unit tests for individual functions
├── integration/          # API endpoint tests
└── fixtures/             # Test data and mocks
```

### Testing Conventions
- Name test files with `.test.ts` or `.spec.ts` suffix
- Co-locate tests with source or use `tests/` directory
- Mock external APIs (Spotify, GPT) in tests
- Aim for high coverage on business logic

## Common Tasks

### Adding a New API Endpoint
1. Create/update route file in `src/routes/`
2. Create controller function in `src/controllers/`
3. Add any required types in `src/types/`
4. Register route in main router
5. Add tests for the endpoint

### Integrating with Spotify API
- Use the `spotify-web-api-node` library
- Handle OAuth token refresh automatically
- Respect rate limits (retry with exponential backoff)
- Store tokens securely (not in client-side code)

### Adding GPT Integration
- Use OpenAI API for playlist recommendations
- Create prompts that consider wedding context
- Handle API errors gracefully
- Consider caching for similar requests

## Missing Configuration (Action Items)

The following configurations need to be created:

1. **tsconfig.json** - TypeScript compiler configuration
   ```json
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "commonjs",
       "lib": ["ES2020"],
       "outDir": "./dist",
       "rootDir": "./src",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true,
       "resolveJsonModule": true
     },
     "include": ["src/**/*"],
     "exclude": ["node_modules", "dist"]
   }
   ```

2. **Jest configuration** - Testing setup
3. **ESLint/Prettier** - Code linting and formatting
4. **Dockerfile** - Container configuration
5. **CI/CD pipeline** - GitHub Actions or similar

## Git Workflow

- **Main branch**: Production-ready code
- **Feature branches**: Use `feature/` or `claude/` prefix
- **Commit messages**: Use conventional commits (feat:, fix:, docs:, etc.)

## Security Considerations

- Never commit `.env` files or secrets
- Validate and sanitize all user inputs
- Use HTTPS in production
- Implement rate limiting on API endpoints
- Store OAuth tokens securely
- Follow OWASP security guidelines

## Dependencies Summary

### Runtime Dependencies
| Package | Version | Purpose |
|---------|---------|---------|
| express | ^4.18.2 | Web framework |
| cors | ^2.8.5 | CORS middleware |
| dotenv | ^16.0.3 | Environment config |
| spotify-web-api-node | ^5.0.2 | Spotify API client |

### Dev Dependencies
| Package | Version | Purpose |
|---------|---------|---------|
| typescript | ^5.0.4 | TypeScript compiler |
| ts-node-dev | ^2.0.0 | Dev server with hot-reload |
| @types/node | ^18.15.11 | Node.js type definitions |
| @types/express | ^4.17.17 | Express type definitions |
| @types/cors | ^2.8.13 | CORS type definitions |

## Quick Reference

```bash
# Development
npm install          # Install dependencies
npm run dev          # Start dev server (port 3000)

# Production
npm run build        # Compile TypeScript
npm start            # Run compiled code

# Testing
npm test             # Run tests (needs Jest setup)
```

---

*Last updated: January 2025*
*Project status: Pre-MVP scaffolding phase*
