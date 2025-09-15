# vLLM Integration Guide for Open Lovable

## Overview

This document provides comprehensive instructions for ML ops teams to integrate vLLM with Open Lovable, enabling use of local/self-hosted large language models instead of commercial AI services.

## Prerequisites

### Infrastructure Requirements
- **GPU Server**: NVIDIA A100/H100 or equivalent with sufficient VRAM
- **Network**: Accessible endpoint for Open Lovable to reach vLLM server
- **vLLM**: Latest stable version installed
- **Model**: Compatible LLM (Llama, Mistral, Mixtral, etc.)

### Open Lovable Requirements
- Open Lovable repository access
- Vercel project access (for sandbox deployment)
- Firecrawl API key (for web scraping)

---

## vLLM Server Setup

### 1. **Server Installation**

```bash
# Install vLLM with GPU support
pip install vllm

# Or with specific CUDA version
pip install vllm[cuda12]
```

### 2. **Model Configuration**

**Recommended Models for Code Generation:**
- `meta-llama/Llama-2-70b-chat-hf`
- `codellama/CodeLlama-70b-hf`
- `mistralai/Mixtral-8x7B-Instruct-v0.1`
- `meta-llama/Meta-Llama-3-70B-Instruct`

### 3. **vLLM Server Launch Command**

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-70b-chat-hf \
  --host 0.0.0.0 \
  --port 8000 \
  --max-model-len 8192 \
  --max-num-batched-tokens 8192 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --tensor-parallel-size 2 \
  --pipeline-parallel-size 1 \
  --dtype float16 \
  --quantization none \
  --disable-log-requests \
  --max-log-len 100
```

**Key Parameters Explained:**
- `--max-model-len 8192`: Required for code generation context
- `--enable-auto-tool-choice`: Enables function calling support
- `--tool-call-parser hermes`: Compatible with OpenAI function calling
- `--tensor-parallel-size 2`: Number of GPUs to use (adjust based on hardware)

### 4. **Optional: Authentication Setup**

If you need API key authentication:

```bash
# Install additional dependencies
pip install fastapi uvicorn

# Add to vLLM startup
--api-key your-secret-api-key
```

---

## Open Lovable Configuration

### 1. **Environment Variables**

Create/Update `.env.local` in Open Lovable root:

```env
# vLLM Configuration
VLLM_BASE_URL=http://your-vllm-server:8000/v1
VLLM_MODEL_NAME=meta-llama/Llama-2-70b-chat-hf
VLLM_API_KEY=your-secret-api-key  # Optional if authentication enabled

# Existing Open Lovable Configuration
FIRECRAWL_API_KEY=your_firecrawl_api_key

# Vercel Sandbox Configuration
VERCEL_TEAM_ID=nova
VERCEL_PROJECT_ID=prj_xxxxxxxxx
VERCEL_OIDC_TOKEN=auto_generated_by_vercel_env_pull
```

### 2. **Code Modifications**

#### **File: `config/app.config.ts`**

Add vLLM model to available models (around line 57):

```typescript
availableModels: [
  'openai/gpt-5',
  'moonshotai/kimi-k2-instruct-0905',
  'anthropic/claude-sonnet-4-20250514',
  'google/gemini-2.0-flash-exp',
  'vllm/llama-2-70b',  // Add this line
],
```

Add vLLM display name (around line 65):

```typescript
modelDisplayNames: {
  'openai/gpt-5': 'GPT-5',
  'moonshotai/kimi-k2-instruct-0905': 'Kimi K2 (Groq)',
  'anthropic/claude-sonnet-4-20250514': 'Sonnet 4',
  'google/gemini-2.0-flash-exp': 'Gemini 2.0 Flash (Experimental)',
  'vllm/llama-2-70b': 'Llama 2 70B (Local vLLM)',  // Add this line
} as Record<string, string>,
```

#### **File: `app/api/generate-ai-code-stream/route.ts`**

Add vLLM client configuration (after line 45):

```typescript
// Add after line 45
const vllm = createOpenAI({
  apiKey: process.env.VLLM_API_KEY || 'dummy-key',
  baseURL: process.env.VLLM_BASE_URL,
});
```

Update model provider logic (around line 1191):

```typescript
// Find the modelProvider assignment and add vLLM detection
const isVLLM = model.startsWith('vllm/');

const modelProvider = isAnthropic ? anthropic :
                      (isOpenAI ? openai :
                      (isGoogle ? googleGenerativeAI :
                      (isKimiGroq ? groq :
                      (isVLLM ? vllm : groq))));
```

Update model name handling (around line 1197):

```typescript
// Add vLLM case to model name handling
let actualModel: string;
if (isVLLM) {
  actualModel = model.replace('vllm/', '') || process.env.VLLM_MODEL_NAME || 'default-model';
} else if (isAnthropic) {
  actualModel = model.replace('anthropic/', '');
} else if (isOpenAI) {
  actualModel = model.replace('openai/', '');
} else if (isKimiGroq) {
  actualModel = 'moonshotai/kimi-k2-instruct-0905';
} else if (isGoogle) {
  actualModel = model.replace('google/', '');
} else {
  actualModel = model;
}
```

#### **File: `app/api/analyze-edit-intent/route.ts`**

Apply similar changes to the edit intent analysis route if you want vLLM to handle edit analysis.

---

## Network Configuration

### 1. **Firewall Rules**

Ensure Open Lovable can reach your vLLM server:

```bash
# Allow inbound traffic to vLLM server
sudo ufw allow 8000/tcp

# Or specific IP whitelist if Open Lovable runs on known IPs
sudo ufw allow from OPEN_LOVABLE_IP to any port 8000
```

### 2. **SSL/TLS (Recommended for Production)**

```bash
# Install nginx and Let's Encrypt
sudo apt install nginx certbot python3-certbot-nginx

# Configure reverse proxy
cat > /etc/nginx/sites-available/vllm << 'EOF'
server {
    listen 443 ssl;
    server_name your-vllm-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-vllm-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-vllm-domain.com/privkey.pem;

    location /v1/ {
        proxy_pass http://localhost:8000/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for streaming
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF

# Enable and restart nginx
sudo ln -s /etc/nginx/sites-available/vllm /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

Update environment variable to use HTTPS:
```env
VLLM_BASE_URL=https://your-vllm-domain.com/v1
```

---

## Testing and Validation

### 1. **vLLM Server Health Check**

```bash
# Check if vLLM is running
curl http://your-vllm-server:8000/health

# Test OpenAI compatibility
curl -X POST http://your-vllm-server:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{
    "model": "meta-llama/Llama-2-70b-chat-hf",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

### 2. **Open Lovable Integration Test**

1. Start Open Lovable development server:
```bash
npm run dev
```

2. Access the application at `http://localhost:3000`

3. Select "Llama 2 70B (Local vLLM)" from the model dropdown

4. Test with a simple prompt like "Create a simple React component"

### 3. **Monitoring vLLM Performance**

```bash
# Check vLLM metrics
curl http://your-vllm-server:8000/metrics

# Monitor GPU usage
nvidia-smi

# Check vLLM logs
journalctl -u vllm -f
```

---

## Troubleshooting

### Common Issues

**1. Connection Refused**
```bash
# Check vLLM is running
ps aux | grep vllm

# Check port availability
netstat -tulpn | grep 8000

# Verify firewall rules
sudo ufw status
```

**2. Model Loading Issues**
```bash
# Check model exists
ls -la /path/to/your/model

# Verify GPU memory
nvidia-smi

# Check vLLM logs for errors
journalctl -u vllm --since "10 minutes ago"
```

**3. Authentication Errors**
```bash
# Verify API key
curl -X POST http://your-vllm-server:8000/v1/chat/completions \
  -H "Authorization: Bearer your-api-key" \
  -d '{"model": "test", "messages": [{"role": "user", "content": "test"}]}'
```

**4. Performance Issues**
```bash
# Monitor GPU usage during requests
watch -n 1 nvidia-smi

# Check vLLM queue status
curl http://your-vllm-server:8000/metrics | grep waiting
```

---

## Production Deployment

### 1. **Systemd Service**

Create `/etc/systemd/system/vllm.service`:

```ini
[Unit]
Description=vLLM API Server
After=network.target

[Service]
Type=simple
User=vllm
WorkingDirectory=/opt/vllm
Environment=PATH=/opt/conda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/opt/conda/bin/python -m vllm.entrypoints.openai.api_server \
  --model /models/meta-llama/Llama-2-70b-chat-hf \
  --host 0.0.0.0 \
  --port 8000 \
  --max-model-len 8192 \
  --enable-auto-tool-choice \
  --tensor-parallel-size 2 \
  --api-key your-production-api-key

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable vllm
sudo systemctl start vllm
```

### 2. **Load Balancing (Multiple vLLM Instances)**

```nginx
upstream vllm_backend {
    server vllm-1:8000;
    server vllm-2:8000;
    server vllm-3:8000;
}

server {
    listen 443 ssl;
    server_name vllm-cluster.yourdomain.com;

    location /v1/ {
        proxy_pass http://vllm_backend/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # Load balancing
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Health check
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    }
}
```

---

## Performance Optimization

### 1. **vLLM Configuration**

```bash
# For better throughput
--max-num-batched-tokens 8192 \
--max-num-seqs 256 \
--enable-chunked-prefill \
--use-v2-block-manager

# For lower latency
--swap-space 4 \
--gpu-memory-utilization 0.90 \
--disable-log-stats
```

### 2. **Model Quantization**

```bash
# AWQ quantization (recommended for 70B models)
--quantization awq \
--load-format awq

# Or GPTQ quantization
--quantization gptq \
--load-format gptq
```

### 3. **Monitoring Setup**

```bash
# Install Prometheus and Grafana for monitoring
docker run -d --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus

# Configure Prometheus to scrape vLLM metrics
```

---

## Security Considerations

### 1. **Network Security**
- Use VPN or private network for vLLM server
- Implement IP whitelisting
- Use SSL/TLS encryption
- Enable rate limiting

### 2. **API Security**
- Use strong API keys
- Implement key rotation
- Monitor access logs
- Set up intrusion detection

### 3. **Data Privacy**
- vLLM processes data locally (no external API calls)
- Configure audit logging
- Implement data retention policies

---

## Support

### **Internal Resources**
- Open Lovable Repository: `/data/projects/open-lovable`
- Configuration Files: `/config/app.config.ts`
- API Routes: `/app/api/generate-ai-code-stream/route.ts`

### **External Resources**
- vLLM Documentation: https://vllm.readthedocs.io/
- OpenAI API Reference: https://platform.openai.com/docs/api-reference
- Open Lovable README: `/README.md`

### **Contact**
- ML Ops Team: For vLLM deployment issues
- Development Team: For Open Lovable integration issues
- Infrastructure Team: For network and deployment support

---

## Checklist

- [ ] vLLM server installed and configured
- [ ] Model downloaded and accessible
- [ ] Firewall rules configured
- [ ] SSL/TLS certificates set up (production)
- [ ] Environment variables configured in Open Lovable
- [ ] Code modifications applied
- [ ] Basic connectivity tested
- [ ] Open Lovable integration tested
- [ ] Performance monitoring configured
- [ ] Security measures implemented
- [ ] Documentation updated
- [ ] Team training completed

---

*Document maintained by ML Ops Team. Last updated: $(date)*