
# Deploy a Python Flask Application with MySQL on AWS EC2

> **Project:** Deploy a Flask application with a MySQL backend, access
> it from a web browser, verify data locally, configure automated
> backups, and run the application as a background service.

------------------------------------------------------------------------

# Architecture

``` text
                Internet
                     |
                     v
              Browser (HTTP)
                     |
                     v
             Ubuntu EC2 Instance
          +-----------------------+
          | Flask Application     |
          | (Port 5000)           |
          +-----------+-----------+
                      |
                      v
              MySQL Database
                      |
                      v
                 users table
                      |
                      v
         mysqldump -> /backup/*.sql
```

# Technologies

-   AWS EC2 (Ubuntu 24.04 LTS)
-   Python 3
-   Flask
-   MySQL Server
-   mysql-connector-python
-   systemd
-   cron
-   mysqldump

------------------------------------------------------------------------

# Step 1 -- Launch EC2

-   AMI: Ubuntu Server 24.04 LTS
-   Instance Type: t3.micro
-   Security Group:
    -   TCP 22 (SSH)
    -   TCP 80 (HTTP)
-   Connect:

``` bash
ssh -i IAM_saim.pem ubuntu@<PUBLIC-IP>
```

Update packages:

``` bash
sudo apt update
sudo apt upgrade -y
```

------------------------------------------------------------------------

# Step 2 -- Install Python

``` bash
sudo apt install python3 python3-pip python3-venv -y

python3 --version
pip3 --version
```

------------------------------------------------------------------------

# Step 3 -- Install MySQL

``` bash
sudo apt install mysql-server -y

sudo systemctl enable mysql
sudo systemctl start mysql

sudo systemctl status mysql
```

Secure MySQL:

``` bash
sudo mysql_secure_installation
```

------------------------------------------------------------------------

# Step 4 -- Configure Database

Login:

``` bash
sudo mysql
```

Create database, user and table:

``` sql
CREATE DATABASE userdb;

CREATE USER 'appuser'@'localhost'
IDENTIFIED BY 'Password@123';

GRANT ALL PRIVILEGES
ON userdb.*
TO 'appuser'@'localhost';

FLUSH PRIVILEGES;

USE userdb;

CREATE TABLE users(
id INT AUTO_INCREMENT PRIMARY KEY,
username VARCHAR(100),
email VARCHAR(150),
age INT,
gender VARCHAR(20)
);
```

------------------------------------------------------------------------

# Step 5 -- Create Flask Project

``` bash
mkdir ~/flaskapp
cd ~/flaskapp

python3 -m venv venv

source venv/bin/activate

pip install flask mysql-connector-python
```

Project structure:

``` text
flaskapp/
├── app.py
├── templates/
│   ├── index.html
│   └── users.html
└── venv/
```

------------------------------------------------------------------------

# app.py

``` python
from flask import Flask, render_template, request, redirect
import mysql.connector

app = Flask(__name__)

db = mysql.connector.connect(
    host="localhost",
    user="appuser",
    password="Password@123",
    database="userdb"
)

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/add", methods=["POST"])
def add():
    username = request.form["username"]
    email = request.form["email"]
    age = request.form["age"]
    gender = request.form["gender"]

    cursor = db.cursor()
    cursor.execute(
        "INSERT INTO users(username,email,age,gender) VALUES(%s,%s,%s,%s)",
        (username,email,age,gender)
    )
    db.commit()

    return redirect("/users")

@app.route("/users")
def users():
    cursor = db.cursor()
    cursor.execute("SELECT * FROM users")
    data = cursor.fetchall()
    return render_template("users.html", users=data)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

------------------------------------------------------------------------

# templates/index.html

``` html
<!DOCTYPE html>
<html>
<head>
<title>User Form</title>
</head>
<body>

<h2>Add User</h2>

<form action="/add" method="POST">

Username<br>
<input type="text" name="username" required><br><br>

Email<br>
<input type="email" name="email" required><br><br>

Age<br>
<input type="number" name="age"><br><br>

Gender<br>

<select name="gender">
<option>Male</option>
<option>Female</option>
<option>Other</option>
</select>

<br><br>

<input type="submit" value="Save">

</form>

</body>
</html>
```

------------------------------------------------------------------------

# templates/users.html

``` html
<!DOCTYPE html>
<html>
<head>
<title>Users</title>
</head>
<body>

<h2>All Users</h2>

<table border="1">

<tr>
<th>ID</th>
<th>Username</th>
<th>Email</th>
<th>Age</th>
<th>Gender</th>
</tr>

{% for user in users %}
<tr>
<td>{{ user[0] }}</td>
<td>{{ user[1] }}</td>
<td>{{ user[2] }}</td>
<td>{{ user[3] }}</td>
<td>{{ user[4] }}</td>
</tr>
{% endfor %}

</table>

</body>
</html>
```

------------------------------------------------------------------------

# Step 6 -- Run Application

``` bash
source venv/bin/activate
python app.py
```

Access:

``` text
http://<EC2-PUBLIC-IP>:5000
```
<img width="2993" height="1618" alt="image" src="https://github.com/user-attachments/assets/6cff367b-cb5e-4619-86d4-60630ec7c7ce" />
<img width="1898" height="1218" alt="image" src="https://github.com/user-attachments/assets/232e7031-d2f8-4bb4-834b-aa3b8a9111e8" />

------------------------------------------------------------------------

# Step 7 -- Verify Database

``` bash
mysql -u appuser -p
```

``` sql
USE userdb;

SELECT * FROM users;
```
<img width="2841" height="1729" alt="image" src="https://github.com/user-attachments/assets/ecdb1ad5-4b9f-4d4f-892d-17556bbd7e3f" />

------------------------------------------------------------------------

# Step 8 -- Backup

Create directory:

``` bash
sudo mkdir /backup
```

Create script:

``` bash
sudo nano /backup/mysql_backup.sh
```

Contents:

``` bash
#!/bin/bash

DATE=$(date +%F-%H-%M-%S)

mysqldump \
--no-tablespaces \
-u appuser \
-p'Password@123' \
userdb > /backup/userdb-$DATE.sql
```

Permissions:

``` bash
sudo chmod +x /backup/mysql_backup.sh
```

Run:

``` bash
sudo /backup/mysql_backup.sh
```

Verify:

``` bash
ls -lh /backup

head -20 /backup/userdb-*.sql
```

<img width="2346" height="1033" alt="image" src="https://github.com/user-attachments/assets/9a768507-1f5a-49bb-a8b1-dfa2e2d45b71" />


------------------------------------------------------------------------

# Step 9 -- Restore

``` bash
sudo mysql
```

``` sql
CREATE DATABASE restoredb;
EXIT;
```

``` bash
sudo mysql -u root -p restoredb < /backup/userdb-YYYY-MM-DD-HH-MM-SS.sql
```

Verify:

``` sql
USE restoredb;
SELECT * FROM users;
```

<img width="2918" height="1994" alt="image" src="https://github.com/user-attachments/assets/acd4e611-afb4-456e-9370-d8800605f7a4" />


------------------------------------------------------------------------

# Step 10 -- Automate Backup

``` bash
crontab -e
```

Add:

``` cron
0 2 * * * /backup/mysql_backup.sh >> /backup/backup.log 2>&1
```

Verify:

``` bash
crontab -l
```

------------------------------------------------------------------------

# Step 11 -- Run Flask in Background

Create service:

``` bash
sudo nano /etc/systemd/system/flaskapp.service
```

``` ini
[Unit]
Description=Flask User Management Application
After=network.target mysql.service

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/flaskapp
Environment="PATH=/home/ubuntu/flaskapp/venv/bin"
ExecStart=/home/ubuntu/flaskapp/venv/bin/python /home/ubuntu/flaskapp/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable flaskapp
sudo systemctl start flaskapp
```

Check:

``` bash
sudo systemctl status flaskapp

journalctl -u flaskapp -f
```

------------------------------------------------------------------------

# Troubleshooting

## MySQL backup error

    Access denied; PROCESS privilege

Use:

``` bash
mysqldump --no-tablespaces
```

## Flask not reachable

-   Check service status.
-   Verify Security Group allows HTTP (or port 5000 if accessing
    directly).
-   Confirm the app is listening:

``` bash
ss -tulnp | grep 5000
```

------------------------------------------------------------------------

# Final Verification

-   EC2 launched ✅
-   Python installed ✅
-   MySQL installed ✅
-   Database created ✅
-   Flask app deployed ✅
-   Browser access working ✅
-   Data stored in MySQL ✅
-   mysqldump backup successful ✅
-   Cron configured ✅
-   systemd service configured ✅
