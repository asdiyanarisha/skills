# Web API / Backend di Go

## Pilihan framework — tanya/deteksi dulu sebelum menulis kode

1. Cek apakah project sudah punya dependency framework tertentu (lihat `go.mod`) — ikuti yang sudah dipakai, jangan campur framework.
2. Kalau project baru dan user belum sebut framework, tawarkan pilihan singkat:
   - **`net/http` stdlib** (Go 1.22+ dengan `http.ServeMux` pattern routing) → pilihan default untuk service kecil-menengah, tanpa dependency tambahan.
   - **Gin** → paling populer, performant, banyak middleware siap pakai.
   - **Echo** → mirip Gin, API sedikit lebih rapi.
   - **Fiber** → API mirip Express.js, berbasis fasthttp (bukan net/http, perhatikan kompatibilitas middleware stdlib).
   - **gRPC** (`google.golang.org/grpc`) → kalau butuh komunikasi service-to-service performant/strongly-typed, bukan API publik berbasis browser.

## Struktur handler yang idiomatis

- Handler tipis: validasi input → panggil service/usecase layer → format response. Logika bisnis TIDAK ditulis langsung di handler.
- Struct handler menerima dependency (service, repo) lewat constructor, bukan variabel global.

### Contoh dengan net/http stdlib (Go 1.22+)

```go
type UserHandler struct {
	svc UserService
}

func NewUserHandler(svc UserService) *UserHandler {
	return &UserHandler{svc: svc}
}

func (h *UserHandler) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("GET /users/{id}", h.GetUser)
	mux.HandleFunc("POST /users", h.CreateUser)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	user, err := h.svc.GetByID(r.Context(), id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			http.Error(w, "user not found", http.StatusNotFound)
			return
		}
		http.Error(w, "internal server error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(user)
}
```

### Contoh dengan Gin

```go
func (h *UserHandler) GetUser(c *gin.Context) {
	id := c.Param("id")

	user, err := h.svc.GetByID(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
		return
	}

	c.JSON(http.StatusOK, user)
}
```

## Middleware

- Middleware untuk concern lintas-endpoint: logging, recover-panic, auth, request-id, CORS, rate limiting.
- Urutan middleware penting: umumnya `recover → request-id/logging → auth → handler`.
- Jangan taruh business logic di middleware.

## Request validation & response

- Validasi payload dengan tag struct (`validator/v10` populer di ekosistem Gin) ATAU manual validation function — konsisten dengan yang sudah dipakai project.
- Response error terstruktur konsisten, contoh bentuk:
```go
type ErrorResponse struct {
	Error   string `json:"error"`
	Code    string `json:"code,omitempty"`
}
```
- Jangan bocorkan detail internal (stack trace, query SQL) di response ke client — log detail di server, kirim pesan umum ke client.

## Status code

| Situasi | Status |
|---|---|
| Sukses ambil/list data | 200 |
| Sukses buat resource baru | 201 |
| Sukses tanpa body | 204 |
| Input tidak valid | 400 |
| Tidak terautentikasi | 401 |
| Terautentikasi tapi tidak berhak | 403 |
| Resource tidak ditemukan | 404 |
| Konflik (mis. duplicate) | 409 |
| Error tak terduga di server | 500 |

## Graceful shutdown

Selalu sertakan graceful shutdown untuk service HTTP produksi:
```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatalf("listen: %v", err)
	}
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
	log.Fatalf("forced shutdown: %v", err)
}
```

## gRPC (kalau relevan)

- Definisikan service di `.proto`, generate lewat `protoc`/`buc`, jangan tulis manual.
- Implementasi service menerima dependency lewat constructor sama seperti handler HTTP.
- Gunakan interceptor (unary/stream) untuk cross-cutting concern, setara middleware di HTTP.