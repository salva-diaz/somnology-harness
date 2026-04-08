# Testing Guidelines

## Current Patterns (follow these for existing code)

Tests use **Jest** with tests in `jest/unit/` and `jest/integration/`. The project uses **PostgreSQL** via Docker (not in-memory SQLite).

```bash
# Run specific test
cd somnorest && npx jest --detectOpenHandles --forceExit jest/unit/path/to/test.unit.test.js

# Run all unit tests
npm run test:unit

# Run all tests
npm test
```

**Actual test example** (`jest/unit/middleware/authMiddleware.unit.test.js`):

```js
const { ensureApiToken } = require('@middlewares/authMiddleware');
const models = require('@models');

jest.mock('@models', () => ({ APIClient: { findOne: jest.fn() } }));

describe('Auth middleware test suite', () => {
  it('should pass with valid API key', async () => {
    models.APIClient.findOne.mockResolvedValue({
      MinSupportedMajorVersion: 1,
      APIKeyHash: 'hashed'
    });
    await ensureApiToken(req, res, next);
    expect(next).toHaveBeenCalled();
  });
});
```

**Path aliases:** Tests use `@middlewares`, `@models`, etc. via `tsconfig-paths`.

---

## Target Patterns (for greenfield modules)

## Principles

- Use Jest with supertest for integration tests — these are the primary test type.
- Use unit tests for edge cases, error handling, and pure business logic in actions.
- Mock external dependencies (email, third-party APIs, etc.) but test against a real database using an in-memory SQLite instance.
- Test middleware in isolation with unit tests.

## Integration Test Template (supertest)

```typescript
import request from 'supertest';
import createApp from '../app';

describe('Users API', () => {
  const app = createApp();

  it('should create a user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'John', email: 'john@example.com', password: 'password123' })
      .expect(201);

    expect(response.body).toHaveProperty('id');
    expect(response.body.name).toBe('John');
  });

  it('should return 400 for invalid input', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: '', email: 'not-an-email' })
      .expect(400);

    expect(response.body.error).toBe('Validation failed');
  });

  it('should return 404 for unknown user', async () => {
    const response = await request(app)
      .get('/api/users/non-existent-id')
      .expect(404);

    expect(response.body.error).toBe('NOT_FOUND');
  });
});
```

## Unit Test Template (actions / middleware)

```typescript
import { validateRequest } from '../middleware/validateRequest';
import { createUserSchema } from '../users/users.validators';
import { getMockReq, getMockRes } from '@jest-mock/express';

describe('validateRequest middleware', () => {
  const middleware = validateRequest(createUserSchema);

  it('should call next() for valid input', async () => {
    const req = getMockReq({ body: { name: 'John', email: 'john@example.com', password: 'password123' } });
    const { res, next } = getMockRes();

    await middleware(req, res, next);

    expect(next).toHaveBeenCalledWith();
  });

  it('should return 400 for invalid input', async () => {
    const req = getMockReq({ body: { name: '' } });
    const { res, next } = getMockRes();

    await middleware(req, res, next);

    expect(res.status).toHaveBeenCalledWith(400);
    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ error: 'Validation failed' }));
  });
});
```

## In-Memory SQLite Setup

```typescript
// tests/setup.ts
import Database from 'better-sqlite3';
import { runMigrations } from '../db/migrations';

let db: Database.Database;

beforeAll(() => {
  db = new Database(':memory:');
  runMigrations(db);
});

afterEach(() => {
  db.exec('DELETE FROM users'); // reset between tests
});

afterAll(() => {
  db.close();
});

export { db };
```

## Rules

- Use supertest for happy-path API tests — they cover the full middleware stack.
- Use unit tests for edge cases, error branches, and pure logic.
- Never mock the database — use `:memory:` SQLite for real query execution.
- Mock only external services (email providers, payment APIs, third-party HTTP calls).
- Each test must be independent — reset DB state in `afterEach`, not `afterAll`.
- Test file naming: `<resource>.test.ts` colocated with the resource, or `tests/<resource>.test.ts`.
