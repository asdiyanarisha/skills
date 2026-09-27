.# Testing di Go

## Konvensi dasar

- File test: `xxx_test.go`, di package yang sama (`package user`) untuk test internal, atau `package user_test` untuk test yang hanya pakai exported API (lebih disarankan untuk library/public API karena memaksa test dari sudut pandang pemakai).
- Nama fungsi test: `func TestXxx(t *testing.T)`, subtest pakai `t.Run("nama kasus", func(t *testing.T){...})`.
- Selalu panggil `t.Parallel()` di test yang independen (tidak share state) untuk mempercepat suite.

## Table-driven test (pola standar Go)

Ini pola DEFAULT yang harus dipakai kecuali test-nya benar-benar cuma 1 skenario:

```go
func TestValidateEmail(t *testing.T) {
	tests := []struct {
		name    string
		email   string
		wantErr bool
	}{
		{name: "valid email", email: "a@b.com", wantErr: false},
		{name: "missing at sign", email: "ab.com", wantErr: true},
		{name: "empty string", email: "", wantErr: true},
	}

	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			err := ValidateEmail(tt.email)
			if (err != nil) != tt.wantErr {
				t.Errorf("ValidateEmail(%q) error = %v, wantErr %v", tt.email, err, tt.wantErr)
			}
		})
	}
}
```

## Assertion

- Stdlib `testing` polos sudah cukup untuk kebanyakan kasus (`t.Errorf`, `t.Fatalf`).
- Kalau project sudah pakai `stretchr/testify`, ikuti konvensi yang ada (`assert.Equal`, `require.NoError`) — `require` untuk kondisi yang harus stop test kalau gagal, `assert` untuk yang boleh lanjut.
- Jangan tambahkan dependency testing library baru kalau project belum pakai apa pun — stdlib cukup.

## Mocking

- Definisikan dependency eksternal sebagai **interface kecil** di sisi consumer (bukan di sisi implementasi) supaya gampang di-mock:

```go
// di package service, bukan di package repository
type UserRepository interface {
	FindByID(ctx context.Context, id string) (*User, error)
}

type fakeUserRepo struct {
	users map[string]*User
	err   error
}

func (f *fakeUserRepo) FindByID(ctx context.Context, id string) (*User, error) {
	if f.err != nil {
		return nil, f.err
	}
	return f.users[id], nil
}
```
- Untuk mock yang lebih kompleks/generate otomatis, `go.uber.org/mock` (mockgen) adalah pilihan umum — sarankan ini hanya kalau user sudah oke menambah dependency/tooling.
- Hindari mocking library HTTP/DB langsung; lebih baik mock di level interface milik sendiri (repository/client interface), bukan mock `http.Client` atau `*sql.DB` mentah-mentah.

## Test HTTP handler

```go
func TestGetUserHandler(t *testing.T) {
	repo := &fakeUserRepo{users: map[string]*User{"1": {ID: "1", Name: "Ada"}}}
	h := NewUserHandler(repo)

	req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
	rec := httptest.NewRecorder()

	h.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("got status %d, want %d", rec.Code, http.StatusOK)
	}
}
```

## Golden file & benchmark (opsional, sebutkan bila relevan)

- Golden file cocok untuk output besar (JSON/HTML) — bandingkan dengan file referensi di `testdata/`.
- Benchmark: `func BenchmarkXxx(b *testing.B)`, jalankan dengan `go test -bench=. -benchmem`.

## Coverage

Sarankan `go test -cover ./...` atau `go test -coverprofile=cover.out ./... && go tool cover -html=cover.out` saat user minta cek coverage — jangan targetkan 100% secara membabi buta, fokus pada logic penting (business rule, edge case), bukan getter/setter trivial.