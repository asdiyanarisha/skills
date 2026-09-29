# Error Handling, Logging & Alerting komcards-api

## Pola utama: sentinel error di `pkg/constants/error.go`

Semua error domain didefinisikan sebagai sentinel `errors.New(...)`, contoh:

```go
var (
    CardNotFound               = errors.New("card not found")
    TransactionAlreadyInserted = errors.New("transaction already inserted")
    InsufficientBalance        = errors.New("insufficient balance")
    WrongPinNumber             = errors.New("wrong pin numbers")
    InternalServerError        = errors.New("internal server errors")
    RedisKeyIsExists           = errors.New("redis key exists")
)
```

- Pemanggil membandingkan dengan `errors.Is(err, constants.X)`.
- `constants.InternalServerError` adalah sentinel khusus yang dipetakan ke HTTP 500 (lihat `DefinedErrorStatusCode`).
- **Jangan** membuat `errors.New` ad-hoc di dalam service untuk kondisi yang sudah punya sentinel. Tambahkan sentinel baru ke file yang sesuai jika belum ada.

## Mengembalikan error (service & repository)

- Repository: boleh mengembalikan error DB apa adanya (termasuk `gorm.ErrRecordNotFound`). Service yang menerjemahkan ke sentinel domain:
  ```go
  card, err := s.CardRepository.FindOne(ctx, "id = ?", cardId)
  if err != nil {
      if errors.Is(err, gorm.ErrRecordNotFound) {
          return constants.CardNotFound
      }
      return err
  }
  ```
- Gunakan **guard clause / early return**. Hindari sarang `if`.
- Untuk error teknis yang perlu konteks, wrap dengan `%w`:
  ```go
  return fmt.Errorf("failed marshal verify pin payload: %w", err)
  ```
- Error non-domain yang tidak informatif biasanya diubah menjadi `constants.InternalServerError` lewat helper:
  ```go
  return util.CatchInternalServerError(err) // log ke file + Slack, kembalikan InternalServerError
  ```
- Jangan `panic` untuk error input/DB/network. `panic` hanya untuk invariant internal yang tidak bisa dilanjutkan (dan akan di-recover di boundary).

## Petakan error ke HTTP di handler

Helper `helper.DefinedErrorStatusCode(err)` mengembalikan **500** untuk `constants.InternalServerError`, selain itu **400**:

```go
response, err := h.service.ListCard(g, userSess, payload)
if err != nil {
    g.JSON(http.StatusBadRequest, dto.Response{
        Meta: dto.Meta{
            Status:  "failed",
            Code:    int64(helper.DefinedErrorStatusCode(err)),
            Message: err.Error(),
        },
    })
    return
}
```

Catatan penting: pola ini membuat **semua error non-internal menjadi 400**, termasuk `CardNotFound`/`Unauthorized`. Saat menambah endpoint yang butuh status berbeda (401/403/404/409), tetap konsisten dengan file sekitarnya, dan bila memungkinkan tambahkan pemetaan status yang lebih tepat alih-alih mengandalkan 400 default.

## Error dari service eksternal

- Decode body error upstream: `helper.CatchErrorMessage(res)` → `error` (dari `dto.Error`).
- Konversi kode status upstream ke sentinel, contoh `AuthClient.GetProfileUser`:
  - `400` → `constants.UserNotFound`
  - `401` → `constants.AuthServiceUnautheticated`
- `helper.CustomErrorMessage(tag, param)` menerjemahkan tag validator (`min`, `max`) menjadi pesan.

## Response envelope

Tiga bentuk envelope (lihat `internal/dto/common.go` & `error.go`):

```go
// envelope meta+data
dto.Response{ Meta: dto.Meta{Status, Code, Message}, Data: any }
// envelope untuk list kartu
dto.ResponseCard{ Meta: dto.MetaCard{...}, Data: any }
// envelope paginasi
dto.ResponsePaginate{ Meta: dto.MetaPaginate{...}, Data: any }
// envelope error / serbaguna
dto.Error{ Status, Code, Message, Data, Errors }
// dipakai middleware untuk menolak request
dto.Common{ Status: "failed", Code: 401, Message: "Unauthenticated" }
```

- `Status` biasanya `"success"` atau `"failed"`.
- Jangan mengarang envelope baru; pakai yang sudah ada.
- Untuk list, jaga `Data` tetap `[]`, bukan `null` → gunakan `util.FormattingAnyToSlice(response)` / `util.CheckEmptySlice`.

## Logging terstruktur (zap)

- Gunakan `s.Log` (`*zap.Logger`) dari factory. Pesan **Bahasa Inggris**, sertakan konteks:
  ```go
  s.Log.Error("error get card logs", zap.Any("filter", filter), zap.Error(err))
  s.Log.Info("transaction has been locked, please wait another processed", zap.String("cardId", cardId))
  ```
- Logger dikonfigurasi di `pkg/log/logger.go` (console encoder, stacktrace disembunyikan). `zap.NewNop()` dipakai di unit test.
- Jangan log DAN return error di layer yang sama berulang-ulang (duplikasi). Log di titik yang menangani (handler/consumer boundary) atau di helper error.

## Error log file + notifikasi Slack

`pkg/util/util.go` menyediakan:
- `util.CreateErrorLog(err)` → tulis `./storage/error_logs/error-YYYY-MM-DD.log` + notifikasi Slack (goroutine).
- `util.CatchInternalServerError(err)` → seperti di atas, tapi mengembalikan `constants.InternalServerError`.
- `util.NotifyCatchError(err, action, detail)`, `util.CatchCallbackError(...)`, `util.CatchMonitServerError(...)`.

`pkg/log/error.go` juga punya `log.WriteErrorLog(err)` (hanya tulis file).

Gunakan alert Slack untuk kejadian penting financial/ops (`SlackClient.SlackNotify`, `SlackNotifyWithCaller`, `SlackNotifyDecline`) — mis. gagal topup/refund/transfer, transaksi duplikat, rate-limit terpicu.

## Merahasiakan detail dari client

- **Jangan** kembalikan pesan mentah dari DB/upstream/stack trace ke client. Kembalikan sentinel/pesan umum; detail lengkap masuk log server.
- Sebelum mencatat header request ke log, buang yang sensitif dengan `helper.GetSafeHeaders` (menghapus `CLIENT-ID`, `CLIENT-SECRET`, `Authorization`, `CALLBACK-TOKEN`).
- Jangan pernah `fmt.Println`/log payload yang memuat nomor kartu penuh, CVV, PIN, atau token.

## Freezer pola yang benar

```go
tx := s.CardRepository.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()
// ... operasi ...
if err := tx.Commit().Error; err != nil {
    s.Log.Error("Error tx commit", zap.Error(err))
    return err
}
```
- Selalu ada jalan `Rollback` (defer) dan `Commit` yang dicek.
- Jangan ada `tx.Rollback()` tanpa `Commit()` pada jalur sukses, dan sebaliknya.
