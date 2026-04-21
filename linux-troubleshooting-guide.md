# Linux Troubleshooting Systematic Guide

## 🎯 Troubleshooting Workflow

```
1. Identify the symptom
2. Check system health
3. Review logs
4. Verify configuration
5. Test connectivity/permissions
6. Fix and validate
7. Document solution
```

---

## 📋 Essential Diagnostic Commands

### Quick Health Check
```bash
# System uptime and load average
uptime

# CPU usage
top -bn1 | head -20
htop  # If installed

# Memory usage
free -h
cat /proc/meminfo

# Disk usage
df -h
df -i  # Inode usage

# Current processes
ps aux | head -20
ps aux --sort=-%mem | head -10  # Top memory consumers
ps aux --sort=-%cpu | head -10  # Top CPU consumers

# System messages
dmesg | tail -50
journalctl -xe --no-pager | tail -50

# Network status
ip addr show
netstat -tuln
ss -tuln
```

---

## 🔥 Common Issues & Troubleshooting

### 1. High CPU Usage

**Symptoms:**
- System becomes slow/unresponsive
- Commands take long to execute
- Load average > number of CPU cores

**Diagnostic Commands:**
```bash
# Check current load
uptime
# Output: 15:30:01 up 10 days, 3:45, 2 users, load average: 8.50, 7.20, 6.80
# If load > CPU cores (check with: nproc), system is overloaded

# Identify CPU-consuming processes
top -bn1
# Press '1' to see per-CPU usage
# Press 'P' to sort by CPU

# Or use:
ps aux --sort=-%cpu | head -20

# Check for zombie processes
ps aux | grep -w Z

# See what each CPU core is doing
mpstat -P ALL 1 5

# Check I/O wait (if high, disk is bottleneck)
iostat -x 1 5
```

**What to Check in Logs:**
```bash
# System logs for errors
journalctl -p err -b  # Errors since last boot
dmesg | grep -i "error\|fail\|warn"

# Check for OOM killer
grep -i "killed process" /var/log/syslog
journalctl | grep -i "killed process"

# Application logs
tail -100 /var/log/syslog | grep -i cpu
```

**Common Patterns to Look For:**
- `CPU: X% us, Y% sy, Z% wa` → High 'wa' = I/O bottleneck
- `load average: 15.0, 12.0, 10.0` → Sustained high load
- `Out of memory: Killed process` → OOM killer activated
- `[PID] segfault at` → Application crash

**Practice Scenario:**
```bash
# Simulate high CPU usage
# Generate load on all cores
stress --cpu $(nproc) --timeout 60s

# In another terminal, diagnose
top
# Watch CPU% column

ps aux --sort=-%cpu | head -5
# Identify 'stress' process

# Kill it
pkill stress

# Verify
uptime  # Load should decrease
```

**Real-World Fix Examples:**

```bash
# Scenario 1: Java application consuming 100% CPU
# Identify process
ps aux | grep java
# PID 12345, user: tomcat

# Check what it's doing
top -p 12345
# Shows 99.9% CPU

# Get thread dump
sudo -u tomcat jstack 12345 > /tmp/thread-dump.txt
cat /tmp/thread-dump.txt | grep -A 10 "RUNNABLE"

# Check application logs
tail -100 /opt/tomcat/logs/catalina.out

# Common fixes:
# 1. Restart application
sudo systemctl restart tomcat

# 2. Increase heap size (if OOM)
# Edit: /etc/default/tomcat
# Add: JAVA_OPTS="-Xmx4096m -Xms2048m"

# Scenario 2: Script stuck in infinite loop
ps aux | grep script.sh
# Shows high CPU

# Check the script
cat /path/to/script.sh
# Find the loop

# Kill it
kill -9 <PID>

# Prevent auto-restart if managed by systemd
sudo systemctl stop script.service
```

---

### 2. High Memory Usage / OOM (Out of Memory)

**Symptoms:**
- "Cannot allocate memory" errors
- Processes getting killed randomly
- System swap usage high
- Applications crash unexpectedly

**Diagnostic Commands:**
```bash
# Check memory usage
free -h
# Look at 'available' column, not just 'free'

# Detailed memory info
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree"

# Top memory consumers
ps aux --sort=-%mem | head -20

# Check swap usage
swapon --show
cat /proc/swaps

# Per-process memory details
cat /proc/<PID>/status | grep -i mem
pmap <PID>

# Check for memory leaks over time
watch -n 5 'free -h'

# System memory pressure
cat /proc/pressure/memory  # If kernel supports PSI
```

**What to Check in Logs:**
```bash
# OOM killer messages
dmesg | grep -i "out of memory"
journalctl | grep -i "out of memory"
grep -i "killed process\|oom" /var/log/kern.log

# Which processes were killed
journalctl -k | grep "Killed process"

# Memory warnings
dmesg | grep -i "memory\|oom"
```

**Common Log Patterns:**
```
Out of memory: Killed process 12345 (java) total-vm:8388608kB
oom-killer: gfp_mask=0x6200ca, order=0
Killed process 2468 (mysqld) total-vm:4194304kB, anon-rss:2097152kB
```

**Practice Scenario:**
```bash
# Simulate memory exhaustion (CAREFUL - can freeze system!)
# Limit memory first in a cgroup or use a small VM

# Generate memory pressure
stress --vm 2 --vm-bytes 1G --timeout 30s

# Watch memory usage
watch -n 1 'free -h'

# Check for OOM
dmesg | tail -20

# Identify the stress process
ps aux --sort=-%mem | head -5

# Clean up
pkill stress
```

**Real-World Fixes:**

```bash
# Scenario 1: Application memory leak

# Identify leaking process
ps aux --sort=-%mem | head -5
# Shows: PID 5678, 'node', 85% memory

# Check process details
pmap -x 5678 | tail -20
# Shows growing heap

# Immediate fix: Restart
sudo systemctl restart nodejs-app

# Long-term: Monitor memory over time
# Add to cron:
# */5 * * * * ps -p 5678 -o %mem,vsz,rss >> /var/log/app-memory.log

# Configure memory limits with systemd
sudo systemctl edit nodejs-app
# Add:
[Service]
MemoryMax=2G
MemoryHigh=1.5G

sudo systemctl daemon-reload
sudo systemctl restart nodejs-app

# Scenario 2: MySQL consuming too much memory

# Check MySQL memory usage
ps aux | grep mysql
mysql -e "SHOW VARIABLES LIKE '%buffer%';"

# Tune MySQL configuration
sudo vi /etc/mysql/my.cnf
# Adjust:
[mysqld]
innodb_buffer_pool_size = 2G  # Instead of 8G
key_buffer_size = 256M

sudo systemctl restart mysql

# Verify
mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

---

### 3. Disk Space Issues

**Symptoms:**
- "No space left on device"
- Applications can't write files
- Logs not updating
- Services failing to start

**Diagnostic Commands:**
```bash
# Check disk space
df -h
# Look for 100% or near 100%

# Check inode usage (can be full even with space available)
df -i

# Find large files
du -sh /* | sort -rh | head -10
du -sh /var/* | sort -rh | head -10

# Find files larger than 1GB
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# Check what's using deleted files (still holding space)
lsof +L1
lsof | grep deleted

# Find recently modified files (where growth is happening)
find /var/log -type f -mtime -1 -exec ls -lh {} \;

# Disk usage by directory (sorted)
du -hx --max-depth=1 /var | sort -rh | head -10
```

**What to Check in Logs:**
```bash
# System logs for disk errors
dmesg | grep -i "disk\|sda\|error"
journalctl -p err | grep -i "disk\|space"

# Application errors
grep -r "No space left on device" /var/log/

# Check log rotation
ls -lh /var/log/*.gz
cat /etc/logrotate.conf
ls /etc/logrotate.d/
```

**Common Log Patterns:**
```
write error: No space left on device
OSError: [Errno 28] No space left on device
SQLSTATE[HY000]: General error: 1021 Disk full
Can't create/write to file '/tmp/#sql_xxx.MAI' (Errcode: 28 - No space left on device)
```

**Practice Scenario:**
```bash
# Create test partition (or use existing filesystem)
# WARNING: This can fill disk - use test VM

# Fill disk partially
dd if=/dev/zero of=/tmp/testfile bs=1M count=1000

# Check space
df -h /tmp

# Find it
du -sh /tmp/* | sort -rh | head -5

# Remove it
rm /tmp/testfile

# Verify space reclaimed
df -h /tmp

# Test inode exhaustion
# Create many small files
mkdir /tmp/inode-test
for i in {1..100000}; do touch /tmp/inode-test/file$i; done

# Check inodes
df -i /tmp

# Clean up
rm -rf /tmp/inode-test
```

**Real-World Fixes:**

```bash
# Scenario 1: Log files filling disk

# Identify large log files
du -sh /var/log/* | sort -rh | head -10
# Shows: /var/log/application.log = 45G

# Check file size
ls -lh /var/log/application.log

# Immediate fix: Truncate (keeps file handle open)
sudo truncate -s 0 /var/log/application.log
# OR rotate manually:
sudo logrotate -f /etc/logrotate.d/application

# Long-term: Configure log rotation
sudo vi /etc/logrotate.d/application
/var/log/application.log {
    daily
    rotate 7
    size 100M
    compress
    delaycompress
    notifempty
    missingok
    create 0644 app app
}

# Test logrotate
sudo logrotate -d /etc/logrotate.d/application

# Scenario 2: Docker images filling disk

# Check Docker disk usage
docker system df

# Remove unused containers/images
docker system prune -a --volumes

# Or selectively:
docker container prune
docker image prune -a
docker volume prune

# Scenario 3: Deleted files still holding space

# Find processes holding deleted files
lsof +L1
# Shows: PID 1234 holding 10GB deleted file

# Restart the service to release
sudo systemctl restart service-name

# Or kill process
sudo kill -HUP 1234

# Verify space reclaimed
df -h

# Scenario 4: /tmp filled by old files

# Find old files
find /tmp -type f -mtime +7

# Remove them
find /tmp -type f -mtime +7 -delete

# Configure tmpwatch/tmpreaper for auto-cleanup
sudo apt install tmpreaper  # Ubuntu/Debian
# Edit /etc/tmpreaper.conf
```

---

### 4. Network Connectivity Issues

**Symptoms:**
- Cannot ping external hosts
- DNS resolution fails
- Application can't connect to services
- SSH connections timeout

**Diagnostic Commands:**
```bash
# Check network interfaces
ip addr show
ip link show

# Check if interface is up
ip link show eth0
# Look for "UP" in output

# Check routing table
ip route show
route -n

# Check default gateway
ip route | grep default

# Test connectivity
ping -c 3 8.8.8.8  # Google DNS
ping -c 3 google.com  # Tests DNS too

# Test DNS resolution
nslookup google.com
dig google.com
host google.com

# Check DNS servers
cat /etc/resolv.conf

# Test specific port
telnet google.com 80
nc -zv google.com 80

# Check listening ports
netstat -tuln
ss -tuln

# Check established connections
netstat -tupn
ss -tupn

# Check firewall rules
sudo iptables -L -n -v
sudo ufw status  # If using UFW

# Trace route
traceroute google.com
mtr google.com  # Better than traceroute

# Check for packet loss
ping -c 100 8.8.8.8 | tail -5
```

**What to Check in Logs:**
```bash
# Network-related errors
journalctl -u NetworkManager
dmesg | grep -i "network\|eth0\|link"

# Firewall logs
journalctl -u ufw
tail -100 /var/log/ufw.log

# Connection attempts
grep -i "connection" /var/log/syslog
journalctl | grep -i "refused\|timeout"
```

**Common Log Patterns:**
```
Network is unreachable
No route to host
Connection refused
Connection timed out
Name or service not known (DNS failure)
eth0: link down
eth0: Link is Up
```

**Practice Scenario:**
```bash
# Test DNS resolution
nslookup google.com
# If fails, check /etc/resolv.conf

# Simulate DNS failure
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
ping google.com  # Will fail

# Restore
sudo mv /etc/resolv.conf.backup /etc/resolv.conf
ping google.com  # Should work

# Test port connectivity
nc -zv localhost 22  # Should succeed if SSH running
nc -zv localhost 9999  # Should fail

# Check firewall
sudo iptables -L -n
```

**Real-World Fixes:**

```bash
# Scenario 1: Cannot reach internet

# Test basic connectivity
ping 8.8.8.8
# If works: DNS issue
# If fails: Network/routing issue

# Check interface status
ip addr show
# eth0 should show UP and have IP address

# If no IP address (DHCP):
sudo dhclient eth0
# Or restart networking:
sudo systemctl restart networking

# Check default gateway
ip route | grep default
# If missing, add manually:
sudo ip route add default via 192.168.1.1 dev eth0

# Scenario 2: DNS not resolving

# Check DNS servers
cat /etc/resolv.conf
# Should have: nameserver 8.8.8.8 or similar

# If empty or wrong, add:
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
echo "nameserver 8.8.4.4" | sudo tee -a /etc/resolv.conf

# For permanent fix with systemd-resolved:
sudo vi /etc/systemd/resolved.conf
# Add: DNS=8.8.8.8 8.8.4.4
sudo systemctl restart systemd-resolved

# Scenario 3: Firewall blocking connection

# Check if firewall is blocking
sudo iptables -L -n | grep 3306  # Example: MySQL port

# Allow port
sudo iptables -A INPUT -p tcp --dport 3306 -j ACCEPT
# Or with UFW:
sudo ufw allow 3306/tcp

# Make permanent
sudo iptables-save > /etc/iptables/rules.v4
# Or:
sudo ufw enable

# Scenario 4: Service not listening on expected interface

# Check what service is listening on
sudo netstat -tlnp | grep 8080

# If shows 127.0.0.1:8080 but need 0.0.0.0:8080
# Edit service config to bind to 0.0.0.0

# For example, nginx:
sudo vi /etc/nginx/sites-available/default
# Change: listen 127.0.0.1:80;
# To:     listen 80;

sudo systemctl restart nginx
```

---

### 5. Permission & Ownership Issues

**Symptoms:**
- "Permission denied" errors
- Cannot read/write files
- Cannot execute scripts
- Service fails to start

**Diagnostic Commands:**
```bash
# Check file permissions
ls -l /path/to/file

# Check directory permissions
ls -ld /path/to/directory

# Check ownership
stat /path/to/file

# Check current user and groups
whoami
id
groups

# Check who owns a process
ps aux | grep process-name

# Check file access control lists (ACL)
getfacl /path/to/file

# Find files with specific permissions
find /var/www -type f -perm 777

# Find files owned by specific user
find /home -user username

# Check sudo access
sudo -l
```

**What to Check in Logs:**
```bash
# Permission-related errors
journalctl | grep -i "permission denied"
grep -r "permission denied" /var/log/

# Sudo attempts
grep sudo /var/log/auth.log
journalctl -u sudo

# Failed authentication
grep "Failed" /var/log/auth.log
```

**Common Log Patterns:**
```
Permission denied
Operation not permitted
You do not have permission to access
sudo: unable to open /var/run/sudo: Permission denied
```

**Practice Scenario:**
```bash
# Create test file
echo "test" > /tmp/testfile.txt

# Remove all permissions
chmod 000 /tmp/testfile.txt

# Try to read
cat /tmp/testfile.txt
# Permission denied

# Check permissions
ls -l /tmp/testfile.txt
# Shows: ----------

# Fix permissions
chmod 644 /tmp/testfile.txt

# Verify
cat /tmp/testfile.txt

# Test ownership
sudo chown root:root /tmp/testfile.txt
cat /tmp/testfile.txt  # Still works (644 allows read)

# Try to modify
echo "more" >> /tmp/testfile.txt
# Permission denied (not owner)

# Fix
sudo chown $USER:$USER /tmp/testfile.txt
echo "more" >> /tmp/testfile.txt  # Works

# Clean up
rm /tmp/testfile.txt
```

**Real-World Fixes:**

```bash
# Scenario 1: Web application can't write to directory

# Check directory permissions
ls -ld /var/www/uploads
# Shows: drwxr-xr-x root root

# Check web server user
ps aux | grep nginx
# Shows: nginx user is 'www-data'

# Fix ownership
sudo chown -R www-data:www-data /var/www/uploads

# Fix permissions
sudo chmod -R 755 /var/www/uploads

# For upload directory, need write
sudo chmod 775 /var/www/uploads

# Verify
sudo -u www-data touch /var/www/uploads/test.txt
ls -l /var/www/uploads/test.txt

# Scenario 2: Script won't execute

# Try to run
./script.sh
# bash: ./script.sh: Permission denied

# Check permissions
ls -l script.sh
# Shows: -rw-r--r-- (no execute)

# Add execute permission
chmod +x script.sh

# Verify
ls -l script.sh
# Shows: -rwxr-xr-x

./script.sh  # Now works

# Scenario 3: Service can't access config file

# Service fails
sudo systemctl start myapp
# Job failed

# Check logs
sudo journalctl -u myapp -n 50
# Shows: Permission denied: '/etc/myapp/config.yml'

# Check file permissions
ls -l /etc/myapp/config.yml
# Shows: -rw------- root root

# Check service user
cat /etc/systemd/system/myapp.service | grep User
# Shows: User=myapp

# Fix: Add group read
sudo chgrp myapp /etc/myapp/config.yml
sudo chmod 640 /etc/myapp/config.yml

# Or add user to group
sudo usermod -aG myapp myapp

# Restart service
sudo systemctl start myapp
sudo systemctl status myapp
```

---

### 6. Service Management Issues

**Symptoms:**
- Service won't start
- Service crashes after starting
- Service stuck in activating state
- Service restarts continuously

**Diagnostic Commands:**
```bash
# Check service status
sudo systemctl status service-name

# View service logs
sudo journalctl -u service-name -n 50
sudo journalctl -u service-name -f  # Follow

# Check if service is enabled
sudo systemctl is-enabled service-name

# List all services
systemctl list-units --type=service
systemctl list-units --type=service --state=failed

# Check service dependencies
systemctl list-dependencies service-name

# View service configuration
systemctl cat service-name

# Check for errors
systemctl status service-name --no-pager

# Reload systemd daemon (after editing service file)
sudo systemctl daemon-reload
```

**What to Check in Logs:**
```bash
# Service-specific logs
sudo journalctl -u service-name --since "10 minutes ago"

# Boot logs
sudo journalctl -b

# Errors only
sudo journalctl -u service-name -p err

# With timestamps
sudo journalctl -u service-name -o short-precise
```

**Common Log Patterns:**
```
Failed to start service-name.service
service-name.service: Main process exited, code=exited, status=1/FAILURE
service-name.service: Failed with result 'exit-code'
Dependency failed for service-name.service
Job service-name.service/start timed out
```

**Practice Scenario:**
```bash
# Create test service
sudo cat > /etc/systemd/system/testapp.service << 'EOF'
[Unit]
Description=Test Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/sleep infinity
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
sudo systemctl daemon-reload

# Start service
sudo systemctl start testapp

# Check status
sudo systemctl status testapp

# Stop service
sudo systemctl stop testapp

# Remove service
sudo rm /etc/systemd/system/testapp.service
sudo systemctl daemon-reload
```

**Real-World Fixes:**

```bash
# Scenario 1: Service fails to start

# Check status
sudo systemctl status nginx
# Shows: Failed

# Check detailed logs
sudo journalctl -u nginx -n 50
# Shows: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# Find what's using port 80
sudo lsof -i :80
# Shows: apache2

# Fix: Stop conflicting service
sudo systemctl stop apache2
sudo systemctl start nginx

# Verify
sudo systemctl status nginx

# Scenario 2: Service keeps restarting

# Check status
sudo systemctl status myapp
# Shows: Active: activating (auto-restart)

# Check logs
sudo journalctl -u myapp -n 100
# Shows repeated: "Configuration file not found: /etc/myapp/config.yml"

# Create missing file
sudo touch /etc/myapp/config.yml
sudo chmod 644 /etc/myapp/config.yml

# Restart service
sudo systemctl restart myapp
sudo systemctl status myapp

# Scenario 3: Service stuck in activating

# Check status
sudo systemctl status myapp
# Shows: Active: activating (start) since ...

# Check what it's waiting for
systemctl list-dependencies myapp
# Shows dependency on database

# Check database service
sudo systemctl status postgresql
# Shows: inactive (dead)

# Start dependency first
sudo systemctl start postgresql
sudo systemctl start myapp

# Scenario 4: Permission issues with service

# Service fails
sudo systemctl start myapp
sudo journalctl -u myapp -n 20
# Shows: Permission denied: '/var/log/myapp/app.log'

# Check service user
grep User /etc/systemd/system/myapp.service
# Shows: User=myapp

# Fix log directory permissions
sudo chown -R myapp:myapp /var/log/myapp
sudo chmod 755 /var/log/myapp

# Restart
sudo systemctl restart myapp
```

---

## 🔍 Advanced Debugging Techniques

### System Call Tracing
```bash
# Trace what a command is doing
strace ls -la

# Trace a running process
strace -p <PID>

# Trace system calls for file operations
strace -e trace=open,read,write ls

# Trace network calls
strace -e trace=network nc -l 8080
```

### Process Debugging
```bash
# Monitor process in real-time
watch -n 1 'ps aux | grep myapp'

# Generate core dump for analysis
gcore <PID>

# Check process limits
cat /proc/<PID>/limits

# Check what files process has open
lsof -p <PID>

# Check process environment
cat /proc/<PID>/environ | tr '\0' '\n'
```

### Performance Analysis
```bash
# I/O statistics
iostat -x 1 5

# Network statistics
netstat -s
ss -s

# CPU per-core usage
mpstat -P ALL 1

# Detailed process monitoring
pidstat -p <PID> 1

# System activity report
sar -u 1 10  # CPU
sar -r 1 10  # Memory
sar -n DEV 1 10  # Network
```

---

## 📊 Systematic Troubleshooting Checklist

When a Linux system issue occurs, follow this order:

```bash
# 1. Check system health
uptime && free -h && df -h

# 2. Check for errors in system log
dmesg | tail -50
journalctl -xe | tail -50

# 3. Identify problematic processes
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10

# 4. Check network connectivity
ping -c 3 8.8.8.8
cat /etc/resolv.conf

# 5. Check service status
sudo systemctl status <service>

# 6. Review application logs
sudo journalctl -u <service> -n 100

# 7. Check disk I/O
iostat -x 1 3

# 8. Check for recent changes
journalctl --since "1 hour ago" -p err

# 9. Verify permissions
ls -la /path/to/problem/file

# 10. Test fix
# Apply fix, then verify all checks above
```

---

## 💡 Key Takeaways

1. **Check system health first** - uptime, free, df
2. **Logs are your friend** - journalctl and dmesg
3. **Process identification** - ps, top, htop
4. **Network basics** - ping, netstat, ss
5. **Permission verification** - ls -l, stat
6. **Service management** - systemctl
7. **Monitor resource usage** - CPU, memory, disk, network
8. **Trace system calls** - strace for deep debugging
9. **Check for recent changes** - journalctl timeline
10. **Document solutions** - Build your runbook

---

## 📚 Quick Reference Card

| Issue Type | First Command | Key Log Location |
|------------|---------------|------------------|
| High CPU | `top` or `htop` | journalctl -p err |
| High Memory | `free -h` | dmesg \| grep oom |
| Disk Full | `df -h` | /var/log/syslog |
| Network Down | `ip addr; ping 8.8.8.8` | journalctl -u NetworkManager |
| Permission Denied | `ls -l file; id` | /var/log/auth.log |
| Service Failed | `systemctl status service` | journalctl -u service |

---

**Remember**: Linux troubleshooting is systematic. Don't guess - check logs, verify state, test hypothesis, fix, and validate. The system tells you what's wrong if you know where to look.
