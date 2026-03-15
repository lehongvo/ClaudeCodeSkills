# Part 2: REST → GraphQL Mapping & Go Patterns

Reference file for `shopify-rest-to-graphql-go` skill. Read when performing conversion steps 3-5.

---

## 2. REST → GraphQL Mapping

### Core Principle
REST: many endpoints, HTTP verbs, numeric IDs, flat responses.
GraphQL: single POST endpoint (`/admin/api/{version}/graphql.json`), queries + mutations, GID format, nested `data` responses.

### Products

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /products.json` | `products(first:N)` query | Use `nodes` pattern, not `edges{node}` |
| `GET /products/{id}.json` | `product(id:"gid://shopify/Product/{id}")` query | |
| `GET /products/count.json` | `productsCount` query | |
| `POST /products.json` | `productCreate(product:$input)` mutation | Input type: `ProductCreateInput`, NOT deprecated `ProductInput` |
| `PUT /products/{id}.json` | `productUpdate(product:$input)` mutation | Input type: `ProductUpdateInput!`, NOT deprecated `ProductInput`. Arg name is `product`, NOT `input` |
| `DELETE /products/{id}.json` | `productDelete(input:{id:$id})` mutation | |
| N/A | `productSet(input:$input)` mutation | Unified create/update; omitted list fields = deleted |

### Orders

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /orders.json` | `orders(first:N, query:"...")` query | Filter syntax: `status:open`, `created_at:>2024-01-01` |
| `GET /orders/{id}.json` | `order(id:"gid://shopify/Order/{id}")` query | |
| `PUT /orders/{id}.json` | `orderUpdate(input:$input)` mutation | Only: email, shipping address, tags, notes, metafields |
| `POST /orders/{id}/close.json` | `orderClose(input:$input)` mutation | Input type: `OrderCloseInput!` (contains `id` field) |
| `POST /orders/{id}/cancel.json` | `orderCancel(orderId:$id, reason:$reason, restock:$restock)` mutation | Individual args, NOT input object. `reason` (OrderCancelReason!) and `restock` (Boolean!) are required |
| N/A | `orderEditBegin` / `orderEditCommit` | For line item changes (REST PUT can't do this either) |

### Customers

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /customers.json` | `customers(first:N)` query | |
| `GET /customers/{id}.json` | `customer(id:"gid://shopify/Customer/{id}")` query | |
| `POST /customers.json` | `customerCreate(input:$input)` mutation | Input type: `CustomerInput!` (NOT `CustomerCreateInput`) |
| `PUT /customers/{id}.json` | `customerUpdate(input:$input)` mutation | Input type: `CustomerInput!` (same type as create, include `id` in input) |
| `DELETE /customers/{id}.json` | `customerDelete(input:$input)` mutation | Input type: `CustomerDeleteInput!` (contains `id` field) |

### Inventory

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /inventory_levels.json` | `inventoryItem(id:$id) { inventoryLevels(...) }` | |
| `POST /inventory_levels/adjust.json` | `inventoryAdjustQuantities(input:$input)` mutation | Requires `reason`, `name` ("available"), `changes[]` with `delta` |
| `POST /inventory_levels/set.json` | `inventorySetQuantities(input:$input)` mutation | |

### Fulfillment

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /orders/{id}/fulfillments.json` | `order(id:$id) { fulfillments {...} }` | |
| `POST /fulfillments.json` | `fulfillmentCreate(fulfillment:$input)` mutation | Works on FulfillmentOrder, NOT Order line items directly |
| `PUT /fulfillments/{id}/update_tracking.json` | `fulfillmentTrackingInfoUpdate(...)` mutation | |

### Collections, Metafields, Webhooks

| REST | GraphQL | Notes |
|------|---------|-------|
| `GET /custom_collections.json` | `collections(first:N, query:"collection_type:custom")` | |
| `GET /smart_collections.json` | `collections(first:N, query:"collection_type:smart")` | |
| `GET /{resource}/{id}/metafields.json` | Query resource with `metafields(first:N)` connection | Inline on parent object |
| `POST /{resource}/{id}/metafields.json` | `metafieldsSet(metafields:[$input])` mutation | Unified across all resource types |
| `GET /webhooks.json` | `webhookSubscriptions(first:N)` query | |
| `POST /webhooks.json` | `webhookSubscriptionCreate(...)` mutation | |

---

## 3. Field Name Mapping (Critical Differences)

REST uses `snake_case`, GraphQL uses `camelCase`. Some fields are renamed or restructured:

| REST Field | GraphQL Field | Breaking Change |
|------------|--------------|-----------------|
| `body_html` | `descriptionHtml` | `bodyHtml` exists but is **deprecated** |
| `tags` (comma-separated string) | `tags` (`[String!]!` array) | Must split/join when mapping |
| `status` (lowercase string) | `status` (enum: `ACTIVE`, `ARCHIVED`, `DRAFT`) | Must uppercase/lowercase in mapper |
| `image` / `images` | `featuredMedia` / `media` (MediaConnection) | `images` field is **deprecated**; media supports video, 3D |
| `id` (int64) | `id` (GID string) + `legacyResourceId` (uint64) | Always convert via GID helpers |
| `product_type` | `productType` | |
| `published_at` | `publishedAt` | To publish: use `publishablePublish` mutation |
| `variants` (array) | `variants` (ProductVariantConnection) | Paginated connection, max 2048 per product |
| N/A | `category` (TaxonomyCategory) | New in GraphQL only |

---

## 4. Gotchas & Traps

These will bite you if you don't handle them explicitly:

1. **`nodes` vs `edges{node}`** — Use `nodes` by default. Only use `edges{node}` when you need per-item `cursor`. Shopify docs: "querying `nodes` and `pageInfo` is preferred."

2. **Pagination limit: 250 max per page.** For datasets >10K items, use `bulkOperationRunQuery` instead of cursor pagination.

3. **Query cost limit: 1000 points max per query.** Nested connections multiply cost: `products(first:50) { nodes { variants(first:20) { nodes { id } } } }` costs ~1152 points and will be **rejected**. Reduce `first` values or flatten queries.

4. **Rate limit restore rates vary by plan:** Standard=100pts/s, Advanced=200, Plus=1000, Enterprise=2000. Read `extensions.cost.throttleStatus` from every response.

5. **`productSet` deletes omitted list items.** If you omit variants from input, existing variants get deleted. Only use `productSet` for full sync operations.

6. **Fulfillments use FulfillmentOrder, not Order.** REST creates fulfillments from order line items. GraphQL creates from `FulfillmentOrder` objects. Must fetch fulfillment orders first.

7. **`orderUpdate` is limited.** Can only change email, shipping address, tags, notes, metafields. For line item changes, use `orderEditBegin` → make edits → `orderEditCommit`.

8. **Status filter syntax changed.** REST: `status=any`. GraphQL: must list explicitly `status:active,archived,draft`. The value `UNLISTED` causes validation warnings.

9. **Permission scopes may differ.** Example: REST Redirect uses `write_content`; GraphQL equivalent needs `write_online_store_navigation`.

10. **Products with >100 variants.** REST endpoints **fail** for these products. GraphQL handles up to 2048 variants per product.

11. **GraphQL returns `null` where REST returns `""`.** Mapper functions must handle nil safely — don't assume non-nil for optional fields.

12. **Mutation errors are NOT HTTP errors.** GraphQL always returns HTTP 200. Check `userErrors` array in every mutation response. Check top-level `errors` array for API/throttle errors.

13. **`productCreate` vs `productUpdate` argument names differ.** `productCreate` uses arg `product: ProductCreateInput!`. `productUpdate` uses arg `product: ProductUpdateInput!`. Both use `product` as arg name, but different input types. The old `input: ProductInput` arg exists on both but is **deprecated** — using it in production will break when Shopify removes it.

14. **`customerCreate` and `customerUpdate` share the same input type.** Both use `CustomerInput!`. But `customerDelete` uses `CustomerDeleteInput!`. Don't assume all CRUD mutations follow the same naming pattern.

15. **`orderCancel` uses individual arguments, not an input object.** Unlike most mutations, `orderCancel(orderId: ID!, reason: OrderCancelReason!, restock: Boolean!, ...)` takes separate args. Wrapping them in an input object will cause a schema validation error.

16. **REST `_rest.go` must not be deleted.** Keep the old implementation as rollback path. Mark deprecated but preserve the code.

---

## 5. Go Implementation Patterns

### 5.1 GID Helpers

```go
func toGID(resource string, id int64) string {
    return fmt.Sprintf("gid://shopify/%s/%d", resource, id)
}

func fromGID(gid string) (int64, error) {
    // GID format: "gid://shopify/Product/123" → splits to ["gid:", "", "shopify", "Product", "123"]
    parts := strings.Split(gid, "/")
    if len(parts) < 5 || parts[0] != "gid:" {
        return 0, fmt.Errorf("invalid GID format: %s", gid)
    }
    return strconv.ParseInt(parts[len(parts)-1], 10, 64)
}

// Resource constants
const (
    GIDProduct          = "Product"
    GIDProductVariant   = "ProductVariant"
    GIDOrder            = "Order"
    GIDCustomer         = "Customer"
    GIDCollection       = "Collection"
    GIDInventoryItem    = "InventoryItem"
    GIDFulfillmentOrder = "FulfillmentOrder"
)
```

### 5.2 GraphQL Client

Two approaches — choose based on existing codebase style:

**Option A: `hasura/go-graphql-client` (struct-based, type-safe)**
```go
import graphql "github.com/hasura/go-graphql-client"

client := graphql.NewClient(
    fmt.Sprintf("https://%s/admin/api/%s/graphql.json", shop, version), nil,
).WithRequestModifier(func(r *http.Request) {
    r.Header.Set("X-Shopify-Access-Token", token)
})

var query struct {
    Product struct {
        ID    string
        Title string
    } `graphql:"product(id: $id)"`
}
vars := map[string]interface{}{
    "id": graphql.ID(toGID(GIDProduct, productID)),
}
err := client.Query(ctx, &query, vars)
```

**Option B: Raw `net/http` client (string-based, flexible)**
```go
type GraphQLClient struct {
    endpoint    string
    accessToken string
    httpClient  *http.Client
}

func (c *GraphQLClient) Execute(ctx context.Context, query string,
    vars map[string]interface{}, result interface{}) error {
    // POST JSON {query, variables} → parse {data, errors, extensions}
    // Check errors array → unmarshal data into result
}
```

Use Option A when the codebase uses struct-based patterns. Use Option B for raw control.

### 5.3 Response Mapping Pattern

The conversion contract: **domain structs don't change**. Create internal GraphQL response types that map to existing domain types.

```go
// Domain struct — UNCHANGED (this is the contract)
type Product struct {
    ID          int64      `json:"id"`
    Title       string     `json:"title"`
    BodyHTML    string     `json:"body_html"`
    Tags        []string   `json:"tags"`
    Status      string     `json:"status"`        // lowercase: "active"
    Variants    []*Variant `json:"variants"`
}

// Internal GraphQL type — maps to domain
type gqlProduct struct {
    ID              string   `json:"id"`              // GID
    Title           string   `json:"title"`
    DescriptionHtml string   `json:"descriptionHtml"` // was body_html
    Tags            []string `json:"tags"`             // already array in GQL
    Status          string   `json:"status"`           // uppercase: "ACTIVE"
    Variants        struct {
        Nodes []gqlVariant `json:"nodes"`
    } `json:"variants"`
}

func (g *gqlProduct) toDomain() *Product {
    id, _ := fromGID(g.ID)
    p := &Product{
        ID:       id,
        Title:    g.Title,
        BodyHTML:  g.DescriptionHtml,
        Tags:     g.Tags,
        Status:   strings.ToLower(g.Status),
    }
    if p.Tags == nil { p.Tags = []string{} } // null safety
    for _, v := range g.Variants.Nodes {
        p.Variants = append(p.Variants, v.toDomain())
    }
    return p
}
```

### 5.4 Cursor Pagination Helper

```go
func paginate[T any](ctx context.Context, client *GraphQLClient, query string,
    pageSize int, extractFn func(json.RawMessage) ([]T, bool, string, error),
) ([]T, error) {
    var all []T
    var cursor string
    for {
        vars := map[string]interface{}{"first": pageSize}
        if cursor != "" { vars["after"] = cursor }
        var raw json.RawMessage
        if err := client.Execute(ctx, query, vars, &raw); err != nil {
            return nil, err
        }
        items, hasNext, endCursor, err := extractFn(raw)
        if err != nil { return nil, err }
        all = append(all, items...)
        if !hasNext { break }
        cursor = endCursor
    }
    return all, nil
}
```

### 5.5 UserErrors Check

```go
type UserError struct {
    Field   []string `json:"field"`
    Message string   `json:"message"`
}

func checkUserErrors(errs []UserError) error {
    if len(errs) == 0 { return nil }
    msgs := make([]string, len(errs))
    for i, e := range errs {
        msgs[i] = fmt.Sprintf("[%s] %s", strings.Join(e.Field, "."), e.Message)
    }
    return fmt.Errorf("shopify: %s", strings.Join(msgs, "; "))
}
```

### 5.6 Bulk Operations (for large datasets)

When paginating >10K items, use bulk operations instead:

```go
const bulkQuery = `mutation { bulkOperationRunQuery(
    query: """{ products { edges { node { id title variants { edges { node { id sku price } } } } } } }"""
) { bulkOperation { id status } userErrors { field message } } }`

// 1. Submit bulk operation
// 2. Poll bulkOperation(id:) until status=COMPLETED
// 3. Download JSONL from url field
// 4. Parse JSONL: one object per line, nested items have __parentId
```

Constraints: max 5 concurrent ops per shop (API 2026-01+), max 5 connections in query, max 2 levels nesting, results available 7 days.

---

## Query Filter Syntax Reference

GraphQL uses string-based filters in the `query` argument:

```graphql
products(first: 50, query: "status:active vendor:Nike created_at:>2024-01-01")
orders(first: 20, query: "financial_status:paid fulfillment_status:unfulfilled")
customers(first: 10, query: "email:*@example.com tag:vip")
```

| Operator | Example | Meaning |
|----------|---------|---------|
| `:` | `status:active` | Equals |
| `:>` | `created_at:>2024-01-01` | Greater than |
| `:<` | `price:<100` | Less than |
| `:>=` | `inventory_total:>=200` | Greater or equal |
| `*` | `email:*@example.com` | Wildcard |
| `,` | `status:active,draft` | OR within field |
| ` ` (space) | `status:active vendor:Nike` | AND between fields |
| `-` | `-tag:clearance` | NOT |
