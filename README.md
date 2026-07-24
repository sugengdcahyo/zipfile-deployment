# Zipline deployment

Deployment Docker Compose untuk [Zipline](https://zipline.diced.sh/) dengan
PostgreSQL dan penyimpanan lokal, AWS S3, atau MinIO/S3-compatible eksternal.

MinIO **tidak** dijalankan oleh Compose ini. Siapkan bucket dan kredensial pada
layanan object storage yang sudah tersedia terlebih dahulu.

## Prasyarat

- Docker Engine dan Docker Compose plugin
- Reverse proxy HTTPS (misalnya NGINX) bila instance akan diakses dari internet
- Bucket S3 atau MinIO, bila tidak menggunakan penyimpanan lokal

## Menyiapkan konfigurasi

Salin template lalu ganti seluruh nilai contoh, terutama password PostgreSQL dan
`CORE_SECRET` (minimal 32 karakter):

```bash
cp .env.example .env
```

`CORE_SECRET` dapat dibuat dengan:

```bash
openssl rand -base64 48
```

Konfigurasi dasar yang diperlukan:

```env
POSTGRESQL_USER=zipline
POSTGRESQL_DB=zipline
POSTGRESQL_PASSWORD=ganti-dengan-password-kuat

CORE_SECRET=ganti-dengan-random-string-minimal-32-karakter
CORE_HOSTNAME=0.0.0.0
CORE_PORT=3000
CORE_TRUST_PROXY=true
CORE_RETURN_HTTPS_URLS=true
```

## Jaringan dan akses port

Compose menggunakan network default Docker yang dialokasikan otomatis, sehingga
tidak ada subnet statis atau network kustom yang dapat bertabrakan dengan
network lain.

Zipline dipublikasikan hanya pada `127.0.0.1:3000`, bukan pada semua interface
host. Gunakan reverse proxy lokal untuk menyediakan HTTPS publik.

```env
ZIPLINE_BIND_ADDRESS=127.0.0.1
ZIPLINE_HOST_PORT=3000
```

Jangan gunakan `0.0.0.0` sebagai `ZIPLINE_BIND_ADDRESS` kecuali firewall dan
kontrol akses jaringan sudah dikonfigurasi dengan benar.

## Pilihan storage

### Penyimpanan lokal

Ini adalah konfigurasi bawaan. Berkas akan disimpan pada direktori `./uploads`
di host dan dimount ke container sebagai `/zipline/uploads`.

```env
DATASOURCE_TYPE=local
DATASOURCE_LOCAL_DIRECTORY=/zipline/uploads
```

Pastikan direktori `uploads`, `public`, dan `themes` tersedia serta dapat ditulis
oleh Docker.

### AWS S3

Buat bucket dan IAM access key dengan izin baca, tulis, hapus, serta daftar
objek pada bucket tersebut. Aktifkan S3 dengan nilai berikut di `.env`:

```env
DATASOURCE_TYPE=s3
DATASOURCE_S3_ACCESS_KEY_ID=AKIA...
DATASOURCE_S3_SECRET_ACCESS_KEY=...
DATASOURCE_S3_BUCKET=zipline
DATASOURCE_S3_REGION=ap-southeast-1
DATASOURCE_S3_FORCE_PATH_STYLE=false
```

Jangan set `DATASOURCE_S3_ENDPOINT` untuk AWS S3 standar. Region harus sama
dengan region bucket.

### MinIO atau layanan S3-compatible eksternal

Buat bucket dan access key khusus untuk Zipline pada MinIO yang sudah ada.
Endpoint wajib dapat dijangkau dari container `zipline`; gunakan hostname/IP
yang dapat diakses dari jaringan Docker, bukan `localhost` kecuali MinIO
memang berjalan di container yang sama (tidak direkomendasikan).

```env
DATASOURCE_TYPE=s3
DATASOURCE_S3_ACCESS_KEY_ID=zipline-access-key
DATASOURCE_S3_SECRET_ACCESS_KEY=zipline-secret-key
DATASOURCE_S3_BUCKET=zipline
DATASOURCE_S3_REGION=us-east-1
DATASOURCE_S3_ENDPOINT=https://minio.example.com
DATASOURCE_S3_FORCE_PATH_STYLE=true
```

Jika MinIO berada pada Docker network lain, sambungkan service `zipline` ke
network eksternal tersebut atau publikasikan endpoint MinIO agar dapat diakses
dari host/container ini. Path-style (`true`) biasanya diperlukan oleh MinIO.

## Menjalankan

Tarik image lalu mulai service di background:

```bash
docker compose pull
docker compose up -d
```

Cek status dan log:

```bash
docker compose ps
docker compose logs -f zipline
```

Secara default Zipline hanya dipublikasikan ke `127.0.0.1:3000`. Akses melalui
reverse proxy yang meneruskan trafik ke alamat tersebut. Setelah server aktif,
buka URL publik Zipline untuk menyelesaikan pembuatan akun super administrator.

## Memperbarui

```bash
docker compose pull
docker compose up -d
```

## Catatan penting

- Jangan commit `.env`; berkas tersebut berisi rahasia.
- Ganti datasource hanya setelah object storage dan bucket siap. File lama pada
  `uploads/` tidak dipindahkan otomatis ke S3/MinIO.
- Sebelum mengganti ke S3/MinIO pada instance yang sudah berjalan, backup volume
  PostgreSQL dan direktori `uploads/`.
- Untuk konfigurasi Zipline terbaru, lihat [dokumentasi konfigurasi resmi](https://zipline.diced.sh/docs/config).
