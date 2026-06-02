# Deployment Guide

## Clone Repository

```bash
git clone https://github.com/pratikkamble14/django-notes-app.git
```

## Install Docker
```bash
sudo apt update

sudo apt install -y docker.io

sudo systemctl enable docker

sudo systemctl start docker

sudo usermod -aG docker $USER

newgrp docker

docker --version

docker-compose --version
```

## Configure Environment Variables

.env file example

## Build Containers
```bash
docker-compose up -d --build
```

## Verify Deployment
```bash
docker ps

curl localhost:8000
```
