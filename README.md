# Prime Academy Django API Deployment Guide

This documentation provides step-by-step instructions for deploying the Prime Academy Django API on a production server.

## Prerequisites
- Ubuntu server (20.04+ recommended)
- Domain name pointed to server IP (prime-api.enghasan.com)
- GitHub repository with Django project

## Step 1: Install Python, pip, and virtualenv

```bash
sudo apt update
sudo apt install python3-pip python3-venv
```

## Step 2: Create and Activate Python Virtual Environment

```bash
cd /var/www/primeacademy/api
sudo python3 -m venv venv
source venv/bin/activate
```

> **Note**: This must be done at least once for the project setup.

## Step 3: Update pip and Fix Permissions

```bash
sudo chown -R prime:prime /var/www/primeacademy/api/venv
pip install --upgrade pip
```

## Step 4: Set Up PostgreSQL Database

### Install PostgreSQL
```bash
sudo apt install postgresql postgresql-contrib
```

### Create Database and User
```bash
sudo -i -u postgres
createdb primeacademydb
createuser primeacademy --pwprompt
```

### Configure Database Permissions
```bash
psql
```

Inside psql, run these commands:
```sql
GRANT ALL PRIVILEGES ON DATABASE primeacademydb TO primeacademy;
GRANT CREATE ON DATABASE primeacademydb TO primeacademy;
ALTER USER primeacademy CREATEDB;
\q
```

Exit postgres user:
```bash
exit
```

### Password Management (if required)
```bash
sudo -u postgres psql -c "ALTER USER primeacademy PASSWORD 'X7vpQ2reL9zT4sbW1uK6m';"
```

## Step 5: GitHub Actions Deployment Setup

### Generate SSH Key Pair
```bash
ssh-keygen -t ed25519 -C "prime-deploy-key"
```
- When prompted, save as: `~/.ssh/id_ed25519`
- Leave passphrase empty for automation

### Add Public Key to Server
```bash
cat ~/.ssh/id_ed25519.pub
echo "[public_key_contents]" >> ~/.ssh/authorized_keys
```

### Add Private Key to GitHub Secrets
1. Copy the private key:
```bash
cat ~/.ssh/id_ed25519
```

2. Add to GitHub:
   - Go to your repository on GitHub
   - Click on **Settings** → **Secrets and variables** → **Actions**
   - Click **New repository secret**
   - Name: `PRIME_SSH_PRIVATE_KEY`
   - Paste the private key contents
   - Click **Add secret**

### Directory Permissions (Important)
```bash
# Fix directory ownership permanently
sudo chown -R prime:prime /var/www/primeacademy/api/
sudo chmod -R 755 /var/www/primeacademy/api/

# Ensure prime user has write access to parent directory
sudo chown prime:prime /var/www/primeacademy/
```

## Step 6: Create Gunicorn Service

Create service file:
```bash
sudo nano /etc/systemd/system/primeacademy.service
```

Add the following content:
```ini
[Unit]
Description=Gunicorn service for Prime Academy Django project
After=network.target

[Service]
User=prime
Group=prime
WorkingDirectory=/var/www/primeacademy/api
Environment="PATH=/var/www/primeacademy/api/venv/bin"
ExecStart=/var/www/primeacademy/api/venv/bin/gunicorn core.wsgi:application --bind 127.0.0.1:8000

[Install]
WantedBy=multi-user.target
```

### Enable and Start Service
```bash
# Reload systemd to recognize new service
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable primeacademy

# Start the service
sudo systemctl start primeacademy

# Check status
sudo systemctl status primeacademy
```

## Step 7: Configure Nginx

Create nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/prime-api.enghasan.com
```

Add the following configuration:
```nginx
server {
    listen 80;
    server_name prime-api.enghasan.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /media/ {
        alias /var/www/primeacademy/api/media/;
    }

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://127.0.0.1:8000;
    }
}
```

### Enable Site and Test Configuration
```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/prime-api.enghasan.com /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx

# Check status
sudo systemctl status nginx
```

### Install SSL Certificate with Certbot
```bash
sudo certbot --nginx -d prime-api.enghasan.com
```

## Step 8: Final Deployment

Push your code to GitHub. The GitHub Actions workflow will automatically:
- SSH into your server
- Pull the latest code
- Install dependencies
- Run migrations
- Restart services

## Verification Steps

1. Check Gunicorn service status:
```bash
sudo systemctl status primeacademy
```

2. Check nginx status:
```bash
sudo systemctl status nginx
```

3. Test your API endpoint:
```bash
curl https://prime-api.enghasan.com/api/health-check/
```

## Troubleshooting

### Common Issues:

1. **Permission denied errors**: Ensure directory permissions are correctly set as in Step 5
2. **Database connection issues**: Verify PostgreSQL user permissions and password
3. **Service not starting**: Check logs with `sudo journalctl -u primeacademy -f`
4. **Nginx configuration errors**: Test with `sudo nginx -t`

### Logs Location:
- Gunicorn logs: `sudo journalctl -u primeacademy`
- Nginx logs: `/var/log/nginx/error.log`

This documentation maintains all original steps while providing clearer organization and additional troubleshooting guidance.
