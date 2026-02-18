Below is the **clean, consolidated, copy-paste setup** using:

* Local Docker Compose
* Production Docker Compose
* Ubuntu 24.04 on AWS Lightsail
* Default user: `ubuntu`
* MariaDB
* Automated deployment via GitHub Actions

No Bitnami. No preinstalled WordPress. Clean VM.

---

# ARCHITECTURE

```
Local Machine
(Docker Compose)
      │
      │ git push
      ▼
GitHub (main branch)
      │
      ▼
GitHub Actions (SSH)
      │
      ▼
Lightsail Ubuntu 24.04
Docker Compose
  ├── WordPress
  └── MariaDB
```

Single VM. Two containers. Automated pull + restart.

---

# PART 1 — LOCAL DEVELOPMENT SETUP

---

## 1️⃣ Install Local Prerequisites

Install:

* Docker Desktop (or Docker Engine)
* Git

Verify:

```
docker --version
docker compose version
git --version
```

---

## 2️⃣ Create Project Structure

```
wordpress-project/
├── docker-compose.local.yml
├── docker-compose.prod.yml
├── .env.local
├── wp-content/
│   └── themes/
│       └── your-theme/
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 3️⃣ Create `.env.local`

```
DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=wppassword
DB_ROOT_PASSWORD=rootpassword
WP_PORT=8000
```

---

## 4️⃣ Create `docker-compose.local.yml`

```yaml
version: '3.8'

services:

  db:
    image: mariadb:11
    restart: unless-stopped
    environment:
      MARIADB_DATABASE: ${DB_NAME}
      MARIADB_USER: ${DB_USER}
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:php8.2-apache
    depends_on:
      - db
    restart: unless-stopped
    ports:
      - "${WP_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    volumes:
      - ./wp-content:/var/www/html/wp-content

volumes:
  db_data:
```

---

## 5️⃣ Run Locally

```
docker compose --env-file .env.local -f docker-compose.local.yml up -d
```

Visit:

```
http://localhost:8000
```

Complete WordPress setup.

---

## 6️⃣ Initialise Git

```
git init
git add .
git commit -m "Initial setup"
git branch -M main
```

Create GitHub repo, then:

```
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

---

# PART 2 — LIGHTSAIL (UBUNTU 24.04) SETUP

---

## 1️⃣ Create Instance

In AWS Lightsail:

* Platform: Linux
* OS: Ubuntu 24.04
* Plan: 1 GB minimum (2 GB recommended)
* Attach Static IP

Default user: `ubuntu`

---

## 2️⃣ SSH Into Server

```
ssh ubuntu@YOUR_STATIC_IP
```

---

## 3️⃣ Install Docker (Official Method for Ubuntu 24.04)

Run:

```
sudo apt update
sudo apt install ca-certificates curl gnupg -y

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io \
docker-buildx-plugin docker-compose-plugin git -y
```

Add user to Docker group:

```
sudo usermod -aG docker ubuntu
exit
```

Reconnect via SSH.

Verify:

```
docker --version
docker compose version
```

---

## 4️⃣ Clone Repository

```
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
```

---

## 5️⃣ Create `.env.prod` (ON SERVER ONLY)

```
nano .env.prod
```

Paste:

```
DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=VeryStrongProductionPassword
DB_ROOT_PASSWORD=VeryStrongRootPassword
WP_PORT=80
```

Save and exit.

Do NOT commit this file.

---

## 6️⃣ Create `docker-compose.prod.yml`

(Commit this file from your local repo)

```yaml
version: '3.8'

services:

  db:
    image: mariadb:11
    restart: always
    environment:
      MARIADB_DATABASE: ${DB_NAME}
      MARIADB_USER: ${DB_USER}
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:php8.2-apache
    depends_on:
      - db
    restart: always
    ports:
      - "${WP_PORT}:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    volumes:
      - ./wp-content:/var/www/html/wp-content

volumes:
  db_data:
```

Push it to GitHub before continuing.

---

## 7️⃣ Start Production

On server:

```
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Visit:

```
http://YOUR_STATIC_IP
```

Complete WordPress setup.

---

# PART 3 — AUTOMATED DEPLOYMENT

---

## 1️⃣ Add GitHub Secrets

In GitHub → Settings → Secrets → Actions:

Add:

* `SERVER_IP`
* `SERVER_USER` → `ubuntu`
* `SSH_KEY` → private SSH key contents

---

## 2️⃣ Create Deployment Workflow

`.github/workflows/deploy.yml`

```yaml
name: Deploy to Lightsail

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: SSH Deploy
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd YOUR_REPO
            git pull origin main
            docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Commit and push.

Deployment is now automatic on every push to `main`.

---

# PART 4 — REGULAR WORKFLOW

---

## Develop Locally

```
docker compose --env-file .env.local -f docker-compose.local.yml up -d
```

Edit theme files.

Test at:

```
http://localhost:8000
```

---

## Deploy

```
git add .
git commit -m "Describe changes"
git push origin main
```

GitHub Actions:

* SSH to Lightsail
* Pull latest code
* Restart containers

Production updates automatically.

---

# RESET COMMANDS

Local reset:

```
docker compose down -v
```

Production reset (DESTRUCTIVE — deletes database):

```
docker compose -f docker-compose.prod.yml down -v
```

---

# RESULT

You now have:

* Clean Ubuntu 24.04 server
* Docker-managed WordPress
* MariaDB container
* Environment separation via `.env`
* Automated CI/CD
* Minimal AWS complexity
* Fully reproducible setup

If needed, next steps could include:

* SSL via Certbot
* Automated database backups
* Rate limiting + firewall hardening
* A condensed student-facing handout version
