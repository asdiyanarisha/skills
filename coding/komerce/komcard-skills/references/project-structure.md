# Struktur Direktori & Layering komcards-api

## Layout nyata repositori (WAJIB diikuti)

```
komcards-api/
├── config/
│   └── config.go              # AppConfig + loader env (godotenv + envconfig). Global: config.Env
├── database/
│   ├── database.go            # koneksi MySQL (GORM), MongoDB, Redis (singleton + sync.Once)
│   ├── type.go                # custom DB type
│   └── migration/             # file .sql bernomor: 000001_*.up.sql / 000001_*.down.sql
├── internal/
│   ├── app/<feature>/         # satu folder per domain
│   │   ├── handler.go         # HTTP handler (bind request → service → response)
│   │   ├── router.go          # daftar route feature (method, path, middleware)
│   │   ├── service.go         # business logic; sering berisi interface Service + struct service
│   │   ├── interface.go       # interface Service (dipisah, mis. di card) — sebagian fitur taruh di service.go
│   │   ├── formatter.go       # fungsi mapping model<->dto / agregasi dto (tanpa business logic)
│   │   ├── consumer.go        # RabbitMQ consumer (untuk fitur yang punya worker)
│   │   └── cron.go            # entrypoint scheduler fitur
│   ├── client/                # HTTP client ke service eksternal (Auth, Monit, Xendit, Komship, Slack, WA, ...)
│   │   └── mocks/             # mockery mock untuk client
│   ├── dto/                   # request/response API + struct hasil service (BUKAN entity DB)
│   ├── factory/
│   │   └── factory.go         # SATU-SATUNYA tempat wiring semua repository/client/service
│   ├── http/
│   │   └── http.go            # setup Gin engine, register semua router + middleware
│   ├── middleware/            # auth, api key, callback token, CORS, monitoring
│   ├── models/                # ENTITY DB (GORM/Mongo) — plural, satu file per entity
│   ├── repository/            # query DB; interface + implementasi; ada cache.go (Redis)
│   │   └── mocks/             # mockery mock untuk repository
│   └── scheduler/
│       └── scheduler.go       # registrasi cron (robfig/cron), graceful shutdown
├── pkg/
│   ├── constants/             # sentinel error + konstanta domain (status, event, key cache)
│   ├── helper/                # helper lintas domain (validate, error, client, format, goroutine)
│   ├── log/                   # zap logger + writer error log file
│   └── util/                  # helper generik (crypt, db, format, version)
├── metrics/metrics.go         # Prometheus
├── rabbitmq/connection.go     # koneksi RabbitMQ
├── main.go                    # entrypoint multi-mode (lihat bawah)
├── .env.example               # template env (yang boleh dibaca)
├── .env                       # JANGAN DIBACA
└── Makefile                   # run/test/build
```

Perbedaan penting dari skill Go generik: **tidak ada `cmd/`**, `config/` ada di root (bukan `pkg/config`), model di `internal/models` (plural), konstanta di `pkg/constants` (bukan `consts`).

## Aturan tiap layer

### `internal/app/<feature>/handler.go`
- Hanya: bind request (`ShouldBindJSON`/`ShouldBindQuery`), ambil session `g.Value("user")` (dan `g.Value("bearer")` bila perlu), panggil service, tulis response.
- Tidak ada query DB, tidak ada aturan bisnis, tidak ada panggilan client langsung.
- Struct handler sering **unexported**: `type handler struct { service Service }`, konstruktor `func NewHandler(f *factory.Factory) *handler`. Perhatikan: beberapa fitur memakai `Handler` exported (mis. `transaction`). Ikuti gaya file/fitur sekitar.
- Router memakai method receiver yang sama, contoh di `router.go`:
  ```go
  func (h *handler) CardRouter(g *gin.RouterGroup) {
      g.GET("/list", h.ListCardHandler)
      g.POST("/:card_id/show", h.ShowCardHandler)
  }
  ```

### `internal/app/<feature>/router.go`
- Hanya deklarasi route + middleware feature. Router didaftarkan dari `internal/http/http.go`.
- Middleware auth dipasang per group. Jangan menduplikasi middleware global di sini.

### `internal/app/<feature>/service.go`
- Business logic. Interface `Service` bisa di `interface.go` (card) atau di atas struct di `service.go` (transaction).
- Struct `service` menerima seluruh dependency dari `factory.Factory`:
  ```go
  func NewService(f *factory.Factory) Service {
      return &service{
          CardRepository: f.CardRepository,
          MonitClient:    f.MonitClient,
          Log:            f.Logger,
          Queue:          f.QueueRepository,
      }
  }
  ```
- Service boleh memanggil service fitur lain lewat konstruktor (mis. `monitService.NewService(f)` di card), tetapi hindari circular import — kalau muncul, pecah helper atau panggil lewat interface.

### `internal/app/<feature>/formatter.go`
- Hanya fungsi konversi: `model` → `dto`, atau komposisi antar `dto`. Tanpa struct baru, tanpa business logic.
- Contoh nyata: `formatLogActionRiskReview(...) dto.LogRequest`.

### `internal/dto/`
- Semua struct request/response API dan struct hasil olahan service. Satu file per fitur (`card.go`, `transaction.go`, `user.go`, ...), plus `common.go` untuk envelope & pagination bersama.
- Konvensi nama: `Request*`/`Payload*` untuk input, `Query*`/`*Parameter*` untuk query param, `Response*` untuk output. Gunakan blok `type ( ... )`.
- Tag: `json` + `form` (untuk query) + `binding:"required"` (validator).
- **Struct yang 1:1 dengan tabel DB bukan DTO** — taruh di `internal/models`.

### `internal/models/`
- Entity DB (GORM/Mongo). Satu file per entity, sering memakai blok `type ( ... )` berisi beberapa varian (`Card`, `CardUpdate`, `CardSimple`, `TotalCardByUser`).
- Tag: `json:"..."` dan `gorm:"column:..."`. Query builder butuh `TableName()`:
  ```go
  func (CardUpdate) TableName() string { return "cards" }
  ```
- Model TIDAK dipakai langsung sebagai response API; lewat `formatter.go` → `dto`.

### `internal/repository/`
- Satu file per domain (`card.go`, `card_transaction.go`, ...), plus `cache.go` untuk Redis dan `queue_repository.go` untuk RabbitMQ.
- Interface dan implementasi dalam file yang sama. Interface besar boleh dipecah (`CardFind`, `CardTx`, `CardAggregate`) lalu di-embed ke `CardRepositoryInterface`.
- Return type = `models`, bukan `dto`. Tanpa business logic.
- Metode transaksi: `Begin()`, `CreateTx(tx, ...)`, `FindOneTx(tx, ...)`, `UpdateTx(tx, ...)`.

### `internal/client/`
- HTTP client ke service eksternal. Punya interface (`AuthClientInterface`, `MonitClientInterface`, dst) supaya bisa di-mock.
- `NewXxxClient(logger)` membaca konfigurasi dari `config.Env` (base URL, path, key) — bukan hardcode.
- Selalu `defer helper.ClientClose(res)` setelah request sukses dibuat; cek `res.StatusCode`.

### `internal/factory/factory.go`
- Satu-satunya tempat wiring. `NewFactory(mgDb)` menginisialisasi DB, RabbitMQ, Redis, logger, lalu mengembalikan struct `Factory` berisi semua dependency.
- Menambah dependency baru = tambah field di `Factory` + konstruksi di `NewFactory`, lalu pakai di service via `f.NamaDependency`.

### `internal/http/http.go`
- Setup `gin.Engine`, urutan middleware global (`CORSMiddleware`, `MonitoringActivity`), lalu register semua router per group dengan middleware auth masing-masing.
- Saat menambah endpoint: daftarkan router di sini, pilih group + middleware yang tepat (jangan lupa auth!).

### `main.go` (multi-mode)
Satu binary menjalankan banyak peran lewat flag `-m`:
- `-m consumer_callback` → Monit webhook consumer
- `-m consumer_operation` / `-m consumer_balance_monitor`
- `-m consumer_ticket -t create|solved`
- `-m scheduler` → cron
- default → HTTP API (Gin)

Saat menambah peran baru, tambahkan cabang di `main.go` dan Dockerfile/Makefile terkait.

## Penamaan

| Elemen | Aturan | Contoh |
|---|---|---|
| Package | lowercase, satu kata | `card`, `repository`, `helper` |
| File | lowercase, underscore antar kata | `card_transaction.go` |
| Handler struct | ikuti fitur (`handler` atau `Handler`) | `type handler struct` |
| Konstruktor | `NewHandler(f)`, `NewService(f)`, `NewXxxRepository(db)` | `NewCardRepository(db)` |
| Method service | akhiran `Service`/`Handler` pada nama method publik | `CreateCardService`, `ListCardHandler` |
| Const | PascalCase / SCREAMING sesuai file | `CARD_STATUS_ACTIVE`, `CardLevelGold` |
| Sentinel error | PascalCase, di `pkg/constants/error.go` | `constants.CardNotFound` |
| Field JSON | snake_case | `card_id`, `transaction_type` |
| Field GORM | `gorm:"column:snake_case"` | `gorm:"column:card_id"` |

## Dependency & import
- Module: `module komcards-api` (bukan path github). Import internal: `komcards-api/internal/...`.
- Framework: **Gin**. ORM: **GORM (MySQL)**. NoSQL: **MongoDB**. Cache/lock: **Redis**. Queue: **RabbitMQ**. Validasi: **validator/v10**. Log: **zap** (+ logrus legacy). Cron: **robfig/cron**.
- Setelah menambah/menghapus import jalankan `go mod tidy`.
- Jangan menambah dependency berat untuk hal yang bisa dilakukan stdlib.

## Migration
- Simpan di `database/migration/`, format `00000N_nama.up.sql` dan `.down.sql` (berurutan, berpasangan).
- Perubahan skema harus disertai down migration yang benar-benar membalikkan perubahan.
- Jangan mengubah migration yang sudah dirilis; buat migration baru.
