# CLAUDE.md — AI Assistant Guide for hhh-website-server

## Project Overview

A modern Express + TypeScript REST API server boilerplate with structured layered architecture, OpenAPI documentation auto-generated from Zod schemas, and a comprehensive testing/quality pipeline.

**Tech Stack:** Node.js 22.8.0, Express 4, TypeScript 5, Zod, Vitest, Biome

---

## Repository Structure

```
src/
├── index.ts                  # Entry point (starts server)
├── server.ts                 # Express app factory (middleware stack)
├── api/
│   ├── healthCheck/          # Health check endpoint module
│   └── user/                 # User API module (template for new modules)
│       ├── userModel.ts      # Zod schemas
│       ├── userController.ts # Request handlers
│       ├── userService.ts    # Business logic
│       ├── userRepository.ts # Data access (currently in-memory)
│       ├── userRouter.ts     # Express routes + OpenAPI registration
│       └── __tests__/
├── common/
│   ├── middleware/           # errorHandler, rateLimiter, requestLogger
│   ├── models/               # ServiceResponse<T> generic wrapper
│   └── utils/                # commonValidation, envConfig, httpHandlers
└── api-docs/                 # OpenAPI/Swagger generation + UI route
```

---

## Development Commands

```bash
npm run dev      # Start with tsx watch + pino-pretty (hot reload)
npm run build    # Bundle with tsup → dist/
npm run start    # Run dist/index.js (production)
npm run test     # Run all tests with Vitest
npm run lint     # Biome lint check
npm run lint:fix # Biome lint + auto-fix
npm run format   # Biome format check
```

> **Node version:** 22.8.0 (use `nvm use` to apply `.nvmrc`)

---

## Architecture: Layered Pattern

```
HTTP Request
    → Router (validates request via Zod middleware)
    → Controller (extracts data, calls service)
    → Service (business logic, error handling)
    → Repository (data access)
    → ServiceResponse<T> (standardized response wrapper)
    → HTTP Response
```

### Adding a New API Module

Follow the user module as a template. Create a directory under `src/api/<feature>/` with:

1. **`<feature>Model.ts`** — Zod schemas + inferred TypeScript types
2. **`<feature>Repository.ts`** — Data access with async interface
3. **`<feature>Service.ts`** — Business logic, wraps results in `ServiceResponse`
4. **`<feature>Controller.ts`** — Calls service, calls `handleServiceResponse()`
5. **`<feature>Router.ts`** — Express Router, registers routes with OpenAPI registry
6. **`__tests__/<feature>Service.test.ts`** — Unit tests (mock repository)
7. **`__tests__/<feature>Router.test.ts`** — Integration tests (Supertest)

Register the new router in `src/server.ts`.

---

## Response Format

All endpoints must use `ServiceResponse<T>` from `src/common/models/serviceResponse.ts`:

```typescript
{
  success: boolean;
  message: string;
  responseObject: T | null;
  statusCode: number;
}
```

Use factory methods:
- `ServiceResponse.success("message", data, StatusCodes.OK)`
- `ServiceResponse.failure("error message", StatusCodes.NOT_FOUND)`

Use `handleServiceResponse(serviceResponse, res)` from `src/common/utils/httpHandlers.ts` in controllers to send the response.

---

## Validation Pattern (Zod + OpenAPI)

1. Define base schema in `*Model.ts`:
   ```typescript
   export const UserSchema = z.object({ id: z.number(), name: z.string(), age: z.number(), email: z.string().email(), createdAt: z.date(), updatedAt: z.date() });
   ```

2. Define request schemas:
   ```typescript
   export const GetUserSchema = z.object({ params: z.object({ id: commonValidations.id }) });
   ```

3. Register with OpenAPI registry in `*Router.ts` using `@asteasolutions/zod-to-openapi`

4. Apply `validateRequest(GetUserSchema)` middleware in routes — this validates params/query/body at runtime.

---

## Environment Configuration

Copy `.env.template` to `.env` and fill in values. Variables are validated at startup via `envalid` in `src/common/utils/envConfig.ts`.

| Variable | Default | Purpose |
|---|---|---|
| `NODE_ENV` | `development` | Run mode (`development`, `production`, `test`) |
| `PORT` | `8080` | Server port |
| `HOST` | `localhost` | Server hostname |
| `CORS_ORIGIN` | `http://localhost:*` | Allowed CORS origins |
| `COMMON_RATE_LIMIT_MAX_REQUESTS` | `20` | Max requests per window |
| `COMMON_RATE_LIMIT_WINDOW_MS` | `1000` | Rate limit window in ms |

---

## Testing

**Framework:** Vitest 2 with Supertest for HTTP integration tests.

```bash
npm run test                    # Run once
npm run test -- --watch         # Watch mode
npm run test -- --coverage      # Coverage report
```

**Patterns:**
- Unit tests (`*Service.test.ts`): mock repository with `vi.mock()`
- Integration tests (`*Router.test.ts`): use `supertest(app)` for full HTTP round-trips
- Use Arrange-Act-Assert structure
- Mock data lives in test files

**Pre-push hook** automatically runs `npm run build && npm run test` — fix failures before pushing.

---

## Code Quality

**Linter/Formatter:** Biome (replaces ESLint + Prettier)
- Line width: 120 characters
- Import organization: enabled (alphabetical)
- **Pre-commit hook** runs `lint-staged` which auto-fixes staged files with Biome

**TypeScript:** Strict mode enabled. Path alias `@/*` maps to `src/*`.

**Do not** use `eslint`, `prettier`, or `jest` — this project uses Biome and Vitest exclusively.

---

## API Documentation

Swagger UI is served at `/` (root). JSON spec at `/swagger.json`.

Documentation is **auto-generated** from Zod schemas — never write OpenAPI YAML manually. Register schemas in routers using the OpenAPI registry pattern from `src/api-docs/openAPIDocumentGenerator.ts`.

---

## Security Conventions

- **Helmet** sets security HTTP headers (configured in `server.ts`)
- **CORS** origin controlled via `CORS_ORIGIN` env var
- **Rate limiting** via `express-rate-limit` middleware
- **Trust proxy** enabled for reverse proxy environments
- Validate all input with Zod before processing

---

## Logging

Uses **Pino** for structured JSON logging. In development, `pino-pretty` formats output for readability.

```typescript
import { logger } from "@/server";
logger.info("message");
logger.error(err, "error message");
```

Request logging (method, url, status, duration) is handled automatically by the `requestLogger` middleware. Logging is disabled in test environments.

---

## Docker

```bash
docker build -t hhh-website-server .
docker run -p 8080:8080 hhh-website-server
```

The Dockerfile uses `node:22.8.0-slim`, runs `npm ci`, builds with tsup, then starts with `npm run start`. Exposes port 8080.

---

## CI/CD (GitHub Actions)

On push/PR to `master`:
- **build.yml** — TypeScript build check
- **test.yml** — Run test suite
- **code-quality.yml** — Biome lint check
- **docker-image.yml** — Build and push Docker image to GHCR (on push to master only)

**Renovate** auto-merges non-major dependency updates and groups related packages.

---

## Key Conventions Summary

| Concern | Convention |
|---|---|
| File naming | camelCase (`userService.ts`) |
| Class/Type naming | PascalCase (`UserService`, `UserSchema`) |
| Schema naming | `<Name>Schema` suffix |
| Singletons | Export instantiated service/controller/repo from module |
| Async | All repository methods are `async` (even if in-memory) |
| Error handling | Try-catch in service, `ServiceResponse.failure()` on error |
| Imports | Use `@/` path alias for `src/` imports |
| Tests location | `__tests__/` subdirectory within feature module |
