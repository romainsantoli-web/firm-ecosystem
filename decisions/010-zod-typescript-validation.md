# 010 — Zod for TypeScript Validation (over Pydantic)

- **Status**: Accepted
- **Date**: 2026-03-03
- **Context**: TaskFlow Pro SaaS MVP (TypeScript stack)
- **Decision**: Use Zod for input validation instead of Pydantic
- **Drivers**: Native TypeScript type inference, frontend/backend sharing

## Context

The Firm ecosystem uses Pydantic as the default validation layer (Python stack). When
`firm init --sector saas --stack typescript` generates a project, the validation strategy
must change to a TypeScript-native solution.

## Decision

Use **Zod 3.23** with inferred types:

```typescript
const CreateTaskSchema = z.object({
  title: z.string().min(1).max(500),
  priority: z.enum(['low', 'medium', 'high', 'urgent']).default('medium'),
  tags: z.array(z.string().max(50)).max(10).default([]),
});
type CreateTask = z.infer<typeof CreateTaskSchema>; // auto-generated type
```

Combined with a generic validation middleware:

```typescript
export function validate(schema: ZodSchema, source: 'body' | 'query' | 'params' = 'body') {
  return (req, res, next) => { /* parse + 400 on error */ };
}
```

## Consequences

- **Positive**: Types inferred at compile time — no manual interface maintenance
- **Positive**: Same validation logic works client-side and server-side
- **Positive**: `.regex()`, `.min()`, `.max()` composable like Pydantic validators
- **Negative**: Not 1:1 with Pydantic — the `TOOL_MODELS` pattern doesn't apply
- **Mapping**: `BaseModel` → `z.object()`, `Field(min_length=)` → `.min()`, `@validator` → `.refine()`
