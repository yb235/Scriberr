# Deployment Guide

## Deployment Options

Scriberr can be deployed in multiple ways depending on your infrastructure and requirements.

## Option 1: Homebrew (macOS/Linux)

The easiest way to install on macOS or Linux.

### Installation

```bash
# Add Scriberr tap
brew tap rishikanthc/scriberr

# Install
brew install scriberr

# Start server
scriberr
```

**Access**: http://localhost:8080

### Configuration

Create `.env` file in the same directory:

```env
PORT=8080
DATABASE_PATH=./data/scriberr.db
UPLOAD_DIR=./data/uploads
```

### Running as Service

**macOS (launchd)**:
```xml
<!-- ~/Library/LaunchAgents/com.scriberr.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.scriberr</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/scriberr</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/scriberr.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/scriberr.error.log</string>
</dict>
</plist>
```

Load service:
```bash
launchctl load ~/Library/LaunchAgents/com.scriberr.plist
```

**Linux (systemd)**:
```ini
# /etc/systemd/system/scriberr.service
[Unit]
Description=Scriberr Transcription Service
After=network.target

[Service]
Type=simple
User=scriberr
WorkingDirectory=/opt/scriberr
ExecStart=/usr/local/bin/scriberr
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable scriberr
sudo systemctl start scriberr
sudo systemctl status scriberr
```

## Option 2: Docker

Run Scriberr in a container with all dependencies included.

### Quick Start

```bash
docker run -d \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  --restart unless-stopped \
  ghcr.io/rishikanthc/scriberr:latest
```

**Access**: http://localhost:8080

### Docker Compose

**Basic Setup**:

```yaml
# docker-compose.yml
version: '3.9'

services:
  scriberr:
    image: ghcr.io/rishikanthc/scriberr:latest
    container_name: scriberr
    ports:
      - "8080:8080"
    volumes:
      - scriberr_data:/app/data
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped

volumes:
  scriberr_data:
```

Run:
```bash
docker-compose up -d
```

### Custom Port

```bash
docker run -d \
  --name scriberr \
  -p 3000:8080 \
  -v scriberr_data:/app/data \
  ghcr.io/rishikanthc/scriberr:latest
```

Access on http://localhost:3000

### GPU Support (NVIDIA)

**Requirements**:
- NVIDIA GPU with CUDA support
- NVIDIA Container Toolkit installed

**Docker Run**:
```bash
docker run -d \
  --name scriberr \
  --gpus all \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  ghcr.io/rishikanthc/scriberr:cuda
```

**Docker Compose**:
```yaml
version: '3.9'

services:
  scriberr:
    image: ghcr.io/rishikanthc/scriberr:cuda
    container_name: scriberr
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    ports:
      - "8080:8080"
    volumes:
      - scriberr_data:/app/data
    restart: unless-stopped

volumes:
  scriberr_data:
```

### Environment Variables

```bash
docker run -d \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  -e PUID=1000 \
  -e PGID=1000 \
  -e PORT=8080 \
  -e LOG_LEVEL=info \
  ghcr.io/rishikanthc/scriberr:latest
```

**Available Variables**:
- `PUID`: User ID for file permissions (default: 1000)
- `PGID`: Group ID for file permissions (default: 1000)
- `PORT`: HTTP port (default: 8080)
- `LOG_LEVEL`: Logging level (debug, info, warn, error)
- `QUEUE_WORKERS`: Number of worker threads

### Data Persistence

**Named Volume** (recommended):
```bash
docker volume create scriberr_data
docker run -d \
  -v scriberr_data:/app/data \
  ghcr.io/rishikanthc/scriberr:latest
```

**Bind Mount**:
```bash
docker run -d \
  -v /path/on/host:/app/data \
  ghcr.io/rishikanthc/scriberr:latest
```

**Volume Contents**:
```
/app/data/
├── scriberr.db          # Database
├── uploads/             # Audio files
├── transcripts/         # JSON outputs
├── whisperx-env/        # Python environment
└── nvidia-env/          # NVIDIA environment
```

## Option 3: Binary Download

Download pre-compiled binaries from GitHub releases.

### Download

Visit: https://github.com/rishikanthc/Scriberr/releases

Download for your platform:
- `scriberr-linux-amd64`
- `scriberr-linux-arm64`
- `scriberr-darwin-amd64` (macOS Intel)
- `scriberr-darwin-arm64` (macOS Apple Silicon)
- `scriberr-windows-amd64.exe`

### Install

**Linux/macOS**:
```bash
# Download (replace VERSION)
wget https://github.com/rishikanthc/Scriberr/releases/download/vVERSION/scriberr-linux-amd64

# Make executable
chmod +x scriberr-linux-amd64

# Move to PATH (optional)
sudo mv scriberr-linux-amd64 /usr/local/bin/scriberr

# Run
scriberr
```

**Windows**:
1. Download `scriberr-windows-amd64.exe`
2. Rename to `scriberr.exe`
3. Run from Command Prompt or PowerShell

### Prerequisites

**All Platforms**:
- Python 3.11+
- Internet connection (first run - downloads models)

**Install Python**:

macOS:
```bash
brew install python@3.11
```

Linux:
```bash
sudo apt-get install python3.11
```

Windows:
Download from python.org

## Option 4: Build from Source

Build Scriberr yourself for maximum control.

### Prerequisites

- Go 1.24+
- Node.js 20+
- Python 3.11+
- Git

### Build Steps

```bash
# Clone repository
git clone https://github.com/rishikanthc/Scriberr.git
cd Scriberr

# Run build script
chmod +x build.sh
./build.sh

# Binary created: ./scriberr
./scriberr
```

**Manual Build**:

```bash
# Build frontend
cd web/frontend
npm install
npm run build
cd ../..

# Copy frontend to embed location
rm -rf internal/web/dist
cp -r web/frontend/dist internal/web/dist

# Build Go binary
go build -o scriberr cmd/server/main.go
```

### Custom Build Options

**Static Binary** (no external dependencies):
```bash
CGO_ENABLED=0 go build -o scriberr -ldflags="-s -w" cmd/server/main.go
```

**With Version Info**:
```bash
VERSION=$(git describe --tags)
COMMIT=$(git rev-parse --short HEAD)
DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

go build -o scriberr \
  -ldflags="-X main.version=$VERSION -X main.commit=$COMMIT -X main.date=$DATE" \
  cmd/server/main.go
```

## Reverse Proxy Setup

### Nginx

```nginx
server {
    listen 80;
    server_name scriberr.example.com;

    # SSL configuration (recommended)
    # listen 443 ssl http2;
    # ssl_certificate /path/to/cert.pem;
    # ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Increase timeout for long-running requests
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
        
        # Increase max body size for file uploads
        client_max_body_size 500M;
    }
}
```

Reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Apache

```apache
<VirtualHost *:80>
    ServerName scriberr.example.com

    ProxyPreserveHost On
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/

    # WebSocket support
    RewriteEngine on
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) "ws://localhost:8080/$1" [P,L]

    # Increase timeout
    ProxyTimeout 300

    # Increase max body size
    LimitRequestBody 524288000
</VirtualHost>
```

Enable modules and reload:
```bash
sudo a2enmod proxy proxy_http proxy_wstunnel rewrite
sudo systemctl reload apache2
```

### Caddy

```caddyfile
scriberr.example.com {
    reverse_proxy localhost:8080
}
```

Run Caddy:
```bash
caddy run --config Caddyfile
```

## SSL/TLS Setup

### Let's Encrypt with Certbot

**Nginx**:
```bash
sudo certbot --nginx -d scriberr.example.com
```

**Apache**:
```bash
sudo certbot --apache -d scriberr.example.com
```

**Standalone** (if not using reverse proxy):
```bash
sudo certbot certonly --standalone -d scriberr.example.com
```

### Manual Certificate

Place certificates and update Nginx config:

```nginx
server {
    listen 443 ssl http2;
    server_name scriberr.example.com;

    ssl_certificate /etc/ssl/certs/scriberr.crt;
    ssl_certificate_key /etc/ssl/private/scriberr.key;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # ... rest of config
}
```

## Cloud Deployment

### AWS EC2

1. **Launch Instance**:
   - Ubuntu 22.04 LTS
   - t3.medium (2 vCPU, 4GB RAM minimum)
   - 20GB storage
   - Security group: Allow port 80, 443, 22

2. **Install Docker**:
```bash
sudo apt-get update
sudo apt-get install docker.io docker-compose
sudo systemctl enable docker
```

3. **Deploy Scriberr**:
```bash
sudo docker run -d \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  --restart unless-stopped \
  ghcr.io/rishikanthc/scriberr:latest
```

4. **Setup Nginx** (see reverse proxy section)

5. **Configure DNS**: Point domain to EC2 public IP

### Google Cloud Run

Not recommended due to:
- Stateful nature (database, uploads)
- Long-running transcription jobs
- Large file uploads

Better alternatives: Compute Engine or GKE

### DigitalOcean Droplet

Similar to AWS EC2:

1. Create Droplet (Ubuntu 22.04, 4GB RAM)
2. Install Docker
3. Deploy with Docker Compose
4. Setup Nginx reverse proxy
5. Configure DNS

### Kubernetes

**Deployment** (`k8s/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scriberr
spec:
  replicas: 1
  selector:
    matchLabels:
      app: scriberr
  template:
    metadata:
      labels:
        app: scriberr
    spec:
      containers:
      - name: scriberr
        image: ghcr.io/rishikanthc/scriberr:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: scriberr-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: scriberr
spec:
  selector:
    app: scriberr
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: scriberr-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

Deploy:
```bash
kubectl apply -f k8s/deployment.yaml
```

## Monitoring

### Logging

**Docker logs**:
```bash
docker logs -f scriberr
```

**Binary logs**:
```bash
# Set debug level
export LOG_LEVEL=debug
scriberr >> scriberr.log 2>&1
```

### Health Checks

```bash
# Health endpoint
curl http://localhost:8080/health

# Expected response
{"status": "ok"}
```

### Metrics (Future)

Prometheus endpoint planned for future release.

## Backup & Restore

### Backup

**Full backup**:
```bash
# Docker volume
docker run --rm \
  -v scriberr_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/scriberr-backup.tar.gz /data

# Local installation
tar czf scriberr-backup.tar.gz ./data
```

**Database only**:
```bash
cp data/scriberr.db scriberr-$(date +%Y%m%d).db
```

### Restore

```bash
# Extract backup
tar xzf scriberr-backup.tar.gz

# Docker: Copy to volume
docker run --rm \
  -v scriberr_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/scriberr-backup.tar.gz -C /"
```

## Upgrading

### Docker

```bash
# Pull latest image
docker pull ghcr.io/rishikanthc/scriberr:latest

# Stop and remove old container
docker stop scriberr
docker rm scriberr

# Start new container (with same volume)
docker run -d \
  --name scriberr \
  -p 8080:8080 \
  -v scriberr_data:/app/data \
  --restart unless-stopped \
  ghcr.io/rishikanthc/scriberr:latest
```

### Homebrew

```bash
brew upgrade scriberr
```

### Binary

1. Download new version
2. Stop old version
3. Replace binary
4. Start new version

## Security Hardening

### Firewall

**UFW (Ubuntu)**:
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

### Fail2ban

Protect against brute force:

```bash
sudo apt-get install fail2ban

# Configure jail
sudo nano /etc/fail2ban/jail.local
```

### User Permissions

Run as non-root user:

```bash
# Create user
sudo useradd -r -s /bin/false scriberr

# Run as user
sudo -u scriberr /usr/local/bin/scriberr
```

## Troubleshooting

### Container won't start

```bash
# Check logs
docker logs scriberr

# Check permissions
docker exec scriberr ls -la /app/data
```

### Out of disk space

```bash
# Check usage
df -h

# Clean old Docker images
docker system prune -a

# Remove old transcriptions
```

### Database locked

```bash
# Stop all instances
docker stop scriberr

# Remove lock files
docker exec scriberr rm /app/data/scriberr.db-wal
```

### Permission errors

```bash
# Fix ownership (Docker)
docker exec scriberr chown -R appuser:appuser /app/data
```

## Performance Tuning

### Worker Threads

```bash
# Increase workers for better parallelism
docker run -d \
  -e QUEUE_WORKERS=4 \
  ghcr.io/rishikanthc/scriberr:latest
```

### Resource Limits

```yaml
# Docker Compose
services:
  scriberr:
    image: ghcr.io/rishikanthc/scriberr:latest
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
```

## Best Practices

1. **Use Docker**: Simplest deployment method
2. **Enable SSL**: Always use HTTPS in production
3. **Regular Backups**: Automate database backups
4. **Monitor Logs**: Check logs regularly
5. **Limit Access**: Use reverse proxy, firewall
6. **Update Regularly**: Stay on latest version
7. **Use Volumes**: Persist data properly
8. **Set Resource Limits**: Prevent resource exhaustion

## Support

- GitHub Issues: Bug reports
- GitHub Discussions: Questions and help
- Documentation: Full docs in `docs/` folder
