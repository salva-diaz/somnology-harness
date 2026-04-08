---
name: context7-mcp-usage
description: Guide for using Context7 MCP tools to fetch up-to-date library documentation. Load this before implementing code that uses any external library (Sequelize, express-validator, etc.) to avoid hallucinated APIs and outdated patterns.
---

# Context7 MCP Usage Guide

## What I do

- Explain which Context7 tools to use and when
- Prevent hallucinated API signatures by fetching live library docs
- Guide agents to look up real, version-specific documentation before writing code

---

## Why Context7

AI models rely on training data, which becomes stale. Context7 fetches current documentation directly from library sources and injects it into context — so agents write code against real APIs, not memorized (possibly wrong) ones.

---

## Tools Reference

### `resolve-library-id`

Converts a human library name into a Context7-compatible library ID.

**Use this first** — always resolve before fetching docs.

| Parameter | Required | Notes |
|-----------|----------|-------|
| `libraryName` | Yes | Library name (e.g., `"sequelize"`, `"express-validator"`, `"kafkajs"`) |
| `query` | Yes | What you're trying to do (e.g., `"model associations"`) |

**Example call:** `resolve-library-id({ libraryName: "sequelize", query: "model associations hasMany belongsTo" })`

---

### `get-library-docs`

Fetches relevant documentation for a specific library ID and topic.

| Parameter | Required | Notes |
|-----------|----------|-------|
| `libraryId` | Yes | ID returned by `resolve-library-id` |
| `query` | Yes | Specific topic or API you need (e.g., `"findAll with include eager loading"`) |
| `tokens` | No | Max tokens to return. Omit for default. |

**Example call:** `get-library-docs({ libraryId: "/sequelize/sequelize", query: "findAll with include eager loading" })`

---

## Standard Workflow

Always follow this two-step sequence:

```
1. resolve-library-id  →  get the Context7 library ID
2. get-library-docs    →  fetch the relevant documentation
```

Then use that documentation to guide your implementation.

---

## Common Libraries

### Backend (Express / Node.js)

| Library | `libraryName` value | When to look up |
|---------|---------------------|-----------------|
| Sequelize | `sequelize` | Models, associations, queries, migrations, transactions |
| Express | `express` | Router, middleware, error handling |
| express-validator | `express-validator` | Validation chains, custom validators, sanitization |
| Winston | `winston` | Logger configuration, transports |
| KafkaJS | `kafkajs` | Producer, consumer, admin client |
| AWS SDK v3 | `aws-sdk` | Cognito, S3, SQS, Lambda, SES clients |
| Jest | `jest` | Testing syntax, mocking, assertions |
| Supertest | `supertest` | HTTP testing with Express apps |
| jsonwebtoken | `jsonwebtoken` | JWT sign, verify, decode |
| jwks-rsa | `jwks-rsa` | JWKS client for Cognito JWT verification |

---

## When to Use Context7

**Always consult Context7 before:**
- Writing code that calls an external library's API
- Implementing patterns you're less than 100% certain about
- Using a library version that may have breaking changes
- Working with Sequelize associations, scopes, or hooks

**Skip Context7 for:**
- Plain TypeScript/JavaScript syntax (no library involved)
- Internal project code (use explorer context files instead)
- HTML/CSS only work

---

## Common Pitfalls

1. **Never call `get-library-docs` without resolving the ID first.** The `libraryId` must be an exact Context7 ID, not a guessed name.
2. **Use specific queries, not generic ones.** `"hasMany vs belongsTo"` is better than `"associations"`.
3. **One topic per call.** Fetch docs for one specific concept at a time for best results.
4. **The returned docs are authoritative.** If docs conflict with your training data, trust the docs.

---

## Example: Express Implementer

```
Goal: Implement a Sequelize model with associations and eager loading.

Step 1: resolve-library-id({ libraryName: "sequelize", query: "model associations hasMany belongsTo" })
→ returns libraryId: "/sequelize/sequelize"

Step 2: get-library-docs({ libraryId: "/sequelize/sequelize", query: "hasMany belongsTo include eager loading" })
→ returns current Sequelize docs with exact method signatures and usage examples

Step 3: Implement using the documented API (not training data)
```

## Example: Validation

```
Goal: Add express-validator chain with custom async validator.

Step 1: resolve-library-id({ libraryName: "express-validator", query: "custom async validator" })
→ returns libraryId

Step 2: get-library-docs({ libraryId: "...", query: "custom validator async withMessage" })
→ returns current express-validator docs

Step 3: Implement using the documented API
```
