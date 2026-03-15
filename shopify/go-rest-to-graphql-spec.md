# Spec: Chuyển Shopify REST sang GraphQL cho codebase Go

Tài liệu này mô tả yêu cầu khi phân tích và chuyển đổi source Go từ Shopify REST Admin API sang GraphQL Admin API. Dùng khi user yêu cầu phân tích code Go, tìm service dùng REST, convert sang GraphQL và viết unit test.

## Yêu cầu tổng quát

1. **Đọc và phân tích toàn bộ code Go** trong repo hiện tại.
2. **Liệt kê các service** đang gọi Shopify REST Admin API (HTTP client, endpoint `/admin/api/...`, thư viện REST).
3. **Đánh giá từng service**: có thể chuyển sang GraphQL hay không (theo [Admin GraphQL API](https://shopify.dev/docs/api/admin-graphql/latest)).
4. **Chuyển sang GraphQL** với ràng buộc:
   - **Input và output** của từng service/handler **giữ nguyên** so với khi dùng REST (contract không đổi cho caller).
   - Dùng [Admin REST API](https://shopify.dev/docs/api/admin-rest) để hiểu resource/field hiện tại, [Admin GraphQL API (latest)](https://shopify.dev/docs/api/admin-graphql/latest) để implement.
5. **Unit test**: mọi service/hàm đã convert phải có unit test đầy đủ (cover logic gọi GraphQL, map request/response, xử lý lỗi). Test phải pass và chuyên nghiệp (table-driven, mock client, assertions rõ ràng).

## Tài liệu tham chiếu

| Mục đích | URL |
|----------|-----|
| REST (hiện tại) | https://shopify.dev/docs/api/admin-rest |
| GraphQL (đích) | https://shopify.dev/docs/api/admin-graphql/latest |
| Migrate REST → GraphQL | https://shopify.dev/docs/apps/build/graphql/migrate |

## Quy trình thực hiện (cho agent)

### Bước 1: Quét codebase Go

- Tìm mọi file `.go` có:
  - HTTP client gọi domain Shopify (e.g. `myshopify.com`, `admin/api`).
  - Chuỗi URL chứa `/admin/api/`, `admin-rest`, REST endpoint (products, orders, customers, inventory, ...).
  - Thư viện hoặc wrapper gọi Shopify REST (nếu có).
- Liệt kê **package** và **function/service** tương ứng (tên file, tên hàm, resource REST đang dùng).

### Bước 2: Map REST resource → GraphQL

- Với từng resource REST (e.g. `GET /products`, `POST /orders`, ...):
  - Tra [Admin REST](https://shopify.dev/docs/api/admin-rest) để biết request/response hiện tại.
  - Tra [Admin GraphQL](https://shopify.dev/docs/api/admin-graphql/latest) để chọn query/mutation tương đương (type, fields, GID).
- Ghi rõ: REST endpoint ↔ GraphQL operation (tên query/mutation + type trả về). Đảm bảo dữ liệu trả về có thể map 1–1 về format cũ (hoặc giữ struct/API response không đổi cho caller).

### Bước 3: Thiết kế interface và giữ contract

- Định nghĩa (hoặc giữ nguyên) **interface** mà service implement (input args, output struct, error).
- Implement mới bằng GraphQL client; **bên trong** đổi từ REST call sang GraphQL call, **bên ngoài** vẫn cùng signature và kiểu trả về (để caller không phải sửa).
- Dùng GID khi GraphQL yêu cầu (`gid://shopify/ResourceName/id`). Convert từ numeric ID sang GID trong layer gọi GraphQL nếu cần.

### Bước 4: Implement và test

- Viết code gọi GraphQL (query/mutation), parse response, map vào output struct giống REST.
- Viết **unit test** cho từng service/hàm đã convert:
  - Mock GraphQL client (hoặc mock HTTP nếu dùng client HTTP cho GraphQL).
  - Test case: success (đủ field), lỗi API, edge case (empty, null).
  - Assert input → output và error handling.
- Đảm bảo toàn bộ test pass và coverage phù hợp (ít nhất cho path đã convert).

### Bước 5: Tài liệu và review

- Cập nhật comment/doc trong code (REST → GraphQL, link doc GraphQL nếu cần).
- Liệt kê ngắn: service nào đã convert, resource REST → operation GraphQL tương ứng, file test.

## Lưu ý kỹ thuật (Go + Shopify)

- **GraphQL endpoint**: `POST https://{shop}.myshopify.com/admin/api/{version}/graphql.json`.
- **Version**: dùng cùng version với REST hoặc version supported (latest) theo [GraphQL API versioning](https://shopify.dev/docs/api/usage/versioning).
- **ID**: REST dùng numeric ID; GraphQL dùng GID. Trong Go: hàm helper `toGID("Product", id)` / `fromGID(gid)` để không leak GID ra ngoài interface nếu contract vẫn dùng số.
- **Rate limit**: GraphQL dùng cost; cần xử lý `extensions.cost` và throttle nếu gọi nhiều.

## Khi nào dùng spec này

- User gửi prompt có nội dung: đọc toàn bộ code, phân tích Go, tìm service dùng REST Shopify, chuyển sang GraphQL, input/output giữ nguyên, unit test đầy đủ.
- Prompt có đề cập `v1.md` hoặc skill Shopify REST→GraphQL: kết hợp với skill `shopify/restToGrapql.md` và spec này để thực hiện đúng ràng buộc (same I/O, full tests).

---

**Lưu ý**: Spec này chạy trên **repo có source Go** gọi Shopify. Repo ClaudeCodeSkills không chứa Go; mở repo Go tương ứng rồi áp dụng spec (và skill REST→GraphQL) tại đó.
