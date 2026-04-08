# Request Validation Guidelines

## Current Patterns (follow these for existing code)

The codebase uses **express-validator** (not Zod) with a custom `MiddleWares` class container.

**Middleware file example** (`src/api/auth/auth.middleware.js`):

```js
const { body } = require('express-validator');
const { MiddleWares } = require('../../middlewares');
const { validateInput } = require('../../utils/validation');

const middlewares = MiddleWares();
middlewares.addMany('POST', '/signIn',
  { name: 'validateEmail', fn: body('email').isEmail().toLowerCase().trim() },
  { name: 'validatePassword', fn: body('password').isString() },
  validateInput
);
module.exports = middlewares;
```

**Rule:** New validation uses this `MiddleWares` + express-validator pattern.

---

## Target Patterns (for greenfield modules)

## Principles

- Validate all incoming requests before they reach controllers.
- Use Zod for schema definition — it provides TypeScript inference and runtime validation in one.
- The `validateRequest` middleware is the single validation entry point; never validate manually inside controllers.

## validateRequest Middleware

```typescript
import { z } from 'zod';
import { Request, Response, NextFunction } from 'express';

export const validateRequest = (schema: z.ZodSchema) => {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      schema.parse({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        res.status(400).json({
          error: 'Validation failed',
          details: error.errors,
        });
        return;
      }
      next(error);
    }
  };
};
```

## Schema Definition

Place schemas in a `<resource>.validators.ts` file next to the router.

```typescript
import { z } from 'zod';

export const createUserSchema = z.object({
  body: z.object({
    name: z.string().min(1),
    email: z.string().email(),
    password: z.string().min(8),
  }),
});

export const updateUserSchema = z.object({
  body: z.object({
    name: z.string().min(1).optional(),
    email: z.string().email().optional(),
  }),
  params: z.object({
    id: z.string().uuid(),
  }),
});

export const listUsersSchema = z.object({
  query: z.object({
    page: z.coerce.number().int().positive().optional(),
    limit: z.coerce.number().int().positive().max(100).optional(),
  }),
});
```

## Rules

- Wrap `body`, `query`, and `params` in the top-level schema object — this matches how `validateRequest` passes data.
- Use `z.coerce` for query params (they arrive as strings).
- Export inferred types for use in actions: `type CreateUserInput = z.infer<typeof createUserSchema>['body']`.
- Never use `any` or skip validation on "trusted" internal routes that still accept external input.
