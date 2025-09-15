# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Core Development
```bash
# Start development server with Turbopack
npm run dev

# Build for production
npm run build

# Start production server
npm run start

# Lint code
npm run lint
```

### Testing
```bash
# Run API endpoint tests
npm run test:api

# Run code execution tests
npm run test:code

# Run all tests
npm run test:all
```

## Project Architecture

### Overview
Open Lovable is a Next.js application that allows users to chat with AI to build React apps instantly. The application integrates with Firecrawl for web scraping and provides sandboxed code execution using either Vercel Sandboxes or E2B.

### Key Technology Stack
- **Framework**: Next.js 15.4.3 with App Router
- **Language**: TypeScript
- **Styling**: Tailwind CSS with custom theme system
- **UI Components**: Radix UI + custom components
- **State Management**: Jotai
- **AI Integration**: Multiple providers (Anthropic, OpenAI, Google, Groq)
- **Sandbox Execution**: Vercel Sandboxes (default) or E2B
- **Web Scraping**: Firecrawl

### Directory Structure

```
├── app/                          # Next.js App Router
│   ├── api/                     # API routes
│   │   ├── analyze-edit-intent/
│   │   ├── apply-ai-code-stream/
│   │   ├── apply-ai-code/
│   │   ├── check-vite-errors/
│   │   ├── create-ai-sandbox-v2/
│   │   ├── create-ai-sandbox/
│   │   ├── detect-and-install-packages/
│   │   ├── generate-ai-code-stream/
│   │   ├── install-packages-v2/
│   │   ├── kill-sandbox/
│   │   ├── run-command-v2/
│   │   └── ...                  # Various AI and sandbox management endpoints
│   ├── generation/              # Code generation interface
│   ├── page.tsx                 # Main landing page with URL input/search
│   └── layout.tsx               # Root layout
├── components/                  # Shared React components
│   ├── app/                     # App-specific components
│   │   └── (home)/              # Homepage sections
│   ├── shared/                  # Reusable shared components
│   │   ├── header/              # Navigation components
│   │   ├── layout/              # Layout primitives
│   │   └── effects/             # Visual effects
│   └── ui/                      # Base UI components (shadcn-based)
├── config/                      # Configuration files
│   └── app.config.ts            # Central application configuration
├── lib/                         # Utility libraries
│   ├── context-selector.ts      # AI context selection
│   ├── edit-examples.ts         # Code editing examples
│   ├── edit-intent-analyzer.ts  # User intent analysis
│   ├── file-parser.ts          # File parsing utilities
│   └── file-search-executor.ts  # File search functionality
├── atoms/                       # Jotai state atoms
├── hooks/                       # Custom React hooks
├── styles/                      # Global styles and CSS
├── utils/                       # Utility functions
└── types/                       # TypeScript type definitions
```

### Configuration System

The application uses a centralized configuration system in `config/app.config.ts`:

- **Sandbox Configuration**: Settings for Vercel and E2B sandboxes including timeouts, ports, and startup delays
- **AI Model Configuration**: Available models, display names, and API settings
- **Code Application Settings**: Refresh delays, truncation recovery, and package installation behavior
- **UI Configuration**: Animation settings, toast durations, and chat message limits
- **Development Settings**: Debug logging, performance monitoring, and API response logging

### Key Features

#### 1. Multi-Provider AI Integration
- Supports Anthropic, OpenAI, Google, and Groq models
- Configurable via environment variables
- Model-specific settings and display names

#### 2. Dual Sandbox Support
- **Vercel Sandboxes**: Default option, uses Node.js runtime
- **E2B Sandboxes**: Alternative with Vite integration
- Automatic sandbox creation, management, and cleanup

#### 3. Web Scraping & Search
- Firecrawl integration for website scraping
- Built-in search functionality with carousel UI
- Screenshot capture and markdown generation

#### 4. Real-time Code Execution
- Streaming code application
- Package installation and management
- Hot module replacement with Vite
- Error detection and recovery

#### 5. Styling System
- Custom Tailwind configuration with extensive theme support
- CSS variables for dynamic theming
- Responsive design with custom breakpoints
- Animation utilities and transitions

### Environment Variables

Required environment variables for full functionality:

```env
# Core Services
FIRECRAWL_API_KEY=your_firecrawl_api_key

# AI Providers (choose one or more)
ANTHROPIC_API_KEY=your_anthropic_api_key
OPENAI_API_KEY=your_openai_api_key
GEMINI_API_KEY=your_gemini_api_key
GROQ_API_KEY=your_groq_api_key

# Sandbox Provider
SANDBOX_PROVIDER=vercel  # or 'e2b'

# Vercel Sandbox (if using vercel)
VERCEL_OIDC_TOKEN=auto_generated_by_vercel_env_pull

# E2B Sandbox (if using e2b)
E2B_API_KEY=your_e2b_api_key
```

### Code Style and Patterns

#### TypeScript
- Strict mode enabled
- Path aliases configured (`@/*`)
- Comprehensive type definitions in `types/`

#### Component Organization
- Co-located components with their features
- Shared components in `components/shared/`
- UI primitives in `components/ui/`

#### Styling Conventions
- Utility-first approach with Tailwind
- Custom CSS variables for theming
- Responsive design with mobile-first approach
- Animation classes for smooth transitions

#### State Management
- Jotai for global state
- Context providers for component state
- Session storage for persistence

### Development Guidelines

#### API Routes
- All API routes in `app/api/` directory
- Streaming responses for real-time updates
- Error handling with appropriate HTTP status codes

#### Component Development
- Use TypeScript interfaces for props
- Implement proper loading states
- Handle errors gracefully with toast notifications
- Follow accessibility best practices

#### Configuration Changes
- Modify `config/app.config.ts` for application settings
- Use environment variables for sensitive data
- Test configuration changes thoroughly

### Common Development Tasks

#### Adding New AI Models
1. Update `config/app.config.ts` with model details
2. Add display name in `modelDisplayNames`
3. Configure API settings if needed
4. Update environment variables documentation

#### Modifying Sandbox Behavior
1. Adjust timeout settings in sandbox configuration
2. Modify startup delays as needed
3. Update working directory paths
4. Test with both providers if applicable

#### Styling Changes
1. Modify `tailwind.config.ts` for new utilities
2. Update CSS variables in `styles/`
3. Test across different screen sizes
4. Verify dark mode compatibility

### Testing

The project includes automated tests for:
- API endpoint functionality (`test:api`)
- Code execution reliability (`test:code`)

Tests can be run individually or all together using the provided npm scripts.