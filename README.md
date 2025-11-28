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
* Size: **t3.micro (Free Tier)**
* Storage: **16GB+**
![EC2](/screenshots/EC2.png)


## **Step 2 — Configure Security Groups**
![Security-group](/screenshots/security-group.jpeg)
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
![dockerhub-user](/screenshots/dockerhub-user.png)


## **Step 2 — Create Repositories**

Create these two:

```
navabdarveshali/backend-image
navabdarveshali/frontend-image
```

## **Step 3 — Create Docker Hub Access Token**
![dockerhub](/screenshots/docker-accestoken-1.png)

Docker Hub → Account Settings → **Security** → Create Token

Save:

* `DOCKERHUB_USERNAME`
* `DOCKERHUB_TOKEN`

You will add these in GitHub Actions secrets.
![dockerhub](/screenshots/docker%20access%20token%202%20copy.png)

---

# 3️⃣ **GitHub Actions CI/CD Setup**

## **Step 1 — Add GitHub Secrets**

Go to:

`GitHub Repo → Settings → Secrets → Actions`
![github secrets](/screenshots/github-secrets.png)
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
name: CRUD-DD-TASK-MEAN-APP

on:
  push:
    branches: [ "main" ]

jobs:

  build-and-push:
    name: Build & Push Docker Images
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Backend Image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/backend-image:latest ./backend

      - name: Push Backend Image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/backend-image:latest

      - name: Build Frontend Image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/frontend-image:latest ./frontend

      - name: Push Frontend Image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/frontend-image:latest


  deploy:
    name: Deploy to AWS EC2
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Copy All Project Files to EC2 (excluding .git & node_modules)
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "./"
          target: ${{ secrets.EC2_PROJECT_PATH }}
          overwrite: true
          rm: true
          skip_symlinks: true


      - name: Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ${{ secrets.EC2_PROJECT_PATH }}

            echo "Rebuilding and restarting containers..."
            docker-compose down
            docker-compose up --build -d


```
![githubactions](/screenshots/actions.png)
✔ This will automatically run every time you `git push`.

---

# 5️⃣ **Nginx Setup (Reverse Proxy)**

## Install Nginx

```bash
sudo apt install nginx -y
```

## Configure Proxy

```bash
sudo vi /etc/nginx/nginx.conf
```

Paste:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:4200;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    
    location /api/ {
        proxy_pass http://localhost:8080/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
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
