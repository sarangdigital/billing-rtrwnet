REST API Portal Pelanggan
=========================

Dokumen ini mendeskripsikan REST-like API yang digunakan oleh Portal Pelanggan.
Semua endpoint diimplementasikan melalui satu URL:

- Base URL: `/api/pelanggan_portal.php`
- Metode utama: `POST` (kecuali disebutkan lain)
- Format request: `application/x-www-form-urlencoded`
- Format respons: `application/json`

Setiap operasi dibedakan dengan parameter `action`.

Struktur Respons Umum
---------------------

Sebagian besar respons menggunakan pola berikut:

```json
{
  "status": "success | error | otp_required",
  "message": "Pesan singkat (opsional)",
  "data": { "..." }          // untuk operasi yang mengembalikan objek/data
}
```

Nilai `status`:

- `success`  → operasi berhasil
- `error`    → terjadi kesalahan (detail di `message`)
- `otp_required` → khusus login, menandakan OTP WhatsApp diperlukan/masih aktif

Autentikasi dan Keamanan
------------------------

- **Session pelanggan** disimpan di `$_SESSION['pelanggan_logged_in']`, `$_SESSION['pelanggan_id']`, `$_SESSION['pelanggan_company_id']`.
- **Device trust**: cookie `pelanggan_device_id` digunakan untuk mengingat perangkat terpercaya dan login tanpa OTP.
- **OTP WhatsApp** digunakan untuk verifikasi login.
- **CSRF**: hampir semua action memanggil `require_valid_csrf()`, sehingga wajib mengirim `csrf_token` yang valid di body request.

Jika sesi pelanggan tidak valid, sebagian besar action akan mengembalikan:

```json
{ "status": "error", "message": "Sesi pelanggan tidak valid" }
```

Jika parameter `action` tidak dikenal, API akan mengembalikan:

```json
{ "status": "error", "message": "Aksi tidak dikenal" }
```

Catatan Umum Contoh curl
------------------------

- Semua contoh menggunakan host `http://sample.sarang.id`. Sesuaikan dengan domain/host yang Anda pakai.
- Pastikan cookie session dan `pelanggan_device_id` sudah terbentuk (misalnya setelah membuka halaman portal).
- Ganti `TOKEN_CSRF_VALID` dengan nilai `csrf_token` yang benar sesuai sesi pengguna.
- Nilai seperti ID (`pelanggan_id`, `tagihan_id`, `ticket_id`, `notif_id`) hanyalah contoh dan harus disesuaikan dengan data nyata.

Grup Endpoint
-------------

1. Autentikasi & OTP
2. Profil & Perangkat Terpercaya
3. Notifikasi
4. Tagihan & Riwayat Tagihan
5. Pembayaran Tripay
6. Tiket Bantuan Pelanggan
7. Endpoint Admin terkait Tiket


1. Autentikasi & OTP
--------------------

### [DOC 1.1] Request Login OTP

- Action: `pelanggan_login_request`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_login_request`
- `pelanggan_id`: integer, ID pelanggan di tabel `pelanggan`
- `phone`: string, nomor WhatsApp pelanggan (akan dinormalisasi ke format 62)
- `csrf_token`: string, token CSRF valid

Respons sukses (OTP baru dikirim):

```json
{
  "status": "otp_required",
  "message": "Kode verifikasi telah dikirim ke WhatsApp Anda"
}
```

Respons jika OTP sebelumnya masih aktif:

```json
{
  "status": "otp_required",
  "message": "Kode verifikasi masih aktif. Silakan gunakan kode yang sudah dikirim ke WhatsApp Anda."
}
```

Contoh respons error:

- Data pelanggan tidak ditemukan:

```json
{ "status": "error", "message": "Data pelanggan tidak ditemukan" }
```

- Nomor telepon tidak sesuai:

```json
{ "status": "error", "message": "Nomor telepon tidak sesuai dengan data pelanggan" }
```

- Layanan OTP tidak tersedia:

```json
{ "status": "error", "message": "Layanan OTP sementara tidak tersedia" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_login_request' \
  --data-urlencode 'pelanggan_id=123' \
  --data-urlencode 'phone=081234567890' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

Jika perangkat sudah tercatat sebagai perangkat terpercaya dan masih aktif, login langsung berhasil tanpa OTP:

```json
{
  "status": "success",
  "message": "Login berhasil (perangkat terpercaya)"
}
```

### [DOC 1.2] Konfirmasi OTP

- Action: `pelanggan_confirm_otp`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_confirm_otp`
- `code`: string, kode OTP 6 digit
- `csrf_token`: string

Prasyarat:

- Sesi pending OTP harus ada di server:
  - `$_SESSION['pelanggan_pending_id']`
  - `$_SESSION['pelanggan_pending_company_id']`

Respons sukses:

```json
{
  "status": "success",
  "message": "Verifikasi berhasil"
}
```

Efek samping:

- Menandai OTP sebagai digunakan.
- Menyimpan device sebagai perangkat terpercaya.
- Meng-set session login pelanggan.

Respons error yang mungkin:

- Kode OTP tidak valid (format salah, bukan 6 digit):

```json
{ "status": "error", "message": "Kode verifikasi tidak valid" }
```

- Kode salah atau sudah kedaluwarsa:

```json
{ "status": "error", "message": "Kode verifikasi salah atau sudah kedaluwarsa" }
```

- Terlalu banyak percobaan:

```json
{ "status": "error", "message": "Terlalu banyak percobaan. Coba lagi beberapa menit lagi." }
```

- Terlalu cepat mencoba ulang:

```json
{ "status": "error", "message": "Tunggu beberapa detik sebelum mencoba lagi." }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_confirm_otp' \
  --data-urlencode 'code=123456' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 1.3] Kirim Ulang OTP

- Action: `pelanggan_resend_otp`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_resend_otp`
- `csrf_token`: string

Prasyarat:

- Sesi pending OTP masih ada:
  - `$_SESSION['pelanggan_pending_id']`
  - `$_SESSION['pelanggan_pending_company_id']`
  - `$_SESSION['pelanggan_pending_phone']`

Respons sukses:

```json
{
  "status": "success",
  "message": "Kode verifikasi telah dikirim ulang"
}
```

Respons error:

```json
{ "status": "error", "message": "Sesi verifikasi tidak ditemukan" }
```

atau

```json
{ "status": "error", "message": "Layanan OTP sementara tidak tersedia" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_resend_otp' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 1.4] Logout Pelanggan

- Action: `pelanggan_logout`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_logout`
- `csrf_token`: string

Respons sukses:

```json
{
  "status": "success",
  "message": "Berhasil keluar"
}
```

Efek samping:

- Menghapus semua data sesi terkait pelanggan (`pelanggan_logged_in`, `pelanggan_id`, dll).

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_logout' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


2. Profil & Perangkat Terpercaya
-------------------------------

### [DOC 2.1] Data Profil Pelanggan

- Action: `pelanggan_profile`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_profile`
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login (`pelanggan_require_logged_in()`).

Respons sukses:

```json
{
  "status": "success",
  "data": {
    "id": 1,
    "nama": "Nama Pelanggan",
    "no_wa": "6281234567890",
    "alamat": "Alamat pelanggan",
    "paket_nama": "Nama Paket",
    "paket_speed": "Deskripsi kecepatan (baris pertama deskripsi paket)",
    "harga_paket": 250000,
    "status_label": "Aktif | Nonaktif | ...",
    "jatuh_tempo_hari": 10,
    "created_at": "2024-01-01 10:00:00",
    "trusted_devices": [
      {
        "device_id": "abcdef...",
        "ip": "1.2.3.4",
        "os": "Android",
        "browser": "Chrome",
        "device": "Mobile",
        "created_at": "2024-01-01 10:00:00",
        "expires_at": "2025-01-01 10:00:00",
        "active": true,
        "current": true
      }
    ]
  }
}
```

Respons error jika sesi tidak valid:

```json
{ "status": "error", "message": "Sesi tidak valid" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_profile' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 2.2] Hapus Perangkat Terpercaya

- Action: `pelanggan_revoke_trust_device`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_revoke_trust_device`
- `device_id`: string, ID perangkat yang akan dihapus
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "message": "Perangkat terpercaya dihapus"
}
```

Contoh error:

```json
{ "status": "error", "message": "Perangkat tidak valid" }
```

```json
{ "status": "error", "message": "Perangkat tidak ditemukan atau sudah dihapus" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_revoke_trust_device' \
  --data-urlencode 'device_id=abcdef123456' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


3. Notifikasi
-------------

### [DOC 3.1] Detail Notifikasi Pelanggan

- Action: `pelanggan_notif_detail`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_notif_detail`
- `notif_id`: integer, ID notifikasi (`pelanggan_notifications.id`)
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "data": {
    "id": 1,
    "title": "Judul notifikasi",
    "message": "Isi notifikasi",
    "type": "info | warning | ...",
    "priority": 1,
    "is_read": 1,
    "created_at": "2024-01-01 10:00:00"
  }
}
```

Jika notifikasi ditemukan dan sebelumnya `is_read = 0`, sistem akan otomatis menandainya sebagai dibaca.

Contoh error:

```json
{ "status": "error", "message": "Data tidak valid" }
```

```json
{ "status": "error", "message": "Notifikasi tidak ditemukan" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_notif_detail' \
  --data-urlencode 'notif_id=1' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


4. Tagihan & Riwayat Tagihan
----------------------------

### [DOC 4.1] Daftar Tagihan Pelanggan

- Action: `pelanggan_tagihan_list`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_tagihan_list`
- `status`: string (opsional), salah satu:
  - `all` (default) → semua tagihan
  - `unpaid` → hanya tagihan belum lunas (`status = 0`)
  - `paid` → hanya tagihan lunas (`status = 1`)
- `limit`: integer (opsional), jumlah maksimum data, default 60, maksimum 200
- `offset`: integer (opsional), posisi awal data untuk paging (default 0)
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "data": [
    {
      "id": 123,
      "periode_bulan": 1,
      "periode_tahun": 2025,
      "total_tagihan": 250000,
      "tanggal_jatuh_tempo": "2025-01-10",
      "tanggal_bayar": "2025-01-08 12:34:56",
      "status": 1,
      "metode_bayar": "TRIPAY",
      "tambahan_biaya_lain": 2500,
      "catatan_pembayaran": "Sudah dibayar via Tripay",
      "id_invoice": "INV-202501-0001"
    }
  ],
  "total": 12,
  "limit": 60,
  "offset": 0
}
```

Keterangan field penting:

- `status`: 0 = belum lunas, 1 = lunas
- `tanggal_jatuh_tempo`: tanggal jatuh tempo tagihan
- `tanggal_bayar`: terisi saat tagihan sudah dibayar
- `metode_bayar`: metode pembayaran yang dicatat (`TRIPAY`, `CASH`, dll)
- `tambahan_biaya_lain`: biaya tambahan lain yang tercatat pada pembayaran
- `catatan_pembayaran`: catatan internal terkait pembayaran
- `id_invoice`: kode invoice untuk membuka halaman invoice detail

Contoh error:

```json
{ "status": "error", "message": "Sesi tidak valid" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_tagihan_list' \
  --data-urlencode 'status=unpaid' \
  --data-urlencode 'limit=10' \
  --data-urlencode 'offset=0' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 4.2] Ringkasan Tagihan Pelanggan

- Action: `pelanggan_tagihan_summary`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_tagihan_summary`
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "data": {
    "outstanding_count": 2,
    "outstanding_total": 500000,
    "overdue_count": 1
  }
}
```

Keterangan field:

- `outstanding_count`: jumlah tagihan dengan status belum lunas (`status = 0`)
- `outstanding_total`: total nilai `total_tagihan` dari semua tagihan belum lunas
- `overdue_count`: jumlah tagihan belum lunas yang tanggal jatuh temponya sudah lewat hari ini

Contoh error:

```json
{ "status": "error", "message": "Sesi tidak valid" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_tagihan_summary' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


5. Pembayaran Tripay
--------------------

### [DOC 5.1] Daftar Kanal Pembayaran Tripay

- Action: `pelanggan_tripay_channels`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_tripay_channels`
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "channels": [
    {
      "code": "BRIVA",
      "name": "BRI Virtual Account",
      "group": "Bank Transfer",
      "fee_flat": 2500,
      "fee_percent": 0.0,
      "addon_fee_flat": 0.0,
      "addon_fee_percent": 0.0,
      "icon_url": "https://..."
    }
  ]
}
```

Contoh error:

```json
{ "status": "error", "message": "Company tidak valid" }
```

```json
{ "status": "error", "message": "Payment gateway Tripay belum diaktifkan" }
```

```json
{ "status": "error", "message": "API key Tripay belum dikonfigurasi" }
```

```json
{ "status": "error", "message": "Tidak ada kanal Tripay yang aktif" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_tripay_channels' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 5.2] Membuat Transaksi Tripay untuk Tagihan

- Action: `pelanggan_create_tripay`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_create_tripay`
- `tagihan_id`: integer, ID tagihan pelanggan
- `method`: string (opsional), kode kanal Tripay; jika kosong akan dipilih default
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.
- Konfigurasi Tripay sudah lengkap dan diaktifkan.

Perilaku:

- Mengambil data tagihan (`tagihan` + `pelanggan` + `paket`) dan menghitung `amount` termasuk biaya tambahan kanal pembayaran (bila ada).
- Jika dalam beberapa jam terakhir sudah ada transaksi Tripay aktif untuk tagihan yang sama dan kanal yang sama, transaksi lama akan digunakan ulang dan API akan mengembalikan `reuse: true`.

Respons sukses (transaksi baru atau reuse):

```json
{
  "status": "success",
  "reference": "REF123456",
  "checkout_url": "https://tripay.co.id/checkout/...",
  "reuse": true   // hanya ada jika menggunakan transaksi lama
}
```

Contoh error:

- Data request tidak lengkap:

```json
{ "status": "error", "message": "Data tidak lengkap" }
```

- Tripay belum diaktifkan / konfigurasi tidak lengkap:

```json
{ "status": "error", "message": "Payment gateway Tripay belum diaktifkan" }
```

```json
{ "status": "error", "message": "Konfigurasi Tripay belum lengkap" }
```

- Kanal pembayaran tidak tersedia:

```json
{ "status": "error", "message": "Kanal pembayaran tidak tersedia" }
```

- Tagihan sudah lunas:

```json
{ "status": "error", "message": "Tagihan sudah lunas" }
```

- Nomor telepon tidak valid:

```json
{ "status": "error", "message": "Nomor telepon pelanggan tidak valid" }
```

- Gagal membuat transaksi di Tripay:

```json
{ "status": "error", "message": "Gagal membuat transaksi Tripay", "http_code": 400 }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_create_tripay' \
  --data-urlencode 'tagihan_id=123' \
  --data-urlencode 'method=BRIVA' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


6. Tiket Bantuan Pelanggan
--------------------------

### [DOC 5.1] Membuat Tiket Baru

- Action: `pelanggan_ticket_create`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_ticket_create`
- `subject`: string, judul tiket (wajib)
- `description`: string, deskripsi keluhan/permintaan (wajib)
- `priority`: string (opsional), salah satu dari: `low`, `medium`, `high` (default: `medium`)
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "message": "Tiket berhasil dikirim"
}
```

Contoh error:

```json
{ "status": "error", "message": "Subjek dan deskripsi wajib diisi" }
```

```json
{ "status": "error", "message": "Sesi tidak valid" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_ticket_create' \
  --data-urlencode 'subject=Internet lambat' \
  --data-urlencode 'description=Sejak kemarin koneksi sering putus' \
  --data-urlencode 'priority=high' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 5.2] Daftar Tiket Open

- Action: `pelanggan_ticket_list_open`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_ticket_list_open`
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "data": [
    {
      "id": 1,
      "subject": "Internet lambat",
      "priority": "high",
      "status": "open",
      "created_at": "2024-01-01 10:00:00"
    }
  ]
}
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_ticket_list_open' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 5.3] Daftar Tiket Closed

- Action: `pelanggan_ticket_list_closed`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_ticket_list_closed`
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.

Respons sukses:

```json
{
  "status": "success",
  "data": [
    {
      "id": 2,
      "subject": "Perubahan paket",
      "priority": "medium",
      "status": "closed",
      "created_at": "2024-01-01 10:00:00",
      "closed_at": "2024-01-02 11:00:00"
    }
  ]
}
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_ticket_list_closed' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 5.4] Menutup Tiket + Feedback

- Action: `pelanggan_ticket_close`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `pelanggan_ticket_close`
- `ticket_id`: integer, ID tiket yang akan ditutup
- `rating`: integer 1–5 (wajib, penilaian layanan)
- `feedback`: string (opsional), komentar pelanggan
- `csrf_token`: string

Prasyarat:

- Pelanggan sudah login.
- Tiket berstatus `open` dan dibuat oleh pelanggan tersebut.

Respons sukses:

```json
{
  "status": "success",
  "message": "Tiket berhasil ditutup. Terima kasih atas feedback Anda!"
}
```

Jika `feedback` atau `rating` dikirim, sistem akan menyimpan ke tabel `ticket_feedback`.

Contoh error:

```json
{ "status": "error", "message": "Penilaian harus antara 1-5 bintang" }
```

```json
{ "status": "error", "message": "Tiket tidak ditemukan atau sudah ditutup" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=pelanggan_ticket_close' \
  --data-urlencode 'ticket_id=1' \
  --data-urlencode 'rating=5' \
  --data-urlencode 'feedback=Petugas datang dan perbaikan sudah selesai' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


7. Endpoint Admin Terkait Tiket
------------------------------

Endpoint berikut berada di file yang sama (`pelanggan_portal.php`) tetapi ditujukan untuk panel admin, bukan untuk Portal Pelanggan publik. Disertakan di sini untuk referensi.

Semua endpoint admin:

- Menggunakan `require_valid_csrf();`
- Menggunakan `require_auth();`
- Bergantung pada `$_SESSION['company_id']` dan `$_SESSION['user_role']`

### [DOC 6.1] Daftar Tiket Closed (Admin)

- Action: `admin_tickets_closed_list`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Parameter utama:

- `action`: `admin_tickets_closed_list`
- `search`: string (opsional), pencarian berdasarkan id, subject, company, nama pelanggan
- `limit`: int (opsional, default 25)
- `offset`: int (opsional, default 0)
- `sort_by`: string (opsional, kolom yang diizinkan: `id`, `subject`, `priority`, `status`, `created_at`, `closed_at`, `company_name`, `created_by_name`)
- `sort_dir`: `asc` atau `desc` (default `desc`)
- `csrf_token`: string

Respons:

```json
{
  "status": "success",
  "data": [ { ... } ],
  "total": 10,
  "limit": 25,
  "offset": 0
}
```

Setiap elemen `data` berisi informasi tiket, feedback, dan data pelanggan terkait.

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=admin_tickets_closed_list' \
  --data-urlencode 'search=Internet' \
  --data-urlencode 'limit=25' \
  --data-urlencode 'offset=0' \
  --data-urlencode 'sort_by=closed_at' \
  --data-urlencode 'sort_dir=desc' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```

### [DOC 6.2] Detail Tiket (Admin)

- Action: `admin_ticket_detail`
- Method: `POST`
- URL: `/api/pelanggan_portal.php`

Request body:

- `action`: `admin_ticket_detail`
- `ticket_id`: integer
- `csrf_token`: string

Respons sukses:

```json
{
  "status": "success",
  "data": {
    "id": 1,
    "subject": "Internet lambat",
    "priority": "high",
    "status": "closed",
    "description": "Detail keluhan",
    "created_at": "...",
    "closed_at": "...",
    "company_name": "Nama ISP",
    "created_by_user_id": "pel-1",
    "rating": 5,
    "feedback": "Pelayanan bagus",
    "created_by_name": "Nama Pelanggan",
    "created_by_phone": "62812..."
  }
}
```

Contoh error:

```json
{ "status": "error", "message": "ID tiket tidak valid" }
```

```json
{ "status": "error", "message": "Tiket tidak ditemukan" }
```

Contoh request `curl`:

```bash
curl -X POST 'http://sample.sarang.id/api/pelanggan_portal.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'action=admin_ticket_detail' \
  --data-urlencode 'ticket_id=1' \
  --data-urlencode 'csrf_token=TOKEN_CSRF_VALID'
```


Catatan Implementasi Client
---------------------------

- Di frontend bawaan (`pelanggan_portal.js`), semua request dikirim ke `../api/pelanggan_portal.php` dengan `fetch` dan body `URLSearchParams`.
- `csrf_token` biasanya disediakan melalui objek global `window.PELANGGAN_PORTAL.csrf_token`.
- Client eksternal (misalnya aplikasi mobile) dapat mengikuti spesifikasi di atas dengan:
  - Menjaga cookie session dan `pelanggan_device_id`.
  - Mengirim `csrf_token` sesuai mekanisme yang disediakan backend (perlu endpoint tambahan jika ingin pure API tanpa halaman HTML).

