# komcard-skills

Skill/panduan engineering khusus repositori **komcards-api** (Komerce) — API Go/Gin untuk virtual debit card & payment.

Skill ini adalah hasil pembacaan langsung struktur, pola error handling, dan aturan keamanan kode di repo ini, sehingga berbeda dari panduan Go generik. Isinya menyesuaikan layout nyata (`config/` di root, `internal/models`, `pkg/constants`, dst) dan menekankan dua hal yang kritis untuk produk finansial: **kebenaran saldo/idempotensi transaksi** dan **keamanan (auth, anti-IDOR, kerahasiaan data kartu)**.

## Isi

- `SKILL.md` — prinsip inti + alur kerja menulis/review kode.
- `references/project-structure.md` — layout folder, layering, penamaan, migration.
- `references/error-handling.md` — sentinel error, response envelope, logging & alert.
- `references/security.md` — middleware auth, PIN, anti-IDOR, rate limit, enkripsi kartu, secret.
- `references/web-api.md` — handler/router Gin, validasi, format & status response.
- `references/repository-database.md` — repository, GORM, transaksi DB, row lock, Redis, Mongo.
- `references/integration-messaging.md` — HTTP client, RabbitMQ consumer, cron/scheduler, goroutine.
- `references/testing.md` — mockery, table-driven test, `repository.DummyQueue`.

## Cara memakai

- Minta agent membaca `komcard-skills/SKILL.md` (dan reference terkait) sebelum mengerjakan kode Go di repo ini.
- Lihat juga `skills.md` (unit testing) dan `pentest-instructions.md` (aturan security scan) di root repo.

### Agar otomatis terdeteksi OpenCode

OpenCode memuat skill dari direktori skills global/project. Bila ingin skill ini otomatis aktif tanpa diminta, salin/symlink foldernya ke lokasi skill OpenCode, mis.:

```bash
ln -s "$(pwd)/komcard-skills" ~/.config/opencode/skills/komcard-skills
```

(`SKILL.md` sudah memiliki frontmatter `name`/`description` yang sesuai.)
