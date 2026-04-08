# Error Handling Guidelines

## Current Patterns (follow these for existing code)

The codebase uses `SomnoRESTError` (not `AppError`) defined in `src/errors/baseError.js`.

```js
class SomnoRESTError extends Error {
  constructor(errorName, message, httpStatusCode, validator, endpointMethod, endpointPath) {
    super(message);
    this.errorName = errorName;
    this.httpStatusCode = httpStatusCode;
    // ...
  }
  toObject() {
    return {
      statusCode: this.httpStatusCode,
      errorName: this.errorName,
      errorMessage: this.message,
      context: `${this.endpointMethod} ${this.endpointPath}`
    };
  }
}
```

Domain-specific errors extend this class and live in `src/errors/` or colocated as `{domain}.errors.js` in `src/api/`.

**Centralized handler** (bottom of `app.js`):

```js
app.use((error, req, res, next) => {
  if (error instanceof SomnoRESTError) {
    logger.error(error.toString(), { additionalInfo: { stack: error.stack } });
    return res.status(error.httpStatusCode || 500).json(error.toObject());
  } else {
    logger.error(error.message, { additionalInfo: { stack: error.stack } });
    return res.status(error.errorCode || 500).json({ msg: error.message });
  }
});
```

**Rule:** New error classes extend `SomnoRESTError`, not `AppError`.

---

## Target Patterns (for greenfield modules)

## Principles

- Use custom error classes — never throw raw `Error` objects.
- All errors flow through the centralized `errorHandler` middleware.
- Return consistent JSON error shapes across the entire API.

## Error Classes

```typescript
// src/errors.ts

export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public code: string = 'INTERNAL_ERROR'
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, `${resource} not found`, 'NOT_FOUND');
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message, 'UNAUTHORIZED');
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, message, 'FORBIDDEN');
  }
}

export class ConflictError extends AppError {
  constructor(resource: string) {
    super(409, `${resource} already exists`, 'CONFLICT');
  }
}
```

## Error Handler Middleware

Register this as the last middleware in the Express app.

```typescript
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors';

export const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction): void => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: err.code,
      message: err.message,
    });
    return;
  }

  console.error(err);
  res.status(500).json({
    error: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred',
  });
};
```

## App Wiring

```typescript
import { errorHandler } from './middleware/errorHandler';

// ... all routes ...
app.use(errorHandler); // must be last
```

## Rules

- Throw `AppError` subclasses from actions — never from controllers directly unless it's a simple 404 check.
- Never expose stack traces or internal error details in responses.
- Log unexpected errors (non-`AppError`) with enough context to debug; suppress logging for known operational errors.
- `next(error)` is the only way to reach `errorHandler` from async handlers — never swallow errors silently.
