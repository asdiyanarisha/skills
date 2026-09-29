# Web API (Gin) — Komcards API

Framework: **Gin** (`github.com/gin-gonic/gin`). Setup di `internal/http/http.go`, entrypoint `main.go`.

## Struktur handler

Handler tipis, dependency lewat factory:

```go
type handler struct {
    service Service
}

func NewHandler(f *factory.Factory) *handler {
    return &handler{service: NewService(f)}
}

func (h *handler) ListCardHandler(g *gin.Context) {
    userSess := g.Value("user")

    var payload dto.QueryParameterListCard
    if err := g.ShouldBindQuery(&payload); err != nil {
        g.JSON(http.StatusBadRequest, dto.Response{
            Meta: dto.Meta{
                Status:  "failed",
                Code:    http.StatusBadRequest,
                Message: constants.QueryNotValidate.Error(),
            },
        })
        return
    }

    response, totalCard, maximalQuotaCard, filteredCount, err := h.service.ListCard(g, userSess, payload)
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

    g.JSON(http.StatusOK, dto.ResponseCard{
        Meta: dto.MetaCard{
            Status:        "success",
            Code:          http.StatusOK,
            Message:       "success fetch data",
            Offset:        payload.Offset,
            Limit:         payload.Limit,
            FilteredTotal: filteredCount,
            Total:         totalCard,
            Quota:         maximalQuotaCard,
        },
        Data: util.FormattingAnyToSlice(response),
    })
}
```

Aturan handler:
- Nama method handler berakhiran `Handler` (atau nama jelas) dan cocok dengan yang didaftarkan di router.
- Bind dengan `g.ShouldBindJSON` (body) / `g.ShouldBindQuery` (query). Selalu tangani error bind dengan `constants.BodyNotValidate` / `constants.QueryNotValidate`.
- Ambil identitas dari `g.Value("user")` (di-set middleware) — jangan dari body untuk endpoint user.
- Kirim `g` (gin.Context) sebagai `ctx` ke service bila service butuh (`h.service.ListCard(g, ...)`); Gin context meng-implement `context.Context`.
- Jangan menaruh business logic/query di handler.
- Response selalu memakai envelope `dto.*` (lihat `references/error-handling.md`).

## Router

Router per fitur ada di `router.go` fitur, didaftarkan di `http.go`:

```go
func (h *handler) CardRouter(g *gin.RouterGroup) {
    g.GET("/mutation", h.MutationTransactionCard)
    g.GET("/list", h.ListCardHandler)
    g.GET("/:card_id/detail", h.DetailCardHandler)
    g.POST("/create", h.CreateCard) // dipasang di group WithPin
}
```

- Path param Gin: `:card_id`, `:transaction_id`, `:user_id` — ambil dengan `g.Param("card_id")`.
- Kelompokkan router berdasarkan middleware: user (`BearerToken`), with PIN (`BearerTokenWithPin`), admin (`BearerTokenOnlyAdmin`), open (`OpenKeyMiddleware`), webhook (`CallbackTokenMonit`), api key (`ApiKey`/`ApiKeyV2`).
- Router admin: taruh di group `/api/v1/admin`.
- Saat menambah route, **wajib** pilih middleware yang tepat (lihat `references/security.md`).

Contoh pendaftaran di `http.go`:
```go
v1 := g.Group("/api/v1")
routerGroupWithPin := g.Group("/api/v1")
routerGroupWithPin.Use(middleware.BearerTokenWithPin(f), gin.Logger(), gin.Recovery())
{
    card.NewHandler(f).CardRouterWithPin(routerGroupWithPin.Group("/card"))
}

adminGroup := g.Group("/api/v1/admin")
adminGroup.Use(middleware.BearerTokenOnlyAdmin(f), gin.Logger(), gin.Recovery())
{
    card.NewHandler(f).CardRouterAdmin(adminGroup.Group("/card"))
}
```
Pola ini memanggil `NewHandler(f)` lagi untuk tiap group — tidak masalah karena handler ringan; ikuti gaya yang ada.

## Validasi request

- Gunakan tag `binding:"required"` (validator/v10) pada field DTO.
- Pesan error validasi: `helper.Validate(err)` (map field→pesan) atau `helper.ErrorMessage(err)` (string). Tag `min`/`max` diterjemahkan `helper.CustomErrorMessage`.
- Validasi domain (aturan bisnis) di **service**, bukan di handler.

Contoh DTO:
```go
type RequestTopupCard struct {
    CardId    int    `json:"card_id"`
    Balance   uint   `json:"balance"`
    Pin       string `json:"pin"`
    IsVoucher bool   `json:"is_voucher"`
}

type AdminParamListCard struct {
    UserId int    `json:"user_id" form:"user_id"`
    Limit  int    `json:"limit" form:"limit" binding:"required"`
    ...
}
```

## Format response & status code

Envelope tersedia di `internal/dto`: `dto.Response`, `dto.ResponseCard`, `dto.ResponsePaginate`, `dto.Error`, `dto.Common`.

| Situasi | Status yang dipakai |
|---|---|
| Sukses ambil/list | 200 |
| Sukses buat | 200 (proyek ini jarang 201) |
| Sukses tanpa body | 200/204 sesuai konvensi file |
| Input tidak valid | 400 (`constants.BodyNotValidate`/`QueryNotValidate`) |
| Tidak terautentikasi | 401 (middleware) |
| Tidak berhak | 403/401 |
| Tidak ditemukan | 400 (default `DefinedErrorStatusCode`) — kecuali endpoint sudah punya pemetaan khusus |
| Terlalu banyak request | 429 |
| Error tak terduga | 500 (`constants.InternalServerError`) |

Middleware menolak request dengan:
```go
c.AbortWithStatusJSON(http.StatusUnauthorized, dto.Common{
    Status: "failed", Code: 401, Message: "Unauthenticated",
})
```

## Middleware

- Concern lintas endpoint: `CORSMiddleware`, `MonitoringActivity` (Prometheus), auth, token callback.
- Urutan umum: CORS/monitoring global → logger/recovery → auth → handler.
- Middleware feature-spesifik tetap di `internal/middleware/` dengan nama jelas.
- Jangan taruh business logic di middleware.
- Catatan: `CORSMiddleware` saat ini memakai `Access-Control-Allow-Origin: *` bersama `Allow-Credentials: true`. Jika menyentuh CORS, pertimbangkan whitelist origin (wildcard + credentials bermasalah secara keamanan/browser).

## Consumer / scheduler bukan HTTP

Fitur seperti `monit`, `operation`, `ticket`, `card` juga punya `consumer.go`; scheduler ada di `internal/scheduler`. Ini bukan HTTP — lihat `references/integration-messaging.md`.

## Graceful shutdown

HTTP API dijalankan `g.Run(port)`. Scheduler punya graceful shutdown sendiri (`internal/scheduler/scheduler.go`). Jika menambah server/worker baru, sediakan penanganan sinyal `SIGINT/SIGTERM` yang menutup resource dengan rapi.

## gRPC

Tidak digunakan di repo ini. Integrasi antar-service memakai HTTP client (`internal/client`), bukan gRPC.
