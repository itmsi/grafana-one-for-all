# Grafana One-for-all (Observability Stack)

Repository ini adalah pusat monitoring dan observability stack berbasis Docker Compose. Dirancang khusus untuk menerima metrics, logs, dan traces dari berbagai microservices (seperti Express.js) yang berjalan di Docker, tanpa menjalankan service aplikasi di dalamnya.

---

## 📁 Struktur Direktori

```
.
├── docker-compose.yml          # Konfigurasi utama stack observability
├── .env                        # Environment variables (Postgres credentials, dll)
├── configs/                    # File konfigurasi tiap service
│   ├── prometheus/             # prometheus.yml (konfigurasi scrape metrics)
│   ├── loki/                   # loki-config.yml (storage logs + limits pino-loki)
│   ├── promtail/               # promtail-config.yml (collection pipeline from docker)
│   ├── tempo/                  # tempo.yml (tracing storage & receiver)
│   └── blackbox/               # blackbox.yml (konfigurasi HTTP probes /health)
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/        # datasources.yml (Auto-tambah Prometheus, Loki, Tempo)
│   │   └── dashboards/         # dashboard.yml (Auto-load dari folder dashboards/)
│   └── dashboards/
│       ├── overview.json       # Dashboard general overview
│       ├── postgres-dashboard.json
│       └── netsuite/           # ← DASHBOARD KHUSUS NETSUITE MIDDLEWARE
│           └── netsuite-middleware.json
└── README.md
```

---

## 🚀 Cara Menjalankan Stack Ini

### 1. Buat Docker External Network (jika belum ada)

```bash
docker network create infra_net
```

### 2. Setup Environment Variables

```bash
cp .env.example .env
# Edit sesuai kredensial PostgreSQL Anda
```

### 3. Jalankan Stack

```bash
docker-compose up -d
```

### 4. Akses Grafana

Buka browser: `http://localhost:3000`

| Field    | Value   |
|----------|---------|
| Username | `admin` |
| Password | `admin` |

---

## 🔍 NetSuite Middleware - Log Monitoring

### Cara Kerja

```
netsuite-middleware (pino-loki) ──push──► Loki :3100
                                               │
Promtail (docker.sock) ──scrape──► Loki :3100  │
                                               │
                               Grafana :3000 ◄─┘
```

Terdapat **dua cara** log dapat masuk ke Loki:

1. **pino-loki (push langsung)**: Service middleware push log langsung ke Loki via HTTP
2. **Promtail (scrape docker)**: Promtail membaca stdout container lalu parse JSON

### Dashboard Siap Pakai

Dashboard Grafana untuk NetSuite Middleware tersedia di:
`grafana/dashboards/netsuite/netsuite-middleware.json`

Dashboard ini akan **auto-load** di folder **"NetSuite Middleware"** setelah stack berjalan.

#### Panel yang Tersedia:

| Panel | Deskripsi |
|-------|-----------|
| 📊 Total Requests | Jumlah total request dalam rentang waktu |
| 🔴 Error Count | Jumlah log level=error |
| ⏱ Avg Latency | Rata-rata duration_ms |
| 👥 Active Clients | Jumlah client_id unik |
| 📋 Request Logs Table | Tabel: timestamp, client_id, method, url, status_code, duration_ms |
| 👥 Request Count per Client | Time series request per client_id |
| ⏱ Latency by Client | Avg & max latency per client_id |
| 📊 Log Level Distribution | Pie chart: info/warn/error |
| 📊 Status Code Distribution | Time series per HTTP status code |
| 🚨 Error Logs | Log viewer filter level=error saja |
| 🔍 Raw JSON Viewer | Full log dengan request_body, response_body, curl |

#### Filter/Variable yang Tersedia:

| Variable | Fungsi |
|----------|--------|
| `$client_id` | Filter per client — multi-select, dropdown dinamis dari label Loki |
| `$level` | Filter log level (info/warn/error/debug) |
| `$service_name` | Filter service — default: netsuite-middleware |

---

## 📊 Contoh LogQL Queries

### 1. Semua Log dari Middleware

```logql
{service_name="netsuite-middleware"}
```

### 2. Parse JSON + Filter by Client ID

```logql
{service_name="netsuite-middleware"}
| json
| client_id="CLIENT_ABC"
```

### 3. Hanya Error Log

```logql
{service_name="netsuite-middleware"}
| json
| level="error"
```

### 4. Filter Multi-label (client + level)

```logql
{service_name="netsuite-middleware", client_id="CLIENT_ABC"}
| json
| level="error"
```

### 5. Filter berdasarkan URL / Endpoint

```logql
{service_name="netsuite-middleware"}
| json
| url=~"/api/netsuite/.*"
```

### 6. Tampilkan Curl Request (untuk debugging)

```logql
{service_name="netsuite-middleware"}
| json
| curl != ""
| line_format "{{.curl}}"
```

### 7. Tampilkan Request Body

```logql
{service_name="netsuite-middleware"}
| json
| request_body != ""
| line_format "CLIENT: {{.client_id}} | BODY: {{.request_body}}"
```

### 8. Filter Response dengan Status Code 4xx/5xx

```logql
{service_name="netsuite-middleware"}
| json
| status_code >= 400
```

### 9. Request Count per Client (Rate)

```logql
sum by (client_id) (
  count_over_time(
    {service_name="netsuite-middleware"}
    | json
    | method != ""
    [5m]
  )
)
```

### 10. Avg Latency per Client

```logql
avg by (client_id) (
  avg_over_time(
    {service_name="netsuite-middleware"}
    | json
    | duration_ms > 0
    | unwrap duration_ms
    | __error__=""
    [5m]
  )
)
```

---

## 🔌 Konfigurasi pino-loki di Service API

Tambahkan ke `package.json`:

```bash
npm install pino-loki
```

Di kode Express.js (contoh dengan pino):

```javascript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service_name: process.env.SERVICE_NAME || 'netsuite-middleware',
  },
  transport: {
    targets: [
      // Output ke stdout (agar Promtail bisa scrape)
      {
        target: 'pino-pretty',
        level: 'debug',
        options: { destination: 1, colorize: false }
      },
      // Push langsung ke Loki
      {
        target: 'pino-loki',
        level: 'info',
        options: {
          host: process.env.LOKI_URL || 'http://loki:3100',
          labels: {
            service_name: process.env.SERVICE_NAME || 'netsuite-middleware',
          },
          // Label dinamis dari log line
          propsToLabels: ['level', 'client_id'],
          replaceTimestamp: true,
          silenceErrors: false,
        }
      }
    ]
  }
});
```

Contoh log request:

```javascript
logger.info({
  msg: 'Outbound HTTP Request',
  client_id: req.headers['x-client-id'],
  method: req.method,
  url: req.originalUrl,
  status_code: res.statusCode,
  duration_ms: Date.now() - startTime,
  request_body: JSON.stringify(req.body),
  response_body: JSON.stringify(responseData),
  curl: generateCurl(req),
  traceId: req.traceId,
});
```

---

## ➕ Cara Menambahkan Service Baru ke Monitoring

### 1. Pastikan Service Terkoneksi ke `infra_net`

Di dalam `docker-compose.yml` milik **microservice API**:

```yaml
networks:
  default:
    name: infra_net
    external: true
```

### 2. Menambahkan Metrics Scraping (Prometheus)

Edit `configs/prometheus/prometheus.yml`:

```yaml
  - job_name: 'microservices'
    metrics_path: '/metrics'
    static_configs:
      - targets:
          - 'netsuite-middleware:3001'
          - 'new-service:3002' # ← Tambah di sini
```

```bash
docker-compose restart prometheus
```

### 3. Log (Promtail & Loki)

Log service akan **otomatis terbaca** karena Promtail terhubung langsung dengan `docker.sock`.
Label `service_name` di-generate otomatis dari nama container Docker.

> Tips: Pastikan logging outputnya berupa JSON agar mudah di-parse oleh Loki.

### 4. Distributed Tracing (Tempo)

Tempo berfungsi sebagai **OpenTelemetry OTLP Receiver** di port `4318` (HTTP):
- **OTLP Endpoint**: `http://tempo:4318/v1/traces`

### 5. API Health Check (Blackbox Exporter)

Edit `configs/prometheus/prometheus.yml`:

```yaml
    static_configs:
      - targets:
        - 'http://netsuite-middleware:3001/health'
        - 'http://new-service:3002/health' # ← Tambahan
```

---

## ✅ Cara Memastikan Service Terbaca di Grafana

1. **Cek Koneksi Datasource:**
   Buka Grafana → Data Sources → Klik Prometheus/Loki/Tempo → **Save & Test**

2. **Cek Metrics (Prometheus):**
   - Menu **Explore** → Data Source: `Prometheus`
   - Query: `up`

3. **Cek Logs (Loki):**
   - Menu **Explore** → Data Source: `Loki`
   - Query: `{service_name="netsuite-middleware"}`

4. **Cek Dashboard:**
   - Menu **Dashboards** → Folder: **"NetSuite Middleware"**
   - Dashboard: "NetSuite Middleware - Monitoring"

5. **Cek Tracing (Tempo):**
   - Buka log yang mengandung `traceId`
   - Klik link TraceID untuk membuka di Tempo
