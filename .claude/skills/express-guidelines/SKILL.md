---
name: express-guidelines
description: Comprehensive guidelines for implementing Express API routes, controllers, actions, models, and tests. Use when implementing or reviewing Express code
---

## Reference Guide

| Topic | File | When to Read |
|-------|------|--------------|
| Routes | [references/routes.md](references/routes.md) | When defining or reviewing router files and route-level middleware ordering |
| Controllers | [references/controllers.md](references/controllers.md) | When writing handler functions that translate HTTP ↔ actions |
| Validation | [references/validation.md](references/validation.md) | When defining Zod schemas or the `validateRequest` middleware |
| Error Handling | [references/error-handling.md](references/error-handling.md) | When throwing errors, creating error classes, or wiring `errorHandler` |
| Testing | [references/testing.md](references/testing.md) | When writing integration tests with supertest or unit tests for middleware/actions |