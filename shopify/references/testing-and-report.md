# Part 3: Live API Testing & HTML Report

Reference file for `shopify-rest-to-graphql-go` skill. Read when performing steps 6-8.

---

## 6. Unit Test Requirements — LIVE API, NO MOCKING

**Core principle: All tests must call the real Shopify API. DO NOT mock data.**

Reason: Mock data only proves the code works with data you made up. Calling the real API proves the code works with actual Shopify production responses — including edge cases you did not think of.

### 6.0 External Test Directory

Tests written outside the main repo at `$TEST_DIR` (e.g. `~/Desktop/shopify/test/unittest/`):

```
$TEST_DIR/
├── go.mod                          # Separate module, imports main repo via replace directive
├── go.sum
├── .env                            # Shopify credentials (DO NOT commit)
├── helpers/
│   ├── shopify_client.go           # Real REST + GraphQL client
│   ├── test_config.go              # Load .env, skip if no credentials
│   ├── cleanup.go                  # Cleanup test data after each test
│   ├── compare.go                  # Deep comparison helpers
│   └── reporter.go                 # HTML report generator (Section 9)
├── product/
│   ├── product_live_test.go        # Live API tests — call real Shopify
│   ├── product_parity_test.go      # REST vs GraphQL same input, compare output
│   └── product_cases_test.go       # Multiple different test cases
├── order/
│   ├── order_live_test.go
│   ├── order_parity_test.go
│   └── order_cases_test.go
├── customer/...
├── inventory/...
├── fulfillment/...
├── collection/...
├── webhook/...
├── metafield/...
├── shop/...
├── shipping/...
└── scripts/
    ├── run_all.sh
    └── run_service.sh
```

### 6.0.1 Setup

**`.env` (DO NOT commit — add to `.gitignore`):**
```env
SHOPIFY_SHOP_DOMAIN=your-dev-store.myshopify.com
SHOPIFY_ACCESS_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SHOPIFY_API_VERSION=2025-01
```

**`go.mod`:**
```go
module shopify-graphql-tests

go 1.22

require (
    YOUR_MODULE_PATH v0.0.0
    github.com/stretchr/testify v1.9.0
    github.com/joho/godotenv v1.5.1
)

replace YOUR_MODULE_PATH => /path/to/your/main/repo
```

### 6.0.2 Shared Test Infrastructure

**`helpers/test_config.go`:**
```go
package helpers

import (
    "os"
    "testing"

    "github.com/joho/godotenv"
)

type ShopifyConfig struct {
    ShopDomain  string
    AccessToken string
    APIVersion  string
}

// LoadConfig loads Shopify credentials from .env
// Skips test if no credentials (allows CI to run without secrets)
func LoadConfig(t *testing.T) ShopifyConfig {
    t.Helper()
    _ = godotenv.Load("../.env") // load from root test dir

    shop := os.Getenv("SHOPIFY_SHOP_DOMAIN")
    token := os.Getenv("SHOPIFY_ACCESS_TOKEN")
    version := os.Getenv("SHOPIFY_API_VERSION")

    if shop == "" || token == "" {
        t.Skip("SHOPIFY_SHOP_DOMAIN and SHOPIFY_ACCESS_TOKEN required — skipping live test")
    }
    if version == "" {
        version = "2025-01"
    }

    return ShopifyConfig{
        ShopDomain:  shop,
        AccessToken: token,
        APIVersion:  version,
    }
}
```

**`helpers/shopify_client.go`:**
```go
package helpers

import (
    "testing"

    "YOUR_MODULE_PATH/internal/shopify/rest"
    "YOUR_MODULE_PATH/internal/shopify/graphql"
)

func NewLiveRESTClient(t *testing.T, cfg ShopifyConfig) *rest.Client {
    t.Helper()
    return rest.NewClient(cfg.ShopDomain, cfg.AccessToken, cfg.APIVersion)
}

func NewLiveGraphQLClient(t *testing.T, cfg ShopifyConfig) *graphql.Client {
    t.Helper()
    return graphql.NewClient(cfg.ShopDomain, cfg.AccessToken, cfg.APIVersion)
}
```

**`helpers/compare.go`:**
```go
package helpers

import (
    "fmt"
    "reflect"
    "strings"
    "testing"

    "github.com/stretchr/testify/assert"
)

type CompareResult struct {
    Field     string
    RESTValue interface{}
    GQLValue  interface{}
    Match     bool
}

func DeepCompare(t *testing.T, restObj, gqlObj interface{}) []CompareResult {
    t.Helper()
    var results []CompareResult

    rv := reflect.ValueOf(restObj)
    gv := reflect.ValueOf(gqlObj)
    if rv.Kind() == reflect.Ptr { rv = rv.Elem() }
    if gv.Kind() == reflect.Ptr { gv = gv.Elem() }

    for i := 0; i < rv.NumField(); i++ {
        fieldName := rv.Type().Field(i).Name
        rVal := rv.Field(i).Interface()
        gVal := gv.Field(i).Interface()
        match := reflect.DeepEqual(rVal, gVal)
        results = append(results, CompareResult{Field: fieldName, RESTValue: rVal, GQLValue: gVal, Match: match})
        assert.Equal(t, rVal, gVal, "field %s mismatch", fieldName)
    }
    return results
}

func PrintCompareReport(results []CompareResult) {
    fmt.Printf("%-25s %-6s %-40s %-40s\n", "FIELD", "MATCH", "REST VALUE", "GRAPHQL VALUE")
    fmt.Println(strings.Repeat("-", 115))
    for _, r := range results {
        status := "✓ PASS"
        if !r.Match { status = "✗ FAIL" }
        fmt.Printf("%-25s %-6s %-40v %-40v\n", r.Field, status, r.RESTValue, r.GQLValue)
    }
}
```

### 6.1 Live Parity Test Pattern

The most important test: same input, call both REST and GraphQL on real Shopify, compare output.

```go
func TestLiveParity_GetProduct(t *testing.T) {
    cfg := helpers.LoadConfig(t)
    restSvc := restimpl.NewProductService(helpers.NewLiveRESTClient(t, cfg))
    gqlSvc := graphqlimpl.NewProductService(helpers.NewLiveGraphQLClient(t, cfg))
    ctx := context.Background()

    // Get a real product from store
    products, err := restSvc.ListProducts(ctx, product.ListOpts{Limit: 1})
    require.NoError(t, err)
    require.NotEmpty(t, products, "store needs at least 1 product")

    // Call BOTH APIs with the same input
    restResult, err := restSvc.GetProduct(ctx, products[0].ID)
    require.NoError(t, err)
    gqlResult, err := gqlSvc.GetProduct(ctx, products[0].ID)
    require.NoError(t, err)

    // Compare every field
    assert.Equal(t, restResult.ID, gqlResult.ID, "ID")
    assert.Equal(t, restResult.Title, gqlResult.Title, "Title")
    assert.Equal(t, restResult.BodyHTML, gqlResult.BodyHTML, "BodyHTML")
    assert.Equal(t, restResult.Tags, gqlResult.Tags, "Tags")
    assert.Equal(t, restResult.Status, gqlResult.Status, "Status")
    require.Equal(t, len(restResult.Variants), len(gqlResult.Variants), "Variants count")
}
```

### 6.2 Multi-Case Test Pattern

Each service must be tested with multiple different cases on real data (at least 5-10 resources):

```go
func TestLive_GetProduct_MultipleCases(t *testing.T) {
    cfg := helpers.LoadConfig(t)
    restSvc := restimpl.NewProductService(helpers.NewLiveRESTClient(t, cfg))
    gqlSvc := graphqlimpl.NewProductService(helpers.NewLiveGraphQLClient(t, cfg))
    ctx := context.Background()

    products, err := restSvc.ListProducts(ctx, product.ListOpts{Limit: 10})
    require.NoError(t, err)

    for _, p := range products {
        t.Run(fmt.Sprintf("Product_%d", p.ID), func(t *testing.T) {
            restResult, _ := restSvc.GetProduct(ctx, p.ID)
            gqlResult, _ := gqlSvc.GetProduct(ctx, p.ID)
            helpers.DeepCompare(t, restResult, gqlResult)
        })
    }
}
```

### 6.3 Mandatory Test Cases per Service

Each service must have these tests, **calling the real Shopify API**:

| # | Test Case | Description | Expected |
|---|-----------|-------------|----------|
| 1 | **Get single — parity** | Same ID, call REST + GraphQL, compare | All fields match |
| 2 | **Get multiple — parity** | 5-10 resources, compare each | All fields match |
| 3 | **List — parity** | Same params, compare count + data | Same count and data |
| 4 | **List with filters** | Query filters (`status:active`) | Results match filter |
| 5 | **List pagination** | Paginate through all, compare total | No duplicates, correct total |
| 6 | **CRUD cycle** | Create → Read → Update → Delete | Full cycle succeeds |
| 7 | **Create invalid input** | Empty title, invalid field | `userErrors` returned, no panic |
| 8 | **Get not found — error parity** | Non-existent ID, call BOTH | `errors.Is` returns same error type |
| 8b | **Error type preservation** | All REST error types tested | `errors.Is`/`errors.As` match |
| 8c | **Nil return preservation** | REST `(nil, nil)` cases | GraphQL also `(nil, nil)` |
| 9 | **Partial update** | Update only 1 field | Only that field changed |
| 10 | **Tags handling** | Complex tags (spaces, special chars) | Comma-string = array content |
| 11 | **Status mapping** | active, draft, archived | Lowercase in domain struct |
| 12 | **Variants/Connections** | Product with 5+ variants | All present, correct order |
| 13 | **Empty collections** | Product with 0 variants | `[]` not nil |
| 14 | **Large data** | Long description, many tags | No truncation |
| 15 | **Concurrent reads** | 5 goroutines GetProduct | All succeed, race-free |

### 6.4 Service-Specific Test Cases

**Orders:**
| # | Test | Reason |
|---|------|--------|
| O1 | Filter `financial_status:paid` | GraphQL filter syntax differs |
| O2 | Filter `fulfillment_status:unfulfilled` | Verify filter works |
| O3 | `orderUpdate` change only tags | Verify limited update scope |
| O4 | `orderClose` → check status | Verify mutation works |

**Inventory:**
| # | Test | Reason |
|---|------|--------|
| I1 | Get inventory levels for 1 variant | GID conversion for InventoryItem |
| I2 | Adjust +1 then -1 | Verify delta-based, restore original |
| I3 | Set quantity → Get → verify | Set vs adjust behavior |

**Customers:**
| # | Test | Reason |
|---|------|--------|
| C1 | Create → Get → compare | `CustomerInput` shared between create/update |
| C2 | Update email → Get → verify | Partial update |
| C3 | Search by email wildcard | GraphQL filter `email:*@example.com` |

### 6.5 Run Commands

```bash
cd $TEST_DIR

# Run all live tests
go test ./... -race -count=1 -v -timeout=5m

# Run 1 service
go test ./product/... -race -count=1 -v -timeout=2m

# Run only parity tests
go test ./... -run "LiveParity" -v -timeout=5m

# Run with verbose output for report
go test ./... -v -json 2>&1 | tee test_results.json
```

### 6.6 Live API Testing Notes

1. **Use development store.** Never test on production. Shopify Partner accounts allow free dev stores.
2. **Cleanup after each test.** All created resources must be deleted in `t.Cleanup()`.
3. **Rate limiting.** Use `t.Parallel()` carefully — may need sequential for mutations.
4. **Timeout.** Set `-timeout=5m` because API calls take time.
5. **Skip when no credentials.** `LoadConfig()` auto-skips if `.env` is missing.
6. **Idempotent tests.** Must be runnable multiple times. Create data → test → cleanup.

---

## 8. Conversion Execution Checklist

For each service being converted, verify all items before marking complete:

### Scope Discipline (check FIRST)
- [ ] `git diff --stat` only contains files in the "ONLY modify" list
- [ ] Domain structs, service interfaces, handlers NOT modified
- [ ] `_rest.go` still intact, only added deprecated comment
- [ ] No refactoring of unrelated code

### Error Preservation (CRITICAL)
- [ ] All error types preserved: `ErrNotFound`, `ErrUnauthorized`, `ErrRateLimited`, etc.
- [ ] Error message format preserved (callers may string-match)
- [ ] Error wrapping chain preserved (`%w` wrap same way)
- [ ] Nil return behavior preserved: `(nil, nil)` stays `(nil, nil)`
- [ ] Each REST HTTP status mapped correctly to GraphQL error
- [ ] Live test: `errors.Is(restErr, X)` = `errors.Is(gqlErr, X)` for all cases
- [ ] No "graphql:" or "gql:" prefix added to messages

### Technical Correctness
- [ ] All REST endpoints mapped to correct GraphQL operations
- [ ] Mutation arg names correct (`product:` vs `input:` — see gotcha #13-15)
- [ ] Input types correct (ProductCreateInput, ProductUpdateInput, CustomerInput, CustomerDeleteInput)
- [ ] GID conversion in all directions
- [ ] Field mapping correct (`body_html`→`descriptionHtml`)
- [ ] Tags comma-string ↔ array handled
- [ ] Status lowercase ↔ uppercase handled
- [ ] Pagination cursor-based with `pageInfo`
- [ ] `userErrors` + `errors` array checked
- [ ] Null safety for all optional fields

### Tests — Live API
- [ ] Tests call real Shopify API, no mocking
- [ ] Tests in external directory, not in main repo
- [ ] Parity tests pass: same input → same output
- [ ] Multi-case: at least 5 resources per service
- [ ] CRUD cycle: Create → Read → Update → Delete
- [ ] Error tests: not found, invalid input
- [ ] Cleanup: no garbage on dev store

### Report & Rollback
- [ ] HTML report generated and reviewed
- [ ] REST implementation still compiles
- [ ] DI/provider can switch back to REST in < 5 minutes

---

## 9. Conversion Report — Professional HTML Dashboard

After converting each service, generate a professional HTML report for stakeholder review. Based on live API test results, not mocks. Standalone file with embedded CSS — open in any browser.

Output: `$TEST_DIR/reports/[service]_report.html`

### 9.1 Report Generator

```go
// File: $TEST_DIR/helpers/reporter.go
package helpers

import (
    "encoding/json"
    "fmt"
    "html/template"
    "os"
    "strings"
    "time"
)

type FieldResult struct {
    Field    string      `json:"field"`
    RESTVal  interface{} `json:"rest_value"`
    GQLVal   interface{} `json:"gql_value"`
    Match    bool        `json:"match"`
}

type EndpointReport struct {
    Name          string        `json:"name"`
    RESTMethod    string        `json:"rest_method"`
    RESTPath      string        `json:"rest_path"`
    GraphQLOp     string        `json:"graphql_op"`
    InputDesc     string        `json:"input_desc"`
    Fields        []FieldResult `json:"fields"`
    AllMatch      bool          `json:"all_match"`
    TestCases     int           `json:"test_cases"`
    TestsPassed   int           `json:"tests_passed"`
}

type ConversionReport struct {
    Service    string           `json:"service"`
    Date       string           `json:"date"`
    Store      string           `json:"store"`
    APIVersion string           `json:"api_version"`
    Endpoints  []EndpointReport `json:"endpoints"`
}

func (r ConversionReport) TotalEndpoints() int { return len(r.Endpoints) }
func (r ConversionReport) PassedEndpoints() int {
    n := 0; for _, e := range r.Endpoints { if e.AllMatch { n++ } }; return n
}
func (r ConversionReport) TotalTests() int {
    n := 0; for _, e := range r.Endpoints { n += e.TestCases }; return n
}
func (r ConversionReport) PassedTests() int {
    n := 0; for _, e := range r.Endpoints { n += e.TestsPassed }; return n
}
func (r ConversionReport) TotalFields() int {
    n := 0; for _, e := range r.Endpoints { n += len(e.Fields) }; return n
}
func (r ConversionReport) MatchedFields() int {
    n := 0; for _, e := range r.Endpoints { for _, f := range e.Fields { if f.Match { n++ } } }; return n
}
func (r ConversionReport) AllPassed() bool { return r.PassedEndpoints() == r.TotalEndpoints() }

func GenerateHTML(report ConversionReport, outPath string) error {
    tmpl, err := template.New("report").Funcs(template.FuncMap{
        "statusIcon":  func(ok bool) string { if ok { return "PASS" }; return "FAIL" },
        "statusClass": func(ok bool) string { if ok { return "pass" }; return "fail" },
        "fmtVal":      func(v interface{}) string { return fmt.Sprintf("%v", v) },
        "truncate": func(v interface{}, n int) string {
            s := fmt.Sprintf("%v", v); if len(s) > n { return s[:n] + "..." }; return s
        },
    }).Parse(htmlTemplate)
    if err != nil { return fmt.Errorf("parse template: %w", err) }
    f, err := os.Create(outPath)
    if err != nil { return fmt.Errorf("create file: %w", err) }
    defer f.Close()
    return tmpl.Execute(f, report)
}

func SaveReport(report ConversionReport, dir string) error {
    os.MkdirAll(dir, 0755)
    jsonData, _ := json.MarshalIndent(report, "", "  ")
    os.WriteFile(fmt.Sprintf("%s/%s_report.json", dir, report.Service), jsonData, 0644)
    return GenerateHTML(report, fmt.Sprintf("%s/%s_report.html", dir, report.Service))
}

func CollectEndpointReport(name, restMethod, restPath, gqlOp, inputDesc string,
    compareResults []FieldResult, testCases, testsPassed int) EndpointReport {
    allMatch := true
    for _, r := range compareResults { if !r.Match { allMatch = false; break } }
    return EndpointReport{
        Name: name, RESTMethod: restMethod, RESTPath: restPath, GraphQLOp: gqlOp,
        InputDesc: inputDesc, Fields: compareResults, AllMatch: allMatch,
        TestCases: testCases, TestsPassed: testsPassed,
    }
}
```

### 9.2 HTML Template

The `htmlTemplate` constant (embedded in `reporter.go`) generates a standalone dashboard:

```go
const htmlTemplate = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{.Service}} — REST → GraphQL Conversion Report</title>
<style>
  :root{--green:#10b981;--red:#ef4444;--blue:#3b82f6;--gray:#6b7280;--bg:#f9fafb;--card:#fff;--border:#e5e7eb}
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;background:var(--bg);color:#1f2937;line-height:1.6}
  .container{max-width:1200px;margin:0 auto;padding:24px}
  .header{background:linear-gradient(135deg,#1e293b,#334155);color:#fff;padding:32px;border-radius:12px;margin-bottom:24px}
  .header h1{font-size:28px;font-weight:700;margin-bottom:4px}
  .header .subtitle{opacity:.8;font-size:14px}
  .header .meta{display:flex;gap:24px;margin-top:16px;font-size:13px;opacity:.7}
  .cards{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:16px;margin-bottom:24px}
  .card{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:20px;text-align:center}
  .card .value{font-size:36px;font-weight:800}
  .card .label{font-size:13px;color:var(--gray);text-transform:uppercase;letter-spacing:.5px;margin-top:4px}
  .card.pass .value{color:var(--green)}.card.fail .value{color:var(--red)}
  .banner{padding:16px 24px;border-radius:10px;font-size:18px;font-weight:700;text-align:center;margin-bottom:24px}
  .banner.pass{background:#d1fae5;color:#065f46;border:2px solid var(--green)}
  .banner.fail{background:#fee2e2;color:#991b1b;border:2px solid var(--red)}
  .endpoint{background:var(--card);border:1px solid var(--border);border-radius:10px;margin-bottom:16px;overflow:hidden}
  .endpoint-header{display:flex;align-items:center;justify-content:space-between;padding:16px 20px;cursor:pointer;user-select:none}
  .endpoint-header:hover{background:#f3f4f6}
  .endpoint-header h3{font-size:16px;font-weight:600}
  .badge{padding:4px 12px;border-radius:20px;font-size:12px;font-weight:700;text-transform:uppercase}
  .badge.pass{background:#d1fae5;color:#065f46}.badge.fail{background:#fee2e2;color:#991b1b}
  .endpoint-header .method{background:var(--blue);color:#fff;padding:2px 8px;border-radius:4px;font-size:11px;font-weight:700;margin-right:8px}
  .endpoint-header .arrow{transition:transform .2s;font-size:12px;color:var(--gray)}
  .endpoint.open .arrow{transform:rotate(90deg)}
  .endpoint-body{display:none;padding:0 20px 20px;border-top:1px solid var(--border)}
  .endpoint.open .endpoint-body{display:block}
  .api-info{display:grid;grid-template-columns:1fr 1fr;gap:16px;margin:16px 0}
  .api-box{background:#f8fafc;border:1px solid var(--border);border-radius:8px;padding:14px}
  .api-box h4{font-size:12px;text-transform:uppercase;color:var(--gray);letter-spacing:.5px;margin-bottom:8px}
  .api-box code{font-size:13px;background:#e2e8f0;padding:2px 6px;border-radius:4px}
  .api-box .detail{font-size:13px;color:#374151;margin-top:4px}
  table{width:100%;border-collapse:collapse;font-size:14px;margin-top:16px}
  thead{background:#f1f5f9}
  th{text-align:left;padding:10px 14px;font-weight:600;font-size:12px;text-transform:uppercase;letter-spacing:.5px;color:var(--gray);border-bottom:2px solid var(--border)}
  td{padding:10px 14px;border-bottom:1px solid var(--border);font-family:'SF Mono','Fira Code',monospace;font-size:13px;max-width:300px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  tr:hover{background:#f9fafb}
  .match-icon{display:inline-block;width:22px;height:22px;border-radius:50%;text-align:center;line-height:22px;font-size:12px;font-weight:700}
  .match-icon.pass{background:#d1fae5;color:#065f46}.match-icon.fail{background:#fee2e2;color:#991b1b}
  .footer{text-align:center;padding:24px;font-size:12px;color:var(--gray)}
</style>
</head>
<body>
<div class="container">
  <div class="header">
    <h1>{{.Service}} — REST → GraphQL Conversion Report</h1>
    <div class="subtitle">Shopify Admin API Migration — Live API Parity Verification</div>
    <div class="meta"><span>Date: {{.Date}}</span><span>Store: {{.Store}}</span><span>API: {{.APIVersion}}</span></div>
  </div>
  <div class="banner {{statusClass .AllPassed}}">
    {{if .AllPassed}}ALL ENDPOINTS PASSED — Migration verified{{else}}FAILURES DETECTED — Review required{{end}}
  </div>
  <div class="cards">
    <div class="card {{statusClass .AllPassed}}"><div class="value">{{.PassedEndpoints}}/{{.TotalEndpoints}}</div><div class="label">Endpoints</div></div>
    <div class="card {{statusClass (eq .PassedTests .TotalTests)}}"><div class="value">{{.PassedTests}}/{{.TotalTests}}</div><div class="label">Tests</div></div>
    <div class="card {{statusClass (eq .MatchedFields .TotalFields)}}"><div class="value">{{.MatchedFields}}/{{.TotalFields}}</div><div class="label">Fields</div></div>
    <div class="card"><div class="value" style="color:var(--blue)">{{.APIVersion}}</div><div class="label">API Version</div></div>
  </div>
  {{range .Endpoints}}
  <div class="endpoint open">
    <div class="endpoint-header" onclick="this.parentElement.classList.toggle('open')">
      <h3><span class="method">{{.RESTMethod}}</span>{{.Name}}</h3>
      <div style="display:flex;align-items:center;gap:12px">
        <span style="font-size:13px;background:#f1f5f9;padding:4px 10px;border-radius:6px">{{.TestsPassed}}/{{.TestCases}} tests</span>
        <span class="badge {{statusClass .AllMatch}}">{{statusIcon .AllMatch}}</span>
        <span class="arrow">▶</span>
      </div>
    </div>
    <div class="endpoint-body">
      <div class="api-info">
        <div class="api-box"><h4>REST</h4><code>{{.RESTMethod}} {{.RESTPath}}</code><div class="detail">Input: {{.InputDesc}}</div></div>
        <div class="api-box"><h4>GraphQL</h4><code>{{.GraphQLOp}}</code></div>
      </div>
      <table>
        <thead><tr><th style="width:30px"></th><th>Field</th><th>REST Value</th><th>GraphQL Value</th></tr></thead>
        <tbody>{{range .Fields}}
          <tr>
            <td><span class="match-icon {{statusClass .Match}}">{{if .Match}}✓{{else}}✗{{end}}</span></td>
            <td style="font-weight:600">{{.Field}}</td>
            <td title="{{fmtVal .RESTVal}}">{{truncate .RESTVal 60}}</td>
            <td title="{{fmtVal .GQLVal}}">{{truncate .GQLVal 60}}</td>
          </tr>{{end}}
        </tbody>
      </table>
    </div>
  </div>
  {{end}}
  <div class="footer">Generated by Shopify REST→GraphQL Migration Tool — All data from live API calls</div>
</div>
<script>
document.querySelectorAll('.endpoint').forEach(el=>{if(!el.querySelector('.badge.fail'))el.classList.remove('open')});
</script>
</body>
</html>` + "`"
```

### 9.3 Usage in Tests

```go
func TestGenerateReport(t *testing.T) {
    cfg := helpers.LoadConfig(t)
    restSvc := restimpl.NewProductService(helpers.NewLiveRESTClient(t, cfg))
    gqlSvc := graphqlimpl.NewProductService(helpers.NewLiveGraphQLClient(t, cfg))
    ctx := context.Background()

    report := helpers.ConversionReport{
        Service: "ProductService", Date: time.Now().Format("2006-01-02 15:04"),
        Store: cfg.ShopDomain, APIVersion: cfg.APIVersion,
    }

    // Collect parity data for each endpoint
    products, _ := restSvc.ListProducts(ctx, product.ListOpts{Limit: 1})
    restResult, _ := restSvc.GetProduct(ctx, products[0].ID)
    gqlResult, _ := gqlSvc.GetProduct(ctx, products[0].ID)
    fields := helpers.DeepCompare(t, restResult, gqlResult)

    report.Endpoints = append(report.Endpoints, helpers.CollectEndpointReport(
        "GetProduct", "GET", "/admin/api/2025-01/products/{id}.json",
        "query { product(id: $id) { ... } }",
        fmt.Sprintf("id = %d", products[0].ID),
        fields, 5, 5,
    ))
    // ... repeat for each endpoint ...

    helpers.SaveReport(report, "../reports")
    t.Logf("Report: ../reports/ProductService_report.html")
}
```

### 9.4 What the Report Shows

- **Header**: service name, date, store, API version
- **Status banner**: green "ALL PASSED" or red "FAILURES DETECTED"
- **Summary cards**: endpoints passed, test cases, fields matched
- **Collapsible endpoints**: click to expand — REST/GraphQL info + field comparison table
- **Auto-collapse**: passed endpoints start collapsed, failures start expanded
- **Standalone**: embedded CSS, no external dependencies, works offline

### 9.5 When to Generate

- After Step 6 (Test): auto-generate after live tests pass
- Before Step 8 (Verify): user must review in browser and approve
- If any field FAILS: STOP, fix mapping, re-test, re-generate
- Open: `open $TEST_DIR/reports/ProductService_report.html`
