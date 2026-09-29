# Integrasi: HTTP Client, RabbitMQ, Cron, Goroutine

## HTTP client ke service lain (`internal/client/`)

- Setiap service eksternal punya client + interface supaya bisa di-mock: `AuthClientInterface`, `MonitClientInterface`, `XenditClientInterface`, `KomshipClientInterface`, `SlackClientInterface`, `EasyWaClientInterface`, `WaServiceClientInterface`, `LogClientInterface`, `ApiClientInterface`, `GlobalConsumerClientInterface`, `MonitGraphqlClientInterface`.
- Konstruktor membaca konfigurasi dari `config.Env` (base URL, path, key), mis. `NewAuthClient(logger)`.
- Pola pemanggilan:
  ```go
  func (c *AuthClient) GetProfileAuth(bearerToken string, path string) (dto.ResponseGetProfileUser, error) {
      source := c.BaseUrl + path
      headersMap := map[string]string{"Authorization": bearerToken}
      c.Http.Timeout = 30 * time.Second

      res, err := helper.GetRequest(c.Http, source, nil, headersMap)
      if err != nil {
          return dto.ResponseGetProfileUser{}, err
      }
      defer helper.ClientClose(res)

      if res.StatusCode != 200 {
          return dto.ResponseGetProfileUser{}, helper.CatchErrorMessage(res)
      }

      var response dto.ResponseGetProfileUser
      if err := json.NewDecoder(res.Body).Decode(&response); err != nil {
          log.WriteErrorLog(err)
          return dto.ResponseGetProfileUser{}, constants.InternalServerError
      }
      return response, nil
  }
  ```
- Helper HTTP tersedia di `pkg/helper/client.go`: `GetRequest`, `PostRequest`, `PutRequest`, `DeleteRequest`, `ClientClose`, `GetSafeHeaders`.
- Aturan: set timeout, `defer helper.ClientClose(res)` tepat setelah sukses, petakan status upstream ke sentinel, jangan bocorkan body mentah ke client, dan **redaksi header sensitif** jika di-log (`GetSafeHeaders`).
- Path endpoint eksternal berasal dari env (`PATH_AUTH_*`, `MONIT_*`, `XENDIT_*`). Jangan hardcode URL/path.

## RabbitMQ

Koneksi: `rabbitmq/connection.go` (`InitConnection`). Repository antrean: `internal/repository/queue_repository.go` (`QueueInterface`, `Queue`).

- Exchange tipe `x-delayed-message` (delayed retry), queue durable, `Qos` prefetch.
- Interface: `InitializeQueue(queue, exchange)`, `DeclareExchangeWithQueue(prefetch)`, `Consume(consumerName) <-chan amqp.Delivery`, `Publish(ctx, amqp.Publishing)`, `WithExchange(exchange)`.
- `Publish` sudah punya retry-reconnect internal (5x) — gunakan itu, jangan bikin loop publish sendiri.
- Consumer didefinisikan di `internal/app/<feature>/consumer.go` dengan struct `Consumer`, konstruktor `NewConsumer(f *factory.Factory)`, dan method entry (mis. `CallbackConsumer()`) yang dipanggil dari `main.go` berdasarkan flag `-m`.

Pola consumer (dari `internal/app/monit/consumer.go`):
```go
jobs := queue.Consume("consumerTransaction")
var forever chan struct{}
go func() {
    for job := range jobs {
        var body dto.MonitWebhook
        _ = json.Unmarshal(job.Body, &body)
        if err := c.Service.TransactionService(context.Background(), body); err != nil {
            go c.HandlingConsumerError(context.Background(), body, err)
        } else {
            go c.DeleteErrorCallbackLogs(context.Background(), body)
        }
        _ = job.Ack(true)
    }
}()
<-forever
```

Aturan consumer:
- **Idempotensi**: webhook/event bisa datang lebih dari sekali. Cek `transaction_id` & status sebelum memproses; gunakan `constants.TransactionAlreadyInserted`.
- **Retry berjenjang**: `HandlingConsumerError` menentukan `publish` ulang berdasarkan tipe error + `callbackLog.Retries` vs `config.Env.MaxRetries*`; catat ke `callback_logs` (insert/update) dan notifikasi Slack. Ikuti pola ini; jangan retry tanpa batas.
- Ack job setelah diproses (atau setelah dijadwalkan retry), supaya tidak infinite redelivery.
- Simpan jejak error di log + `util.CreateErrorLog`.
- Jangan mempercayai payload webhook mentah (validasi tipe event, nominal, kartu).

Publishing dari service:
```go
queue := s.Queue.WithExchange(config.Env.RabbitmqExchange)
// ... marshal payload ...
if err := queue.Publish(ctx, amqp.Publishing{
    ContentType: "application/json",
    Body:        payloadByte,
    Timestamp:   time.Now(),
}); err != nil { ... }
```

## Cron & Scheduler

- Registrasi cron di `internal/scheduler/scheduler.go` (`cron.New()` + `c.AddFunc(config.Env.*_CRON, handler)`), dijalankan dengan flag `-m scheduler`.
- Handler fitur ada di `cron.go` fitur (mis. `transaction/cron.go` → `SchedulerSpendingHandler` memanggil `Service.SpendingUserScheduler(ctx)`).
- Cron melakukan graceful shutdown pada `SIGINT`/`SIGTERM`/`SIGHUP`.
- Saat menambah job: tambah entri `*_CRON` di env/`config.go` + registrasi di scheduler + handler di fitur. Pastikan job **idempotent** dan tahan kalau berjalan ganda.

## Goroutine

- Gunakan `helper.SafeGo(s.Log, func(){ ... })` (memasang `recover` + stack trace) untuk pekerjaan latar, bukan `go func(){}()` mentah:
  ```go
  helper.SafeGo(s.Log, func() {
      if err := s.SomeBackgroundTask(); err != nil {
          s.Log.Error("background task failed", zap.Error(err))
      }
  })
  ```
- Batasi goroutine dengan `sync.WaitGroup` + `errgroup` (`golang.org/x/sync/errgroup`) dan timeout (`context.WithTimeout`), seperti `GetDetailUserByIdsService`.
- State bersama (map, counter) wajib dilindungi `sync.Mutex`/`sync.RWMutex` atau `sync/atomic`. Contoh: `limiters` di `middleware/auth.go` dijaga `mu`.
- Jangan menyimpan `context.Context` di struct; teruskan lewat parameter.
- Jangan `context.Background()` di tengah call chain untuk mengganti ctx dari handler. Consumer/cron boleh memakai `context.Background()` sebagai akar karena tidak ada request HTTP.
- Jalankan `go test -race ./...` / `go run -race` saat menyentuh kode ber-goroutine.

## Logging ke layanan log eksternal

- `client.LogClientInterface` (`f.LogClient`) dipakai mengirim log/aktivitas (`SendMonitClientLog`, `SendLogAutoTopup`, dll) — mis. user activity dari middleware `BearerTokenWithPin` (`catchUserActivity`).
- `SlackClient` untuk alert opsional (`SlackNotify`, `SlackNotifyWithCaller`, `SlackNotifyDecline`).
- Kirim data minimum yang diperlukan; jangan kirim secret/data kartu/PIN.
