# Deployment Guide

## Clone Repository

git clone https://github.com/pratikkamble14/django-notes-app.git

## Install Docker

sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker-compose --version

## Configure Environment Variables

.env file example

## Build Containers

docker-compose up -d --build

## Verify Deployment

docker ps

curl localhost:8000
