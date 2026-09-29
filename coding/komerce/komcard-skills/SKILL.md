---
name: komcard-skills
description: Panduan engineering untuk project komcards-api (Komerce) — API Go/Gin untuk virtual debit card, payment, topup/refund, transfer saldo, disbursement, voucher, dan kompoints. Gunakan skill ini SETIAP kali menulis, mereview, memperbaiki, atau merancang kode di repositori komcards-api — handler, router, service, repository, DTO, model, client, consumer RabbitMQ, cron, middleware, migration, unit test, atau file .go apa pun di repo ini. Wajib dipakai juga saat menyentuh alur keuangan (saldo, topup, refund, transfer, disbursement, fee, kompoints), keamanan (auth, PIN, API key, webhook token, enkripsi data kartu, anti-IDOR), atau saat menambah endpoint baru. Jangan pakai panduan struktur Go generik — komcards-api punya konvensi layout, error handling, dan aturan keamanan tersendiri yang dijelaskan di sini.
---

# Komcards API Engineering Standards

Skill ini adalah panduan konvensi nyata repositori **komcards-api** — service Go (Gin + GORM + MongoDB + Redis + RabbitMQ) yang menangani **virtual debit card & payment**: pembuatan kartu, topup, refund, transfer saldo, disbursement, voucher, kompoints, dan webhook Monit.

Karena ini produk **finansial**, dua hal tidak bisa ditawar:
1. **Kebenaran saldo & idempotensi transaksi** (uang tidak boleh hilang/berlipat).
2. **Keamanan** (auth, otorisasi kepemilikan resource, kerahasiaan data kartu/PIN/secret).

Skema skill ini mengikuti pola skill `golang-coding` (Prinsip Inti + reference per topik). Baca reference yang relevan **sebelum** menulis kode.

## Cara pakai skill ini

1. Terapkan **Prinsip Inti** di bawah pada setiap perubahan.
2. Baca reference sesuai tugas:

| Tugas | Baca |
|---|---|
| Menaruh file baru, penamaan, layering, layout folder, DTO vs model | `references/project-structure.md` |
| Error handling, sentinel error, response envelope, logging & alert | `references/error-handling.md` |
| Auth middleware, PIN, API key, webhook token, anti-IDOR, enkripsi kartu, secret | `references/security.md` |
| REST API Gin: handler, router, validasi, format response, status code | `references/web-api.md` |
| Repository/GORM, query, transaksi DB, row lock, Redis lock, Mongo, model | `references/repository-database.md` |
| HTTP client ke service lain, RabbitMQ consumer/producer, cron & scheduler, goroutine | `references/integration-messaging.md` |
| Unit test, mockery, table-driven test, `repository.DummyQueue` | `references/testing.md` |

Untuk tugas lintas topik (mis. "endpoint topup baru"), baca `project-structure.md` + `web-api.md` + `repository-database.md` + `security.md`.

> Catatan: repo ini juga punya `skills.md` (khusus unit testing) dan `pentest-instructions.md` (aturan security scan read-only) di root. Jangan hapus/ubah; skill ini melengkapinya.

## Prinsip Inti (selalu berlaku)

- **JANGAN PERNAH membaca isi `.env`** (atau `.env.*`). File itu berisi kredensial asli (DB, Monit, Xendit, Slack, API key). Untuk mengetahui variabel yang dibutuhkan, baca **`.env.example`** atau `config/config.go`. Jika butuh nilai, minta user mengisinya sendiri — jangan pernah menyalin nilai secret dari `.env` ke chat, kode, log, atau test.
- **Ikuti layout nyata repo**, bukan layout Go generik. Ringkasnya: `config/` di root, `database/`, `internal/app/<feature>/`, `internal/models` (plural), `internal/repository`, `internal/client`, `internal/dto`, `pkg/constants` + `pkg/helper` + `pkg/log` + `pkg/util`. Detail di `references/project-structure.md`.
- **Layering ketat**: `handler` hanya bind request → panggil service → format response. `service` berisi business logic. `repository` hanya persistence (query/insert/update), memakai **model** (bukan DTO), tanpa business logic. Jangan taruh query DB di handler atau aturan bisnis di repository.
- **Wiring lewat `factory.Factory`**: dependency dikonstruksi lewat `NewHandler(f *factory.Factory)` / `NewService(f *factory.Factory)`. Jangan bikin koneksi DB/Redis/queue/client baru di dalam service; ambil dari `f`.
- **Uang & saldo**: tidak ada magic number/string. Pakai `pkg/constants` (`constants.TRANSACTION_STATUS_SUCCESS`, `constants.EVENT_TOPUP`, `constants.CARD_STATUS_ACTIVE`, dst). Operasi yang mengubah saldo WAJIB dalam transaksi DB (`tx := repo.Begin(); defer tx.Rollback(); ...; tx.Commit()`) dan ambil baris dengan row lock (`FindOneTx` / `clause.Locking{Strength: "UPDATE"}`).
- **Idempotensi**: sebelum memproses webhook/transaksi, cek eksistensi berdasarkan `transaction_id` (mis. `constants.TransactionAlreadyInserted`). Proses per kartu dibungkus Redis lock (`lockTransaction`/`unLockTransaction`, `constants.RedisKeyIsExists`).
- **Keamanan**: setiap endpoint wajib middleware auth yang sesuai (`BearerToken`, `BearerTokenOnlyAdmin`, `BearerTokenWithPin`, `ApiKey`/`ApiKeyV2`, `CallbackTokenMonit`, `CallbackTokenXenditSrv`, `OpenKeyMiddleware`). Endpoint milik user WAJIB memfilter kepemilikan dengan `user_id` dari session (`g.Value("user")`), bukan dari body/query. Endpoint admin dipisah di group `/api/v1/admin`. Detail di `references/security.md`.
- **Validasi input**: gunakan `binding:"required"` pada DTO + `c.ShouldBindJSON`/`ShouldBindQuery`. Jangan percaya `card_id`, `user_id`, atau `amount` dari client tanpa cek kepemilikan & rentang nilai.
- **Error adalah value**: gunakan sentinel error di `pkg/constants/error.go`. Handler memetakan error ke HTTP via `helper.DefinedErrorStatusCode(err)`. Jangan bocorkan detail internal (SQL, stack, pesan upstream mentah) ke client; log detail di server.
- **Logging**: pakai `s.Log` (`*zap.Logger`, English message) untuk log terstruktur; `util.CreateErrorLog(err)` / `util.CatchInternalServerError(err)` untuk error log ke file + notifikasi Slack. Redaksi header sensitif sebelum log dengan `helper.GetSafeHeaders`.
- **Background work**: gunakan `helper.SafeGo(s.Log, func(){...})` (recover panic) untuk goroutine, bukan `go func(){}()` mentah. Jangan pakai `context.Background()` untuk mengganti `ctx` yang sudah ada — propagate `ctx` dari handler.
- **Context**: fungsi I/O (HTTP, DB, Mongo, Redis) menerima `ctx context.Context` sebagai parameter pertama.
- **Jangan mengekspos data kartu sensitif**: nomor kartu/CVV hanya lewat mekanisme reveal yang terenkripsi (`util.EncryptGCM`/`EncryptManualCardNumber`) dan hanya ke pemilik kartu/admin berwenang.
- **Comment kode berbahasa Inggris** (konvensi Go), percakapan dengan user boleh Bahasa Indonesia. Response message API yang sudah ada banyak berbahasa Indonesia — ikuti konvensi sekitar file yang diubah.

## Alur kerja menulis kode baru

0. Buat **implementation plan singkat** (scope, file yang disentuh, dependency, risiko saldo/keamanan) dan tampilkan ke user sebelum menulis kode. Jika user sudah setuju, eksekusi tanpa konfirmasi ulang tiap langkah.
1. Tentukan layer & folder (lihat `references/project-structure.md`).
2. Tulis kode mengikuti pola fitur terdekat (mis. `internal/app/card`) sebagai referensi gaya.
3. Untuk perubahan yang menyentuh saldo/transaksi: sebutkan secara eksplisit invariant yang dijaga (idempotensi, lock, tx, arah debit/kredit).
4. Sertakan unit test bila logika non-trivial (lihat `references/testing.md`). Pastikan mock sudah ada; jika belum, **infokan ke user** perintah mockery-nya sebelum generate.
5. Jalankan `gofmt`/`goimports`, `go vet`, `go test ./...` (opsional `golangci-lint run`).

## Alur kerja review/perbaikan kode

1. Baca seluruh file relevan sebelum mengubah.
2. Jalankan `go test ./...` pada package terkait dulu untuk baseline; perhatikan test yang menyentuh saldo/PIN/auth.
3. Prioritaskan temuan: **celah keamanan (IDOR/auth) > bug saldo/idempotensi/race > error handling > desain API > kosmetik**.
4. Berikan perbaikan sebagai patch/diff yang jelas; jangan tulis ulang seluruh file bila tidak perlu.
5. Jalankan ulang test; untuk kode ber-goroutine pertimbangkan `go test -race`.
6. Untuk regresi keamanan, jelaskan dampak (siapa bisa mengakses apa) dan cara memverifikasi perbaikannya.
