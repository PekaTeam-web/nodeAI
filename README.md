# 🧩 Dria Nodes Setup Script

**File:** `dria.sh`
**Repo:** [ai-nodes-setup](https://github.com/direkturcrypto/ai-nodes-setup)
**Credits:** powered by **direkturcrypto** 🙏

This script automates the installation and setup of:

* Docker & Docker Compose 🐳
* Node.js, npm, pm2 📦
* Vikey Inference 🔑
* EVM crypto wallets 💰
* Dria nodes ⚡ (configurable nodes per wallet)

---

## 🔧 Prerequisites

* Ubuntu/Debian-based Linux machine
* `sudo` access
* Internet connection
* Valid `VIKEY_API_KEY` from [Vikey API](https://api.vikey.ai)

### Prerequisites Check

Before running the script, verify you have the basic requirements:

```bash
# Check if you have sudo access
sudo -l

# Check internet connectivity
ping -c1 google.com

# Check if you're on a compatible system
lsb_release -a
```

---

## 📥 Installation

Clone the repository:

```bash
git clone https://github.com/direkturcrypto/ai-nodes-setup
cd ai-nodes-setup
```

Make the script executable:

```bash
chmod +x dria.sh
```

---

## 🚀 Usage

Run the setup script:

```bash
./dria.sh
```

### During setup you will be asked:

1. **Enter your `VIKEY_API_KEY`** 🔐
   This key will be saved in the Vikey `.env` file.

2. **Choose wallet option** 💰
   - Option 1: Generate new wallets
   - Option 2: Use existing wallet.json file

3. **How many wallets to generate** (if option 1)
   Example: if you enter `5`, it will generate **5 wallets**.

4. **How many nodes per wallet**
   Example: if you enter `3`, each wallet will run **3 Dria nodes**.

---

## 📂 What Gets Created

* `~/vikey-inference/` → Vikey inference server + `.env`
* `~/crypto-generator/wallets.json` → Generated wallets (address & private key)
* `~/dria-nodes/dria-node-<wallet_address>/docker-compose.yml` → Node configs (multiple nodes per wallet)
* `~/dria-nodes/manage-dria.sh` → Helper script to manage Dria nodes

---

## ⚡ Managing Dria Nodes

After setup, use the helper script:

### Start all nodes

```bash
cd ~/dria-nodes
./manage-dria.sh start
```

✅ Brings up all Dria nodes in the background.

### Restart all nodes

```bash
cd ~/dria-nodes
./manage-dria.sh restart
```

♻️ Restarts all running nodes.

---

## 🔍 Verify Vikey

Run:

```bash
curl http://localhost:14441
```

Expected response:

```json
{"error":"Endpoint not supported"}
```

This confirms Vikey is running correctly.

---

## 🪵 Logs

To check Vikey logs:

```bash
tail -f ~/vikey-inference/vikey.log
```

To check Dria node logs:

```bash
cd ~/dria-nodes/dria-node-<WALLET_ADDRESS>
docker-compose logs -f
```

---

## 🔧 Troubleshooting

### Common Issues

**1. Permission denied errors**
```bash
sudo usermod -aG docker $USER
# Log out and log back in
```

**2. Vikey not responding**
```bash
# Check if Vikey is running
ps aux | grep vikey
# Restart Vikey if needed
cd ~/vikey-inference
./vikey-inference-linux &
```

**3. Docker network issues**
```bash
# Recreate the network
docker network rm dria-nodes
docker network create --subnet=10.172.0.0/16 dria-nodes
```

**4. Node not starting**
```bash
# Check Docker logs
cd ~/dria-nodes/dria-node-<WALLET_ADDRESS>
docker-compose logs
```

### Verification Commands

**Check all services are running:**
```bash
# Vikey status
curl -s http://localhost:14441 | jq .

# Docker containers
docker ps | grep dkn-compute-node

# Network status
docker network ls | grep dria-nodes
```

---

## 🙏 Credits

This setup is proudly **powered by direkturcrypto**.
