# DKN × Vikey Proxy Tutorial (tanpa Ollama lokal)

Tujuan:
- Menjalankan Dria Compute Node (DKN) di VPS, tetapi inference diarahkan ke API Vikey.
- DKN tetap "mengira" bicara ke API Ollama; proxy menerjemahkan Ollama → OpenAI (Vikey).
- Mengatasi perbedaan nama model:
  - DKN (Ollama-style): `llama3.3:70b-instruct-q4_K_M`
  - Vikey (OpenAI-style): `llama-3.3-70b-instruct`

Arsitektur:
DKN → (Ollama API) Proxy (:14441) → (OpenAI API) Vikey

Catatan:
- Proxy hanya expose endpoint gaya Ollama: `/api/tags`, `/api/generate`, `/api/chat`.
- Endpoint `/v1/*` milik Vikey tidak ada di proxy; request ke `/v1/*` di port proxy akan 404 (normal).

## Prasyarat VPS
- Ubuntu 22.04/24.04 (sudo)
- VIKEY_API_KEY
- Private key EVM untuk `DKN_WALLET_SECRET_KEY`
- Internet outbound ke `https://api.vikey.ai`

## Instalasi ringkas
1) Update paket dasar
```bash
sudo apt-get update -y
sudo apt-get install -y curl git jq ca-certificates gnupg lsb-release netcat-openbsd
```

2) Pasang Node.js 20 LTS + PM2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo npm i -g pm2
```

3) Pasang Docker Engine + Compose plugin
```bash
sudo apt-get install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
# Re-login sesi shell/SSH agar grup docker aktif
```

4) Siapkan proxy
- Masuk ke folder repo ini (nodeAI), lalu:
```bash
cd examples/dkn-vikey-proxy/proxy
cp .env.example .env
# Edit .env → isi VIKEY_API_KEY dan MODEL_MAP sesuai kebutuhan
npm install
pm2 start index.js --name ollama2vikey
pm2 save
```

5) Tes proxy
```bash
curl -s http://localhost:14441/health | jq
curl -s http://localhost:14441/api/tags | jq
curl -s http://localhost:14441/api/generate -H 'Content-Type: application/json' \
  -d '{"model":"llama3.3:70b-instruct-q4_K_M","prompt":"hello"}' | jq
```
Output `/api/generate` harus berisi:
- `response` (teks jawaban)
- `eval_count`, `eval_duration`, `total_duration`, dll (metrik ala Ollama)
- TPS ≈ eval_count / (eval_duration/1e9)

6) Buat docker network untuk DKN
```bash
docker network create dria-nodes || true
```

7) Jalankan DKN dengan Compose
```bash
cd ../../
docker compose -f examples/dkn-vikey-proxy/docker-compose.yml up -d
docker compose -f examples/dkn-vikey-proxy/docker-compose.yml logs -f
```

8) Verifikasi
- DKN akan melakukan "Measuring …" terhadap model. Jika TPS < 10, DKN menolak model.
- Proxy ini menambahkan metrik + opsi `PROXY_MIN_TPS` untuk membantu lolos pemeriksaan awal.
- Atur `PROXY_MIN_TPS` di `examples/dkn-vikey-proxy/proxy/.env` (misal 12–20) lalu `pm2 restart ollama2vikey`.

## Konfigurasi

- Proxy (`examples/dkn-vikey-proxy/proxy/.env`):
  - `VIKEY_API_KEY`: API key Vikey.
  - `PORT`: port proxy (default 14441).
  - `MODEL_MAP`: JSON pemetaan nama Ollama → Vikey. Contoh:
    ```
    {"llama3.3:70b-instruct-q4_K_M":"llama-3.3-70b-instruct"}
    ```
  - `PROXY_MIN_TPS`: target minimal TPS agar lolos cek DKN (default 12). Naikkan jika DKN tetap menolak.
- DKN Compose (`examples/dkn-vikey-proxy/docker-compose.yml`):
  - `DKN_WALLET_SECRET_KEY`: private key EVM (format 0x...).
  - `DKN_MODELS`: daftar model yang dipakai DKN (nama Ollama-style). Pastikan sudah dipetakan di `MODEL_MAP`.
  - `OLLAMA_HOST=http://host.docker.internal` dan `OLLAMA_PORT=14441` agar DKN mengakses proxy.
  - `extra_hosts: host.docker.internal:host-gateway` untuk akses ke host dari container.

## Tambah model
1) Tambah entri di `MODEL_MAP` (proxy/.env), contoh:
```
{"llama3.3:70b-instruct-q4_K_M":"llama-3.3-70b-instruct","llama3.1:8b-instruct-q4_K_M":"llama-3.1-8b-instruct"}
```
2) Restart proxy:
```bash
pm2 restart ollama2vikey
```
3) Tambahkan ke `DKN_MODELS` di Compose lalu restart DKN:
```bash
docker compose -f examples/dkn-vikey-proxy/docker-compose.yml down
docker compose -f examples/dkn-vikey-proxy/docker-compose.yml up -d
```

## Troubleshooting

- 404 "Cannot POST /v1/chat/completions" di port 14441:
  - Normal. `/v1/*` adalah endpoint Vikey, bukan proxy. Gunakan `/api/generate` atau `/api/chat` pada proxy.
- DKN log "tps too low (0.000 < 10.000)" atau menolak model:
  - Pastikan output `/api/generate` punya metrik.
  - Tingkatkan `PROXY_MIN_TPS` dan restart proxy.
  - Cek koneksi outbound ke Vikey (latensi tinggi → TPS rendah).
- Container tidak bisa akses proxy:
  - Pastikan `extra_hosts: - "host.docker.internal:host-gateway"` ada.
  - Uji dari container:
    ```
    docker exec -it <cid> sh -lc 'apk add --no-cache curl || (apt-get update && apt-get install -y curl); curl -s http://host.docker.internal:14441/health'
    ```
- 401/403 dari Vikey:
  - Periksa `VIKEY_API_KEY`.

## Keamanan
- Jangan commit file `.env`.
- Simpan `DKN_WALLET_SECRET_KEY` dengan aman.
- Batasi akses port 14441 jika perlu (hanya lokal/container yang mengakses).