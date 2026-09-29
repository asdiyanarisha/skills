# Repository, Database & Transaksi (GORM, MongoDB, Redis)

## Repository = persistence saja

- Satu file per domain di `internal/repository/` (`card.go`, `card_transaction.go`, ...). Interface + implementasi diletakkan dalam file yang sama.
- Return type = `models`, **bukan** `dto`. Tidak ada aturan bisnis, tidak ada keputusan (mis. pemilihan nilai fallback, perhitungan saldo) di repository.
- Interface boleh dipecah dan di-embed:
  ```go
  type CardRepositoryInterface interface {
      Create(ctx context.Context, data *models.Card) error
      Count(ctx context.Context, query string, args ...interface{}) (int, error)
      Update(ctx context.Context, fields string, updatedField models.Card, query string, args ...interface{}) error
      CardFind
      CardTx
      CardAggregate
  }
  ```
- Konstruktor: `func NewCardRepository(db *gorm.DB) *CardRepository`.
- Query memakai `WithContext(ctx)` supaya cancellation/timeout ikut:
  ```go
  func (r *CardRepository) FindOne(ctx context.Context, query string, args ...interface{}) (models.Card, error) {
      var card models.Card
      err := r.Database.WithContext(ctx).Model(models.Card{}).Where(query, args...).First(&card).Error
      if err != nil {
          return models.Card{}, err
      }
      return card, nil
  }
  ```

## Model (GORM)

- Entity di `internal/models/<entity>.go`, sering memakai blok `type ( ... )` dengan varian: entity utama, struct update parsial, struct agregat/query-result, dan DTO-repository.
- Tag `json` + `gorm:"column:..."`.
- Query builder butuh `TableName()`, contoh `func (CardUpdate) TableName() string { return "cards" }`.
- Jangan pakai model langsung sebagai response API — mapping lewat `formatter.go` → `dto`.
- Nilai status/level/event JANGAN hardcode string; ambil dari `pkg/constants` (`constants.CARD_STATUS_ACTIVE`, `constants.TRANSACTION_STATUS_SUCCESS`, `constants.CardLevelGold`, ...).

## Transaksi DB

Pola di repo (lihat juga `references/error-handling.md`):

```go
tx := s.CardRepository.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()

card, err := s.CardRepository.FindOneTx(tx, "id,user_id,card_id,balance", "id = ?", missData.CardId)
if err != nil {
    tx.Rollback()
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return constants.CardNotFound
    }
    return err
}

// ... update/insert via *Tx(tx, ...) ...

if err := tx.Commit().Error; err != nil {
    s.Log.Error("Error tx commit", zap.Error(err))
    return err
}
```

- Interface `CardTx`/`...Tx` menyediakan `Begin()`, `FindOneTx(tx, ...)`, `CreateTx(tx, ...)`, `UpdateTx(tx, ...)`.
- `FindOneTx` memakai `clause.Locking{Strength: "UPDATE"}` → **row lock** untuk mencegah race saldo. Gunakan varian `Tx` (bukan non-Tx) untuk baca yang akan diubah dalam transaksi yang sama.
- Selalu `Commit()` yang dicek; rollback pada setiap jalur error dan panic.
- Jangan membaca saldo di luar transaksi lalu menulis berdasarkan nilai itu.

## Redis (cache & lock)

`internal/repository/cache.go` (`CacheInterface`):
- `SetTransactionLock` / `DelTransactionLock` → lock per kartu (TTL detik), memakai `SetNX`; bentrok → `constants.RedisKeyIsExists`.
- `SetRiskReviewLock` / `GetRiskReviewLock` / `DelRiskReviewLock`.
- `SetBlockedBearer` / `GetBlockedBearer` → rate limit/blocked bearer.
- `SetBearer` / `GetBearer` (token Monit GraphQL), `SetUser` / `GetUser`.

Aturan:
- Lock harus selalu dilepas (`defer s.unLockTransaction(...)`).
- Tangani `constants.RedisKeyIsExists` secara eksplisit (artinya proses lain sedang berjalan, bukan error tak terduga).
- Key Redis dibangun dari konstanta di `pkg/constants` (`constants.TransactionLockCache`, dll) — jangan menulis key ad-hoc.

## MongoDB

- Client dibuka di `main.go` via `database.MongoConnect(ctx, config.Env.MongoDb)`, wajib `defer database.MongoClose(ctx, client)`.
- Di factory, `*mongo.Database` (mgDb) dioper ke repository Mongo (`NewCardLogRepository(logger, mgDb)`, `NewDeclineTransactionRepository`, `NewLogClient`, dst).
- Query memakai `bson.D`/`bson.M` dan `primitive.NewDateTimeFromTime` untuk waktu. Tangani `mongo.ErrNoDocuments` (bandingkan dengan struct kosong seperti `models.UserProfile{}`).

## Konvensi query yang aman

- Selalu pakai **placeholder** (`?`), jangan menyusun string dari input user:
  ```go
  s.CardRepository.FindOne(ctx, "id = ? and user_id = ? and card_status = 'active'", cardId, userId)
  ```
- Sertakan `deleted_at is null` untuk tabel ber-soft-delete bila relevan.
- Untuk endpoint user, sertakan `user_id = ?` (lihat `references/security.md`).
- `.Debug()` banyak dipakai di repo ini untuk memunculkan query di log saat development; pertimbangkan dampak noise/performa bila menambahkannya.

## Error DB

- `gorm.ErrRecordNotFound` diterjemahkan service ke sentinel domain (`constants.CardNotFound`, `constants.TransactionNotFound`, ...).
- Error DB lain: biarkan mengalir atau `util.CatchInternalServerError(err)`; jangan bocorkan SQL ke client.

## Resource management

- Setiap resource yang dibuka di-`defer Close()`: HTTP response body (`helper.ClientClose`), file, dsb.
- Koneksi DB/Redis/RabbitMQ adalah singleton yang dikelola `database/` dan `rabbitmq/` — jangan membuat koneksi baru di service/repository.
