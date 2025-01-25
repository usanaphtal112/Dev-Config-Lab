# Complete Django Server Setup Guide

## Prerequisites
- Ubuntu 20.04 LTS or later
- Root or sudo access to the server
- Domain name (recommended for production)
- SSH access to your server
- Basic knowledge of Linux commands

## 1. Initial Server Setup

### 1.1. Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2. Create Deploy User
```bash
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG sudo deploy
sudo passwd deploy
```

## 2. Install Required Software

### 2.1. Install System Packages
```bash
sudo apt install python3-pip python3-dev python3-venv libpq-dev postgresql postgresql-contrib nginx curl git supervisor redis-server build-essential
```

### 2.2. Create Project Directory
```bash
sudo mkdir -p /var/www/myproject
sudo chown deploy:www-data /var/www/myproject
```

## 3. Project Setup

### 3.1. Change to Project directory
```bash
cd /var/www/myproject
```
or (if using git)
```bash
git clone your-repository-url .
```

### 3.2. Create Virtual Environment
```bash
cd /var/www/myproject
python3 -m venv .venv
source .venv/bin/activate
```

### 3.3. Install Python Packages
```bash
pip install django gunicorn psycopg2-binary celery[redis] django-celery-beat python-dotenv
```
***If You have project created and running loccally you can use ```requirement.txt``` file***
```bash
pip install -r requirements.txt
```


## 4. PostgreSQL Setup

### 4.1. Create Database and User
```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE myproject;
CREATE USER myuser WITH PASSWORD 'strong_password';
ALTER ROLE myuser SET client_encoding TO 'utf8';
ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myuser SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myuser;
\q
```

### 4.2. 4.2. Apply Migrations
```bash
python manage.py migrate
python manage.py collectstatic
```

### 4.3. Create Environment variables File
```bash
nano /var/www/myproject/.env
```

```env
DEBUG=False
SECRET_KEY=your_secret_key_here
DB_NAME=myproject
DB_USER=myuser
DB_PASSWORD=strong_password
DB_HOST=localhost
ALLOWED_HOSTS=your_domain.com,www.your_domain.com
```

## 5. Configure Gunicorn

### 5.1. Create Gunicorn Socket File
```bash
sudo nano /etc/systemd/system/gunicorn.socket
```

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### 5.2. Create Gunicorn Service
```bash
sudo nano /etc/systemd/system/gunicorn.service
```

```ini
[Unit]
Description=Gunicorn daemon for Django Project
Requires=gunicorn.socket
After=network.target

[Service]
User=deploy
Group=www-data
WorkingDirectory=/var/www/myproject
ExecStart=/var/www/myproject/.venv/bin/gunicorn \
    --access-logfile - \
    --workers 3 \
    --bind unix:/run/gunicorn.sock \
    django_project.wsgi:application
    
[Install]
WantedBy=multi-user.target
```

## 6. Configure Nginx

### 6.1. Create Nginx Configuration
```bash
sudo nano /etc/nginx/sites-available/myproject
```

```nginx
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    client_max_body_size 100M;

    # Static files configuration
    location /static/ {
        alias /var/www/myproject/staticfiles_build/;  # Points to STATIC_ROOT
        try_files $uri $uri/ =404;
        expires 30d;
        access_log off;
        add_header Cache-Control "public, no-transform";
    }

    # Media files configuration
    location /media/ {
        alias /var/www/myproject/media/;  # Points to directory containing media folder
        try_files $uri $uri/ =404;
        expires 30d;
        access_log off;
        add_header Cache-Control "public, no-transform";
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }

    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

### 6.2. Enable Nginx Configuration
```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # Remove default configuration
sudo nginx -t  # Test configuration
sudo systemctl restart nginx
```

Check the Nginx status
```bash
sudo systemctl status nginx
sudo ss -tuln | grep :80
```

## 7. Configure Celery and Redis

### 7.1. Configure Celery
Update ```django_project/celery.py:```
```bash
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'django_project.settings')

app = Celery('django_project')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

### 7.2. Setup Supervisor for celery
```bash
sudo nano /etc/supervisor/conf.d/celery.conf
```

```ini
[program:celery]
command=/var/www/myproject/.venv/bin/celery -A django_project worker -l INFO
directory=/var/www/myproject
user=deploy
numprocs=1
stdout_logfile=/var/log/celery/worker.log
stderr_logfile=/var/log/celery/worker.error.log
autostart=true
autorestart=true
startsecs=10

[program:celerybeat]
command=/var/www/myproject/.venv/bin/celery -A django_project beat -l INFO
directory=/var/www/myproject
user=deploy
numprocs=1
stdout_logfile=/var/log/celery/beat.log
stderr_logfile=/var/log/celery/beat.error.log
autostart=true
autorestart=true
startsecs=10
```

### 7.3. Create Log Directory
```bash
sudo mkdir -p /var/log/celery
sudo chown -R deploy:deploy /var/log/celery
```

## 8. SSL Configuration (Let's Encrypt)

**Before this steps, Make sure that your domain Name is connected to your server public IPv4 Address**

### 8.1. Install Certbot
```bash
sudo apt install certbot python3-certbot-nginx
```

### 8.2. Obtain SSL Certificate
```bash
sudo certbot --nginx -d your_domain.com -d www.your_domain.com
```

## 9. Setup Firewall

### 9.1. Configure UFW
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

## 10. CI/CD Setup

### 10.1. Create GitHub Workflows

Create ```.github/workflows/django-ci-cd.yml```:
```yml
name: Django CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  DJANGO_SETTINGS_MODULE: django_project.settings
  PROJECT_PATH: /var/www/myproject

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:latest
        env:
          POSTGRES_DB: github_actions
          POSTGRES_USER: github_actions
          POSTGRES_PASSWORD: github_actions_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        cache: 'pip'
    
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install coverage pytest pytest-django
    
    - name: Run Tests
      env:
        DEBUG: "False"
        SECRET_KEY: "test_key"
        DB_NAME: "github_actions"
        DB_USER: "github_actions"
        DB_PASSWORD: "github_actions_password"
        DB_HOST: "localhost"
        ALLOWED_HOSTS: "localhost,127.0.0.1"
      run: |
        coverage run -m pytest
        coverage report

    - name: Run Linting
      run: |
        pip install flake8 black isort
        flake8 .
        black . --check
        isort . --check-only

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to Server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd ${{ env.PROJECT_PATH }}
          
          # Backup database
          pg_dump ${{ secrets.DB_NAME }} > backup_$(date +%Y%m%d_%H%M%S).sql
          
          # Pull latest changes
          git pull origin main
          
          # Update virtual environment
          source .venv/bin/activate
          pip install -r requirements.txt
          
          # Collect static files
          python manage.py collectstatic --noinput
          
          # Run migrations
          python manage.py migrate
          
          # Clear cache
          python manage.py clearcache
          
          # Restart services
          sudo systemctl restart gunicorn
          sudo supervisorctl restart celery:*
          sudo systemctl restart nginx
          
          # Verify deployment
          curl -f http://localhost || exit 1

    - name: Notify on Success
      if: success()
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        SLACK_MESSAGE: 'Deployment successful! 🚀'

    - name: Notify on Failure
      if: failure()
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        SLACK_MESSAGE: '❌ Deployment failed!'
```

### 10.2. Create necessary GitHub Secrets
```
SERVER_HOST: Your server's IP or domain
SERVER_USER: SSH user (typically 'deploy')
SSH_PRIVATE_KEY: SSH private key for authentication
DB_NAME: Database name
SLACK_WEBHOOK: Slack webhook URL for notifications
```

### 10.3. Add additional project files for quality checks
```
# setup.cfg
[flake8]
max-line-length = 100
exclude = .git,__pycache__,.venv,migrations

# pyproject.toml
[tool.black]
line-length = 100
include = '\.pyw?$'
exclude = '''
/(
    \.git
  | \.venv
  | migrations
)/
'''

[tool.isort]
profile = "black"
multi_line_output = 3
```

## 11. Final Steps

### 11.1. Set Correct Permissions
```bash
# Set ownership for project files
sudo chown -R deploy:www-data /var/www/myproject

# Set permissions for media and static directories
sudo mkdir -p /var/www/myproject/media /var/www/myproject/staticfiles_build/static
sudo chmod -R 755 /var/www/myproject/staticfiles_build
sudo chmod -R 755 /var/www/myproject/media

# Set specific permissions for media files
sudo find /var/www/myproject/media -type d -exec chmod 755 {} \;
sudo find /var/www/myproject/media -type f -exec chmod 644 {} \;
sudo chown -R deploy:www-data /var/www/myproject/media

# Verify permissions
sudo -u www-data ls -la /var/www/myproject/media/
sudo -u www-data ls -la /var/www/myproject/staticfiles_build/static/
```

### 11.2. Django Configuration Check
Ensure your Django settings.py has the correct static and media configurations:

```python
# Static files configuration
STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / "static"]
STATIC_ROOT = os.path.join(BASE_DIR, "staticfiles_build", "static")

# Media files configuration
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, "media")
```

Update your main urls.py to include media serving in all environments:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    # Your other URL patterns...
]

# Serve media files in all environments
urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```


### 11.3. Start All Services
```bash
sudo systemctl start redis-server
sudo systemctl enable redis-server

sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket

sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start all

sudo systemctl restart nginx
```

### 11.4. Check Status
```bash
# Check status
sudo systemctl status nginx gunicorn redis-server
sudo supervisorctl status

# Restart services
sudo systemctl restart gunicorn
sudo systemctl restart nginx
sudo supervisorctl restart all
```
### 11.5. Backup Strategy

```
# Database backup
pg_dump myproject > backup_$(date +%Y%m%d_%H%M%S).sql

# Media files backup
tar -czf media_backup_$(date +%Y%m%d_%H%M%S).tar.gz /var/www/myproject/media/
```

### 11.6. Project Directory Structure
Your final project structure should look like this:
```
/var/www/myproject/
├── manage.py
├── django_project/
│   ├── __init__.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── static/
├── media/
├── tests/
│   ├── __init__.py
│   └── conftest.py
├── .venv/
└── .env
├── .github/
│   └── workflows/
│       └── django-ci-cd.yml
├── setup.cfg
└── pyproject.toml
```



## 12. Maintenance Tipsz

### 12.1. Log Locations
- Nginx: `/var/log/nginx/`
- Gunicorn: `/var/log/gunicorn/`
- Celery: `/var/log/celery/`
- Application: `/var/www/myproject/logs/`

### 12.2. Common Commands
- Restart Gunicorn: `sudo systemctl restart gunicorn`
- Restart Nginx: `sudo systemctl restart nginx`
- View Celery logs: `sudo tail -f /var/log/celery/worker.log`
- View Nginx access logs: `sudo tail -f /var/log/nginx/access.log`

Remember to:
- Regularly update your system packages
- Monitor your logs for errors
- Backup your database regularly
- Keep your Python packages updated
- Monitor system resources