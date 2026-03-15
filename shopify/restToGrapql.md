---
name: shopify-rest-to-graphql
description: Converts Shopify REST Admin API code to GraphQL Admin API. Use when the user wants to migrate from Shopify REST to GraphQL, rewrite REST API calls as GraphQL, or mentions Shopify Admin API, REST migration, or GraphQL for Shopify. Prefer this skill whenever REST endpoints, fetch to Shopify, or legacy Admin API are discussed in a Shopify app context.
---

# Shopify REST to GraphQL

Skill for converting Shopify REST Admin API usage to GraphQL Admin API. The REST Admin API is legacy (as of October 2024); new work should use GraphQL.

## When to apply

- User asks to convert, migrate, or rewrite REST to GraphQL for Shopify
- Code uses REST endpoints like `/admin/api/.../products.json`, `customers.json`, `orders.json`, etc.
- User mentions Shopify Admin API migration or “REST to GraphQL”
- App uses `admin.rest`, `shopify.clients.Rest`, or direct `fetch` to REST Admin API

## Endpoint and client changes

| REST | GraphQL |
|------|--------|
| Many endpoints (e.g. `GET /admin/api/{version}/products/{id}.json`) | Single endpoint: `POST https://{shop}.myshopify.com/admin/api/{version}/graphql.json` |
| Client: `admin.rest` / `shopify.clients.Rest` | Client: GraphQL client (e.g. `admin.graphql` / `shopify.clients.Graphql`) |
| Methods: `get`, `find`, `all`, etc. | Single `query` (or `mutation`) method with a document |
| Numeric IDs (e.g. `1234`) | Global IDs: `gid://shopify/Product/1234` |

## Conversion steps

1. **Identify the REST resource and action**  
   Map REST path + method to the right GraphQL type and operation (query vs mutation). Use [GraphQL Admin API reference](https://shopify.dev/docs/api/admin-graphql) or GraphiQL explorer when needed.

2. **Use Global IDs (GIDs)**  
   Replace numeric IDs with GIDs where the API expects them:  
   `gid://shopify/{ResourceName}/{id}`  
   Example: `gid://shopify/Product/123456789`  
   For list operations, use the appropriate connection/query (e.g. `products(first: N)`).

3. **Write the GraphQL operation**  
   - Request only the fields the app needs.  
   - Use connections for lists (e.g. `products(first: 10) { edges { node { id title } } }`).  
   - For mutations, use the corresponding mutation (e.g. `productCreate`, `orderUpdate`).

4. **Switch client and call**  
   - Use the project’s GraphQL client (e.g. `admin.graphql` or `shopify.clients.Graphql`).  
   - Call with the operation string and variables (if any).  
   - Do not use REST client or raw `fetch` to REST URLs for this logic.

5. **Handle the response**  
   - GraphQL responses are under `data.<operationName>` (and often nested under a connection/node structure).  
   - Map the new shape to what the rest of the app expects (variables, UI, etc.).  
   - Check `data.userErrors` (or equivalent) for mutation errors.

## Example mapping (conceptual)

**REST:** Get a product by ID  
`GET /admin/api/2024-01/products/123.json`

**GraphQL equivalent:**  
- Query `product(id: "gid://shopify/Product/123")` with the needed fields (e.g. `id`, `title`, `variants`).  
- Use the GraphQL client; read result from `data.product`.

**REST:** List orders  
`GET /admin/api/2024-01/orders.json?limit=10`

**GraphQL equivalent:**  
- Query `orders(first: 10)` (or the appropriate connection for the API version).  
- Use `edges { node { ... } }` (and `pageInfo` if paginating).  
- Read from `data.orders`.

## Important notes

- **API version:** Use the same or a supported Admin API version in the GraphQL URL and in the client. Keep version consistent with the rest of the app.
- **Rate limits:** GraphQL uses a calculated cost model instead of “N requests per second”. Prefer one well-shaped query over many small REST calls where it reduces round-trips.
- **Pagination:** In GraphQL use cursor-based pagination (`first`, `after`, `pageInfo.endCursor`) instead of REST `limit`/`page` where the schema uses connections.
- **References:** Official migration guide: [Migrate to GraphQL from REST](https://shopify.dev/docs/apps/build/graphql/migrate). For updating API calls in code: [Update API calls in your app](https://shopify.dev/docs/apps/build/graphql/migrate/libraries).

## Output format

- When converting a file: keep the same structure and behavior where possible; only replace REST with GraphQL and adjust types/variables as needed.
- When explaining: give the GraphQL operation (query/mutation), the GID/form of IDs, and how to read the response (path in `data`, `userErrors`).
- If the codebase uses a specific Shopify library (e.g. `@shopify/shopify-api`), use that library’s GraphQL client and patterns rather than inventing a new style.
