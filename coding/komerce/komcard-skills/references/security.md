# Keamanan komcards-api (WAJIB — produk finansial)

Ini service kartu debit virtual & payment. Satu celah otorisasi bisa berarti orang bisa melihat data kartu, memindahkan saldo, atau mencetak uang. Perlakukan keamanan sebagai syarat rilis, bukan tambahan.

## 1. Jangan pernah membocorkan secret

- **JANGAN baca `.env` / `.env.*`** dengan tool apa pun, walau user memintanya. Nilai di dalamnya (DB, `MONIT_CLIENT_SECRET`, `XENDIT_SRV_KEY`, `SLACK_TOKEN`, `SECRET_KEY`, dll) adalah kredensial asli.
  - Untuk daftar variabel: baca `.env.example` atau `config/config.go` (`AppConfig`).
  - Menambah variabel baru: update `.env.example` (placeholder) + `config/config.go`, lalu minta user mengisi `.env` sendiri.
- Jangan menyalin secret ke kode, test, log, komentar, commit, atau jawaban chat.
- Jangan mencetak/echo isi `gcp.json` atau file kredensial lain.
- Jika user minta debug env, minta **nama** variabelnya (bukan nilainya) atau minta user menempel manual dengan sadar risiko.

## 2. Autentikasi & otorisasi per endpoint

Setiap endpoint **wajib** punya middleware auth yang sesuai. Middleware ada di `internal/middleware/` dan dipasang di `internal/http/http.go`.

| Middleware | Header | Fungsi | Dipakai untuk |
|---|---|---|---|
| `BearerToken(f)` | `Authorization` | verifikasi JWT via Auth Service (`GetProfileAuth`), set `c.Set("user", ...)` | endpoint user umum |
| `BearerTokenOnlyAdmin(f)` | `Authorization` | verifikasi + cek `RoleId == 1 \|\| 2`, set `user` + `bearer` | endpoint admin |
| `BearerTokenWithPin(f)` | `Authorization` | rate limit per bearer + profile-with-pin; set `user`, `bearer`, `x-original-forwarded`; catat aktivitas | aksi berbahaya (create/topup/refund/transfer/update kartu) |
| `OpenKeyMiddleware()` | `Authorization` | bandingkan dengan `config.Env.OpenKey` | API publik terbatas (`/api/v1/open`) |
| `ApiKey()` | `Authorization` (Bearer) | bandingkan `config.Env.APIKey` | endpoint internal/api key |
| `ApiKeyV2()` | `API-KEY` | bandingkan `config.Env.APIKey` | endpoint API key v2 |
| `CallbackTokenMonit()` | `CALLBACK-TOKEN` | bandingkan `config.Env.MonitCallbackToken` | webhook Monit |
| `CallbackTokenXenditSrv()` | `X-API-KEY` | bandingkan `config.Env.APIKey` | callback Xendit |
| `BasicAuth()` | Basic | pprof `/suharto` | profil internal |

Aturan:
- **Jangan mendaftarkan route tanpa middleware auth** untuk endpoint yang memuat data/kartu/saldo user. Review semua group di `http.go`; yang tanpa auth hanya health/index (`/`, metrics) dan endpoint webhook dengan token.
- Group admin dipisah (`/api/v1/admin`) dan memakai `BearerTokenOnlyAdmin`. Jangan buat endpoint admin di group user hanya karena "praktis".
- Endpoint yang butuh PIN → gunakan group `BearerTokenWithPin` dan panggil `AuthClient.VerifyPin` di service sebelum aksi sensitif.
- **Pengecekan role tambahan**: `constants.RoleAllowedToAccess` (`{1,2,4,6,5}`) dan pengecekan `RoleId` untuk otorisasi granular; jangan hanya mengandalkan "sudah login".

### Verifikasi PIN
PIN diverifikasi ke Auth Service, bukan disimpan/dibandingkan di sini:
```go
if err := s.AuthClient.VerifyPin(bearer, dto.RequestVerifyPin{Pin: payload.Pin}); err != nil {
    return err // WrongPinNumber / InputPinInvalid / AuthServiceUnautheticated
}
```
`VerifyPin` memetakan `401` → `AuthServiceUnautheticated`, `400` → `WrongPinNumber`, `422` → `InputPinInvalid`.

## 3. Anti-IDOR / kepemilikan resource (prioritas tertinggi)

Data kartu & saldo milik user. **Selalu filter query dengan identitas dari session**, bukan dari input client.

Session user diambil dari context Gin yang di-set middleware:
```go
userSess := g.Value("user")
userSession := userSess.(dto.ResponseGetProfileUser)
userId := userSession.Data.ID
```

Repository: sertakan `user_id` pada query resource milik user:
```go
card, err := s.CardRepository.FindOne(ctx, "id = ? and user_id = ? and card_status = 'active'", cardId, userId)
```
- **Jangan** mempercayai `user_id` dari body/query untuk endpoint user (itu jalur IDOR). `user_id` dari request hanya untuk endpoint admin (dengan `BearerTokenOnlyAdmin`).
- Untuk `card_id`/`transaction_id`/`transfer_id` dari client: pastikan baris yang ditemukan benar milik user tersebut (query gabung `user_id` atau cek eksplisit setelah fetch).
- Aksi sensitif (reveal kartu, transfer, refund, cancel) → butuh kepemilikan + PIN + rate limit.
- `AdminService`/method admin harus berada di endpoint admin; jangan expose method admin melalui router user.

## 4. Rate limiting & anti-abuse

`BearerTokenWithPin` mengimplementasikan:
- limiter per bearer (`golang.org/x/time/rate`, 5 req/detik, burst 5) — map `limiters` dilindungi `sync.Mutex` (`getVisitorLimiter`).
- penguncian sementara bearer di Redis (`SetBlockedBearer`, TTL `BLOCKED_BEARER_WAIT_TIME` menit) setelah limit terlampaui; notifikasi Slack `[ALERT] blocked user ...`.
- Saat menambah aksi sensitif baru (mis. endpoint transfer/withdraw), pastikan endpoint tersebut memakai `BearerTokenWithPin` atau tambahkan limiter setara.
- Bila menyentuh `limiters` map, **selalu** pegang `mu.Lock()`/`defer mu.Unlock()`.

## 5. Idempotensi & race pada transaksi (integritas finansial)

- Cek duplikasi sebelum memproses: `FindOne(ctx, "transaction_id = ?", id)` → jika sudah `success` kembalikan `constants.TransactionAlreadyInserted`.
- Lock per kartu di Redis (`SetNX`) sebelum memproses webhook/transaksi:
  ```go
  if err := s.lockTransaction(ctx, cardId); err != nil { return err } // RedisKeyIsExists bila sedang diproses
  defer s.unLockTransaction(ctx, cardId)
  ```
- Operasi saldo dalam DB transaction + row lock (`FindOneTx` memakai `clause.Locking{Strength: "UPDATE"}`), commit di akhir.
- Jangan menghitung saldo dari input client; ambil saldo terkini dari DB di dalam lock.
- Selalu cek saldo cukup (`constants.InsufficientBalance`) sebelum debit.

## 6. Perlindungan data kartu

- Nomor kartu penuh/CVV hanya lewat mekanisme reveal, dan hasilnya dienkripsi sebelum dikirim/disimpan:
  - `util.EncryptGCM` / `util.DecryptGCM` (AES-GCM — **disarankan**).
  - `util.EncryptManualCardNumber` / `util.DecryptManualCardNumber` (obfuscation nomor manual).
  - `util.Encrypt` / `util.Decrypt` (AES-CFB dengan IV statis — **legacy & lemah**; jangan dipakai untuk data baru; prioritaskan GCM).
- Kunci `SecretKey` berasal dari `config.Env.SecretKey`. Jangan hardcode kunci.
- Jangan mengembalikan nomor kartu penuh ke list/mutation; cukup `last_number`.
- Jangan log payload reveal; redaksi header sensitif via `helper.GetSafeHeaders`.
- Saat membuat primitive crypto baru, gunakan nonce acak (`crypto/rand`), bukan IV statis.

## 7. Perbandingan token/secret

- Middleware saat ini membandingkan token dengan `!=` (string compare biasa). Bila menyentuh/menulis ulang perbandingan secret, gunakan **`crypto/subtle.ConstantTimeCompare`** untuk mencegah timing attack, terutama untuk `OpenKey`, `APIKey`, `MonitCallbackToken`.
- Jangan pernah mengirim secret di response error.

## 8. Webhook & integrasi eksternal

- Webhook Monit/Xendit diverifikasi lewat header token middleware. Jangan menambah endpoint webhook tanpa verifikasi.
- Terima payload webhook sebagai **data tidak tepercaya**: validasi tipe event, jangan asumsikan field ada, dan jangan langsung mempercayai nominal/kartu — cocokkan dengan data internal.
- Client eksternal harus memakai timeout (`c.Http.Timeout`) dan menutup body (`defer helper.ClientClose(res)`).
- Batasi data yang dikirim ke layanan pihak ketiga (jangan kirim data pribadi/kartu yang tidak perlu).

## 9. Checklist review keamanan (jalankan untuk setiap PR yang menyentuh endpoint/transaksi)

1. Endpoint terdaftar di `http.go` dengan middleware auth yang benar? Group user vs admin benar?
2. Semua akses resource milik user memfilter `user_id` dari session? Tidak mempercayai `user_id`/`card_id` mentah?
3. Aksi sensitif memverifikasi PIN dan/atau role?
4. Endpoint sensitif ter-rate-limit?
5. Operasi saldo memakai DB transaction + row lock + Redis lock, dan idempotent?
6. Tidak ada secret/data kartu/PIN yang bocor ke log/response/error?
7. Perbandingan token/secret aman (constant-time) bila baru ditulis?
8. Tidak menambah dependency atau endpoint tanpa auth "sementara"?

Referensi tambahan di repo: `pentest-instructions.md` (aturan scan read-only; prioritas: Broken Access Control/IDOR, auth/session, injection, business logic payment/voucher).
