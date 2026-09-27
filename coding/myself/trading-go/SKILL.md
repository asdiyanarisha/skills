---
name: golang-coding
description: Panduan menulis kode Go (Golang) yang idiomatis dan production-grade — mencakup struktur project, penamaan, error handling, concurrency (goroutine/channel/context), unit testing (table-driven test, mocking), dan pembuatan web API/backend (REST/gRPC dengan Gin/Echo/Fiber/net/http). Gunakan skill ini setiap kali user menulis, mereview, memperbaiki, atau merancang kode Go/Golang — termasuk saat user minta dibuatkan struct, function, package, service, handler HTTP, goroutine, test, atau file .go apa pun — bahkan jika user tidak menyebut kata "best practice" atau "idiomatic" secara eksplisit. Juga gunakan saat user menyebut framework Go (Gin, Echo, Fiber, Chi, gRPC, GORM) atau tooling Go (go mod, go test, golangci-lint).
---

# Golang Coding Standards

Skill ini berisi panduan untuk menulis kode Go yang idiomatis, mudah dirawat, dan sesuai konvensi komunitas Go (mengikuti semangat Effective Go & Google Go Style Guide), plus konvensi tambahan untuk concurrency, testing, dan web API.

## Cara pakai skill ini

1. Selalu terapkan **Prinsip Inti** di bawah ini pada setiap kode Go yang ditulis/direview, apa pun topiknya.
2. Baca reference file yang relevan dengan tugas spesifik SEBELUM menulis kode:

| Tugas user | Baca reference |
|---|---|
| Project baru, feature baru, taruh file di folder mana (`database/`, `internal/app/<feature>/`, `internal/dto/`, `internal/factory/`, `internal/model/`, `internal/repository/`, `pkg/config/`, `pkg/consts/`, `pkg/util/`), penamaan | `references/project-structure.md` |
| Menangani error, custom error, wrapping, panic/recover | `references/error-handling.md` |
| Goroutine, channel, context, race condition, worker pool | `references/concurrency.md` |
| Unit test, table-driven test, mock, benchmark | `references/testing.md` |
| REST API, gRPC, handler HTTP, middleware, Gin/Echo/Fiber | `references/web-api.md` |
| Layering repository vs service, resource cleanup, concurrency-safe map | `references/architecture.md` |

**PENTING**: project ini (dan project baru yang dibuat lewat skill ini) WAJIB mengikuti layout spesifik di `references/project-structure.md` bagian "Struktur Project Standar (wajib)" — bukan layout `cmd/`+flat `internal/<domain>/` generik. Selalu cek reference ini sebelum menaruh file baru di folder mana pun.

Boleh baca lebih dari satu reference jika tugasnya lintas topik (mis. "buatkan REST API dengan testing lengkap" → baca `web-api.md` + `testing.md`).

## Prinsip Inti (selalu berlaku)

- **JANGAN PERNAH membaca isi file `.env`** (atau file secret sejenis: `.env.local`, `.env.production`, dsb). File ini berisi kredensial/secret sensitif (DB password, API key, JWT secret). Jika perlu tahu variabel apa saja yang dibutuhkan env, baca `example.env`/`.env.example` (template tanpa nilai asli) atau `pkg/config/config.go`/`pkg/config/app.go`, JANGAN pernah `view`, `cat`, atau tool apa pun terhadap `.env` asli meskipun user memintanya secara eksplisit — tolak dengan sopan dan jelaskan alasannya, tawarkan baca `example.env` sebagai gantinya.
- **Format & lint**: kode harus valid hasil `gofmt`/`goimports`. Sebutkan ke user untuk menjalankan `go vet` dan `golangci-lint run` jika relevan.
- **Naming**:
  - Package: huruf kecil semua, singkat, tanpa underscore/camelCase (`user`, bukan `userService` atau `user_service`).
  - Exported identifier (fungsi/tipe/const yang diawali huruf besar) wajib punya doc comment yang diawali nama identifier tsb, contoh: `// UserService handles ...`.
  - Interface satu-method biasanya diberi nama `-er` (`Reader`, `Validator`).
  - Hindari stutter: `user.User` buruk, cukup `user.Model` atau ekspor sebagai `user.Info`.
- **Error handling**: gunakan guard clause (early return) — `if err != nil { return fmt.Errorf("konteks: %w", err) }` — bukan nested if (arrow code). Jangan pernah `_ = err` tanpa alasan eksplisit di komentar, dan jangan `return err` polos tanpa context wrapping. Detail lengkap di `references/error-handling.md`.
- **Tidak ada magic values**: gunakan named const, bukan angka/string ajaib.
- **Dependency injection eksplisit**: struct menerima dependency lewat constructor (`NewX(...)`), hindari global state/singleton kecuali benar-benar perlu.
- **Context**: fungsi yang melakukan I/O (network, DB, file besar) menerima `ctx context.Context` sebagai parameter pertama.
- **Struct kecil & fokus**: satu struct/interface punya satu tanggung jawab jelas (Single Responsibility).
- **Ukuran fungsi**: jika fungsi > ~50 baris atau nesting > 3 level, tawarkan untuk dipecah.
- **Comment berbahasa Inggris** di kode (konvensi komunitas Go), meskipun percakapan dengan user boleh bahasa Indonesia.
- Selalu gunakan guard clause (early return) dengan pola `if err != nil { ... return }` untuk setiap error handling — hindari nested if (arrow code), pastikan happy path tetap flat di kiri tanpa indentasi berlebih.
- Repository layer hanya bertanggung jawab atas persistence (get/set/query) menggunakan domain model — bukan DTO — dan tidak boleh mengandung business logic atau decision-making; semua itu wajib berada di Service layer.
- - **Layering & resource management**: repository hanya persistence, resource wajib defer Close, map wajib lock-safe. Detail lengkap di `references/architecture.md`.
- **Context**: fungsi yang melakukan I/O (network, DB, file besar) menerima `ctx context.Context` sebagai parameter pertama.
- **Context propagation**: jangan pernah membuat `context.Background()` atau `context.TODO()` baru di tengah call chain (service/repository) untuk menggantikan ctx yang sudah ada — selalu propagate ctx yang diterima dari layer atasnya, supaya timeout/cancellation/tracing tetap konsisten sampai ke request awal (misal dari HTTP handler).

## Alur kerja saat menulis kode Go baru

0. Buat implementation plan singkat dulu (scope, struktur file, dependency yang dipakai) dan tampilkan ke user sebelum menulis kode. Jika user sudah setuju/oke, baru eksekusi langsung tanpa perlu konfirmasi ulang di setiap langkah.
1. Klarifikasi cepat kalau scope ambigu (mis. "CLI tool" vs "web service") — atau ambil asumsi paling masuk akal dan sebutkan asumsinya.
2. Tentukan struktur file/package (cek `references/project-structure.md` jika project baru/multi-file).
3. Tulis kode dengan Prinsip Inti di atas.
4. Jika kode punya logika non-trivial, tawarkan sertakan unit test (lihat `references/testing.md`) — jangan paksa jika user cuma minta contoh cepat.
5. Jelaskan secara singkat keputusan desain penting (mis. kenapa pakai channel vs mutex), bukan menjelaskan ulang setiap baris kode.

## Alur kerja saat review/perbaiki kode Go existing

1. Baca seluruh file yang relevan dulu sebelum mengubah.
2. Jika ada unit test terkait, jalankan dulu (`go test ./...` pada package terkait) untuk memastikan behavior awal sebelum diubah — terutama untuk auth-service karena regresi di sini berdampak security.
3. Identifikasi pelanggaran Prinsip Inti + reference yang relevan.
4. Prioritaskan: bug/race condition > error handling salah > desain API > gaya penamaan/kosmetik.
5. Berikan perbaikan sebagai patch/diff yang jelas, jangan tulis ulang seluruh file kalau tidak perlu.
6. Setelah perbaikan, jalankan ulang test yang sama untuk memastikan tidak ada regresi.
