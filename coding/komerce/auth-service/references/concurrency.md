# Concurrency di Go: Goroutine, Channel, Context

## Prinsip dasar

> "Don't communicate by sharing memory; share memory by communicating." — tapi kalau kasusnya cocok, `sync.Mutex` yang simpel lebih baik daripada channel yang dipaksakan. Pilih yang paling jelas dibaca untuk kasusnya, bukan yang paling "canggih".

- Setiap goroutine yang di-`go func(){}()` harus JELAS kapan dan bagaimana dia berhenti. Goroutine yang "lupa" dimatikan = goroutine leak.
- SELALU pikirkan: siapa yang menutup channel? Siapa yang menunggu goroutine selesai (`sync.WaitGroup`)? Apa yang terjadi kalau context di-cancel di tengah jalan?

## Context untuk cancellation & timeout

- Fungsi yang melakukan I/O (HTTP call, DB query, baca file besar) WAJIB menerima `ctx context.Context` sebagai parameter PERTAMA, dan meneruskannya ke pemanggilan I/O di dalamnya.
- JANGAN simpan `context.Context` di dalam struct field — selalu lewat parameter.
- JANGAN pakai `context.Background()` di kode bisnis biasa (hanya valid di `main()`/entrypoint/test); teruskan context yang diterima dari caller.

```go
func FetchUser(ctx context.Context, id string) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("build request: %w", err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fetch user %s: %w", id, err)
	}
	defer resp.Body.Close()
	// ...
}
```

## Goroutine + WaitGroup (pola dasar fan-out)

```go
func ProcessAll(ctx context.Context, items []Item) error {
	var wg sync.WaitGroup
	errCh := make(chan error, len(items))

	for _, item := range items {
		item := item // penting di Go <1.22: hindari capture variabel loop
		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := process(ctx, item); err != nil {
				errCh <- fmt.Errorf("process item %v: %w", item.ID, err)
			}
		}()
	}

	wg.Wait()
	close(errCh)

	var errs []error
	for err := range errCh {
		errs = append(errs, err)
	}
	return errors.Join(errs...)
}
```
Catatan: di Go 1.22+ variabel loop sudah per-iterasi otomatis, jadi `item := item` tidak wajib lagi — tapi tetap aman ditulis kalau target Go version-nya belum jelas.

## errgroup untuk fan-out dengan error + cancellation

Untuk kasus "jalankan N goroutine, hentikan semua begitu satu gagal", `golang.org/x/sync/errgroup` lebih idiomatis daripada WaitGroup manual:

```go
func ProcessAll(ctx context.Context, items []Item) error {
	g, ctx := errgroup.WithContext(ctx)
	for _, item := range items {
		item := item
		g.Go(func() error {
			return process(ctx, item)
		})
	}
	return g.Wait()
}
```

## Worker pool (batasi concurrency)

```go
func RunWorkerPool(ctx context.Context, jobs <-chan Job, workerCount int) error {
	g, ctx := errgroup.WithContext(ctx)
	for i := 0; i < workerCount; i++ {
		g.Go(func() error {
			for {
				select {
				case <-ctx.Done():
					return ctx.Err()
				case job, ok := <-jobs:
					if !ok {
						return nil
					}
					if err := handle(ctx, job); err != nil {
						return err
					}
				}
			}
		})
	}
	return g.Wait()
}
```

## Sinkronisasi state bersama

- State sederhana yang dibaca/ditulis banyak goroutine → `sync.Mutex`/`sync.RWMutex`, mutex sedekat mungkin dengan data yang dilindungi (idealnya sebagai field bersebelahan di struct yang sama).
- Counter sederhana → pertimbangkan `sync/atomic` daripada mutex.
- JANGAN copy struct yang mengandung `sync.Mutex` (termasuk lewat pass-by-value) — selalu pass pointer.

## Deteksi race condition

Selalu sarankan user menjalankan test dengan race detector saat kode melibatkan goroutine:
```
go test -race ./...
go run -race main.go
```