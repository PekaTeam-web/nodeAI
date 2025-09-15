# DKN × Vikey Proxy Starter

Jalankan Dria Compute Node (DKN) tanpa Ollama lokal. DKN akan “mengira” berbicara ke Ollama, tetapi request dialihkan oleh proxy ke API Vikey (OpenAI-style). Ini mengatasi perbedaan penamaan model:
- DKN (Ollama-style): `llama3.3:70b-instruct-q4_K_M`
- Vikey (OpenAI-style): `llama-3.3-70b-instruct`

Proxy memetakan nama model tersebut lewat `MODEL_MAP`.

## Arsitektur Singkat
DKN Compute Node → (Ollama API) → Proxy (host:14441) → (OpenAI API) → Vikey

## Prasyarat VPS
- Ubuntu 22.04/24.04 (sudo)
- VIKEY_API_KEY siap
- EVM Private Key (untuk `DKN_WALLET_SECRET_KEY`)
- Docker Engine + Docker Compose plugin
- Node.js 20 LTS + PM2

## Instalasi Ringkas di VPS Baru

1) Update OS dan paket pendukung
```bash
sudo apt-get update -y
sudo apt-get install -y curl git jq ca-certificates gnupg lsb-release netcat-openbsd
```

2) Pasang Node.js LTS 20.x + PM2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo npm i -g pm2
```

3) Pasang Docker + Compose plugin
```bash
sudo apt-get install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
# Re-login sesi shell/SSH agar grup docker aktif
```

4) Clone repo dan siapkan proxy
```bash
git clone <URL_REPO_KAMU> dkn-vikey-proxy-starter
cd dkn-vikey-proxy-starter/proxy
cp .env.example .env
# Edit .env → isi VIKEY_API_KEY dan mapping MODEL_MAP
npm install
pm2 start index.js --name ollama2vikey
pm2 save
# Tes:
curl -s http://localhost:14441/health | jq
curl -s http://localhost:14441/api/tags | jq
```

5) Siapkan network Docker untuk node
```bash
docker network create dria-nodes || true
```

6) Siapkan ENV untuk DKN
```bash
cd ..
cp .env.example .env
# Edit .env → isi DKN_WALLET_SECRET_KEY (jangan commit)
# DKN_MODELS boleh tetap default: llama3.3:70b-instruct-q4_K_M
```

7) Jalankan Compute Node
```bash
docker compose up -d
docker compose logs -f
```

Jika lancar, DKN akan konek ke jaringan dan memakai proxy untuk inferensi Vikey.

## Konfigurasi

- `proxy/.env`
  - `VIKEY_API_KEY`: API key Vikey kamu.
  - `PORT`: Port proxy (default 14441).
  - `MODEL_MAP`: JSON pemetaan nama “Ollama-style” → “Vikey-style”.
    - Contoh: `{"llama3.3:70b-instruct-q4_K_M":"llama-3.3-70b-instruct"}`
- Root `.env`
  - `DKN_WALLET_SECRET_KEY`: Private key EVM (format 0x...).
  - `DKN_MODELS`: Daftar model untuk DKN, pisah koma. Pastikan semua ada di `MODEL_MAP` proxy.
    - Contoh: `llama3.3:70b-instruct-q4_K_M`

Compose sudah set:
- `OLLAMA_HOST=http://host.docker.internal`
- `OLLAMA_PORT=14441`
- `OLLAMA_AUTO_PULL=false`
- `extra_hosts: host.docker.internal:host-gateway` agar container bisa akses proxy di host.

## Tambah Model
- Tambahkan entry ke `proxy/.env` pada `MODEL_MAP`, contoh:
  ```
  MODEL_MAP={"llama3.3:70b-instruct-q4_K_M":"llama-3.3-70b-instruct","llama3.1:8b-instruct-q4_K_M":"llama-3.1-8b-instruct"}
  ```
- Restart proxy:
  ```bash
  pm2 restart ollama2vikey
  ```
- Tambahkan ke `DKN_MODELS` di root `.env` dan restart container:
  ```bash
  docker compose down && docker compose up -d
  ```

## Troubleshooting
- Proxy hidup?
  - `curl http://localhost:14441/health`
  - `pm2 logs ollama2vikey`
- Container bisa akses proxy?
  - `docker exec -it <cid> sh -lc 'apk add --no-cache curl || (apt-get update && apt-get install -y curl); curl -s http://host.docker.internal:14441/api/tags'`
- "Model not found" di DKN:
  - Pastikan nama di `DKN_MODELS` ada di `MODEL_MAP` dan ID Vikey benar.
- 401/403 dari Vikey:
  - Cek `VIKEY_API_KEY` pada `proxy/.env`.

## Keamanan
- Jangan commit file `.env` (root dan proxy).
- Simpan `DKN_WALLET_SECRET_KEY` dengan aman. Gunakan pengaturan permission ketat pada server.

## Buat Repo di GitHub
- Via UI: buat repo kosong, lalu:
  ```bash
  cd dkn-vikey-proxy-starter
  git init
  git add .
  git commit -m "Initial commit: DKN × Vikey proxy starter"
  git branch -M main
  git remote add origin https://github.com/PekaTeam-web/dkn-vikey-proxy-starter.git
  git push -u origin main
  ```
- Via GitHub CLI:
  ```bash
  gh repo create PekaTeam-web/dkn-vikey-proxy-starter --public --source=. --remote=origin --push
  ```

Butuh aku yang push file ini langsung ke repo (nama apa, public/private)? Bilang ya.
