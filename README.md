# **MEAN Stack Deployment (MongoDB, Express, Angular, Node.js) with Docker, Docker Hub, GitHub Actions CI/CD & AWS EC2**

This project demonstrates a **fully automated CI/CD pipeline** using:

* **Docker & Docker Compose**
* **Docker Hub Container Registry**
* **GitHub Actions CI/CD**
* **AWS EC2 (Ubuntu)**
* **Nginx Reverse Proxy**
* **MEAN Stack Application**

---

# 📌 **Table of Contents**

1. [Project Architecture](#project-architecture)
2. [Setup Instructions](#setup-instructions)

   * [EC2 Setup](#1-ec2-setup)
   * [Docker Hub Setup](#2-docker-hub-setup)
   * [GitHub Actions CI/CD Setup](#3-github-actions-cicd-setup)
   * [Docker Compose Setup](#4-docker-compose-setup-on-ec2)
   * [Nginx Setup](#5-nginx-setup-reverse-proxy)
3. [Deployment Workflow](#deployment-workflow)
4. [Environment Variables](#environment-variables)
5. [Screenshots](#screenshots)
6. [Troubleshooting](#troubleshooting)

---

# 🏗 **Project Architecture**

```
Developer → GitHub → GitHub Actions → Docker Build → Push to Docker Hub
        → SSH to EC2 → docker compose pull → restart containers
        → Nginx → Live Production App
```

---

# ⚙️ **Setup Instructions**

---

# 1️⃣ **EC2 Setup**

## **Step 1 — Launch EC2 Instance**

* OS: **Ubuntu 22.04 LTS**
* Size: **t2.micro (Free Tier)**
* Storage: **20GB+**

## **Step 2 — Configure Security Groups**

Open inbound ports:

| Port  | Description                 | Required          |
| ----- | --------------------------- | ----------------- |
| 22    | SSH Access                  | ✔                 |
| 80    | HTTP (Nginx)                | ✔                 |
| 443   | HTTPS (SSL)                 | Optional          |
| 4200  | Angular Frontend (dev only) | Optional          |
| 8080  | Node Backend                | Optional          |
| 27017 | MongoDB                     | ❌ Not recommended |

## **Step 3 — Connect to EC2**

```bash
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

## **Step 4 — Install Docker**

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
```

## **Step 5 — Install Docker Compose**

```bash
sudo apt install docker-compose -y
```

## **Step 6 — Give non-sudo access to Docker**

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

---

# 2️⃣ **Docker Hub Setup**

## **Step 1 — Create a Docker Hub Account**

👉 [https://hub.docker.com/](https://hub.docker.com/)

## **Step 2 — Create Repositories**

Create these two:

```
yourname/backend
yourname/frontend
```

## **Step 3 — Create Docker Hub Access Token**

Docker Hub → Account Settings → **Security** → Create Token

Save:

* `DOCKERHUB_USERNAME`
* `DOCKERHUB_TOKEN`

You will add these in GitHub Actions secrets.

---

# 3️⃣ **GitHub Actions CI/CD Setup**

## **Step 1 — Add GitHub Secrets**

Go to:

`GitHub Repo → Settings → Secrets → Actions`

Add:

| Secret               | Example              |
| -------------------- | -------------------- |
| `DOCKERHUB_USERNAME` | yourname             |
| `DOCKERHUB_TOKEN`    | token123             |
| `EC2_HOST`           | 13.233.xx.xx         |
| `EC2_USERNAME`       | ubuntu               |
| `EC2_SSH_KEY`        | contents of .pem key |
| `EC2_PROJECT_PATH`   | /home/ubuntu/app     |

---

## **Step 2 — Create GitHub Actions Workflow**

Create:

```
.github/workflows/deploy.yml
```

Paste this:

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:

  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Backend Image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/backend:latest ./backend
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/backend:latest

      - name: Build Frontend Image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/frontend:latest ./frontend
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/frontend:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ${{ secrets.EC2_PROJECT_PATH }}
            docker compose pull
            docker compose down
            docker compose up -d
```

✔ This will automatically run every time you `git push`.

---

# 4️⃣ **Docker Compose Setup on EC2**

Inside EC2:

```
mkdir app
cd app
nano docker-compose.yml
```

Paste the following:

```yaml
version: '3.8'

services:

  mongo:
    image: mongo:6
    container_name: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin@123
    volumes:
      - mongo-data:/data/db
    networks:
      - mean-net

  backend:
    image: yourdockerhub/backend:latest
    container_name: backend
    environment:
      MONGO_URI: mongodb://admin:admin%40123@mongo:27017/dd_db?authSource=admin
      PORT: 8080
    ports:
      - "8080:8080"
    depends_on:
      - mongo
    networks:
      - mean-net

  frontend:
    image: yourdockerhub/frontend:latest
    container_name: frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - mean-net

volumes:
  mongo-data:

networks:
  mean-net:
    driver: bridge
```

Start it manually once:

```bash
docker compose up -d
```

---

# 5️⃣ **Nginx Setup (Reverse Proxy)**

## Install Nginx

```bash
sudo apt install nginx -y
```

## Configure Proxy

```bash
sudo nano /etc/nginx/sites-available/mean-app
```

Paste:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://localhost:80;  # frontend
    }

    location /api/ {
        proxy_pass http://localhost:8080/;  # backend
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/mean-app /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

---

# 🚀 **Deployment Workflow**

1. Developer pushes code → **GitHub**
2. GitHub Actions triggers:

   * Builds frontend & backend images
   * Pushes to Docker Hub
3. Actions SSHs into EC2
4. Pulls latest images
5. Runs `docker compose up -d`
6. Nginx routes incoming traffic
7. App automatically updates 🎉

---

# 🌱 **Environment Variables**

Backend:

```
PORT=8080
MONGO_URI=mongodb://admin:admin%40123@mongo:27017/dd_db?authSource=admin
```

---
