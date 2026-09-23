# Error Handling di Go

## Aturan dasar

- Error adalah **value**, bukan exception. Selalu cek `if err != nil` segera setelah pemanggilan yang bisa gagal — jangan tunda.
- **Wrap** error dengan konteks memakai `%w` (bukan `%v`/`%s`) supaya bisa di-unwrap dengan `errors.Is`/`errors.As`:

```go
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
	row := r.db.QueryRowContext(ctx, query, id)
	var u User
	err := row.Scan(&u.ID, &u.Name)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
		}
		return nil, fmt.Errorf("scan user %s: %w", id, err)
	}
	return &u, nil
}
```

- JANGAN `return err` polos di tengah call stack yang dalam — tambahkan konteks di setiap layer supaya pesan error akhirnya informatif ("layer mana yang gagal").
- JANGAN pernah `if err != nil { }` kosong atau `_ = someCall()` tanpa komentar penjelasan eksplisit kenapa error diabaikan.

## Sentinel error & custom error type

Sentinel error (dengan `errors.New`) untuk kondisi yang perlu dicek pemanggil:
```go
var ErrNotFound = errors.New("resource not found")

// pemanggil:
if errors.Is(err, ErrNotFound) { ... }
```

Custom error type kalau butuh membawa data tambahan:
```go
type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("field %s: %s", e.Field, e.Msg)
}

// pemanggil:
var ve *ValidationError
if errors.As(err, &ve) {
	fmt.Println(ve.Field)
}
```

## Panic & recover

- `panic` HANYA untuk kondisi yang benar-benar tidak bisa dilanjutkan (programmer error, bukan input user salah) — mis. invariant internal rusak saat init.
- Error dari input user, network, DB, file → SELALU error value, JANGAN panic.
- `recover()` hanya dipakai di boundary tertentu (mis. middleware HTTP top-level) untuk mencegah crash total, bukan sebagai pengganti error handling normal:

```go
func RecoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic recovered: %v", rec)
				http.Error(w, "internal server error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

## Multiple errors (Go 1.20+)

Pakai `errors.Join` kalau perlu menggabungkan beberapa error independen (mis. validasi banyak field sekaligus):
```go
var errs []error
if name == "" {
	errs = append(errs, errors.New("name required"))
}
if age < 0 {
	errs = append(errs, errors.New("age must be non-negative"))
}
return errors.Join(errs...)
```

## Logging vs returning

- Jangan log DAN return error di tempat yang sama (menyebabkan duplikasi log di setiap layer). Log sekali di titik paling atas (boundary, mis. handler HTTP) atau paling bawah kalau memang mau di-swallow, bukan keduanya di tengah.