# 🚀 Piki Stan Panel - Linux Server Deployment Guide

**Piki Stan Panel** اصلاح‌شده برای استقرار بر روی سرورهای Linux است.

---

## 📋 پیش‌نیازها

```bash
# بروز رسانی سیستم
sudo apt update && sudo apt upgrade -y

# نصب Python 3.10+
sudo apt install python3.10 python3.10-venv python3-pip -y

# نصب dependencies سیستم
sudo apt install git curl wget -y
```

---

## 🔧 مراحل نصب

### مرحله ۱: Clone پروژه

```bash
# برو به پوشه مناسب
cd /opt
# یا
cd /home/yourusername

# Clone کن
git clone https://github.com/YOUR-USERNAME/piki-stan-panel.git
cd piki-stan-panel
```

### مرحله ۲: Python Virtual Environment

```bash
# Virtual environment بساز
python3 -m venv venv

# فعال کن
source venv/bin/activate

# Pip رو بروز کن
pip install --upgrade pip
```

### مرحله ۳: نصب وابستگی‌ها

```bash
pip install -r requirements.txt
```

### مرحله ۴: متغیرهای محیطی

```bash
# فایل .env بساز (اختیاری)
cat > .env << EOF
# Server Configuration
LISTEN_HOST=0.0.0.0
LISTEN_PORT=8000
PORT=8000

# Data Directory
STANNG_DATA_DIR=/opt/piki-stan-panel/data

# GitHub OTA (برای آپدیت خودکار)
# اختیاری - تغییر بده به GitHub repo خودت
EOF
```

---

## ▶️ راه‌اندازی اولیه

### روش ۱: اجرای مستقیم (برای تست)

```bash
# Virtual environment فعال باشد
source venv/bin/activate

# اجرا کن
python main.py
```

**نتیجه:**
```
🚀 Starting Piki Stan Panel v1.0.0
📍 Listening on http://0.0.0.0:8000
📖 Setup at http://0.0.0.0:8000/setup
```

**مرورگر را باز کن:**
```
http://your-server-ip:8000/setup
```

---

### روش ۲: اجرا در پس‌زمینه (Systemd)

#### ۱. Systemd Service فایل بساز

```bash
sudo nano /etc/systemd/system/piki-stan.service
```

**محتوا:**
```ini
[Unit]
Description=Piki Stan Panel - VLESS Management
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/piki-stan-panel
Environment="PATH=/opt/piki-stan-panel/venv/bin"
Environment="LISTEN_HOST=0.0.0.0"
Environment="LISTEN_PORT=8000"
ExecStart=/opt/piki-stan-panel/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### ۲. Service فایل را فعال کن

```bash
# Daemon reload
sudo systemctl daemon-reload

# Service شروع کن
sudo systemctl start piki-stan

# بر روی بوت فعال کن
sudo systemctl enable piki-stan

# وضعیت را چک کن
sudo systemctl status piki-stan

# Logs را ببین
sudo journalctl -u piki-stan -f
```

---

### روش ۳: استفاده از Nginx Reverse Proxy

#### ۱. Nginx نصب کن

```bash
sudo apt install nginx -y
```

#### ۲. Nginx config بساز

```bash
sudo nano /etc/nginx/sites-available/piki-stan
```

**محتوا:**
```nginx
upstream piki_stan {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name your-domain.com;

    # HTTPS Redirect (توصیه شده)
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # SSL Certificates (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Proxy Settings
    location / {
        proxy_pass http://piki_stan;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }

    # WebSocket Support
    location /ws/ {
        proxy_pass http://piki_stan;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### ۳. Nginx فعال کن

```bash
# تست کن
sudo nginx -t

# فعال کن
sudo systemctl enable nginx
sudo systemctl start nginx

# Reload کن
sudo systemctl reload nginx
```

#### ۴. SSL Certificate (Let's Encrypt)

```bash
# Certbot نصب کن
sudo apt install certbot python3-certbot-nginx -y

# Certificate بگیر
sudo certbot certonly --nginx -d your-domain.com
```

---

## 🔐 امنیت

### Firewall Configuration

```bash
# UFW فعال کن (اگر نصب است)
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw enable
```

### Backup خودکار

```bash
# Backup script بساز
cat > /opt/piki-stan-panel/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/backups/piki-stan"
mkdir -p $BACKUP_DIR
cp -r /opt/piki-stan-panel/data $BACKUP_DIR/data-$(date +%Y%m%d-%H%M%S)
echo "Backup completed!"
EOF

# اجازه دهنده‌ی اجرا کن
chmod +x /opt/piki-stan-panel/backup.sh

# Cron job بساز (روزانه در ساعت ۲ صبح)
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/piki-stan-panel/backup.sh") | crontab -
```

---

## 📊 نگاهی به Logs

### Systemd Logs

```bash
# Real-time logs
sudo journalctl -u piki-stan -f

# آخرین ۱۰۰ خط
sudo journalctl -u piki-stan -n 100

# آخرین یک ساعت
sudo journalctl -u piki-stan --since="1 hour ago"
```

### Application Logs

```bash
# Custom log directory
mkdir -p /var/log/piki-stan

# Redirect logs to file (در service file)
# StandardOutput=append:/var/log/piki-stan/output.log
# StandardError=append:/var/log/piki-stan/error.log
```

---

## 🔄 آپدیت و نگهداری

### Pull Latest Changes

```bash
cd /opt/piki-stan-panel
git fetch origin
git pull origin main
pip install -r requirements.txt
sudo systemctl restart piki-stan
```

### Data Directory Backup قبل از آپدیت

```bash
cp -r data data.backup
```

---

## 🌐 اتصال از دور (Remote Access)

### SSH Tunnel

```bash
# Local machine
ssh -L 8000:127.0.0.1:8000 user@your-server-ip
# سپس: http://localhost:8000
```

### Public Domain

```bash
# Update DNS records
# A record: your-domain.com → your-server-ip

# از Nginx reverse proxy استفاده کن (توصیه شده)
# HTTPS + Custom domain
```

---

## 🆘 Troubleshooting

### خطا: Port 8000 Already in Use

```bash
# پروسس را پیدا کن
lsof -i :8000

# Kill کن
kill -9 <PID>

# یا پورت دیگری استفاده کن
LISTEN_PORT=8001 python main.py
```

### خطا: Permission Denied

```bash
# مالک فایل‌ها را تغییر بده
sudo chown -R www-data:www-data /opt/piki-stan-panel

# اجازات
sudo chmod -R 755 /opt/piki-stan-panel
sudo chmod -R 777 /opt/piki-stan-panel/data
```

### خطا: Database Locked

```bash
# data directory حذف کن (اگر نیاز است)
rm -f /opt/piki-stan-panel/data/db.json

# یا restart کن
sudo systemctl restart piki-stan
```

---

## 📈 Performance Tuning

### uvicorn Workers

```bash
# اگر بیشتر CPU cores دارید
python -m uvicorn main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4  # تعداد workers
```

### Gunicorn (Production)

```bash
pip install gunicorn

# اجرا
gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000
```

---

## ✅ Checklist

- [ ] Python 3.10+ نصب شده
- [ ] Virtual environment فعال است
- [ ] وابستگی‌ها نصب شده‌اند
- [ ] اولین بار تنظیمات کامل شده‌اند
- [ ] SSH keys تنظیم شده‌اند (اختیاری)
- [ ] Firewall configured
- [ ] Backup script فعال است
- [ ] Systemd service فعال است
- [ ] Nginx reverse proxy کار می‌کند
- [ ] SSL certificate صحیح است

---

**نیاز به کمک؟** به Telegram پیام بده: [@piki_vpnbot](https://t.me/piki_vpnbot)

**مشکل یافتی؟** [GitHub Issues](https://github.com/YOUR-USERNAME/piki-stan-panel/issues) باز کن
