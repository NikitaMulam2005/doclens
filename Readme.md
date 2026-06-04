# DocLens — PDF RAG Highlighter

A local RAG (Retrieval-Augmented Generation) system that lets you upload a PDF, ask questions about it, and get back a **FOUND / NOT FOUND / PARTIAL** verdict with the relevant passages highlighted directly in the PDF.

```
┌─────────────────────────────────────────────────────┐
│  PDF  →  chunk + embed  →  FAISS index              │
│  Question  →  embed  →  similarity search           │
│  Top chunk  →  Ollama (llama3.2:3b)  →  Verdict     │
│  Verdict  →  highlight PDF  →  DocLens UI           │
└─────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Stack](#stack)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Local Setup](#local-setup)
5. [Option A — Expose via ngrok](#option-a--expose-via-ngrok)
6. [Option B — Run on Google Colab](#option-b--run-on-google-colab)
7. [Option C — Deploy on a Cloud VM](#option-c--deploy-on-a-cloud-vm)
8. [Option D — Deploy on Hugging Face Spaces](#option-d--deploy-on-hugging-face-spaces)
9. [API Reference](#api-reference)
10. [Troubleshooting](#troubleshooting)

---

## Stack

| Layer | Technology |
|---|---|
| PDF parsing & highlighting | PyMuPDF (`fitz`) |
| Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`) |
| Vector search | FAISS (`faiss-cpu`) |
| LLM inference | Ollama → `llama3.2:3b` |
| API server | Flask + flask-cors |
| Frontend | Vanilla HTML/CSS/JS (single file) |

---

## Prerequisites

Install these before anything else.

### Python packages

```bash
pip install flask flask-cors pymupdf sentence-transformers faiss-cpu
```

### Ollama

Ollama runs the LLM locally. Install it from [https://ollama.com/download](https://ollama.com/download), then pull the model:

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Pull the model (downloads ~2 GB)
ollama pull llama3.2:3b

# Verify it works
ollama run llama3.2:3b "Say hello"
```

> **Windows:** Download the `.exe` installer from the Ollama website. After install, run `ollama pull llama3.2:3b` in PowerShell.

---

## Project Structure

```
doclens/
├── app.py          ← Flask API server
├── index.html      ← Frontend (open in browser or serve statically)
└── README.md
```

---

## Local Setup

### 1. Start Ollama

```bash
ollama serve
```

Keep this terminal open. Ollama listens on `http://localhost:11434` by default.

### 2. Start the Flask API

```bash
python app.py
```

The server starts on `http://localhost:5000`. You should see:

```
✅ Model loaded
 * Running on http://0.0.0.0:5000
```

### 3. Open the frontend

Open `index.html` directly in your browser (double-click the file, or `open index.html` on macOS).

In the **API** field at the top of the UI, enter:

```
http://localhost:5000
```

Click **Connect**. The status pill should turn green.

### 4. Use it

1. Drop or select a PDF → click **Index PDF**
2. Type a question → click **Search Document** (or press `Ctrl+Enter`)
3. The verdict badge appears and the highlighted PDF loads in the viewer

---

## Option A — Expose via ngrok

Use this when you want to access the local server from another device (phone, another laptop, or sharing with someone remotely).

### Step 1 — Install ngrok

```bash
# macOS (Homebrew)
brew install ngrok

# Linux
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok

# Windows — download from https://ngrok.com/download
```

### Step 2 — Create a free account

Sign up at [https://dashboard.ngrok.com/signup](https://dashboard.ngrok.com/signup).

Copy your authtoken from [https://dashboard.ngrok.com/get-started/your-authtoken](https://dashboard.ngrok.com/get-started/your-authtoken).

```bash
ngrok config add-authtoken YOUR_TOKEN_HERE
```

### Step 3 — Start your local server

```bash
# Terminal 1
ollama serve

# Terminal 2
python app.py
```

### Step 4 — Start the tunnel

```bash
# Terminal 3
ngrok http 5000
```

You will see output like:

```
Forwarding    https://a1b2-203-0-113-42.ngrok-free.app -> http://localhost:5000
```

### Step 5 — Connect the frontend

Open `index.html` in your browser. In the **API** field, paste the `https://` ngrok URL (no trailing slash):

```
https://a1b2-203-0-113-42.ngrok-free.app
```

Click **Connect**.

> **Note:** The free ngrok URL changes every time you restart the tunnel. Pin a static domain by upgrading to a paid plan, or re-paste the new URL each session.

> **"Browser Warning" page:** The frontend already sends the `ngrok-skip-browser-warning: true` header on every request, so you will never see the ngrok interstitial page.

---

## Option B — Run on Google Colab

Colab gives you a free GPU and is the easiest way to run this without any local setup.

### Full setup cell — paste into a new notebook

```python
# Cell 1 — Install dependencies
!pip install flask flask-cors pymupdf sentence-transformers faiss-cpu pyngrok -q

# Cell 2 — Install Ollama inside Colab
!curl -fsSL https://ollama.com/install.sh | sh
import subprocess, time, threading

def run_ollama():
    subprocess.Popen(["ollama", "serve"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

threading.Thread(target=run_ollama, daemon=True).start()
time.sleep(5)  # give Ollama time to start

# Pull the model
!ollama pull llama3.2:3b

# Cell 3 — Paste your full app.py code here, then run the Flask server
# (copy the contents of app.py into a cell and run it,
#  or upload app.py and run: !python app.py &)

# Cell 4 — Start ngrok tunnel
from pyngrok import ngrok
public_url = ngrok.connect(5000)
print("🔗 DocLens API URL:", public_url)
```

Copy the printed URL into the **API** field in `index.html`.

> **Colab tip:** Runtime disconnects after ~90 minutes of inactivity on the free tier. Use Colab Pro or keep the tab active for longer sessions.

---

## Option C — Deploy on a Cloud VM

This is the most stable option for continuous availability. Works on AWS EC2, Google Cloud Compute Engine, Azure VM, DigitalOcean Droplet, or any VPS.

### Recommended specs

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 8 GB | 16 GB |
| Disk | 20 GB | 40 GB |
| GPU | not required | optional (speeds up embeddings) |

A **DigitalOcean $24/month droplet** (4 vCPU / 8 GB RAM) or **AWS t3.large** works well.

### Step 1 — Provision and SSH in

```bash
ssh root@YOUR_SERVER_IP
```

### Step 2 — Install dependencies on the server

```bash
# Update system
apt update && apt upgrade -y

# Python
apt install python3-pip python3-venv -y

# Ollama
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2:3b

# Python packages
pip3 install flask flask-cors pymupdf sentence-transformers faiss-cpu
```

### Step 3 — Upload your files

From your local machine:

```bash
scp app.py root@YOUR_SERVER_IP:/root/doclens/
scp index.html root@YOUR_SERVER_IP:/root/doclens/
```

Or clone from your repo:

```bash
git clone https://github.com/yourname/doclens.git
cd doclens
```

### Step 4 — Run with systemd (so it survives reboots)

Create a service file:

```bash
nano /etc/systemd/system/doclens.service
```

Paste:

```ini
[Unit]
Description=DocLens Flask API
After=network.target

[Service]
User=root
WorkingDirectory=/root/doclens
ExecStartPre=/bin/bash -c 'ollama serve &'
ExecStart=/usr/bin/python3 app.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable doclens
systemctl start doclens
systemctl status doclens
```

### Step 5 — Open firewall port

```bash
# UFW (Ubuntu)
ufw allow 5000/tcp
ufw reload

# AWS — also open port 5000 in your EC2 Security Group (Inbound Rules)
```

### Step 6 — Connect the frontend

In `index.html`, set the API field to:

```
http://YOUR_SERVER_IP:5000
```

### Step 7 (optional) — Add HTTPS with nginx + Certbot

```bash
apt install nginx certbot python3-certbot-nginx -y

# Create nginx config
cat > /etc/nginx/sites-available/doclens << 'EOF'
server {
    server_name yourdomain.com;
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 50M;
    }
}
EOF

ln -s /etc/nginx/sites-available/doclens /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# Issue SSL cert
certbot --nginx -d yourdomain.com
```

Now your API is available at `https://yourdomain.com` — paste that into the frontend.

---

## Option D — Deploy on Hugging Face Spaces

Spaces gives you a free CPU container with a public URL. The catch is Ollama cannot run on Spaces, so we swap the LLM call to the Hugging Face Inference API instead.

### Step 1 — Create a Space

1. Go to [https://huggingface.co/new-space](https://huggingface.co/new-space)
2. Choose **SDK: Docker**
3. Set visibility to **Public** (required for free tier)

### Step 2 — Modify app.py to use HF Inference API

Replace the `subprocess.run(["ollama", ...])` call with:

```python
import requests

HF_TOKEN = os.environ.get("HF_TOKEN", "")

def call_llm(prompt):
    response = requests.post(
        "https://api-inference.huggingface.co/models/meta-llama/Llama-3.2-3B-Instruct",
        headers={"Authorization": f"Bearer {HF_TOKEN}"},
        json={"inputs": prompt, "parameters": {"max_new_tokens": 300}}
    )
    result = response.json()
    if isinstance(result, list):
        return result[0].get("generated_text", "")
    return str(result)
```

### Step 3 — Add a Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY . .

RUN pip install flask flask-cors pymupdf sentence-transformers faiss-cpu requests

EXPOSE 7860
CMD ["python", "app.py"]
```

Update `app.py` to use port `7860` (Spaces default):

```python
app.run(host="0.0.0.0", port=7860)
```

### Step 4 — Add your HF token as a secret

In your Space → **Settings** → **Repository secrets**, add:

```
Name:  HF_TOKEN
Value: hf_xxxxxxxxxxxxxxxxxxxx
```

### Step 5 — Push and connect

```bash
git clone https://huggingface.co/spaces/yourname/doclens
cp app.py Dockerfile index.html doclens/
cd doclens && git add . && git commit -m "deploy" && git push
```

Your Space URL will be `https://yourname-doclens.hf.space`. Paste it into the frontend API field.

---

## API Reference

All endpoints accept and return JSON unless noted.

### `GET /health`

Returns server status.

```json
{ "status": "ok", "indexed": true, "pdf": "report.pdf" }
```

### `POST /upload`

Upload and index a PDF. Send as `multipart/form-data` with field name `pdf`.

```json
{ "message": "Indexed successfully", "chunks": 42 }
```

### `POST /query`

Ask a question about the indexed PDF.

**Request:**
```json
{ "question": "What is the refund policy?" }
```

**Response:**
```json
{
  "verdict": "FOUND",
  "answer": "VERDICT: FOUND\nEXPLANATION: ...\nEVIDENCE: ...",
  "pages": [3],
  "top_scores": [0.812]
}
```

### `GET /download`

Returns the highlighted PDF as `application/pdf`.

---

## Troubleshooting

**Status stays "unreachable" after connecting**

- Check that `python app.py` is still running in its terminal
- If using ngrok, make sure the tunnel is active and you copied the correct `https://` URL
- The frontend already sends `ngrok-skip-browser-warning: true` — if you still see an ngrok warning page, hard-refresh the browser

**Verdict always shows UNKNOWN**

- Run a test query and check the Flask terminal for the `📋 Raw LLM output` line
- If it is empty, Ollama may have timed out — try `ollama run llama3.2:3b "hello"` in a terminal to verify it is responsive
- If the output looks correct but parsing fails, check that the LLM is outputting `VERDICT:` (with a colon) on its own line

**Upload fails with 413 error**

Your server or proxy is rejecting large files. Add to `app.py`:

```python
app.config['MAX_CONTENT_LENGTH'] = 50 * 1024 * 1024  # 50 MB
```

For nginx, add `client_max_body_size 50M;` inside the `server {}` block.

**Ollama is slow on first query**

The model loads into memory on first use (~5–10 seconds). Subsequent queries are much faster. On Colab, make sure you are using a GPU runtime (**Runtime → Change runtime type → T4 GPU**).

**`faiss-cpu` install fails on Apple Silicon**

```bash
pip install faiss-cpu --no-binary faiss-cpu
```

---

## License

MIT — do whatever you want, just don't remove attribution.