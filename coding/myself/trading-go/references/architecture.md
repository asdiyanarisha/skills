# Architecture & Layering Rules

## Repository Layer
- Hanya bertanggung jawab atas persistence (get/set/query).
- Gunakan domain model/entity sebagai parameter & return value — BUKAN DTO.
- Tidak boleh berisi business logic atau decision-making (mis. fallback value,
  perhitungan TTL, validasi bisnis). Semua itu wajib di Service/UseCase layer.
- Mapping domain model ↔ DTO dilakukan di Service layer atau handler, bukan di repository.

## Resource Management
- Setiap resource yang di-`open` (file, koneksi DB, HTTP response body, dll)
  wajib langsung diikuti `defer xxx.Close()` tepat setelah pengecekan error open berhasil.

## Concurrency-safe Map Access
- Hindari akses/modifikasi map secara concurrent tanpa proteksi (mutex/sync.Map).
- Pastikan `Lock()`/`RLock()` selalu diikuti `defer Unlock()`/`defer RUnlock()` segera
  setelah lock diambil, untuk mencegah deadlock akibat lupa unlock atau nested locking
  pada map yang sama.