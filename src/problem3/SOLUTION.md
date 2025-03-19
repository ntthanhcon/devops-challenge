# Task

Provide steps to troubleshoot two production issues.

### Scenario

You are working as a DevOps Engineer on a cloud-based infrastructure where a virtual machine (VM), running Ubuntu 24.04, with 64GB of storage is under your management. Recently, your monitoring tools have reported that the VM is consistently running at 99% storage usage. This VM is responsible for only running one service - a NGINX load balancer as a traffic router for upstream services. 

### Challenge

Describe how you would troubleshoot the issue, and what causes/scenario do you expect to encounter, as well as what are the impacts and recovery steps for each possible root cause.

### Additional Setup Suggested (by Thanh)

- I Suggest Install NGINX first using the quickest way (Docker Compose)
- Simulate Scenario (Create a single heavy file in NGNIX) that take up to 98% storage to make VM run with near 99% storage usage (Script create 11GB file, currently my Ubuntu VM only have 12GB)
- Finish the challenge with the most efficiency way, demonstrate my troubleshoot skill

### Simulation Step Summary (Suggested by Thanh)

1. Environment Setup
```bash
# Create project directory and install dependencies in one go
mkdir -p nginx-storage-demo && cd nginx-storage-demo && \
sudo apt update && sudo apt install -y docker.io docker-compose && \
sudo usermod -aG docker $USER && newgrp docker
```

2. NGINX Configuration
```bash
# Create docker-compose.yml
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx_logs:/var/log/nginx
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
EOF

# Create nginx.conf
cat > nginx.conf <<EOF
events {
    worker_connections 1024;
}
http {
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    server {
        listen 80;
        server_name localhost;
        location / {
            return 200 'Hello from NGINX!\n';
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

3. Storage Issue Simulation
```bash
# Script create 11GB file, currently my Ubuntu VM have 12GB only
cat > simulate_storage_issue.sh <<EOF
#!/bin/bash
mkdir -p nginx_logs
dd if=/dev/zero of=nginx_logs/large_access.log bs=1M count=11000
echo "Large file created in nginx_logs directory"
EOF
chmod +x simulate_storage_issue.sh
```

4. Execution and Verification
```bash
# Start NGINX container
docker-compose up -d

# Fix permissions for nginx_logs directory
sudo chown -R $USER:$USER nginx_logs

# Run simulation to create 11GB file
./simulate_storage_issue.sh

# Verify NGINX is running
curl localhost:80
# Expected output: Hello from NGINX!

# Verify file size
ls -lh nginx_logs/large_access.log
# Expected output: -rw-rw-r-- 1 user docker 11G Mar 18 18:11 nginx_logs/large_access.log
```

**Solution**:

# NGINX VM Storage Analysis

## Executive Summary

This document outlines the storage issues discovered on the NGINX VM and provides recommendations for resolution. The system was found to be at 95% capacity with only 1.5GB of free space remaining.

## Disk Usage Analysis

| Filesystem | Size | Used | Available | Use% | Mounted on |
|------------|------|------|-----------|------|------------|
| /dev/mapper/ubuntu--vg-ubuntu--lv | 31G | 28G | 1.5G | 95% | / |

### Main Storage Consumers

1. **Home Directory**: 14GB total
   - `/home/thanh/nginx-storage-demo`: 11GB
     - Contains a single large log file `large_access.log` (11GB)

2. **Docker Storage**: 11GB in `/var/lib/docker`
   - Images: 3.95GB (15 images, with 54% being reclaimable)
   - Containers: 215.9MB
   - Volumes: 3.933GB

3. **Other Notable Usage**:
   - `/var/log`: 134MB
   - `/var/cache`: 157MB

**Diagnostic Methods Used**

The following diagnostic methods and commands were used to identify the storage issues:

Before beginning the storage analysis, we verified how NGINX was deployed on the system:

### Verifying Native Installation
```bash
systemctl status nginx
which nginx
nginx -v
```
These commands did not return a valid path or version, confirming NGINX was not installed natively on Ubuntu.

### Verifying Docker Deployment
```bash
docker ps -a | grep nginx
```
Result: `ba755735d649   nginx:latest`

This confirmed NGINX was running as a Docker container.

### Initial Storage Issue Assessment
```bash
df -h
docker system df
docker exec -it ba755735d649 du -sh /var/log/nginx/
```
The last command revealed: `11G     /var/log/nginx/`


1. **Overall Disk Space Analysis**:
   ```bash
   df -h
   ```
   This command revealed that the root filesystem was at 95% capacity.

2. **Directory Size Analysis**:
   ```bash
   sudo du -h --max-depth=1 / | sort -hr
   ```
   This identified `/home` (14GB) and `/var` (12GB) as the largest consumers of space.

3. **Subdirectory Investigation**:
   ```bash
   sudo du -h --max-depth=1 /var | sort -hr
   sudo du -h --max-depth=1 /var/lib | sort -hr
   sudo du -h --max-depth=1 /home | sort -hr
   du -h --max-depth=1 ~ | sort -hr
   ```
   These commands narrowed down the specific directories consuming the most space.

4. **Docker Space Analysis**:
   ```bash
   sudo docker system df
   ```
   This revealed the breakdown of Docker's disk usage (images, containers, volumes).

5. **Log File Investigation**:
   ```bash
   du -h --max-depth=1 /home/thanh/nginx-storage-demo | sort -hr
   ls -lah /home/thanh/nginx-storage-demo/nginx_logs
   ```
   These commands identified the large 11GB log file.

**Potential Impact**: If the disk reaches 100% capacity, Docker may fail to start new containers or cause NGINX crashes due to log write failures.


**Recommendation**

1. **Immediate Actions to Free Space**:
   - Remove the large demo log file:
     ```bash
     rm /home/thanh/nginx-storage-demo/nginx_logs/large_access.log
     ```

2. **Long-term Solutions**:
   - Implement log rotation for NGINX:
     ```bash
     sudo nano /etc/logrotate.d/nginx
     ```
     Add appropriate rotation settings to prevent logs from growing too large.
   
   - Set up monitoring for disk usage with alerts at 80% capacity.
   
   - Consider extending the volume size if storage needs are expected to grow.

**Recovery Steps (if VM Crash/Lag)**

### For Log File Issues
1. **Immediate Recovery**:
   ```bash
   # Remove or truncate large log files
   sudo truncate -s 0 /home/thanh/nginx-storage-demo/nginx_logs/large_access.log
   # OR delete if not needed
   sudo rm /home/thanh/nginx-storage-demo/nginx_logs/large_access.log
   ```

2. **Configure Log Rotation**:
   ```bash
   # Create proper NGINX log rotation configuration
   sudo tee /etc/logrotate.d/nginx-docker <<EOF
   /home/thanh/nginx-storage-demo/nginx_logs/*.log {
       daily
       missingok
       rotate 7
       compress
       delaycompress
       notifempty
       create 640 root docker
       sharedscripts
       postrotate
           [ -s /var/run/nginx.pid ] && kill -USR1 \$(cat /var/run/nginx.pid)
       endscript
   }
   EOF
   
   # Test the configuration
   sudo logrotate -d /etc/logrotate.d/nginx-docker
   ```

3. **Restart NGINX** (if necessary):
   ```bash
   docker restart $(docker ps -q --filter "ancestor=nginx:latest")
   ```

### For Docker Storage Issues
1. **Clean Unused Resources**:
   ```bash
   # Remove unused containers, networks, images, and volumes
   docker system prune -a --volumes --force
   ```

2. **Limit Docker Log Sizes**:
   ```bash
   # Configure Docker daemon to limit log sizes
   sudo tee /etc/docker/daemon.json <<EOF
   {
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "10m",
       "max-file": "3"
     }
   }
   EOF
   
   # Restart Docker to apply changes
   sudo systemctl restart docker
   ```

3. **Verify Space Freed**:
   ```bash
   df -h
   docker system df
   ```

### Emergency Recovery (If System Becomes Unresponsive)
1. **Create Space for Critical Operations**:
   ```bash
   # Find and remove the largest files to create space for recovery
   sudo find / -type f -size +100M -exec ls -lh {} \; | sort -rh | head -n 10
   ```

2. **Force Remove Docker Containers** (if Docker is unresponsive):
   ```bash
   sudo rm -rf /var/lib/docker/containers/<container-id>
   ```

3. **Clear Journal Logs** (often overlooked):
   ```bash
   sudo journalctl --vacuum-time=1d
   ```

## Ongoing Maintenance

- **Regular Cleanup Schedule**:
  - Weekly: Check and clean Docker images/containers
  - Monthly: Review and rotate logs
  - Quarterly: Full system cleanup

- **Monitoring**:
  - Add disk monitoring to existing monitoring solutions
  - Set up automated reporting of disk usage trends

## Contact

For questions or further assistance, please contact the infrastructure team. 