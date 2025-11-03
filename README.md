# Simple Bank — Kiến trúc dự án

Tài liệu ngắn gọn về kiến trúc và cấu trúc mã của dự án "simple-bank". Mục tiêu: giúp một lập trình viên mới nhanh chóng hiểu luồng dữ liệu, các thành phần chính, nơi đặt schema/migration, mã sinh (sqlc), và cách chạy/gỡ lỗi cục bộ.

## Tổng quan

`simple-bank` là một ứng dụng mẫu viết bằng Go, tách biệt rõ ràng giữa:

- Schema & migration (thư mục `db/migration`).
- Các truy vấn SQL định nghĩa API dữ liệu (thư mục `db/query`).
- Mã Go được sinh bởi `sqlc` (`db/sqlc`).
- Lớp store/transactional wrapper (`db/sqlc/store.go`, `db/sqlc/store_test.go`).
- Các test đơn vị để kiểm tra logic thao tác DB (`db/sqlc/*_test.go`).
- Mã tiện ích dùng chung (`util/random.go`).

Kiến trúc hướng tới tách biệt rõ ràng giữa SQL (nguồn chân thực của truy vấn) và mã Go (đóng gói truy vấn vào các phương thức có tính giao dịch). Điều này giúp: an toàn kiểu (type-safe queries), dễ bảo trì khi schema thay đổi, và test từng phần.

## Sơ đồ kiến trúc (textual)

Client/API layer (tương lai)
-> Store / Service layer (`db/sqlc/store.go`)
-> SQLC generated queries (`db/sqlc/*.go`) <-- queries defined in `db/query/*.sql`
-> PostgreSQL database (migrations in `db/migration`)

Data flow chính:

1. Migrations trong `db/migration/*.up.sql` tạo bảng và ràng buộc.
2. Các truy vấn SQL (SELECT/INSERT/UPDATE...) nằm ở `db/query/*.sql`.
3. `sqlc` đọc `db/query/*.sql` và `sqlc.yaml`, sinh mã Go trong `db/sqlc`.
4. `store.go` (wrapper) sử dụng mã sinh để đóng gói thao tác đơn vị công việc (ví dụ: chuyển tiền trong 1 transaction).
5. Tests khởi tạo state DB (thường bằng migrations) và kiểm tra behavior của store.

## File chính và mục đích

- `go.mod` — quản lý module Go.
- `Makefile` — (nếu được cấu hình) có thể chứa target hữu ích (build, test, migrate). Kiểm tra file để biết lệnh sẵn có.
- `db/migration/` — các migration SQL: `000001_...up.sql` / `...down.sql`.
- `db/query/` — nơi để định nghĩa truy vấn SQL sạch (nguồn để `sqlc` sinh code):
  - `account.sql`, `entry.sql`, `transfer.sql`.
- `db/sqlc/` — mã sinh + wrapper + tests:
  - `models.go` — struct type do sqlc sinh hoặc được đặt chung.
  - `db.go`, `store.go` — phần kết nối DB và logic transaction.
  - `*_test.go` — test cho các thao tác DB.
- `util/random.go` — helper dùng trong test để sinh dữ liệu mẫu.

## Schema & Migrations

Migrations có trong `db/migration/` và được sắp xếp bằng số tiền tố. Áp dụng chúng để tạo schema trước khi chạy ứng dụng hoặc test.

Ví dụ nhanh (dùng Docker Postgres + psql):

```powershell
# Chạy Postgres cục bộ
docker run --name simplebank-db -e POSTGRES_PASSWORD=pass -e POSTGRES_USER=user -e POSTGRES_DB=simple_bank -p 5432:5432 -d postgres:15

# Áp migration (đơn giản bằng psql, thay connection string nếu cần)
psql "postgresql://user:pass@localhost:5432/simple_bank?sslmode=disable" -f db/migration/000001_init_schema.up.sql
psql "postgresql://user:pass@localhost:5432/simple_bank?sslmode=disable" -f db/migration/000002_add_not_null_to_entries_account_id.up.sql
```

Lưu ý: có nhiều công cụ migration (migrate, goose, golang-migrate...). Nếu dự án đã có Makefile target cho migration, ưu tiên dùng target đó.

## SQLC (code generation)

`sqlc` đọc các file `.sql` trong `db/query/` và cấu hình trong `sqlc.yaml` để sinh mã Go type-safe vào `db/sqlc`.

Regenerate code khi thay đổi `.sql` hoặc `sqlc.yaml`:

```powershell
sqlc generate
```

Sau khi chạy, kiểm tra `db/sqlc` để thấy các file `*_sql.go` và models.

## Store layer & giao dịch

`db/sqlc/store.go` đóng gói những thao tác phức tạp cần transactional semantics (ví dụ: chuyển tiền giữa 2 tài khoản).

Contract ngắn gọn của store:

- Inputs: kết nối \*sql.DB hoặc tx, các tham số nghiệp vụ (amount, account IDs,...).
- Outputs: domain models (Account, Entry, Transfer), error nếu thất bại.
- Error modes: trả về lỗi khi giao dịch không thành công, rollback tự động.

Edge cases cần quan tâm:

- Số dư âm / kiểm tra ràng buộc hiển nhiên.
- Deadlocks khi nhiều transaction song song (cần ordering/locking).
- Kiểm tra null/zero values theo migration.

## Tests

Test có sẵn nằm trong `db/sqlc/*_test.go`. Thông thường chúng:

- Khởi tạo hoặc sử dụng DB test instance.
- Chạy migrations để đảm bảo schema.
- Sử dụng helper trong `util/` để tạo dữ liệu ngẫu nhiên.

Chạy tất cả test (có thể cần DB):

```powershell
go test ./...
```

Nếu test phụ thuộc DB, chạy Postgres cục bộ hoặc trong Docker trước khi chạy test, và set biến môi trường kết nối phù hợp (ví dụ `DATABASE_URL` hoặc `DB_SOURCE`).

## Cách chạy nhanh (dev)

1. Start Postgres (Docker).
2. Áp migration (psql hoặc tool migration project dùng).
3. Chạy `sqlc generate` nếu bạn sửa `.sql`.
4. Chạy unit tests: `go test ./...`.

Mở rộng để chạy API (nếu có): thêm phần server/handler, wire store vào HTTP handlers.

## Gợi ý cải tiến / next steps

- Thêm Docker Compose để khởi động Postgres + service cùng lúc.
- Thêm CI workflow (GitHub Actions) để chạy `sqlc generate` check + `go test`.
- Thêm end-to-end tests cho luồng chuyển tiền.
- Thêm swagger/openapi nếu muốn public API.

## Ghi chú cho maintainer

- Khi thay đổi `db/query/*.sql`, luôn chạy `sqlc generate` và commit mã sinh.
- Migrations là nguồn chân thực cho schema — sửa bằng migration mới, không chỉnh trực tiếp bảng production.

---

Nếu bạn muốn, tôi có thể:

1. Thêm một `docker-compose.yml` mẫu để dev nhanh.
2. Tạo Makefile targets (migrate, sqlc, test) nếu chưa có.
3. Viết phần README bằng tiếng Anh nếu cần.

Hãy cho biết bạn muốn tôi commit trực tiếp README này vào repo (tôi có thể tạo patch và apply).
