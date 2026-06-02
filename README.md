# Simple Notes App for TWS Community
This is a simple notes app built with React and Django.

## Requirements
1. Python 3.9
2. Node.js
3. React

## Installation
1. Clone the repository
```
git clone https://github.com/LondheShubham153/django-notes-app.git
```

2. Build the app
```
docker build -t notes-app .
```

3. Run the app
```
docker run -d -p 8000:8000 notes-app:latest
```

## Nginx

Install Nginx reverse proxy to make this application available

`sudo apt-get update`
`sudo apt install nginx`

## Production Troubleshooting Case Study

During deployment, the application failed due to multiple infrastructure and database issues.

Topics covered:

- Docker troubleshooting
- MySQL authentication issues
- Container networking
- Persistent volume behavior
- Swap configuration
- Gunicorn startup debugging
- Root cause analysis

👉 Full report: [docs/troubleshooting-case-study.md](docs/troubleshooting-case-study.md)