# Common Issues and Solutions

This guide covers the most frequently encountered problems when working with GServer-v2, organized by category with detailed solutions and prevention strategies.

## Build and Compilation Issues

### CMake Configuration Failures

#### Problem: CMake can't find dependencies
```bash
CMake Error: Could not find V8 installation
-- V8NPCSERVER requested but V8 not found
```

**Solution:**
```bash
# Install V8 dependencies
cd dependencies/
git submodule update --init --recursive
./build-v8-linux

# For Windows
nuget install v8-v142-x64 -version 7.4.288.26

# Verify V8 installation
ls dependencies/v8/out.gn/x64.release/
# Should contain libv8_*.a files
```

#### Problem: C++20 compiler requirements
```bash
CMake Error: C++ compiler does not support C++20
```

**Solution:**
```bash
# Update compiler (Ubuntu/Debian)
sudo apt update
sudo apt install gcc-10 g++-10

# Set as default
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-10 100

# Verify version
gcc --version  # Should be 10.x or higher
```

### Linking Errors

#### Problem: Undefined V8 symbols
```bash
undefined reference to `v8::Isolate::New(...)`
```

**Solution:**
```bash
# Ensure V8 is built correctly
cd dependencies/v8
gn gen out.gn/x64.release --args='is_debug=false target_cpu="x64"'
ninja -C out.gn/x64.release v8_monolith

# Check CMake finds the right libraries
cmake .. -DV8NPCSERVER=TRUE -DCMAKE_VERBOSE_MAKEFILE=ON
```

#### Problem: Missing system libraries
```bash
/usr/bin/ld: cannot find -lssl
/usr/bin/ld: cannot find -lz
```

**Solution:**
```bash
# Install development packages
sudo apt install libssl-dev zlib1g-dev

# For CentOS/RHEL
sudo yum install openssl-devel zlib-devel

# For Windows (vcpkg)
vcpkg install openssl zlib
```

## Server Startup Issues

### Port Binding Problems

#### Problem: Address already in use
```bash
[ERROR] Failed to bind to port 2001: Address already in use
```

**Solution:**
```bash
# Find what's using the port
netstat -tulpn | grep :2001
lsof -i :2001

# Kill the conflicting process
sudo kill -9 <PID>

# Or change the port in serveroptions.txt
port=2002
```

#### Problem: Permission denied for port binding
```bash
[ERROR] Failed to bind to port 2001: Permission denied
```

**Solution:**
```bash
# Run as root (not recommended)
sudo ./gserver-v2

# Better: Use a high port (>1024)
port=14001

# Or give capability to bind low ports
sudo setcap 'cap_net_bind_service=+ep' ./gserver-v2
```

### Configuration File Issues

#### Problem: Config file not found
```bash
[ERROR] Could not open serveroptions.txt
```

**Solution:**
```bash
# Specify full path
./gserver-v2 --config /full/path/to/serveroptions.txt

# Or create default config
cat > serveroptions.txt << 'EOF'
name=Default Server
port=2001
maxplayers=50
EOF
```

#### Problem: Invalid configuration syntax
```bash
[ERROR] Invalid option 'maxplayer' (did you mean 'maxplayers'?)
```

**Solution:**
```bash
# Check configuration syntax
grep -n "=" serveroptions.txt | head -10

# Common mistakes:
maxplayer=50     # Wrong: should be maxplayers
port = 2001      # Wrong: spaces around =
name="My Server" # Wrong: quotes not needed

# Correct format:
maxplayers=50
port=2001
name=My Server
```

## Client Connection Issues

### Authentication Failures

#### Problem: Client login rejected
```bash
[INFO] Player 'testuser' login rejected: Invalid account
```

**Solution:**
```bash
# Check account file exists
ls accounts/testuser.txt

# Create account file
cat > accounts/testuser.txt << 'EOF'
GRACC001
testuser
password123
0.0.0.0

community=Player
maxchars=1
EOF

# Check file permissions
chmod 644 accounts/testuser.txt
```

#### Problem: RC authentication fails
```bash
[ERROR] RC authentication failed for admin
```

**Solution:**
```bash
# Verify admin account in serveroptions.txt
staff_account=admin
staff_password=correct_password

# Or check accounts/admin.txt contains:
staffaccount

# Test RC connection
python3 << 'EOF'
import socket, struct, zlib
s = socket.socket()
s.connect(('localhost', 14922))
packet = chr(33) + chr(37) + "admin" + chr(44) + "correct_password" + "\n"
compressed = zlib.compress(packet.encode())
s.send(struct.pack('!H', len(compressed)) + compressed)
print("Response:", s.recv(1024))
EOF
```

### Protocol Mismatch Issues

#### Problem: Unsupported client version
```bash
[WARN] Client version 'GNW30000' not supported
```

**Solution:**
```bash
# Add version to allowed list in serveroptions.txt
allowedversions=1.41.1,2.22.122,3.0.0

# Or allow all versions (not recommended)
allowedversions=*

# Check client sends correct version string
tcpdump -i lo -A port 2001
```

#### Problem: Packet format errors
```bash
[ERROR] Invalid packet format from client
```

**Solution:**
```bash
# Enable packet debugging
loglevel=DEBUG
log_packets=true

# Common causes:
# 1. Wrong endianness (little vs big endian)
# 2. Missing compression for RC clients
# 3. Incorrect string length encoding

# Test with known-good client
telnet localhost 2001
# Should connect but not understand protocol
```

## Runtime and Performance Issues

### Memory Problems

#### Problem: Server crashes with out of memory
```bash
[ERROR] std::bad_alloc
terminate called after throwing an instance of 'std::bad_alloc'
```

**Solution:**
```bash
# Monitor memory usage
top -p $(pgrep gserver)

# Reduce cache sizes in serveroptions.txt
levelcache=10      # Reduce from default 32
scriptcache=false  # Disable if not needed

# Check for memory leaks
valgrind --tool=memcheck ./gserver-v2

# Increase system limits
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
```

#### Problem: High memory usage with NPCs
```bash
[WARN] V8 heap size exceeded limit
```

**Solution:**
```bash
# Increase V8 memory limits in code
// In CScriptEngine::Initialize()
v8::ResourceConstraints constraints;
constraints.set_max_old_space_size(256);  // MB
isolate->SetResourceConstraints(&constraints);

# Optimize NPC scripts
// Avoid memory leaks in JavaScript
function onTimer() {
    // Bad: creates new objects each time
    var enemies = findAllEnemies();
    
    // Good: reuse existing objects
    if (!this.enemyCache) this.enemyCache = [];
    updateEnemyCache();
}
```

### Script Execution Issues

#### Problem: JavaScript errors in NPCs
```bash
[ERROR] V8 Script Error: ReferenceError: player is not defined
```

**Solution:**
```bash
# Check script syntax
node --check npc_script.js

# Common NPC script issues:
function onPlayerTouches() {
    player.chat = "Hello";  // Correct: player object available
}

function onTimer() {
    player.chat = "Hello";  // Error: no player context in timer
    // Use: this.level.players[0].chat = "Hello";
}

# Enable script debugging
script_debug=true
script_stack_traces=true
```

#### Problem: Script timeouts
```bash
[WARN] Script execution timeout: npc001.js
```

**Solution:**
```bash
# Increase timeout in serveroptions.txt
script_timeout=5000  # milliseconds

# Optimize expensive operations
function onTimer() {
    // Bad: expensive operation every timer
    for (var i = 0; i < 10000; i++) {
        doExpensiveCalculation();
    }
    
    // Good: spread work over multiple timers
    if (!this.workIndex) this.workIndex = 0;
    for (var i = 0; i < 100; i++) {
        doExpensiveCalculation(this.workIndex++);
        if (this.workIndex >= 10000) this.workIndex = 0;
    }
}
```

### Network and Connectivity Issues

#### Problem: Clients randomly disconnect
```bash
[INFO] Player 'user123' disconnected: Connection lost
```

**Solution:**
```bash
# Check network stability
ping -c 10 server_ip

# Monitor socket errors
netstat -s | grep -i error

# Increase socket timeouts
socket_timeout=30000  # milliseconds
keepalive_interval=60 # seconds

# Check for firewall issues
sudo iptables -L -n
sudo ufw status
```

#### Problem: High latency/lag
```bash
[WARN] High ping detected: 2000ms
```

**Solution:**
```bash
# Monitor server load
uptime
iostat 1 5

# Check packet loss
mtr server_ip

# Optimize network settings
# In serveroptions.txt:
compression_level=1  # Reduce CPU usage
packet_batching=true # Combine small packets
nagle_disable=true   # Reduce latency

# Kernel network tuning
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf
sysctl -p
```

## Database and File System Issues

### Level Loading Problems

#### Problem: Level files not found
```bash
[ERROR] Could not load level 'start.graal'
```

**Solution:**
```bash
# Check level directory structure
ls -la levels/
# Should contain .graal files

# Verify file permissions
chmod 644 levels/*.graal

# Check level file format
head -5 levels/start.graal
# Should start with:
# GRALLEV01
# levelname
```

#### Problem: Corrupted level files
```bash
[ERROR] Invalid level format in 'corrupted.graal'
```

**Solution:**
```bash
# Restore from backup
cp backups/levels/corrupted.graal levels/

# Validate level file
grep -c "GRALLEV01" levels/corrupted.graal
# Should return 1

# Repair common issues
sed -i 's/\r$//' levels/corrupted.graal  # Remove Windows line endings
dos2unix levels/corrupted.graal          # Convert to Unix format

# Recreate level if necessary
cat > levels/fixed.graal << 'EOF'
GRALLEV01
fixed_level

BOARD
40404040404040404040404040404040404040404040404040404040404040404040404040404040
...
BOARDEND
EOF
```

### Account Management Issues

#### Problem: Account corruption
```bash
[ERROR] Invalid account format: accounts/player.txt
```

**Solution:**
```bash
# Check account file format
cat accounts/player.txt
# Should start with GRACC001

# Recreate account
cat > accounts/player.txt << 'EOF'
GRACC001
player
password123
127.0.0.1

community=Player
maxchars=3
attr1=value1
attr2=value2
EOF

# Fix permissions
chmod 600 accounts/player.txt  # Protect password
```

## Security Issues

### Authentication Bypass Attempts

#### Problem: Suspicious login attempts
```bash
[WARN] Failed login attempt from 192.168.1.100: admin/password123
[WARN] Failed login attempt from 192.168.1.100: admin/admin
```

**Solution:**
```bash
# Implement rate limiting
max_login_attempts=3
login_ban_time=300  # seconds

# Use strong passwords
staff_password=$(openssl rand -base64 32)

# Restrict RC access by IP
allowed_rc_ips=127.0.0.1,192.168.1.0/24

# Monitor authentication logs
grep -i "failed login" logs/server.log | tail -20
```

#### Problem: Unauthorized admin access
```bash
[WARN] Non-staff account attempting admin commands
```

**Solution:**
```bash
# Audit admin accounts
grep -r "staffaccount" accounts/
grep -r "community=Admin" accounts/

# Check for privilege escalation
audit_admin_commands=true
log_admin_actions=true

# Remove unauthorized staff flags
sed -i '/staffaccount/d' accounts/suspicious_user.txt
```

## Monitoring and Diagnostics

### Debug Logging Setup

#### Enable Comprehensive Logging
```bash
# In serveroptions.txt
loglevel=DEBUG
logfile=logs/debug.log
log_packets=true
log_script_errors=true
log_admin_actions=true
audit_file_access=true

# Rotate logs to prevent disk space issues
logrotate_size=100MB
logrotate_count=5
```

#### Real-time Monitoring Script
```bash
#!/bin/bash
# monitor.sh - Real-time server monitoring

tail -f logs/server.log | while read line; do
    echo "$line"
    
    # Alert on critical errors
    if echo "$line" | grep -qi "error\|crash\|segfault"; then
        echo "ALERT: Critical error detected!" | mail -s "Server Alert" admin@example.com
    fi
    
    # Monitor player count
    if echo "$line" | grep -q "player.*connected"; then
        players=$(grep -c "connected" logs/server.log)
        echo "Current players: $players"
    fi
done
```

### Performance Profiling

#### CPU Usage Analysis
```bash
# Profile with perf
perf record -g ./gserver-v2
perf report

# Monitor with htop
htop -p $(pgrep gserver)

# Check for high-CPU scripts
grep "script.*timeout\|v8.*error" logs/server.log
```

#### Memory Usage Analysis
```bash
# Memory profiling with Valgrind
valgrind --tool=massif ./gserver-v2
ms_print massif.out.*

# Monitor memory growth
while true; do
    ps -o pid,vsz,rss,comm -p $(pgrep gserver)
    sleep 60
done

# Check for script memory leaks
grep "v8.*heap\|javascript.*memory" logs/server.log
```

## Recovery Procedures

### Emergency Server Recovery

#### Graceful Restart
```bash
#!/bin/bash
# graceful_restart.sh

echo "Initiating graceful restart..."

# Send SIGTERM for graceful shutdown
kill -TERM $(pgrep gserver)

# Wait up to 30 seconds
for i in {1..30}; do
    if ! pgrep gserver > /dev/null; then
        break
    fi
    sleep 1
done

# Force kill if still running
if pgrep gserver > /dev/null; then
    echo "Force killing server..."
    kill -KILL $(pgrep gserver)
fi

# Restart server
echo "Restarting server..."
nohup ./gserver-v2 --config config/serveroptions.txt > logs/restart.log 2>&1 &

echo "Server restarted with PID: $(pgrep gserver)"
```

#### Disaster Recovery
```bash
#!/bin/bash
# disaster_recovery.sh

echo "Starting disaster recovery..."

# Stop any running instances
killall gserver-v2 2>/dev/null

# Restore from backup
BACKUP_DATE=$(date -d "1 day ago" +%Y%m%d)
if [ -f "backups/server-backup-$BACKUP_DATE.tar.gz" ]; then
    echo "Restoring from backup: $BACKUP_DATE"
    tar -xzf "backups/server-backup-$BACKUP_DATE.tar.gz"
else
    echo "No recent backup found!"
    exit 1
fi

# Verify critical files
if [ ! -f "config/serveroptions.txt" ]; then
    echo "Critical files missing after restore!"
    exit 1
fi

# Start in recovery mode
./gserver-v2 --config config/serveroptions.txt --recovery-mode

echo "Recovery complete"
```

---

*This troubleshooting guide covers the most common issues encountered when running GServer-v2. For issues not covered here, check the server logs, enable debug logging, and consider posting to the community forums with detailed error information.*