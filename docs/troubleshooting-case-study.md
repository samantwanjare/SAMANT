# Real Production Troubleshooting Case Study

## Django CI/CD Deployment on AWS EC2 using Jenkins, Docker, Nginx and MySQL

---

# Project Objective

The objective was to deploy a Django Notes Application using:

- Jenkins Pipeline
- Docker
- Docker Compose
- Nginx
- Gunicorn
- MySQL
- AWS EC2

The expectation was simple:

```text
Code Push
    ↓
Jenkins Build
    ↓
Docker Build
    ↓
Container Deployment
    ↓
Application Available
```

However, although Jenkins completed successfully, the application never came online.

This document explains the entire troubleshooting journey.

---

# Stage 1: First Sign of Failure

After Jenkins completed successfully, I attempted to access the application.

```bash
curl http://localhost:8000
```

Result:

```bash
curl: (7) Failed to connect to localhost port 8000
```

Initial assumption:

> Maybe port 8000 is blocked by AWS Security Groups or Firewall.

---

# Stage 2: Network Investigation

Commands executed:

```bash
sudo ss -tulpn | grep 8000
sudo lsof -i :8000
curl http://localhost:8000
```

Result:

No process was listening on port 8000.

Important observation:

The problem was NOT:

- Security Groups
- Nginx
- Firewall
- Port Mapping

The application was crashing before reaching the networking stage.

---

# Stage 3: Docker Investigation

```bash
docker ps
docker ps -a
```

Result:

```text
db_cont      Exited
django_cont  Created
nginx_cont   Created
```

Critical discovery:

The database container was not running.

Failure chain:

```text
MySQL fails
     ↓
Django cannot connect
     ↓
Django startup fails
     ↓
Gunicorn never starts
     ↓
Port 8000 never opens
```

---

# Stage 4: Investigating MySQL Failure

```bash
docker logs db_cont
```

Observed:

```text
InnoDB initialization has started
```

Container stopped shortly afterward.

---

# Stage 5: Memory Investigation

Checking memory:

```bash
free -h
```

Output showed only about 908 MB RAM.

Hypothesis:

Possible Out Of Memory (OOM) condition.

Verification:

```bash
sudo dmesg -T | grep -i -E "killed process|out of memory|oom"
```

---

# Stage 6: Creating Swap Memory

Commands executed:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h
```

Verification:

```text
Swap: 2.0Gi
```

### Mistake Encountered

Commands were accidentally executed twice.

Errors:

```bash
fallocate failed: Text file busy
swapon failed: Device or resource busy
```

Root cause:

Swap was already active.

System behavior was normal.

---

# Stage 7: Making Swap Persistent

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verification:

```bash
grep swapfile /etc/fstab
swapon --show
```

Result:

```text
/swapfile file 2G
```

Swap survived reboot successfully.

---

# Stage 8: MySQL Starts but Django Still Fails

Database container was started:

```bash
docker start db_cont
```

Django still crashed.

Logs:

```bash
docker logs django_cont
```

Error:

```text
Host '172.18.0.3' is not allowed to connect to this MySQL server
```

---

# Stage 9: Investigating MySQL Authentication

Environment variables:

```bash
docker exec db_cont env | grep MYSQL
```

Output:

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=test_db
```

Everything looked correct.

---

# Stage 10: Inspecting MySQL Users

Entered container:

```bash
docker exec -it db_cont bash
mysql -u root
```

Query:

```sql
SELECT User, Host FROM mysql.user;
```

Result:

```text
root | localhost
```

Important learning:

MySQL authentication uses:

```text
user + host
```

not only:

```text
username + password
```

Django container connected from Docker network:

```text
172.18.0.x
```

MySQL considered this a remote host.

Connection was rejected.

---

# Stage 11: Fixing MySQL Host Access

Created remote-access user:

```sql
CREATE USER 'root'@'%' IDENTIFIED BY 'root';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Verification:

```sql
SELECT User, Host FROM mysql.user;
```

Output:

```text
root | localhost
root | %
```

Authentication issue resolved.

---

# Stage 12: New Error Appears

Django logs now showed:

```text
Unknown database 'test_db'
```

Lesson:

Fixing one layer exposed the next hidden problem.

---

# Stage 13: Verifying Database Existence

```sql
SHOW DATABASES;
```

Result:

```text
information_schema
mysql
performance_schema
sys
```

Missing:

```text
test_db
```

---

# Stage 14: Why MYSQL_DATABASE Didn't Work

Docker Compose contained:

```yaml
MYSQL_DATABASE=test_db
```

But Compose also used:

```yaml
volumes:
  - ./data/mysql/db:/var/lib/mysql
```

Explanation:

When MySQL detects existing data inside the mounted volume:

- MYSQL_DATABASE is ignored
- MYSQL_ROOT_PASSWORD is ignored
- Initialization scripts are skipped

This explained why the database was never created.

---

# Stage 15: Creating Database Manually

```sql
CREATE DATABASE test_db;
```

Verification:

```sql
SHOW DATABASES;
```

Output:

```text
test_db
```

Database issue resolved.

---

# Stage 16: Django Finally Starts

Logs changed dramatically:

```text
Applying contenttypes.0001_initial... OK
Applying auth.0001_initial... OK
Applying admin.0001_initial... OK
Applying sessions.0001_initial... OK
```

Meaning:

- Database connection successful
- Migrations running
- Application startup progressing

---

# Stage 17: Gunicorn Startup

Logs:

```text
Starting gunicorn 20.1.0
Listening at: http://0.0.0.0:8000
Booting worker with pid: 9
```

This was the breakthrough moment.

Port 8000 finally became available.

---

# Stage 18: Final Validation

Container status:

```bash
docker ps
```

Output:

```text
db_cont       Up (healthy)
django_cont   Up (healthy)
nginx_cont    Up
```

Application validation:

```bash
curl http://localhost:8000
```

Service responded successfully.

---

# Additional Lessons Learned

## Why `mysql -u root` Worked Without Password

Inside the MySQL container:

```bash
mysql -u root
```

worked because MySQL socket authentication allowed local root access.

This confused troubleshooting initially because:

- Root login worked inside container
- Django login failed from another container

The difference was host-based authentication.

---

## Why Ubuntu Suggested Installing mysql-client

After exiting the container:

```bash
mysql
```

on EC2 host returned:

```text
Command 'mysql' not found
```

Ubuntu suggested:

```bash
sudo apt install mariadb-client-core
```

Reason:

The MySQL client existed only inside the container.

The EC2 host itself had no mysql client installed.

---

## Why Restarting Containers Showed Permission Errors

Commands:

```bash
docker restart <container>
systemctl restart docker
```

returned permission errors.

Reason:

Docker was installed through Snap and required elevated privileges.

Using sudo and proper Docker group permissions resolved access issues.

---

# Complete Failure Chain

```text
Small EC2 Instance
        ↓
Limited RAM
        ↓
Swap Missing
        ↓
MySQL Instability
        ↓
Django Cannot Reach Database
        ↓
MySQL User Restricted To localhost
        ↓
Authentication Failure
        ↓
Database Missing
        ↓
Migration Failure
        ↓
Gunicorn Never Starts
        ↓
Port 8000 Closed
        ↓
Application Down
```

---

# Skills Demonstrated

- Linux Administration
- Memory Management
- Swap Configuration
- Docker Troubleshooting
- Docker Compose Debugging
- Container Lifecycle Analysis
- Log Analysis
- MySQL Administration
- MySQL Authentication
- Persistent Volume Debugging
- Django Deployment
- Gunicorn Troubleshooting
- Nginx Reverse Proxy Architecture
- Jenkins CI/CD Validation
- Root Cause Analysis
- Production Incident Investigation

---

# Final Result

✅ Swap configured and persisted

✅ MySQL healthy

✅ Docker networking fixed

✅ MySQL authentication fixed

✅ Database created

✅ Django migrations successful

✅ Gunicorn running

✅ Nginx running

✅ Jenkins deployment working

✅ Application accessible

---

# Conclusion

This deployment was not fixed by a single command.

The resolution required:

1. Following logs instead of assumptions.
2. Validating each layer independently.
3. Fixing infrastructure before application issues.
4. Understanding Docker networking.
5. Understanding MySQL authentication behavior.
6. Understanding persistent volume initialization behavior.

The final success came from systematic troubleshooting rather than trial-and-error fixes.

This case study represents a realistic DevOps production debugging workflow and demonstrates practical skills in deployment diagnostics, root cause analysis, and infrastructure troubleshooting.
