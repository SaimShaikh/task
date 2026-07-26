# Nginx Reverse Proxy with HTTPS (Self-Signed SSL) for FastAPI Backend on AWS EC2

## Architecture

```
                 Internet
                     │
              HTTPS (443)
              HTTP (80)
                     │
        +------------------------+
        |     Nginx EC2 (Public) |
        |  Public IP             |
        |  13.63.157.42          |
        |  Private IP            |
        | 10.0.13.193            |
        +-----------+------------+
                    │
          Reverse Proxy (HTTP)
                    │
                    ▼
        +------------------------+
        |  Backend EC2 (Private) |
        | Private IP             |
        | 10.0.15.204            |
        | FastAPI :3000          |
        +------------------------+
```

---

# Prerequisites

- AWS EC2 Ubuntu Server
- Nginx EC2 in Public Subnet
- Backend EC2 in Private Subnet
- Backend application running on Port 3000
- Security Group configured

---

# Security Group Configuration

## Nginx EC2

| Type | Port | Source |
|------|------|---------|
| SSH | 22 | Your IP |
| HTTP | 80 | 0.0.0.0/0 |
| HTTPS | 443 | 0.0.0.0/0 |

---

## Backend EC2

| Type | Port | Source |
|------|------|---------|
| SSH | 22 | Nginx Security Group / Bastion |
| Custom TCP | 3000 | Nginx Security Group |

---

# Step 1 - Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

Enable Nginx

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Verify

```bash
sudo systemctl status nginx
```

---

# Step 2 - Connect to Setup the Backend Server and Verify Backend Connectivity

Step 1. Install Python
```bash
sudo apt install python3 python3-pip python3-venv -y
python3 --version
pip3 --version
mkdir -p ~/backend
cd ~/backend
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn
nano main.py


from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {
        "message": "Hello from Backend Server",
        "server": "Private Server Ip"
    }

@app.get("/health")
def health():
    return {
        "status": "UP"
    }


    
```

## Run The App

`uvicorn main:app --host 0.0.0.0 --port 3000`

## Test Locally
From Nginx Server

```bash
curl http://backend-server-private-ip:3000
```

Expected Output

```json
{
  "message":"Hello from Backend Server",
  "server":"private-ip"
}
```



## Create a Production Service

```bash
sudo nano /etc/systemd/system/backend.service
## Add this 
[Unit]
Description=FastAPI Backend
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/backend
Environment="PATH=/home/ubuntu/backend/venv/bin"
ExecStart=/home/ubuntu/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 3000
Restart=always

[Install]
WantedBy=multi-user.target


```
##  Enable the Service

```bash 
sudo systemctl daemon-reload
sudo systemctl enable backend
sudo systemctl start backend
```

## Check the Listening Port

`sudo ss -tulnp | grep 3000`
---

# Step 3 - In Nginx Ec2 Configure Nginx Reverse Proxy

Edit the default configuration.

```bash
sudo nano /etc/nginx/sites-available/default
```

Replace everything with

```nginx
server {
    listen 80;
    server_name _;

    location / {

        proxy_pass http://priavte server ip:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

# Step 4 - Test Configuration

```bash
sudo nginx -t
```

Expected

```
syntax is ok
test is successful
```

Reload nginx

```bash
sudo systemctl reload nginx
```

---

# Step 5 - Verify Reverse Proxy

On Nginx Server

```bash
curl http://localhost
```

Expected Output

```json
{
  "message":"Hello from Backend Server",
  "server":"10.0.15.204"
}
```

Now open in browser

```
http://Nginx Server Public IP
```

You should see

```json
{
  "message":"Hello from Backend Server",
  "server":"Priavte Server IP"
}
```

---

# Enable HTTPS (Self-Signed SSL)

Since we don't have a domain name, we'll use a Self-Signed SSL Certificate.

> **Note:** Browsers will display a security warning because the certificate is not signed by a trusted Certificate Authority.

---

# Step 6 - Install OpenSSL

```bash
sudo apt update
sudo apt install openssl -y
```

---

# Step 7 - Create SSL Directory

```bash
sudo mkdir -p /etc/nginx/ssl
```

---

# Step 8 - Generate SSL Certificate

```bash
sudo openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout /etc/nginx/ssl/nginx.key \
-out /etc/nginx/ssl/nginx.crt
```

During certificate generation

```
Country Name: IN
State: <YOUR STATTE 
Locality: <Add >
Organization: Demo
Organizational Unit: IT
Common Name: Nginx Server Public IP
Email Address: (optional)
```


---

# Step 9 - Configure HTTPS

Open

```bash
sudo nano /etc/nginx/sites-available/default
```

Replace everything with

```nginx
server {
    listen 80;
    server_name _;

    return 301 https://$host$request_uri;
}

server {

    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {

        proxy_pass http://Backend Server Priavte IP :3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

# Step 10 - Validate Configuration

```bash
sudo nginx -t
```

Expected

```
syntax is ok
test is successful
```

Reload nginx

```bash
sudo systemctl reload nginx
```

---

# Step 11 - Verify HTTPS

Browser

```
https://Nginx Server IP 
```

Browser Warning

```
Your connection is not private
```

Click

```
Advanced
↓

Proceed to (unsafe)
```
<img width="6" height="6" alt="image" src="https://github.com/user-attachments/assets/feb6f771-b4ed-4991-8e79-5ca4b1c3f23e" />


Expected Output

```json
{
  "message":"Hello from Backend Server",
  "server":"Backend Server Private IP "
}
```
<img width="3005" height="1688" alt="image" src="https://github.com/user-attachments/assets/32e63083-9c4d-478d-801d-23e24ab5119b" />

---

# Verify Services

Nginx Status

```bash
sudo systemctl status nginx
```

Listening Ports

```bash
sudo ss -tulnp | grep nginx
```

Backend Connectivity

```bash
curl http://Backend Server IP:3000
```

Local Reverse Proxy Test

```bash
curl http://localhost
```

HTTPS Test

```bash
curl -k https://localhost
```

---

# Useful Commands

Restart Nginx

```bash
sudo systemctl restart nginx
```

Reload Configuration

```bash
sudo systemctl reload nginx
```

Stop Nginx

```bash
sudo systemctl stop nginx
```

Enable on Boot

```bash
sudo systemctl enable nginx
```

View Logs

```bash
sudo journalctl -u nginx -f
```

Error Log

```bash
sudo tail -f /var/log/nginx/error.log
```

Access Log

```bash
sudo tail -f /var/log/nginx/access.log
```

Test Configuration

```bash
sudo nginx -t
```

---

# Troubleshooting Checklist

### Backend not reachable

```bash
curl http://Backend Server Private IP :3000
```

---

### Nginx not running

```bash
sudo systemctl status nginx
```

---

### Configuration Error

```bash
sudo nginx -t
```

---

### Port 80

```bash
sudo ss -tulnp | grep :80
```

---

### Port 443

```bash
sudo ss -tulnp | grep :443
```

---

### Test Reverse Proxy

```bash
curl http://localhost
```

---

### Test HTTPS

```bash
curl -k https://localhost
```

---

# Final Result

- ✅ Backend hosted on Private EC2
- ✅ Nginx hosted on Public EC2
- ✅ Reverse Proxy configured
- ✅ HTTP working
- ✅ HTTPS enabled using Self-Signed SSL
- ✅ HTTP automatically redirects to HTTPS
- ✅ Encrypted communication between Client and Nginx
- ✅ Nginx securely proxies requests to the backend application
