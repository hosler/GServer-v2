# GServer-v2 Protocol Specification

The GServer-v2 protocol is a sophisticated binary communication system that supports multiple client types with different authentication flows, packet formats, and feature sets.

## Overview

### Protocol Characteristics
- **Binary Format**: All packets use binary encoding for efficiency
- **Length-Prefixed**: Packets start with a length header
- **Type-Based**: Each packet has a type identifier
- **Compressed**: Optional zlib compression (required for RC clients)
- **Versioned**: Protocol version negotiation during handshake

### Client Type Detection
The server determines client type based on the initial login packet format:

```cpp
// From TServer analysis
enum class ClientType {
    CLIENT,      // Standard game client (v1.41.1)
    CLIENT3,     // Modern game client (v2.22+)
    WEB,         // Web-based client
    RC,          // Remote Control (admin)
    NPCSERVER,   // NPC server connection
    NC           // Level editor/dev tools
};
```

## Packet Structure

### Basic Packet Format
```
┌─────────────┬─────────────┬─────────────────────┐
│   Length    │    Type     │        Data         │
│  (2 bytes)  │  (1 byte)   │    (variable)       │
└─────────────┴─────────────┴─────────────────────┘
```

### Length Header Variations
Different client types use different length encoding:

| Client Type | Length Format | Byte Order | Notes |
|-------------|---------------|------------|-------|
| CLIENT/CLIENT3 | `uint16` | Little-endian | Standard game clients |
| RC | `uint16` | Big-endian | RC protocol uses network byte order |
| NPCSERVER | `uint16` | Platform-dependent | Server-to-server |
| WEB | `uint32` | Little-endian | Extended for large payloads |

### Compression
- **RC Clients**: **REQUIRED** - All packets must be zlib compressed
- **Game Clients**: **OPTIONAL** - Compression flag in capabilities
- **NPC Server**: **CONDITIONAL** - Based on negotiation

## Authentication Flows

### 1. Game Client Authentication

```
Client                           Server
  │                                │
  ├─ PLI_LOGIN ────────────────────┤
  │  ├─ Account name               │
  │  ├─ Password                   │
  │  ├─ Protocol version           │
  │  └─ Client capabilities        │
  │                                │
  │  ┌─ PLO_SIGNATURE ─────────────┤
  │  │  ├─ Server name             │
  │  │  ├─ Welcome message         │
  │  │  └─ Server capabilities     │
  │                                │
  ├─ PLI_REQUESTLEVELWARP ─────────┤
  │  └─ Starting level             │
  │                                │
  │  ┌─ PLO_LEVELNAME ─────────────┤
  │  │  └─ Level information       │
  │                                │
  │  ┌─ PLO_LEVELBOARD ────────────┤
  │  │  └─ Level tile data         │
```

#### PLI_LOGIN Packet Structure
```cpp
struct PLI_LOGIN {
    uint8_t packet_type;        // PLI_LOGIN (0x20 + type)
    uint8_t account_len;        // Length + 32
    char account[];             // Account name
    uint8_t password_len;       // Length + 32  
    char password[];            // Password
    uint8_t version_len;        // Length + 32
    char version[];             // Protocol version (e.g., "GNW22122")
    // Optional: additional capabilities
};
```

### 2. RC Client Authentication

```
RC Client                        Server
  │                                │
  ├─ RC Login Packet ──────────────┤
  │  ├─ Magic byte (33)            │
  │  ├─ Account (length + 32)      │
  │  ├─ Password (length + 32)     │
  │  └─ Newline terminator         │
  │  [All zlib compressed]         │
  │                                │
  │  ┌─ Server List Response ──────┤
  │  │  ├─ Type (32)               │
  │  │  ├─ Header bytes            │
  │  │  └─ Server information      │
  │  [All zlib compressed]         │
```

#### RC Login Packet
```python
# Python implementation example
def create_rc_login(username, password):
    packet = chr(33)  # Magic identifier
    packet += chr(len(username) + 32) + username
    packet += chr(len(password) + 32) + password  
    packet += "\n"  # Terminator
    
    compressed = zlib.compress(packet.encode('utf-8'))
    header = struct.pack('!H', len(compressed))  # Big-endian
    return header + compressed
```

### 3. NPC Server Authentication

```cpp
// NPC Server connects as a special client
struct NPCSERVER_LOGIN {
    uint8_t packet_type;        // PLI_NPCSERVERCONNECTION
    uint8_t server_name_len;
    char server_name[];
    uint8_t verification_code_len;
    char verification_code[];
};
```

## Packet Types

### Game Client Packets (PLI_/PLO_)

#### Player Input (PLI_)
```cpp
enum PLI_PacketTypes {
    PLI_LOGIN = 0,              // Initial authentication
    PLI_SERVERLISTPING = 1,     // Server list request
    PLI_MOVE = 2,               // Player movement
    PLI_CHAT = 3,               // Chat message
    PLI_SHOOT = 4,              // Weapon/projectile
    PLI_HIT = 5,                // Hit detection
    PLI_LEVEL = 6,              // Level change request
    PLI_WANTLEVEL = 7,          // Request level data
    PLI_REQUESTLEVELWARP = 8,   // Warp to level
    PLI_BOARDMODIFY = 9,        // Tile modification
    PLI_PLAYERPROPS = 10,       // Property updates
    // ... many more (158+ total)
};
```

#### Server Output (PLO_)
```cpp
enum PLO_PacketTypes {
    PLO_LEVELBOARD = 0,         // Level tile data
    PLO_LEVELLINK = 1,          // Level connections
    PLO_BADDYPROPS = 2,         // Enemy properties
    PLO_NPCPROPS = 3,           // NPC properties
    PLO_LEVELCHEST = 4,         // Chest contents
    PLO_LEVELSIGN = 5,          // Sign text
    PLO_LEVELNAME = 6,          // Level information
    PLO_BOARDMODIFY = 7,        // Tile changes
    PLO_PLAYERPROPS = 8,        // Player properties
    PLO_OTHERPLPROPS = 9,       // Other player updates
    // ... many more
};
```

### RC Client Packets

#### RC Commands
```cpp
enum RCI_PacketTypes {
    RCI_OPLPROPS = 8,           // Online player properties
    RCI_OSPLPROPS = 9,          // Offline player properties  
    RCI_DISCMSG = 16,           // Disconnect message
    RCI_SIGNATURE = 25,         // Server signature
    RCI_SERVERFLAGS = 28,       // Server configuration
    RCI_GRAALTIME = 42,         // Server time sync
    RCI_ADDPLAYER = 55,         // Player join notification
    RCI_DELPLAYER = 56,         // Player leave notification
    RCI_CHATLOG = 74,           // Chat log access
    // ... administrative commands
};
```

### Data Encoding

#### String Encoding
Strings use a length-prefix system with offset encoding:
```cpp
// Length encoding formula
if (raw_length < 128) {
    encoded_length = raw_length + 32;  // Normal encoding
} else {
    encoded_length = raw_length - 224; // Extended encoding  
}

// Usage example
void encode_string(const std::string& str) {
    uint8_t len = (str.length() < 96) ? 
                  str.length() + 32 : 
                  str.length() + 32 - 256;
    write_byte(len);
    write_data(str.data(), str.length());
}
```

#### Player Properties
Players have 83+ different properties that can be updated:
```cpp
enum PlayerProps {
    NICKNAME = 0,       // Display name
    MAXPOWER = 1,       // Max health
    CURPOWER = 2,       // Current health
    RUPEESCOUNT = 3,    // Currency
    ARROWSCOUNT = 4,    // Ammunition
    BOMBSCOUNT = 5,     // Bombs
    GLOVEPOWER = 6,     // Glove strength
    BOMBPOWER = 7,      // Bomb power
    SWORDPOWER = 8,     // Sword damage
    SHIELDPOWER = 9,    // Shield defense
    PLAYERANI = 10,     // Animation frame
    HEADGIF = 11,       // Head image
    CURCHAT = 12,       // Current chat text
    PLAYERCOLORS = 13,  // Color palette
    PLAYERID = 14,      // Unique ID
    PLAYERX = 15,       // X coordinate
    PLAYERY = 16,       // Y coordinate
    PLAYERSPRITE = 17,  // Sprite image
    STATUS = 18,        // Player status flags
    CURLEVEL = 20,      // Current level
    // ... 60+ more properties
};
```

## Protocol Versions

### Version String Format
```
Protocol versions follow the pattern: [TYPE][VERSION]
- "GNW22122" - Game client v2.22.122
- "GSERV025" - Server v0.25  
- "GNRCU110" - RC client v1.10
- "NPCSERV1" - NPC server v1
```

### Capability Negotiation
During login, clients and servers exchange capability flags:
```cpp
enum Capabilities {
    CAP_COMPRESSION = 0x01,     // Supports zlib compression
    CAP_ENCRYPTION = 0x02,      // Supports encryption
    CAP_LARGELEVELS = 0x04,     // Large level support
    CAP_EXTENDEDCHAT = 0x08,    // Extended chat features
    CAP_FILETRANSFER = 0x10,    // File upload/download
    CAP_V8SCRIPTS = 0x20,       // V8 JavaScript support
};
```

## Error Handling

### Common Error Codes
```cpp
enum ErrorCodes {
    ERR_INVALID_ACCOUNT = 1,    // Bad username/password
    ERR_BANNED_IP = 2,          // IP address banned
    ERR_SERVER_FULL = 3,        // Max players reached
    ERR_INVALID_VERSION = 4,    // Unsupported protocol
    ERR_MAINTENANCE = 5,        // Server maintenance mode
    ERR_INVALID_LEVEL = 6,      // Level doesn't exist
    ERR_PERMISSION_DENIED = 7,  // Insufficient privileges
};
```

### Error Response Format
```cpp
struct PLO_ERROR {
    uint8_t packet_type;        // PLO_ERROR
    uint8_t error_code;         // Error identifier
    uint8_t message_len;        // Length + 32
    char message[];             // Human-readable error
};
```

## Advanced Features

### Scripting Integration
The server supports multiple scripting systems:

1. **V8 JavaScript Engine**
   - Modern ECMAScript support
   - Sandboxed execution environment
   - Hot-reload capabilities
   - Performance optimization

2. **GraalScript 2.0**
   - Legacy compatibility
   - Bytecode compilation and caching
   - Event-driven programming model

### Multi-Server Communication
GServer-v2 can link multiple server instances:
```
Server A ←→ Server B ←→ Server C
    ↓           ↓           ↓
 Players    Players    Players
```

Players can seamlessly move between connected servers using `PLI_SERVERWARP` packets.

## Security Considerations

### Input Validation
- All packet lengths are bounds-checked
- String inputs are sanitized
- Script execution is sandboxed
- File system access is virtualized

### Authentication
- Password hashing with salt
- Account lockout after failed attempts
- IP-based access control
- Admin privilege separation

### Network Security
- Optional packet encryption (Gen 1-5 schemes)
- DDoS protection and rate limiting
- Secure file transfer protocols
- Audit logging for administrative actions

---

*This protocol specification covers the complete communication system used by GServer-v2. Understanding these details is essential for developing compatible clients, tools, and extensions.*