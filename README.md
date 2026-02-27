# Grafana One-for-all (Observability Stack)

Repository ini adalah pusat monitoring dan observability stack berbasis Docker Compose. Dirancang khusus untuk menerima metrics, logs, dan traces dari berbagai microservices (seperti Express.js) yang berjalan di Docker, tanpa menjalankan service aplikasi di dalamnya.

## 📁 Struktur Direktori

```
.
├── docker-compose.yml          # Konfigurasi utama stack observability
├── configs/                    # File konfigurasi tiap service
│   ├── prometheus/             # prometheus.yml (konfigurasi scrape metrics)
│   ├── loki/                   # loki-config.yml (storage logs)
│   ├── promtail/               # promtail-config.yml (collection pipeline dari docker)
│   ├── tempo/                  # tempo.yml (tracing storage & receiver)
│   └── blackbox/               # blackbox.yml (konfigurasi HTTP probes probe /health)
├── grafana/
│   ├── provisioning/           
│   │   ├── datasources/        # datasources.yml (Auto-tambah Prometheus, Loki, Tempo)
│   │   └── dashboards/         # dashboards.yml (Auto-load dari folder dashboards/)
│   └── dashboards/
│       └── overview.json       # (Contoh) Dashboard placeholder
└── README.md
```

---

## 🚀 Cara Menjalankan Stack Ini

1. **Pastikan external network sudah terbuat**
   Semua service microservices dan monitoring harus berada di satu external network bernama `infra_net` (atau sesuaikan dengan nama network yang Anda inginkan di `docker-compose.yml`).
   Jika belum ada, buat dengan perintah:
   ```bash
   docker network create infra_net
   ```

2. **Jalankan container**
   Masuk ke direktori ini, kemudian deploy stack-nya di background:
   ```bash
   docker-compose up -d
   ```

3. **Akses Grafana**
   Grafana akan berjalan di port `3000`. Akses melalui browser Anda di `http://localhost:3000` (atau IP/Domain server Anda).
   - **User**: `admin`
   - **Password**: `admin`

---

## ➕ Cara Menambahkan Service Baru ke Monitoring

### 1. Pastikan Service Terkoneksi ke `infra_net`
Di dalam `docker-compose.yml` milik **microservice API**, tambahkan network yang sama:
```yaml
networks:
  default:
    name: infra_net
    external: true
```

### 2. Menambahkan Metrics Scraping (Prometheus)
Edit file `configs/prometheus/prometheus.yml`, cari bagian `job_name: 'microservices'` dan tambahkan targetnya menggunakan *hostname* dan port service tersebut:
```yaml
  - job_name: 'microservices'
    metrics_path: '/metrics'
    static_configs:
      - targets:
          - 'auth-service:3001'
          - 'crm-service:3002'
          - 'new-service:3003' # <--- Tambah di sini
```
Kemudian restart Prometheus agar membaca ulang config:
```bash
docker-compose restart prometheus
```

### 3. Log (Promtail & Loki)
Log service akan **otomatis terbaca** karena Promtail terhubung langsung dengan `docker.sock` dan membaca seluruh container logger.
Label `<service_name>` di-generate secara otomatis berdasarkan nama container Docker Anda.
> *Tips: Pastikan logging di Express.js (Pino/Winston) outputnya berupa JSON agar mudah di-parse oleh Loki.*

### 4. Distributed Tracing (Tempo)
Tempo berfungsi sebagai **OpenTelemetry OTLP Receiver** jalan di port `4318` (via HTTP).
Di dalam service Express.js Anda, pastikan OpenTelemetry Exporter diset mengirim trace ke *Tempo hostname*:
- **OTLP Endpoint**: `http://tempo:4318/v1/traces`
Jika menggunakan SDK Node.js, tempo otomatis menampung trace Anda.

### 5. API Health Check (Blackbox Exporter)
Edit `configs/prometheus/prometheus.yml` bagian `job_name: 'blackbox-http'`, lalu tambahkan target endpoint `/health`:
```yaml
    static_configs:
      - targets:
        - 'http://auth-service:3001/health'
        - 'http://new-service:3003/health' # <--- Tambahan
```

### 6. PostgreSQL Monitoring
Secara default, metrics dari PostgreSQL diekstrak menggunakan `postgresql-exporter`.
Untuk mengubah target database yang di-monitor, edit environment variable `DATA_SOURCE_NAME` di `docker-compose.yml` pada service `postgres-exporter`:
```yaml
DATA_SOURCE_NAME=postgresql://username:password@db-hostname:5432/dbname?sslmode=disable
```

---

## ✅ Cara Memastikan Service Terbaca di Grafana

1. **Cek Koneksi Datasource:**
   Buka Grafana -> Data Sources (Icon Gear -> Data Sources). Klik Prometheus, Loki, atau Tempo -> Klik **Save & Test**. Harus muncul status sukses.

2. **Cek Metrics (Prometheus):**
   - Buka menu **Explore** (Icon kompas).
   - Pilih Data Source `Prometheus`.
   - Jalankan query sederhana: `up`
   - Anda akan melihat nilai `1` untuk semua service (seperti node-exporter, cadvisor, auth-service, dll.) jika config scrape berhasil.

3. **Cek Logs (Loki):**
   - Di menu **Explore**, ubah Data Source ke `Loki`.
   - Pilih tab **Label browser**.
   - Cari label `service_name` (Ini adalah label yang kita generate dari Container Name menggunakan Promtail config). Pilih container API yang bersangkutan.
   - Run query (Contoh: `{service_name="auth-service"}`). Anda akan melihat log langsung termuncul dari container tersebut.

4. **Cek Tracing (Tempo):**
   - Lakukan satu request API pada Express App.
   - Buka Loki -> Cari HTTP Request log yang mengandung `trace_id` (jika aplikasi Anda inject log menggunakan OpenTelemetry/Pino).
   - Karena file `datasources.yml` sudah disetup, Anda akan bisa mengklik langsung Trace ID tersebut dari tampilan Log, dan **Tempo akan membuka split view** menampilkan hierarki service execution!
