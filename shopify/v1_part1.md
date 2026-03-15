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
- **ONLY modify:** the REST call implementation **in-place** (replace REST code with GraphQL code in the same file), add `gid.go`, `graphql_client.go` helpers if needed.
- **No separate `_graphql.go` files.** Convert directly in the existing file — replace the REST call with GraphQL, keep the same function signature, same return types, same error behavior.
- **Tests in EXTERNAL directory** (`$TEST_DIR` — configure path before starting, e.g. `~/Desktop/shopify/test/unittest/`), never create `_test.go` in the main repo. See `v1_part3.md` Section 6.
- Convert 1 service at a time.
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

For each function being converted, follow this exact sequence:

1. **Scan** — Find REST usage (URL `/admin/api/`, SDK `go-shopify`, header `X-Shopify-Access-Token`).
2. **Report** — Inventory: File:Line, Package, Endpoint, Method, Convertible?.
3. **Deep Review REST** (mandatory) — Read the function line by line, document signature, trace all callers, map every error path, record REST response shape, create "Must-Match Behaviors" checklist. Do NOT write any GraphQL code until this is complete. See `v1_part2.md` for mapping tables.
4. **Test REST first** — Call the real Shopify API using the existing REST code. Record the input and output for each function. This is the baseline.
5. **Convert in-place** — Replace REST call with GraphQL call **in the same file**. Keep the same function signature, same return types, same error behavior. Use GID helpers + response mappers. Refer to `v1_part2.md` for mapping.
6. **Test GraphQL** — Call the real Shopify API using the newly converted GraphQL code, with the **same input** as step 4.
7. **Compare REST vs GraphQL output:**
   - **Output matches** → conversion is correct. Proceed to step 8.
   - **Output differs or errors** → DO NOT proceed. Search docs, deep-think, cross-reference REST code vs GraphQL code vs official Shopify docs (https://shopify.dev/docs/api/admin-graphql/latest). Fix the issue. Go back to step 6.
8. **Review & Refactor** — ONLY review the code you just changed. Check: unused vars, unchecked errors, over-fetching, query cost, missing `userErrors` check. Verify `git diff --stat` — only the converted file + helpers should appear.
9. **Report** — Generate professional HTML dashboard with parity results. See `v1_part3.md` Section 9.
10. **Verify** — Main repo `go test ./...` no breakage, `go vet` clean.

### Step 7 — The Comparison Loop (critical)

This is the most important step. When REST and GraphQL outputs don't match:

```
┌─────────────────────────────────────────────────────┐
│               COMPARISON LOOP                       │
│                                                     │
│  REST output ←──compare──→ GraphQL output           │
│       │                         │                   │
│       │    Match? ─── YES ───→ DONE (step 8)        │
│       │      │                                      │
│       │     NO                                      │
│       │      │                                      │
│       ▼      ▼                                      │
│  1. Identify which fields differ                    │
│  2. Read REST code: what does it send/receive?      │
│  3. Read GraphQL code: what does it send/receive?   │
│  4. Fetch official Shopify GraphQL docs for this     │
│     specific mutation/query                         │
│  5. Cross-reference: field names? GID format?       │
│     Input type? Response path? Error mapping?       │
│  6. Fix the GraphQL code                            │
│  7. Re-run test → back to compare                   │
│                                                     │
│  DO NOT guess. DO NOT skip fields.                  │
│  Every field must match before proceeding.          │
└─────────────────────────────────────────────────────┘
```

### Parallel Strategy

When converting multiple services, spawn parallel agents — each handles 1 service independently:

```
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│ Agent: ProductService   │  │ Agent: OrderService      │  │ Agent: CustomerSvc      │
│ 1. Deep Review REST     │  │ 1. Deep Review REST      │  │ 1. Deep Review REST     │
│ 2. Test REST (baseline) │  │ 2. Test REST (baseline)  │  │ 2. Test REST (baseline) │
│ 3. Convert in-place     │  │ 3. Convert in-place      │  │ 3. Convert in-place     │
│ 4. Test GraphQL         │  │ 4. Test GraphQL          │  │ 4. Test GraphQL         │
│ 5. Compare → fix loop   │  │ 5. Compare → fix loop    │  │ 5. Compare → fix loop   │
│ 6. Review changed code  │  │ 6. Review changed code   │  │ 6. Review changed code  │
└────────────┬────────────┘  └────────────┬─────────────┘  └────────────┬─────────────┘
             └────────────────────────────┼─────────────────────────────┘
                                          ▼
                        Merge → Run all tests → Generate HTML report
                        → Confirm with user
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

- **`v1_part2.md`** — REST→GraphQL endpoint mapping tables (10+ common resources + 15 additional resources + step-by-step process for handling ANY resource not listed), field name mapping, 16 gotchas & traps, Go implementation patterns (GID helpers, GraphQL client, response mapping, pagination, bulk operations)
- **`v1_part3.md`** — Live API test infrastructure, external test directory setup, 15+ mandatory test cases per service, parity test patterns, HTML report generator, execution checklist
