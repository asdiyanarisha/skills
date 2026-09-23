# Struktur Project & Konvensi Penamaan Go

## Struktur Project Standar (wajib)

Semua project baru yang dibuat lewat skill ini WAJIB mengikuti layout berikut. Untuk project existing yang sudah punya layout ini, ikuti konvensi folder yang sama persis — jangan taruh file di folder yang salah lapisan (mis. logic bisnis di `handler.go`, atau query SQL di `service.go`).

```
myservice/
├── database/
│   ├── migration/
│   │   ├── migrations_file/       # file .sql migration, urut & bernomor/timestamp
│   │   └── migration.go           # runner migrasi (up/down)
│   ├── database.go                # koneksi/init DB (mis. *sql.DB / gorm.DB)
│   └── type.go                    # custom DB types (mis. NullString wrapper, JSON column type)
├── internal/
│   ├── app/
│   │   └── <feature>/             # satu folder per feature/domain, mis. app/user, app/order
│   │       ├── handler.go         # HTTP handler: parse request, panggil service, format response
│   │       ├── router.go          # daftar route feature ini (path + method → handler)
│   │       ├── service.go         # business logic / usecase feature ini
│   │       └── formatter.go       # HANYA fungsi mapping antar dto atau model->dto, tanpa struct/business logic
│   ├── dto/
│   │   ├── common.go              # struct dari service (non-repository) yg dipakai lintas feature: pagination, base response, dll
│   │   └── user.go                # struct request/response dari service spesifik feature (satu file per feature, mis. order.go)
│   ├── factory/
│   │   └── factory.go             # wiring/dependency injection: bikin instance repository → service → handler
│   ├── http/
│   │   └── http.go                # setup http server/router utama (register semua router feature, middleware global)
│   ├── middleware/
│   │   └── middleware.go          # auth, logging, recover, CORS, request-id, dll
│   ├── model/
│   │   ├── entity.go              # HANYA struct dari repository (entity DB); satu file per entity juga boleh, mis. user_entity.go
│   │   └── common.go              # field yang berulang di banyak entity repository (base model: ID, CreatedAt, dll)
│   └── repository/
│       └── repository.go          # akses data ke DB (query, insert, update) — satu file per domain bila membesar, mis. user_repository.go
├── pkg/
│   ├── config/
│   │   ├── app.go                 # config aplikasi (port, env, dll)
│   │   ├── config.go              # loader utama (baca env var, validasi, gabungkan semua sub-config)
│   │   └── mysql.go               # config koneksi DB
│   ├── consts/
│   │   └── mysql.go               # konstanta terkait domain/infra tertentu (nama tabel, kolom, dll) — pecah per domain kalau membesar
│   └── util/
│       └── util.go                # helper generic yang tidak spesifik domain (string, hashing, dll)
├── .gitignore
├── go.mod
├── go.sum
├── example.env                    # TEMPLATE variabel env TANPA nilai asli — satu-satunya file env yang boleh dibaca
├── main.go                        # entrypoint: load config → factory → jalankan http server
└── readme.md
```

### Aturan `internal/app/<feature>/`

- Satu feature/domain = satu folder di bawah `internal/app/` (contoh: `internal/app/user/`, `internal/app/order/`).
- Isi wajib per feature: `handler.go`, `router.go`, `service.go`, `formatter.go`. Kalau feature butuh repository sendiri, tetap taruh implementasinya di `internal/repository/`, bukan di dalam folder feature — folder feature hanya untuk layer HTTP+business logic.
- `handler.go`: HANYA parsing request (query/body/param), panggil `service`, dan format response lewat `formatter.go`. TIDAK ADA query DB atau business rule langsung di sini.
- `service.go`: business logic murni. Menerima dependency (repository, dll) lewat constructor (`NewXxxService(repo Repository) *XxxService`), dipanggil dari `factory.go`.
- `router.go`: hanya daftar route (path, method, handler) untuk feature ini — didaftarkan ke router utama lewat `internal/http/http.go`.
- `formatter.go`: lihat definisi lengkap di bagian "Aturan layer lain" di bawah — intinya hanya fungsi mapping antar `dto` atau dari `model` ke `dto`, tanpa struct baru dan tanpa business logic. Jangan campur mapping ini ke `handler.go` atau `service.go`.

### Aturan layer lain

- `internal/model/`: **HANYA** berisi struct yang merepresentasikan data dari `repository` (entity DB, punya tag `db`/`gorm`, dipakai sebagai return type query). TIDAK ADA struct lain di sini — bukan tempat untuk struct request/response API, bukan tempat untuk struct hasil olahan service. `common.go` khusus untuk field yang berulang di banyak entity (mis. `ID`, `CreatedAt`, `UpdatedAt`) yang di-embed ke entity lain. JANGAN dipakai langsung sebagai response API — selalu lewat `formatter.go` → `dto`.
- `internal/dto/`: **HANYA** berisi struct yang berasal dari `service` dan BUKAN dari repository — yaitu semua struct request & response API/HTTP layer (payload yang diterima handler, payload yang dikembalikan ke client), termasuk struct hasil komposisi/agregasi di level service yang tidak 1:1 dengan satu tabel DB. Dikelompokkan per file per feature (`user.go`, `order.go`) + `common.go` untuk yang dipakai bersama (pagination request, base response envelope, dll). Handler & service meng-import dari sini, TIDAK mendefinisikan struct request/response sendiri di file lain. Kalau sebuah struct 1:1 merepresentasikan tabel DB, itu punya `model`, bukan `dto`.
- `internal/app/<feature>/formatter.go`: **HANYA** berisi fungsi mapping/transformasi — antar `dto` (mis. gabungkan beberapa dto jadi satu response), atau dari `model` ke `dto` (mis. `entity.User` → `dto.UserResponse`). TIDAK ADA definisi struct baru di sini (struct-nya tetap didefinisikan di `model`/`dto`), TIDAK ADA business logic (validasi, kalkulasi, pemanggilan repository) — murni fungsi konversi bentuk data supaya `service.go`/`handler.go` tetap ringkas dan readable. Pola penamaan fungsi: `ToUserResponse(u *model.User) *dto.UserResponse`, `ToUserListResponse(users []*model.User) []*dto.UserResponse`.
- `internal/repository/`: satu-satunya layer yang boleh menyentuh `database.go`/melakukan query SQL. Return-nya berupa `model`, bukan `dto`. Interface repository sebaiknya didefinisikan di sisi `service.go` (consumer), implementasinya di `repository.go` — lihat pola di `references/testing.md` bagian Mocking.
- `internal/factory/factory.go`: satu-satunya tempat melakukan wiring (`repo := repository.NewUserRepository(db)`, `svc := user.NewService(repo)`, `handler := user.NewHandler(svc)`). `main.go` memanggil factory, bukan membangun dependency manual satu-satu.
- `internal/http/http.go`: setup `*http.Server`/router utama, register semua `router.go` tiap feature, pasang middleware global dari `internal/middleware/`.
- `internal/middleware/middleware.go`: middleware lintas-feature (auth, logging, recover, CORS). Middleware spesifik satu feature saja sebaiknya tetap di sini juga tapi diberi nama jelas, bukan ditaruh di folder feature.
- `pkg/config/`: loader konfigurasi dari environment variable. `config.go` sebagai entrypoint yang memanggil sub-loader (`app.go` untuk config aplikasi, `mysql.go` untuk config DB, dst — tambah file baru per sumber config, mis. `redis.go`).
- `pkg/consts/`: konstanta yang sifatnya infra/domain-specific (nama kolom, nama tabel, key cache). Pecah jadi beberapa file per domain kalau `mysql.go` membesar (mis. tambah `redis.go`, `queue.go`).
- `pkg/util/`: HANYA helper generic yang tidak bergantung pada domain bisnis (string manipulation, hashing, format tanggal). Kalau suatu fungsi hanya dipakai satu feature spesifik, taruh di `service.go`/`formatter.go` feature itu, bukan di `util`.
- `database/migration/`: file SQL migration di `migrations_file/`, diberi nama urut/timestamp (`202506231200_create_users_table.sql`). `migration.go` berisi runner (biasanya wrapper `golang-migrate` atau sejenis).

### Environment & secret — WAJIB dipatuhi

- **`.env` (dan varian `.env.local`, `.env.production`, dll) TIDAK BOLEH dibaca isinya dengan cara apa pun** — jangan `view`, `cat`, `grep`, atau tool lain terhadap file ini, walau diminta eksplisit oleh user. File ini berisi credential asli (password DB, API key, JWT secret) yang tidak boleh terekspos ke percakapan/model.
- Untuk mengetahui daftar variabel env yang dibutuhkan aplikasi (nama variabel, bukan nilainya), baca **`example.env`** — file ini adalah template publik yang isinya placeholder/kosong, aman dibaca.
- Kalau perlu menambah variabel env baru untuk fitur yang sedang dikerjakan: tambahkan nama variabelnya ke `example.env` (dengan nilai placeholder/contoh, BUKAN nilai asli) dan ke `pkg/config/` (`app.go`/`mysql.go`/file config terkait), lalu beri tahu user untuk mengisi nilai aslinya sendiri di `.env` miliknya.
- Kalau user minta debug masalah terkait environment variable, minta user menyalin/menyebutkan NAMA variabel yang bermasalah (bukan menyuruh Claude membaca file `.env`-nya), atau minta user menempel isi relevan secara manual ke chat kalau memang perlu dan mereka sadar risikonya.

## Penamaan

| Elemen | Aturan | Contoh |
|---|---|---|
| Package | huruf kecil, satu kata, tanpa `_`/camelCase | `user`, `httputil` |
| File | huruf kecil, underscore boleh untuk pemisah kata | `user_repository.go` |
| Exported func/type | PascalCase + doc comment | `func NewUserService(...) *UserService` |
| Unexported func/var | camelCase | `func validateEmail(...)` |
| Interface 1 method | akhiran `-er` | `type Reader interface { Read(...) }` |
| Const | PascalCase seperti biasa (bukan ALL_CAPS ala C) | `const MaxRetries = 3` |
| Getter | TANPA prefix `Get` | `func (u *User) Name() string`, bukan `GetName()` |

## go.mod & dependency

- Module path idealnya mengikuti path repo: `module github.com/org/myservice`.
- Jalankan `go mod tidy` setelah menambah/menghapus import — sebutkan ini ke user kalau kamu menambah dependency baru.
- Hindari dependency berat untuk hal yang bisa dilakukan stdlib (mis. jangan pakai library eksternal cuma untuk hal yang `net/http`, `encoding/json`, `strings` sudah cukup).

## Import grouping

Urutkan import dalam 3 grup dipisah baris kosong, dan `goimports` akan merapikan otomatis:
```go
import (
	"context"
	"fmt"

	"github.com/some/external/pkg"

	"github.com/org/myservice/internal/dto"
)
```