# GServer-v2 Quick Start Guide

This guide will get you up and running with GServer-v2 in the shortest time possible, covering basic setup, configuration, and testing with different client types.

## Prerequisites

### System Requirements
- **Operating System**: Linux (Ubuntu/Debian recommended), Windows, or macOS
- **Compiler**: GCC 9+ or MSVC 2019+ with C++20 support
- **Memory**: 2GB RAM minimum, 4GB+ recommended
- **Disk Space**: 500MB for source + build, 1GB+ for game content

### Dependencies
- **CMake 3.16+**: Build system
- **V8 JavaScript Engine**: For NPC scripting (optional but recommended)
- **OpenSSL**: For secure connections (optional)
- **zlib**: For packet compression

## Step 1: Build the Server

### Linux/macOS Build
```bash
# Clone the repository
git clone <repository-url> GServer-v2
cd GServer-v2

# Install dependencies (Ubuntu/Debian)
sudo apt update
sudo apt install build-essential cmake git zlib1g-dev libssl-dev

# Build V8 dependencies (optional but recommended)
cd dependencies/
./build-v8-linux
cd ..

# Configure and build
mkdir build && cd build
cmake .. -DV8NPCSERVER=TRUE -DTESTS=ON
make -j$(nproc)

# Verify build
./gserver-v2 --version
```

### Windows Build (MSVC)
```bash
# Install dependencies via vcpkg or NuGet
mkdir packages && cd packages
nuget install v8-v142-x64 -version 7.4.288.26
cd ..

# Configure with Visual Studio
mkdir build && cd build
cmake .. -G "Visual Studio 16 2019" -A x64 -DV8NPCSERVER=TRUE
cmake --build . --config Release

# Copy V8 runtime files
robocopy ..\packages\v8.redist-v142-x64.7.4.288.26\lib\Release ..\bin\ *.dll
robocopy ..\packages\v8.redist-v142-x64.7.4.288.26\lib\Release ..\bin\ *.bin
```

### Docker Build (Alternative)
```bash
# Build Docker image
docker build -t gserver-v2 .

# Run container
docker run -p 2001:2001 -p 14922:14922 -v $(pwd)/config:/app/config gserver-v2
```

## Step 2: Basic Configuration

### Create Server Directory Structure
```bash
mkdir -p my-graal-server/{accounts,levels,weapons,logs,config}
cd my-graal-server
```

### Essential Configuration Files

#### `config/serveroptions.txt`
```ini
# Basic server settings
name=My First Graal Server
description=A test server for learning GServer-v2
maxplayers=50
port=2001

# List server configuration (optional)
listserver=listserver.graal.in:14900
listed=false

# Administrative account
staff_account=admin
staff_password=change_me_123
noexpiration=true

# Performance settings
levelcache=20
scriptcache=true
compressionlevel=6

# Security settings
onlystaff=false
allowedversions=1.41.1,2.22.122,3.0.0

# Logging
logfile=logs/server.log
loglevel=INFO
```

#### `config/servers.txt` (Multi-server setup)
```
# Format: IP:PORT:NAME
127.0.0.1:2001:Main Server
# 127.0.0.1:2002:Test Server
# remote.server.com:2001:Production
```

#### `accounts/admin.txt` (Administrative account)
```
GRACC001
admin
change_me_123
127.0.0.1

# Permissions
staffaccount
community=Admin
maxchars=3
#banned
```

### Create a Test Level

#### `levels/start.graal`
```
GRALLEV01
testlevel

# 64x64 tile map (simplified - normally much larger)
BOARD
40404040404040404040404040404040404040404040404040404040404040404040404040404040
40404040404040404040404040404040404040404040404040404040404040404040404040404040
40424242424242424242424242424242424242424242424242424242424242424242424242424040
40420000000000000000000000000000000000000000000000000000000000000000000000004240
40420000000000000000000000000000000000000000000000000000000000000000000000004240
# ... more tiles ...
BOARDEND

# NPCs
NPC 32.5 32.5 npc001.png
NPCEND

# Level script
SCRIPT
function onPlayerEnters() {
  player.chat = "Welcome to the test level!";
}

function onCreated() {
  echo("Test level loaded successfully");
}
SCRIPTEND
```

## Step 3: Start the Server

### Basic Startup
```bash
# From the build directory
./gserver-v2 --config ../my-graal-server/config/serveroptions.txt

# Expected output:
# [INFO] GServer-v2 v2.x.x starting...
# [INFO] Loading configuration from serveroptions.txt
# [INFO] Binding to port 2001
# [INFO] Loading levels from levels/
# [INFO] V8 JavaScript engine initialized
# [INFO] Server ready - accepting connections
```

### Verify Server is Running
```bash
# Check if server is listening
netstat -ln | grep 2001

# Test basic connectivity
telnet localhost 2001
```

## Step 4: Test with Different Clients

### Test with Game Client (Simulated)

#### Python Test Client
```python
#!/usr/bin/env python3
"""Simple test client for GServer-v2"""

import socket
import struct
import time

def test_game_client():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('localhost', 2001))
    
    # Send PLI_LOGIN packet
    account = "testuser"
    password = "testpass"
    version = "GNW22122"
    
    packet = b''
    packet += bytes([0x20])  # PLI_LOGIN (0 + 32)
    packet += bytes([len(account) + 32])
    packet += account.encode('utf-8')
    packet += bytes([len(password) + 32])
    packet += password.encode('utf-8')
    packet += bytes([len(version) + 32])
    packet += version.encode('utf-8')
    
    # Send with little-endian length header
    header = struct.pack('<H', len(packet))
    sock.send(header + packet)
    
    # Receive response
    response_header = sock.recv(2)
    response_length = struct.unpack('<H', response_header)[0]
    response_data = sock.recv(response_length)
    
    print(f"Server response: {response_data[:50]}...")
    sock.close()

if __name__ == "__main__":
    test_game_client()
```

### Test with RC Client

#### Using pyRemoteControl
```bash
# From the pyRemoteControl directory (if available)
python main_simple.py

# Or test with basic RC connection
python3 << 'EOF'
import socket
import struct
import zlib

def test_rc_client():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('localhost', 14922))  # RC port
    
    # Create RC login packet
    username = "admin"
    password = "change_me_123"
    
    packet = chr(33)  # Magic byte
    packet += chr(len(username) + 32) + username
    packet += chr(len(password) + 32) + password
    packet += "\n"
    
    # Compress and send
    compressed = zlib.compress(packet.encode('utf-8'))
    header = struct.pack('!H', len(compressed))
    sock.send(header + compressed)
    
    # Receive response
    response_header = sock.recv(2)
    response_length = struct.unpack('!H', response_header)[0]
    response_data = sock.recv(response_length)
    
    # Decompress
    decompressed = zlib.decompress(response_data)
    print(f"RC response: {decompressed[:100]}...")
    
    sock.close()

test_rc_client()
EOF
```

### Test with Web Client
```html
<!DOCTYPE html>
<html>
<head>
    <title>GServer-v2 Web Test</title>
</head>
<body>
    <div id="status">Connecting...</div>
    <div id="output"></div>
    
    <script>
    // WebSocket connection to GServer-v2 (requires WebSocket proxy)
    const ws = new WebSocket('ws://localhost:8080/graal');
    
    ws.onopen = function() {
        document.getElementById('status').textContent = 'Connected!';
        
        // Send login packet (would need proper encoding)
        const loginData = {
            type: 'login',
            account: 'webuser',
            password: 'webpass',
            version: 'GWEB1.0'
        };
        ws.send(JSON.stringify(loginData));
    };
    
    ws.onmessage = function(event) {
        const output = document.getElementById('output');
        output.innerHTML += '<div>' + event.data + '</div>';
    };
    
    ws.onerror = function(error) {
        document.getElementById('status').textContent = 'Error: ' + error;
    };
    </script>
</body>
</html>
```

## Step 5: Add Content

### Create Your First NPC

#### `levels/npc001.txt`
```javascript
// V8 JavaScript NPC script
function onCreated() {
    // Initialize NPC
    this.name = "Tutorial Guide";
    this.image = "npc001.png";
    this.chat = "Hello! I'm here to help new players.";
}

function onPlayerTouches() {
    // Interact with player
    player.chat = "Welcome to my server! Type 'help' for commands.";
    player.health = player.maxhealth;  // Heal player
    
    // Give starting items
    if (!player.hasFlag("tutorial_complete")) {
        player.giveRupees(100);
        player.giveArrows(30);
        player.setFlag("tutorial_complete", true);
        player.chat = "Here are some starting items!";
    }
}

function onPlayerSays(message) {
    switch(message.toLowerCase()) {
        case "help":
            player.chat = "Available commands: time, heal, save";
            break;
            
        case "time":
            player.chat = "Server time: " + getServerTime();
            break;
            
        case "heal":
            player.health = player.maxhealth;
            player.chat = "You have been healed!";
            break;
            
        case "save":
            player.save();
            player.chat = "Your progress has been saved.";
            break;
            
        default:
            player.chat = "I don't understand '" + message + "'. Try 'help'.";
    }
}
```

### Create a Simple Weapon

#### `weapons/sword.txt`
```javascript
// Basic sword weapon
function onCreated() {
    this.name = "Training Sword";
    this.image = "sword001.png";
    this.defaultpower = 1;
}

function onPlayerHits(target) {
    if (target.type == "player") {
        // PvP combat
        target.health -= this.power;
        target.chat = "Ouch! You hit me for " + this.power + " damage!";
        
        if (target.health <= 0) {
            target.health = target.maxhealth;  // Respawn
            target.x = 32;  // Reset position
            target.y = 32;
            target.chat = "You have been defeated!";
        }
    } else if (target.type == "npc") {
        // NPC combat
        target.health -= this.power;
        if (target.health <= 0) {
            target.destroy();
            player.giveRupees(10);  // Reward
        }
    }
}
```

## Step 6: Monitor and Manage

### Server Logs
```bash
# Monitor server output
tail -f logs/server.log

# Watch for errors
grep -i error logs/server.log

# Monitor connections
grep -i "player.*connected\|player.*disconnected" logs/server.log
```

### Performance Monitoring
```bash
# Check server process
ps aux | grep gserver

# Monitor memory usage
top -p $(pgrep gserver)

# Network connections
netstat -anp | grep :2001
```

### Administrative Commands
```bash
# Graceful shutdown
kill -TERM $(pgrep gserver)

# Reload configuration (if supported)
kill -HUP $(pgrep gserver)

# Emergency stop
kill -KILL $(pgrep gserver)
```

## Common First-Time Issues

### Build Problems
```bash
# Missing dependencies
sudo apt install build-essential cmake zlib1g-dev

# V8 build issues
cd dependencies/
git submodule update --init --recursive
./build-v8-linux --clean

# Permission issues
chmod +x build-v8-linux
```

### Configuration Issues
```bash
# Port already in use
netstat -tulpn | grep :2001
# Kill conflicting process or change port

# File permissions
chmod 644 config/serveroptions.txt
chmod 600 accounts/*.txt  # Protect account files
```

### Connection Issues
```bash
# Firewall blocking
sudo ufw allow 2001  # Game clients
sudo ufw allow 14922 # RC clients

# Binding issues
# Edit serveroptions.txt:
bindip=0.0.0.0  # Listen on all interfaces
# or
bindip=127.0.0.1  # Localhost only
```

### Runtime Issues
```bash
# Script errors
grep -i "script error\|javascript" logs/server.log

# Memory issues
# Reduce cache sizes in serveroptions.txt:
levelcache=10
scriptcache=false

# Performance issues
# Enable profiling:
enable_profiling=true
profile_interval=30
```

## Next Steps

1. **[Multi-Client Setup](multi-client.md)** - Support different client types
2. **[NPC Development](npc-development.md)** - Create interactive NPCs
3. **[RC Administration](rc-administration.md)** - Manage your server
4. **[Level Design](../development/level-design.md)** - Create custom levels
5. **[Performance Tuning](../troubleshooting/performance.md)** - Optimize for scale

## Backup and Maintenance

### Daily Backup Script
```bash
#!/bin/bash
# backup.sh - Daily server backup

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backup/graal-server"
SERVER_DIR="/path/to/my-graal-server"

mkdir -p "$BACKUP_DIR"

# Backup critical files
tar -czf "$BACKUP_DIR/server-backup-$DATE.tar.gz" \
    "$SERVER_DIR/accounts" \
    "$SERVER_DIR/levels" \
    "$SERVER_DIR/weapons" \
    "$SERVER_DIR/config"

# Keep only last 7 days
find "$BACKUP_DIR" -name "server-backup-*.tar.gz" -mtime +7 -delete

echo "Backup completed: server-backup-$DATE.tar.gz"
```

### Update Script
```bash
#!/bin/bash
# update.sh - Update server to latest version

# Stop server gracefully
echo "Stopping server..."
kill -TERM $(pgrep gserver)
sleep 5

# Backup current installation
cp -r build build.backup.$(date +%Y%m%d)

# Pull latest changes
git pull origin main

# Rebuild
cd build
make -j$(nproc)

# Restart server
echo "Starting server..."
nohup ./gserver-v2 --config ../my-graal-server/config/serveroptions.txt > ../logs/server.out 2>&1 &

echo "Server updated and restarted"
```

---

*This quick start guide covers the essential steps to get GServer-v2 running. Once you have a basic server working, explore the advanced features and customization options covered in the other documentation sections.*