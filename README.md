# Review Kantin API - Tugas 3 (Validasi, Error, dan Auth)

Express 5 + Drizzle ORM + SQL Server. Lanjutan Tugas 2, ditambah:

- **Validasi input** dengan **zod** (query, params, body) lewat middleware `validate`
- **Satu format error** untuk semua jenis kegagalan
- **Autentikasi JWT** (register, login, `/auth/me`) + password di-hash **bcrypt**
- **Otorisasi role** `admin` / `owner` / `customer`, dan **401 vs 403** dibedakan
- **Dokumentasi Swagger/OpenAPI** di `/docs`

## Menjalankan

1. `npm install`
2. Salin `.env.example` menjadi `.env`, isi `DB_USER` / `DB_PASSWORD`, dan **`JWT_SECRET`** (wajib; server menolak start kalau kosong).
3. Di SQL Server, jalankan berurutan (folder `database/`):
   `01-create-database-and-user.sql` → `02-schema.sql` → `03-seed.sql` → **`04-seed-password-hashes.sql`**
   (file 04 mengganti password placeholder seed dengan hash bcrypt asli agar akun seed bisa login).
4. `npm run dev` → cek `http://localhost:3000/health`
5. Dokumentasi interaktif: **http://localhost:3000/docs** (spec JSON: `/docs.json`)

### Akun uji (password semuanya `rahasia123`)

| Role     | Email                | Keterangan                                   |
|----------|----------------------|----------------------------------------------|
| admin    | `admin@kantin.test`  | akses semua endpoint                         |
| owner    | `tini@kantin.test`   | pemilik stall id 1 (Warung Bu Tini)          |
| owner    | `slamet@kantin.test` | pemilik stall id 3 (Kedai Pak Slamet)        |
| customer | `bagas@student.test` | customer biasa                               |

## Format error (satu untuk semua)

```json
{
  "status": "fail",
  "message": "Validasi gagal",
  "errors": [
    { "field": "name", "message": "name minimal 3 karakter" },
    { "field": "password", "message": "password minimal 8 karakter" }
  ]
}
```

`status: "fail"` untuk error client (4xx), `status: "error"` untuk error server (5xx). Field `errors` hanya ada saat validasi gagal.

| Kode | Arti                                                                             |
|------|----------------------------------------------------------------------------------|
| 400  | Validasi gagal / JSON rusak                                                      |
| 401  | **Belum/tidak terautentikasi**: token tidak ada, salah format, palsu, kedaluwarsa |
| 403  | **Sudah login, tidak punya akses**: role tidak cukup / bukan pemilik resource     |
| 404  | Data / rute tidak ditemukan                                                      |
| 409  | Duplikat (mis. email sudah terdaftar, review ganda)                              |

## Hak akses endpoint (prefix `/api/v1`)

| Endpoint | Akses |
|---|---|
| `GET /stalls`, `GET /stalls/:id` | publik |
| `POST /auth/register`, `POST /auth/login` | publik |
| `GET /auth/me` | login (semua role) |
| `POST /stalls/:id/reviews` | login (semua role) |
| `POST /stalls` | owner, admin |
| `PUT /stalls/:id` | owner (**hanya miliknya**), admin |
| `DELETE /stalls/:id` | admin |

Endpoint lanjutan dari Tugas 2 ikut diamankan:

| Endpoint | Akses |
|---|---|
| `GET /menu-items`, `GET /menu-items/:id`, `GET /reviews`, `GET /reviews/:id` | publik |
| `POST/PUT/DELETE /menu-items` | owner (hanya kantin miliknya), admin |
| `DELETE /reviews/:id` | penulis review, admin |
| `POST /likes`, `DELETE /likes/:id` | login; hapus hanya pemilik like / admin |
| `GET/POST /users`, `GET/PUT /flags`, `GET/POST /audit-logs` | admin |

**Identitas diambil dari token, bukan dari body.** `owner_id` (stall), `user_id` (review, like, audit log), dan `role` (register) tidak ada di schema body; mengirimnya akan ditolak 400 (schema `strict`).

## Skema validasi (zod)

| Endpoint | Sumber | Aturan |
|---|---|---|
| `GET /stalls` | query | `search` trim ≤100 opsional; `category` trim ≤50 opsional; `page` int ≥1 default 1; `limit` int 1..100 default 10 |
| `GET /stalls/:id` | params | `id` int > 0 |
| `POST /stalls` | body | `name` trim 3..100 wajib; `category` ≤50; `location` ≤100; `description` ≤1000 (opsional) |
| `PUT /stalls/:id` | params + body | `id` int > 0; body = versi parsial POST (minimal satu field) |
| `DELETE /stalls/:id` | params | `id` int > 0 |
| `POST /stalls/:id/reviews` | params + body | `id` int > 0; `rating` int 1..5 wajib; `comment` trim ≤1000 opsional |
| `POST /auth/register` | body | `name` trim 3..100; `email` format email, trim, lowercase, ≤150; `password` 8..72 |
| `POST /auth/login` | body | `email` format email, trim, lowercase; `password` min 1 |
| `GET /auth/me` | header | tanpa body/params (header `Authorization: Bearer <token>`) |

## Arsitektur

`routes -> (authenticate -> authorize -> validate) -> controllers -> services -> repositories -> db (Drizzle)`

```
src/
  dtos/          schema zod (validasi) per resource: *.dto.ts
  middlewares/   auth.ts (authenticate), authorize.ts, validate.ts, errorHandler.ts, notFound.ts
  services/      tokenService.ts (JWT), authService.ts (register/login), + service per resource
  utils/         AppError.ts (AppError, ValidationError, unauthorized, forbidden), ownership.ts, response.ts
  types/         express.d.ts (menambah req.user)
  docs/          openapi.ts (Swagger/OpenAPI, body diambil dari schema zod)
```

Urutan middleware sengaja `authenticate` → `authorize` → `validate`: request tanpa token selalu 401 apa pun isi body-nya, jadi aturan validasi tidak bocor ke pengguna anonim.

Contoh request lain: `docs/requests.http` (VS Code REST Client).
