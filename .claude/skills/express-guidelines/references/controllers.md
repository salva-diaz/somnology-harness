# Controllers Guidelines

## Principles

- Controllers are thin — they translate HTTP requests into action calls and HTTP responses.
- All business logic lives in actions (service layer), not controllers.
- Controllers must not throw raw `Error` objects; wrap unexpected failures in `AppError` or let the action layer handle it.

## Controller Template

```typescript
import { Request, Response, NextFunction } from 'express';
import * as actions from './users.actions';
import { NotFoundError } from '../errors';

export const listUsers = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
  try {
    const users = await actions.listUsers();
    res.json(users);
  } catch (error) {
    next(error);
  }
};

export const getUser = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
  try {
    const user = await actions.getUser(req.params.id);
    if (!user) throw new NotFoundError('User');
    res.json(user);
  } catch (error) {
    next(error);
  }
};

export const createUser = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
  try {
    const user = await actions.createUser(req.body);
    res.status(201).json(user);
  } catch (error) {
    next(error);
  }
};

export const updateUser = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
  try {
    const user = await actions.updateUser(req.params.id, req.body);
    res.json(user);
  } catch (error) {
    next(error);
  }
};

export const deleteUser = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
  try {
    await actions.deleteUser(req.params.id);
    res.status(204).send();
  } catch (error) {
    next(error);
  }
};
```

## Rules

- Always call `next(error)` in catch blocks — never `res.status(500)` manually.
- Return `void` from async handlers and declare the return type explicitly.
- Never access `req.body` after the validation middleware — trust the schema has coerced types.
- Keep controllers free of `if/else` business rules; those belong in actions.
