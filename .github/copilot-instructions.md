# GitHub Copilot Custom Instructions for Motion MCP Server

This document provides guidelines for GitHub Copilot when working on the Motion MCP Server codebase.

## Project Overview

This is an **unofficial** Model Context Protocol (MCP) server for Motion (usemotion.com), enabling AI assistants like Claude to interact with Motion's task management and calendar features. The project is built with TypeScript and uses the MCP SDK to expose Motion's API as tools.

## Code Style & Standards

### TypeScript

- **Strict mode enabled**: All TypeScript strict checks are enforced
- **Target**: ES2022 with NodeNext module resolution
- **No `any` types**: Use proper type definitions. If unavoidable, add a comment explaining why
- **No unused variables/parameters**: Prefix unused parameters with `_` if required by interface
- **Explicit return types**: While not enforced, prefer explicit return types for public APIs
- **Type imports**: Use `.js` extensions in import paths (TypeScript requirement for NodeNext)

### Code Organization

```
src/
├── api/client.ts       # Single Motion API client with rate limiting
├── tools/              # MCP tool implementations (one file per category)
├── types/              # TypeScript type definitions
├── config.ts           # Configuration management
└── index.ts            # Main server entry point
```

### Formatting (Prettier)

- **Line width**: 100 characters
- **Indentation**: 2 spaces (no tabs)
- **Quotes**: Single quotes
- **Semicolons**: Always required
- **Trailing commas**: ES5 style

### Linting (ESLint)

- `@typescript-eslint/no-explicit-any`: warn (work to eliminate)
- `@typescript-eslint/no-unused-vars`: error (except `^_` pattern)
- `no-console`: warn (only allow `console.warn` and `console.error`)

## Architecture Patterns

### Tool Registration

Each tool category (task, project, workspace, etc.) should:
1. Export a `register*Tools(client: MotionApiClient): Tool[]` function
2. Return an array of tool definitions with `name`, `description`, `inputSchema`, and `handler`
3. Use Zod schemas for runtime input validation in handlers
4. Follow naming convention: `motion_<action>_<entity>` (e.g., `motion_create_task`)

Example:
```typescript
export function registerTaskTools(client: MotionApiClient): Tool[] {
  return [
    {
      name: 'motion_create_task',
      description: 'Create a new task in Motion',
      inputSchema: {
        type: 'object',
        properties: { /* JSON Schema */ },
        required: ['name'],
      },
      handler: async (args: unknown) => {
        const schema = z.object({ /* Zod schema */ });
        const validated = schema.parse(args);
        return await client.createTask(validated);
      },
    },
  ];
}
```

### API Client

- **Single client instance**: `MotionApiClient` handles all API communication
- **Rate limiting**: Uses `p-queue` for automatic rate limiting (12 req/min individual, 120 req/min team)
- **Error handling**: Axios interceptor translates HTTP errors to user-friendly messages
- **Type safety**: All API methods should have properly typed parameters and return values

### Error Handling

- Translate Motion API errors to user-friendly messages
- Include rate limit information in 429 responses
- Handle 401/403 with clear authentication guidance
- Use `McpError` with appropriate `ErrorCode` for MCP-level errors

## Development Workflow

### Before Committing

Always run:
```bash
npm run typecheck  # TypeScript type checking
npm run lint       # ESLint
npm run format     # Prettier formatting
npm run build      # Verify production build works
```

### Testing

- Currently no test infrastructure exists (this is a known gap)
- When adding tests, use descriptive names and test edge cases
- Mock external Motion API calls
- Test rate limiting behavior

## Dependencies

### Production Dependencies
- `@modelcontextprotocol/sdk`: MCP protocol implementation
- `axios`: HTTP client for Motion API
- `dotenv`: Environment variable management
- `p-queue`: Rate limiting queue
- `zod`: Runtime type validation

### Development Dependencies
- TypeScript 5.7+
- ESLint with TypeScript plugin
- Prettier for formatting
- `tsx` for development mode with hot reload

## Motion API Specifics

### Authentication
- Uses API key in `X-API-Key` header
- API key stored in `MOTION_API_KEY` environment variable
- Base URL: `https://api.usemotion.com/v1` (configurable)

### Rate Limits
- **Individual accounts**: 12 requests/minute
- **Team accounts**: 120 requests/minute
- Configured via `MOTION_RATE_LIMIT_PER_MINUTE` environment variable

### Common Entities
- **Tasks**: Core entity with auto-scheduling, deadlines, priorities
- **Projects**: Task containers with custom statuses
- **Workspaces**: Top-level organization units
- **Users**: Team members for assignments
- **Comments**: Task discussions
- **Custom Fields**: Extensible task/project metadata
- **Recurring Tasks**: Repeating task patterns

## Documentation

### When to Update Docs

- **README.md**: User-facing changes, new tools, setup instructions
- **CONTRIBUTING.md**: Development workflow changes
- **docs/MOTION_API_REFERENCE.md**: Motion API changes or clarifications
- **Inline comments**: Complex algorithms, workarounds, or non-obvious logic

### Comment Style

- Use JSDoc for public APIs (functions, classes, types)
- Keep comments concise and explain "why" not "what"
- Document any Motion API quirks or limitations

## Common Tasks

### Adding a New Tool

1. Identify the tool category (or create new file in `src/tools/`)
2. Add the tool definition to the appropriate `register*Tools` function
3. Implement the API client method if needed in `src/api/client.ts`
4. Update type definitions in `src/types/motion.ts` if needed
5. Test manually with development mode: `npm run dev`
6. Update README.md to list the new tool

### Adding a New Tool Category

1. Create new file: `src/tools/<category>.ts`
2. Export `register<Category>Tools(client: MotionApiClient): Tool[]`
3. Import and register in `src/index.ts`
4. Add API client methods in `src/api/client.ts`
5. Update README.md with new category and tool count

## Best Practices

1. **Minimize `any` usage**: Work to eliminate existing `any` types when touching related code
2. **Validate inputs**: Always use Zod schemas for runtime validation in tool handlers
3. **Handle errors gracefully**: Provide clear, actionable error messages
4. **Respect rate limits**: Trust the queue implementation, don't bypass it
5. **Keep tools focused**: Each tool should do one thing well
6. **Document API assumptions**: Motion's API may have undocumented behavior
7. **Use semantic versioning**: Follow semver for version bumps
8. **Small, focused commits**: Use conventional commit messages (feat:, fix:, docs:, etc.)

## Helpful Context

- This is a **community project**, not official Motion software
- Used in production by Claude Desktop users for Motion integration
- Published to npm as `@rf-d/motion-mcp`
- Must maintain backward compatibility with MCP SDK 1.0.4+
- Node.js >= 20.0.0 is required for modern JavaScript features
