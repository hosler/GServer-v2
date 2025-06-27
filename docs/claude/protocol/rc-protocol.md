# Remote Control (RC) Protocol

The RC (Remote Control) protocol is a specialized administrative interface for GServer-v2 that allows real-time server monitoring, configuration, and management. This protocol is used by administrative tools and the pyRemoteControl application.

## Protocol Overview

### Key Characteristics
- **Mandatory Compression**: All packets MUST use zlib compression
- **Big-Endian Headers**: Network byte order for length prefixes
- **Binary Protocol**: Efficient binary packet format
- **Administrative Focus**: Enhanced commands for server management
- **Session-Based**: Persistent connection with authentication

### Port Configuration
- **Default RC Port**: 14922 (different from game client port)
- **List Server Port**: 14922 (shared with RC for server discovery)
- **Multi-Server**: Each server instance can have its own RC port

## Authentication Flow

### 1. Initial Connection
```
RC Client                    List Server (14922)
    │                              │
    ├─ RC Login Packet ────────────┤
    │  [Compressed]                │
    │  ├─ Magic Byte (33)          │
    │  ├─ Username (len+32)        │
    │  ├─ Password (len+32)        │
    │  └─ Newline terminator       │
    │                              │
    │  ┌─ Server List Response ────┤
    │  │  [Compressed]             │
    │  │  ├─ Type (32)             │
    │  │  ├─ Header bytes          │
    │  │  └─ Server entries        │
    │                              │
    ├─ Server Selection ───────────┤
    │                              │
    │  ┌─ Connection Info ─────────┤
```

### 2. Direct Server Connection
```
RC Client                    Game Server (2001+)
    │                              │
    ├─ RC Authentication ──────────┤
    │  [Similar to list server]    │
    │                              │
    │  ┌─ Authentication Success ──┤
    │  │  ├─ Session established   │
    │  │  └─ RC commands enabled   │
    │                              │
    ├─ RC Commands ────────────────┤
    │  ├─ Player monitoring        │
    │  ├─ Server configuration     │
    │  └─ Administrative actions   │
```

## Packet Format

### Basic RC Packet Structure
```
┌─────────────┬─────────────────────────────────────┐
│   Length    │        Compressed Payload           │
│  (2 bytes)  │         (zlib stream)               │
│ Big-Endian  │  ┌─────────────┬─────────────────┐  │
│   uint16    │  │    Type     │      Data       │  │
│             │  │  (1 byte)   │   (variable)    │  │
│             │  └─────────────┴─────────────────┘  │
└─────────────┴─────────────────────────────────────┘
```

### Implementation Example
```cpp
// Sending an RC packet
void sendRCPacket(uint8_t type, const void* data, size_t data_length) {
    // Build uncompressed payload
    std::vector<uint8_t> payload;
    payload.push_back(type);
    payload.insert(payload.end(), 
                   static_cast<const uint8_t*>(data),
                   static_cast<const uint8_t*>(data) + data_length);
    
    // Compress payload
    std::vector<uint8_t> compressed = zlibCompress(payload);
    
    // Send with big-endian length header
    uint16_t length = htons(compressed.size());
    send(&length, sizeof(length));
    send(compressed.data(), compressed.size());
}

// Receiving an RC packet
std::vector<uint8_t> receiveRCPacket() {
    // Read big-endian length
    uint16_t compressed_length;
    recv(&compressed_length, sizeof(compressed_length));
    compressed_length = ntohs(compressed_length);
    
    // Read compressed data
    std::vector<uint8_t> compressed(compressed_length);
    recv(compressed.data(), compressed_length);
    
    // Decompress
    return zlibDecompress(compressed);
}
```

## Authentication Packets

### RC Login Request
```cpp
struct RC_LOGIN_REQUEST {
    // Packet structure (before compression):
    uint8_t magic;              // Always 33 (0x21)
    uint8_t username_length;    // strlen(username) + 32
    char username[];            // Account name
    uint8_t password_length;    // strlen(password) + 32
    char password[];            // Account password
    uint8_t terminator;         // Always '\n' (0x0A)
};
```

### Python Implementation
```python
def create_rc_login(username, password):
    # Build login packet
    packet = chr(33)  # Magic byte
    packet += chr(len(username) + 32) + username
    packet += chr(len(password) + 32) + password
    packet += "\n"  # Terminator
    
    # Compress
    compressed = zlib.compress(packet.encode('utf-8'))
    
    # Add big-endian length header
    header = struct.pack('!H', len(compressed))
    
    return header + compressed
```

### C++ Implementation
```cpp
std::vector<uint8_t> createRCLogin(const std::string& username, 
                                   const std::string& password) {
    std::string packet;
    packet += static_cast<char>(33);  // Magic byte
    packet += static_cast<char>(username.length() + 32);
    packet += username;
    packet += static_cast<char>(password.length() + 32);
    packet += password;
    packet += '\n';  // Terminator
    
    // Compress the packet
    auto compressed = zlibCompress(packet);
    
    // Prepend big-endian length
    std::vector<uint8_t> result;
    uint16_t length = htons(compressed.size());
    result.insert(result.end(), 
                  reinterpret_cast<uint8_t*>(&length),
                  reinterpret_cast<uint8_t*>(&length) + 2);
    result.insert(result.end(), compressed.begin(), compressed.end());
    
    return result;
}
```

## Server List Response

### List Server Response Format
When connecting to the list server (port 14922), the response contains available servers:

```cpp
struct RC_SERVER_LIST_RESPONSE {
    uint8_t type;               // 32 (0x20)
    uint8_t header_byte1;       // Usually 34 (0x22)
    uint8_t header_byte2;       // Usually 40 (0x28)
    
    // Server entries follow
    ServerEntry servers[];
};

struct ServerEntry {
    uint8_t server_id;          // Server identifier
    
    // 8 fields with length-encoded strings
    EncodedString fields[8];    // [name, language, description, url, version, players, ip, port]
};

struct EncodedString {
    uint8_t length;             // Encoded length (see string encoding rules)
    char data[];                // String data
};
```

### String Length Encoding
The RC protocol uses the same string encoding as game clients:
```cpp
// Encoding: actual_length -> wire_length
uint8_t encode_length(size_t actual_length) {
    if (actual_length < 96) {
        return actual_length + 32;      // Range: 32-127
    } else {
        return actual_length + 32 - 256; // Wraps to 0-31 for lengths 96-223
    }
}

// Decoding: wire_length -> actual_length
size_t decode_length(uint8_t wire_length) {
    if (wire_length >= 32) {
        return wire_length - 32;        // Normal range
    } else {
        return wire_length + 224;       // Extended range: 224-255
    }
}
```

### Parsing Server List (Python)
```python
def parse_server_list(packet_data):
    """Parse RC server list response"""
    servers = []
    
    # Skip first 3 bytes (type + 2 header bytes)
    data = packet_data[3:]
    pos = 0
    
    while pos < len(data) - 1:
        # Read server ID
        if pos >= len(data):
            break
        server_id = struct.unpack('b', data[pos:pos+1])[0]
        pos += 1
        
        # Read 8 fields
        server_fields = []
        for field_num in range(8):
            if pos >= len(data):
                break
                
            # Read encoded length
            length_byte = struct.unpack('b', data[pos:pos+1])[0]
            pos += 1
            
            # Decode actual length
            if length_byte < 0:
                actual_length = length_byte + 224
            else:
                actual_length = length_byte - 32
                
            if actual_length <= 0 or pos + actual_length > len(data):
                break
                
            # Read field data
            field_data = data[pos:pos + actual_length]
            server_fields.append(field_data)
            pos += actual_length
        
        # Extract server information
        if len(server_fields) >= 8:
            server_info = {
                'id': server_id,
                'name': server_fields[0].decode('utf-8', errors='replace'),
                'language': server_fields[1].decode('utf-8', errors='replace'),
                'description': server_fields[2].decode('utf-8', errors='replace'),
                'url': server_fields[3].decode('utf-8', errors='replace'),
                'version': server_fields[4].decode('utf-8', errors='replace'),
                'players': server_fields[5].decode('utf-8', errors='replace'),
                'ip': server_fields[6].decode('utf-8', errors='replace'),
                'port': server_fields[7].decode('utf-8', errors='replace')
            }
            servers.append(server_info)
    
    return servers
```

## RC Command Packets

### Command Types
```cpp
enum RC_CommandTypes {
    RCI_OPLPROPS = 8,           // Online player properties
    RCI_OSPLPROPS = 9,          // Offline player properties
    RCI_DISCMSG = 16,           // Disconnect message
    RCI_SIGNATURE = 25,         // Server signature
    RCI_SERVERFLAGS = 28,       // Server configuration flags
    RCI_GRAALTIME = 42,         // Server time synchronization
    RCI_ADDPLAYER = 55,         // Player join notification
    RCI_DELPLAYER = 56,         // Player leave notification
    RCI_OPENRIGHT = 62,         // Open rights dialog
    RCI_OPENCMMTS = 63,         // Open comments dialog
    RCI_OPENATTRS = 72,         // Open attributes dialog
    RCI_OPENACCTS = 73,         // Open accounts dialog
    RCI_CHATLOG = 74,           // Chat log access
    RCI_GETSVROPTS = 76,        // Get server options
    RCI_GETFLDRCFG = 77,        // Get folder configuration
};
```

### Player Monitoring Commands
```cpp
// Request online player list
struct RCI_OPLPROPS_REQUEST {
    uint8_t type;               // RCI_OPLPROPS (8)
    // No additional data needed
};

// Response contains player information
struct RCI_OPLPROPS_RESPONSE {
    uint8_t type;               // RCI_OPLPROPS (8)
    uint16_t player_count;      // Number of online players
    
    struct PlayerInfo {
        uint16_t player_id;     // Unique player ID
        uint8_t nickname_len;   // Length + 32
        char nickname[];        // Player name
        uint8_t level_len;      // Length + 32
        char level[];           // Current level
        uint32_t login_time;    // Login timestamp
        uint16_t x, y;          // Position coordinates
        // ... additional properties
    } players[];
};
```

### Chat Commands
```cpp
// Send chat message as RC
struct RC_CHAT_SEND {
    uint8_t type;               // Custom chat type (usually 111)
    uint8_t message_len;        // Length + 32
    char message[];             // Chat message text
};

// Receive chat log
struct RC_CHAT_LOG {
    uint8_t type;               // RCI_CHATLOG (74)
    uint32_t entry_count;       // Number of log entries
    
    struct ChatEntry {
        uint32_t timestamp;     // Unix timestamp
        uint8_t sender_len;     // Length + 32
        char sender[];          // Sender name
        uint8_t message_len;    // Length + 32
        char message[];         // Message text
        uint8_t channel;        // Chat channel (0=say, 1=yell, etc.)
    } entries[];
};
```

### Server Configuration
```cpp
// Get server options
struct RCI_GETSVROPTS_REQUEST {
    uint8_t type;               // RCI_GETSVROPTS (76)
};

struct RCI_GETSVROPTS_RESPONSE {
    uint8_t type;               // RCI_GETSVROPTS (76)
    
    // Server configuration as key-value pairs
    struct ConfigOption {
        uint8_t key_len;        // Length + 32
        char key[];             // Option name
        uint8_t value_len;      // Length + 32
        char value[];           // Option value
    } options[];
};
```

## Advanced RC Features

### Real-Time Monitoring
RC clients can receive real-time updates about server events:

```cpp
// Player join notification
struct RCI_ADDPLAYER {
    uint8_t type;               // RCI_ADDPLAYER (55)
    uint16_t player_id;         // New player ID
    uint8_t nickname_len;       // Length + 32
    char nickname[];            // Player name
    uint32_t login_time;        // Login timestamp
    uint8_t ip_len;             // Length + 32
    char ip_address[];          // Client IP address
};

// Player leave notification
struct RCI_DELPLAYER {
    uint8_t type;               // RCI_DELPLAYER (56)
    uint16_t player_id;         // Leaving player ID
    uint32_t logout_time;       // Logout timestamp
    uint8_t reason;             // Disconnect reason
};
```

### Administrative Actions
```cpp
// Kick player
struct RC_KICK_PLAYER {
    uint8_t type;               // Custom admin command
    uint16_t player_id;         // Target player
    uint8_t reason_len;         // Length + 32
    char reason[];              // Kick reason
};

// Change server settings
struct RC_SET_OPTION {
    uint8_t type;               // Custom admin command
    uint8_t option_len;         // Length + 32
    char option_name[];         // Setting name
    uint8_t value_len;          // Length + 32
    char option_value[];        // New value
};
```

## Security Considerations

### Authentication Security
- **Credential Verification**: RC accounts require special administrative privileges
- **Session Management**: RC sessions are tracked and can be forcibly terminated
- **IP Restrictions**: RC access can be limited to specific IP addresses
- **Audit Logging**: All RC actions are logged for security auditing

### Command Restrictions
```cpp
enum RC_PermissionLevel {
    RC_MONITOR_ONLY = 1,        // Read-only access
    RC_CHAT_ADMIN = 2,          // Chat moderation
    RC_PLAYER_ADMIN = 3,        // Player management
    RC_LEVEL_ADMIN = 4,         // Level editing
    RC_SERVER_ADMIN = 5,        // Server configuration
    RC_SUPER_ADMIN = 6          // Full access
};
```

### Rate Limiting
```cpp
struct RC_RateLimit {
    uint32_t commands_per_minute;   // Command rate limit
    uint32_t bytes_per_second;      // Bandwidth limit
    uint32_t max_concurrent;        // Max concurrent RC connections
};
```

## Troubleshooting RC Connections

### Common Issues

1. **Compression Errors**
   ```cpp
   // Always verify compression is working
   bool testCompression(const std::vector<uint8_t>& data) {
       try {
           auto compressed = zlibCompress(data);
           auto decompressed = zlibDecompress(compressed);
           return decompressed == data;
       } catch (...) {
           return false;
       }
   }
   ```

2. **Endianness Problems**
   ```cpp
   // RC uses big-endian (network byte order)
   uint16_t length = ntohs(received_length);  // Convert from network
   uint16_t send_length = htons(packet_size); // Convert to network
   ```

3. **String Encoding Issues**
   ```cpp
   // Handle both UTF-8 and legacy encodings
   std::string decodeString(const std::vector<uint8_t>& data) {
       try {
           return std::string(data.begin(), data.end());  // UTF-8
       } catch (...) {
           // Fallback to Latin-1 for legacy compatibility
           return decodeLatin1(data);
       }
   }
   ```

### Debugging Tools

1. **Packet Capture**
   ```bash
   # Capture RC traffic
   tcpdump -i any -s 0 -w rc_traffic.pcap port 14922
   ```

2. **Compression Testing**
   ```python
   import zlib
   
   def test_rc_compression(packet_data):
       try:
           compressed = zlib.compress(packet_data)
           decompressed = zlib.decompress(compressed)
           return decompressed == packet_data
       except:
           return False
   ```

3. **Protocol Validation**
   ```cpp
   bool validateRCPacket(const std::vector<uint8_t>& packet) {
       if (packet.size() < 3) return false;  // Minimum size
       
       // Verify compression
       try {
           auto decompressed = zlibDecompress(packet);
           return decompressed.size() > 0;
       } catch (...) {
           return false;
       }
   }
   ```

---

*The RC protocol provides powerful administrative capabilities for GServer-v2. Understanding its unique requirements (mandatory compression, big-endian headers, and specialized packet formats) is essential for developing compatible RC tools and clients.*