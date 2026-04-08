# Routing Guidelines

## Current Patterns (follow these for existing code)

The codebase has two routing systems:

**Legacy system** (`src/routes/`): Auto-registered by `registerRoutes()` in `app.js`. Uses `createCustomRouter(getValidator)` with Swagger-based validation.

**New system** (`src/api/`): Auto-registered by `registerNewRoutes()` in `app.js`. File convention: `<folderName>.routes.js`. Uses `createCustomRouterWithMiddlewares(middlewares)` with `MiddleWares` class for express-validator chains.

**Rule:** All new routes go in `/src/api/` using the new system. Never add to `/src/routes/`.

**Actual route file example** (`src/api/auth/auth.routes.js`):

```js
const controllers = require("./auth.controllers");
const middlewares = require("./auth.middleware");
const routerUtils = require("../../utils/router-utils");
const router = routerUtils.createCustomRouterWithMiddlewares(middlewares);

router.add("POST", "/signIn", async (req, res, next) => {
  try {
    const response = await controllers.signIn(req.body);
    return res.status(response.statusCode).json(response.resp);
  } catch (error) {
    next(error);
  }
});
module.exports = router;
```

---

## Target Patterns (for greenfield modules)

## Principles

- Organize routes by resource/domain entity — one router file per resource.
- Use `Router` from Express for modular route definitions.
- Apply middleware (auth, validation) at the route level, not globally, so each route declares its own requirements explicitly.
- Order middleware left to right: `authenticate` → `validateRequest` → `controller`.

## File Structure

```
src/
  users/
    users.routes.ts      ← Router definition
    users.controller.ts  ← Handler functions
    users.validators.ts  ← Zod schemas
    users.actions.ts     ← Business logic
```

## Route File Template

```typescript
import { Router } from 'express';
import { authenticate } from '../middleware/auth';
import { validateRequest } from '../middleware/validateRequest';
import { createUserSchema, updateUserSchema } from './users.validators';
import * as controller from './users.controller';

const router = Router();

router.get('/', controller.listUsers);
router.get('/:id', controller.getUser);
router.post('/', validateRequest(createUserSchema), controller.createUser);
router.put('/:id', authenticate, validateRequest(updateUserSchema), controller.updateUser);
router.delete('/:id', authenticate, controller.deleteUser);

export default router;
```

## Mounting Routers in App

```typescript
import usersRouter from './users/users.routes';
import postsRouter from './posts/posts.routes';

app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);
```

## Rules

- Never define business logic inside the router file — delegate to controllers.
- Protect mutating routes (`POST`, `PUT`, `PATCH`, `DELETE`) with `authenticate` unless explicitly public.
- Prefix all API routes with `/api` at the app level, not inside the router.
