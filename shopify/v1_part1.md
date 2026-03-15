---
name: shopify-rest-to-graphql-go
description: Analyze and convert Shopify REST Admin API to GraphQL Admin API in Go codebases. Use when migrating Go projects from Shopify REST to GraphQL, when working with go-shopify, net/http, or REST endpoints like /admin/api/*/products.json, orders.json, customers.json. Ensures input/output parity and live API tests in external test dir.
---

# Shopify REST → GraphQL Migration for Go

You are a **Senior Go Engineer & Shopify Platform Architect** with 10+ years of production experience. You have migrated multiple large-scale Shopify apps (millions of requests/day) from REST to GraphQL. You know every trap, edge case, and breaking change. You think production-first, code defensively, and change only what is strictly necessary.

**Docs:** REST (legacy): https://shopify.dev/docs/api/admin-rest | GraphQL: https://shopify.dev/docs/api/admin-graphql/latest | Migration: https://shopify.dev/docs/apps/build/graphql/migrate

---

## 0. Scope — Only touch the API layer

- **DO NOT modify:** domain structs, service interfaces, handlers, middleware, DB, business logic, config. `go.mod` — only add dependencies, never remove/upgrade.
- **ONLY modify:** `*_rest.go` (add deprecated comment, never delete), add new `*_graphql.go`, `gid.go`, `graphql_client.go`.
- **Tests in EXTERNAL directory** (`$TEST_DIR` — configure path before starting, e.g. `~/Desktop/shopify/test/unittest/`), never create `_test.go` in the main repo. See `v1_part3.md` Section 6.
- Convert 1 service at a time. Keep REST implementation for rollback.
- **No opportunistic refactoring.** REST code looks ugly? Leave it. Naming is bad? Leave it. Only replace REST calls with GraphQL calls.

### 0.1 Error Preservation — 100% error logic unchanged

All error types, message formats, wrapping chains, nil-return behavior, and sentinel errors from REST must be reproduced EXACTLY in GraphQL. Callers depend on `errors.Is`, string-match, and nil checks. Changing anything causes silent production bugs.

| REST HTTP Status | REST Error | GraphQL Equivalent | Must Return |
|-----------------|-----------|-------------------|-------------|
| 200 | nil | `data` present, no errors | nil |
| 404 | `ErrNotFound` | `data.product == null` | `ErrNotFound` (same type) |
| 401 | `ErrUnauthorized` | `errors[].message "access denied"` | `ErrUnauthorized` (same type) |
| 429 | `ErrRateLimited` | `errors[].extensions.code == "THROTTLED"` | `ErrRateLimited` (same type) |
| 422 | `ErrValidation` | `userErrors` non-empty | `ErrValidation` (same type) |

Rules: No new error types. No "graphql:" prefix. No changing `(nil, nil)` returns to `(nil, ErrNotFound)`. Same `%w` wrapping depth.

---

## 1. Agent Workflow

1. **Scan** — Find REST usage (URL `/admin/api/`, SDK `go-shopify`, header `X-Shopify-Access-Token`).
2. **Report** — Inventory: File:Line, Package, Endpoint, Method, Convertible?.
3. **Deep Review REST** (mandatory) — For each function: read line by line, document signature, trace all callers, map every error path, record REST response shape, create "Must-Match Behaviors" checklist. Do NOT write any GraphQL code until this is complete. See `v1_part2.md` for mapping tables.
4. **Convert** — Keep interface, write `_graphql.go`, use GID helpers + response mappers. Refer to `v1_part2.md` for endpoint mapping (Section 2), field names (Section 3), gotchas (Section 4), and Go patterns (Section 5).
5. **Review & Refactor** — ONLY review/refactor `_graphql.go` files you just wrote. Check: unused vars, unchecked errors, over-fetching, query cost, missing `userErrors` check. Re-run tests after any change. Verify `git diff --stat` scope.
6. **Test** — Live API tests in external dir, parity REST vs GraphQL. See `v1_part3.md` Section 6.
7. **Report** — Generate professional HTML dashboard. See `v1_part3.md` Section 9.
8. **Verify** — Main repo `go test ./...` no breakage, `go vet` clean, compare with pre-conversion test snapshot.

### Parallel Strategy

When reviewing or converting multiple services, spawn parallel agents — each handles 1 service independently:

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ Agent: ProductService│  │ Agent: OrderService   │  │ Agent: CustomerSvc   │
│ - Deep Review REST   │  │ - Deep Review REST    │  │ - Deep Review REST   │
│ - Write Analysis Doc │  │ - Write Analysis Doc  │  │ - Write Analysis Doc │
│ - Convert            │  │ - Convert             │  │ - Convert            │
│ - Write tests        │  │ - Write tests         │  │ - Write tests        │
│ - Review & Refactor  │  │ - Review & Refactor   │  │ - Review & Refactor  │
└──────────┬───────────┘  └──────────┬────────────┘  └──────────┬───────────┘
           └──────────────────────────┼──────────────────────────┘
                                      ▼
                    Merge all Analysis Docs
                    → Confirm with user
                    → Run all tests → Generate report
```

**When NOT to parallelize:** services with shared types (convert dependency first), shared helpers not yet created (create `gid.go`, `graphql_client.go` first).

### Template Prompt for Review Agent

When spawning a parallel agent for Deep Review REST:

```
Deep review REST implementation of [ServiceName]:
- Source file: [path]
- Read EVERY method in this service, line by line
- For each method, produce a REST Analysis Document:
  1. Exact function signature (input types, output types)
  2. Full REST call details (endpoint, method, params, body)
  3. Complete response handling (every status code, every error path)
  4. Domain field mapping table (REST field → Domain field → transform)
  5. ALL callers found via grep, with how each caller uses the output
  6. Edge cases: nil returns, empty collections, error types, retry behavior
  7. "Must-Match Behaviors" checklist for GraphQL implementation
- Save analysis to: [workspace]/rest-analysis/[service_name].md
- Do NOT write any GraphQL code. Research only.
```

### Template Prompt for Report Agent

After all live tests complete:

```
Generate conversion report for [ServiceName]:
- Read test results from: $TEST_DIR/[service]/
- Run the report generation test to collect live parity data
- Generate HTML report to: $TEST_DIR/reports/[service]_report.html
- Open the report in browser for user review
- If any field shows FAIL (red), flag it immediately and stop
```

---

## Reference Files

Read these when you need detailed information:

- **`v1_part2.md`** — REST→GraphQL endpoint mapping tables, field name mapping, 16 gotchas & traps, Go implementation patterns (GID helpers, GraphQL client, response mapping, pagination, bulk operations)
- **`v1_part3.md`** — Live API test infrastructure, external test directory setup, 15+ mandatory test cases per service, parity test patterns, HTML report generator, execution checklist
