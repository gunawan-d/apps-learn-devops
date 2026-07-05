# Maintenance Page

Halaman maintenance statis yang di-serve via nginx di dalam Docker container.

## Struktur File

```
.
├── index.html       # Halaman maintenance
├── Dockerfile       # Multi-stage: builder (tidy) → runtime (nginx)
├── .dockerignore    # Exclude file yang tidak perlu
└── README.md
```

## Multi-Stage Build

Dockerfile ini punya **2 stage**:

| Stage   | Base Image         | Fungsi                                  |
|---------|--------------------|-----------------------------------------|
| builder | `alpine:3.20`      | Validasi & minify HTML pakai `tidy`     |
| runtime | `nginx:1.27-alpine`| Serve file final (image kecil ~40MB)    |

**Kenapa multi-stage?**
- Stage `builder` di-buang setelah build → image akhir kecil
- HTML di-validate saat build → kalau markup rusak, build langsung gagal
- Source HTML asli tidak masuk ke image akhir → kecil & bersih
- Bisa tambah proses lain di builder (minify CSS/JS, hash asset, dll)

## Cara Build & Run

### Opsi 1: Docker Compose (recommended)

```bash
# Copy env file (opsional, untuk customize port)
cp .env.example .env

# Build + run
docker compose up -d --build

# Lihat status & logs
docker compose ps
docker compose logs -f maintenance

# Stop
docker compose down
```

### Opsi 2: Docker Manual

```bash
# Build image
docker build -t maintenance-page .

# Run container
docker run -d -p 8080:80 --name maintenance maintenance-page

# Akses di browser
open http://localhost:8080
```

## Custom Domain / Production

Untuk production, tambahkan nginx config custom dan mount sebagai volume:

```bash
docker run -d -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf:ro \
  --name maintenance \
  maintenance-page
```

## Status Codes

Secara default nginx mengirim `200 OK`. Jika ingin return `503 Service Unavailable`
(lebih sesuai untuk maintenance), tambahkan di nginx.conf:

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
        add_header Retry-After 3600;
        return 503;
    }

    error_page 503 @maintenance;
    location @maintenance {
        root /usr/share/nginx/html;
        rewrite ^(.*)$ /index.html break;
    }
}
```