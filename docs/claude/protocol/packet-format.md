# Packet Format Specification

This document details the binary packet formats used by GServer-v2 for all client types. Understanding these formats is essential for developing compatible clients and debugging network communication.

## General Packet Structure

All GServer-v2 packets follow a common structure with client-specific variations:

```
┌─────────────┬─────────────┬─────────────────────┐
│   Header    │    Type     │        Data         │
│ (2-4 bytes) │  (1 byte)   │    (variable)       │
└─────────────┴─────────────┴─────────────────────┘
```

### Header Variations by Client Type

| Client Type | Header Size | Byte Order | Format | Notes |
|-------------|-------------|------------|--------|-------|
| CLIENT/CLIENT3 | 2 bytes | Little-endian | `uint16` | Standard game clients |
| RC | 2 bytes | Big-endian | `uint16` | Network byte order |
| NPCSERVER | 4 bytes | Platform | `uint32` | Extended for large scripts |
| WEB | 2 bytes | Little-endian | `uint16` | Web-compatible format |
| NC | 2 bytes | Little-endian | `uint16` | Development tools |

## Compression and Encoding

### Compression Rules
- **RC Clients**: **MANDATORY** zlib compression on all packets
- **Game Clients**: Optional compression based on capabilities negotiation
- **NPC Server**: Conditional compression for large script payloads
- **NC Clients**: Optional compression for large level data

### String Encoding
All strings use a length-prefix system with offset encoding:

```cpp
// Encoding formula
uint8_t encode_length(size_t actual_length) {
    if (actual_length < 96) {
        return actual_length + 32;  // Normal range: 32-127
    } else {
        // Extended encoding for lengths 96-223
        return actual_length + 32 - 256;  // Wraps to negative values
    }
}

// Decoding formula  
size_t decode_length(uint8_t encoded_length) {
    if (encoded_length >= 32) {
        return encoded_length - 32;  // Normal range
    } else {
        return encoded_length + 224;  // Extended range
    }
}
```

### Usage Example
```cpp
void writeString(const std::string& str) {
    uint8_t encoded_len = encode_length(str.length());
    write_byte(encoded_len);
    write_data(str.data(), str.length());
}

std::string readString() {
    uint8_t encoded_len = read_byte();
    size_t actual_len = decode_length(encoded_len);
    return read_string(actual_len);
}
```

## Client-Specific Packet Formats

### Game Client Packets (CLIENT/CLIENT3)

#### Basic Format
```
┌─────────────┬─────────────┬─────────────────────┐
│   Length    │    Type     │        Data         │
│  (2 bytes)  │  (1 byte)   │    (variable)       │
│   LE uint16 │  type + 32  │  String-encoded     │
└─────────────┴─────────────┴─────────────────────┘
```

#### Type Encoding
Game client packet types are encoded with an offset:
```cpp
uint8_t encoded_type = actual_packet_type + 32;
```

#### Example: PLI_LOGIN Packet
```cpp
struct PLI_LOGIN {
    uint16_t length;           // Little-endian packet length
    uint8_t type;              // PLI_LOGIN (0) + 32 = 32
    
    // Account name
    uint8_t account_len;       // Length + 32
    char account[];            // UTF-8 string
    
    // Password
    uint8_t password_len;      // Length + 32
    char password[];           // UTF-8 string
    
    // Protocol version
    uint8_t version_len;       // Length + 32
    char version[];            // e.g., "GNW22122"
    
    // Optional additional data
    uint8_t extra_data[];
};
```

#### Binary Layout Example
```
Packet: PLI_LOGIN with account="test", password="pass", version="GNW22122"

Hex dump:
00 1A              // Length: 26 bytes (little-endian)
20                 // Type: PLI_LOGIN (0+32)
26 74 65 73 74     // Account: length=6+32=38(0x26), "test"  
24 70 61 73 73     // Password: length=4+32=36(0x24), "pass"
28 47 4E 57 32 32 31 32 32  // Version: length=8+32=40(0x28), "GNW22122"
```

### RC Client Packets

#### Basic Format
```
┌─────────────┬─────────────┬─────────────────────┐
│   Length    │             │        Data         │
│  (2 bytes)  │   Compressed zlib stream          │
│   BE uint16 │   Contains type + payload         │
└─────────────┴─────────────────────────────────────┘
```

#### RC Login Packet
```cpp
struct RC_LOGIN {
    uint16_t compressed_length;  // Big-endian length of compressed data
    
    // Compressed payload contains:
    uint8_t magic;              // Magic byte: 33
    uint8_t account_len;        // Account length + 32
    char account[];             // Account name
    uint8_t password_len;       // Password length + 32  
    char password[];            // Password
    uint8_t terminator;         // Newline: '\n'
};
```

#### RC Response Format
```cpp
struct RC_RESPONSE {
    uint16_t compressed_length;  // Big-endian
    
    // Compressed payload:
    uint8_t type;               // Response type (raw byte, no offset)
    uint8_t data[];             // Response-specific data
};
```

#### RC Server List Format
The RC server list response uses a special encoding:
```cpp
struct RC_SERVER_LIST {
    uint8_t type;               // 32 (0x20)
    uint8_t header[2];          // Fixed header bytes
    
    // For each server:
    struct ServerEntry {
        uint8_t server_id;      // Server identifier
        
        // 8 fields with length-encoded strings:
        struct Field {
            uint8_t length;     // Encoded length (see string encoding)
            char data[];        // Field data
        } fields[8];            // name, language, description, url, version, players, ip, port
    } servers[];
};
```

### NPC Server Packets

#### Basic Format
```
┌─────────────┬─────────────┬─────────────────────┐
│   Length    │    Type     │        Data         │
│  (4 bytes)  │  (2 bytes)  │    (variable)       │
│  Platform   │  Extended   │  Script/Command     │
└─────────────┴─────────────┴─────────────────────┘
```

#### Script Upload Packet
```cpp
struct NPC_SCRIPT_UPLOAD {
    uint32_t length;            // Total packet length
    uint16_t type;              // NPC_SCRIPT_UPLOAD
    
    uint8_t npc_name_len;       // NPC name length + 32
    char npc_name[];            // Target NPC name
    
    uint32_t script_length;     // JavaScript code length
    char script_code[];         // V8 JavaScript source
    
    uint8_t flags;              // Execution flags
};
```

## Packet Type Definitions

### Game Client Input (PLI_)
```cpp
enum PLI_PacketTypes {
    PLI_LOGIN = 0,              // 0x20 (0+32)
    PLI_SERVERLISTPING = 1,     // 0x21 (1+32)
    PLI_MOVE = 2,               // 0x22 (2+32)
    PLI_CHAT = 3,               // 0x23 (3+32)
    PLI_SHOOT = 4,              // 0x24 (4+32)
    PLI_HIT = 5,                // 0x25 (5+32)
    PLI_LEVEL = 6,              // 0x26 (6+32)
    PLI_WANTLEVEL = 7,          // 0x27 (7+32)
    PLI_REQUESTLEVELWARP = 8,   // 0x28 (8+32)
    PLI_BOARDMODIFY = 9,        // 0x29 (9+32)
    PLI_PLAYERPROPS = 10,       // 0x2A (10+32)
    PLI_UNKNOWN11 = 11,         // 0x2B
    PLI_CARRYOBJECT = 12,       // 0x2C
    PLI_THROWCARRIED = 13,      // 0x2D
    PLI_UNKNOWN14 = 14,         // 0x2E
    PLI_UNKNOWN15 = 15,         // 0x2F
    PLI_UNKNOWN16 = 16,         // 0x30
    PLI_UNKNOWN17 = 17,         // 0x31
    PLI_UNKNOWN18 = 18,         // 0x32
    PLI_WEAPONFIRE = 19,        // 0x33
    PLI_UPDATEFILE = 20,        // 0x34
    PLI_ADJACENTLEVEL = 21,     // 0x35
    PLI_HITOBJECTS = 22,        // 0x36
    PLI_GETFILE = 23,           // 0x37
    PLI_UNKNOWN24 = 24,         // 0x38
    PLI_SENDTORC = 25,          // 0x39
    PLI_UNKNOWN26 = 26,         // 0x3A
    PLI_UNKNOWN27 = 27,         // 0x3B
    PLI_UNKNOWN28 = 28,         // 0x3C
    PLI_PROCESSLIST = 29,       // 0x3D
    PLI_UNKNOWN30 = 30,         // 0x3E
    PLI_CLIENTVERSION = 31,     // 0x3F
    PLI_UNKNOWN32 = 32,         // 0x40
    PLI_LEVELWARP = 33,         // 0x41
    PLI_SENDPM = 34,            // 0x42
    PLI_SENDTOSERVER = 35,      // 0x43
    PLI_UNKNOWN36 = 36,         // 0x44
    PLI_LANGUAGE = 37,          // 0x45
    PLI_TRIGGERACTION = 38,     // 0x46
    PLI_MAPINFO = 39,           // 0x47
    PLI_UPDATESCRIPT = 40,      // 0x48
    PLI_UNKNOWN41 = 41,         // 0x49
    PLI_NPCSERVERQUERY = 42,    // 0x4A
    PLI_WEAPONLISTGET = 43,     // 0x4B
    PLI_REQUESTSAVE = 44,       // 0x4C
    PLI_UNKNOWN45 = 45,         // 0x4D
    PLI_REQUESTTEXT = 46,       // 0x4E
    PLI_UNKNOWN47 = 47,         // 0x4F
    PLI_UNKNOWN48 = 48,         // 0x50
    PLI_UNKNOWN49 = 49,         // 0x51
    PLI_SERVERWARP = 50,        // 0x52
    PLI_UNKNOWN51 = 51,         // 0x53
    PLI_COUNT = 52              // Total count
};
```

### Server Output (PLO_)
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
    PLO_ISLEADER = 10,          // Leadership status
    PLO_BOMBADD = 11,           // Bomb placement
    PLO_BOMBDEL = 12,           // Bomb removal
    PLO_TOALL = 13,             // Global message
    PLO_PLAYERWARP = 14,        // Player teleport
    PLO_WARPFAILED = 15,        // Warp failure
    PLO_DISCMESSAGE = 16,       // Disconnect message
    PLO_HORSEADD = 17,          // Horse appearance
    PLO_HORSEDEL = 18,          // Horse removal
    PLO_ARROWADD = 19,          // Arrow projectile
    PLO_FIRESPY = 20,           // Firespy effect
    PLO_THROWCARRIED = 21,      // Throw object
    PLO_ITEMADD = 22,           // Item appearance
    PLO_ITEMDEL = 23,           // Item removal
    PLO_NPCMOVED = 24,          // NPC movement
    PLO_SIGNATURE = 25,         // Server signature
    PLO_NPCACTION = 26,         // NPC action
    PLO_BADDYHURT = 27,         // Enemy damage
    PLO_FLAGSET = 28,           // Flag setting
    PLO_NPCDEL = 29,            // NPC removal
    PLO_FILESENDFAILED = 30,    // File transfer failure
    PLO_FLAGDEL = 31,           // Flag deletion
    PLO_SHOWIMG = 32,           // Image display
    PLO_NPCWEAPONADD = 33,      // NPC weapon
    PLO_NPCWEAPONDEL = 34,      // NPC weapon removal
    PLO_RC_ADMINMESSAGE = 35,   // RC admin message
    PLO_EXPLOSION = 36,         // Explosion effect
    PLO_PRIVATEMESSAGE = 37,    // Private message
    PLO_PUSHAWAY = 38,          // Push effect
    PLO_LEVELMODTIME = 39,      // Level modification time
    PLO_HURTPLAYER = 40,        // Player damage
    PLO_STARTMESSAGE = 41,      // Startup message
    PLO_NEWWORLDTIME = 42,      // World time update
    PLO_DEFAULTWEAPON = 43,     // Default weapon
    PLO_HASNPCSERVER = 44,      // NPC server status
    PLO_FILEUPTODATE = 45,      // File version check
    PLO_HITOBJECTS = 46,        // Object collision
    PLO_STAFFGUILDS = 47,       // Staff guild list
    PLO_TRIGGERACTION = 48,     // Trigger activation
    PLO_PLAYERWARP2 = 49,       // Extended warp
    // ... many more (200+ total)
};
```

## Advanced Packet Features

### Player Properties Encoding
Player properties use a special sub-packet format:
```cpp
struct PLO_PLAYERPROPS {
    uint8_t type;              // PLO_PLAYERPROPS (8+32)
    uint16_t player_id;        // Target player ID
    
    // Property updates (multiple allowed per packet)
    struct PropertyUpdate {
        uint8_t property_id;   // See PlayerProps enum
        uint8_t value_length;  // Length + 32
        char value_data[];     // Property-specific data
    } updates[];
};
```

### Level Board Encoding
Level tiles are encoded efficiently:
```cpp
struct PLO_LEVELBOARD {
    uint8_t type;              // PLO_LEVELBOARD (0+32)
    uint16_t board_width;      // Usually 64
    uint16_t board_height;     // Usually 64
    
    // Tile data (width * height * 2 bytes per tile)
    struct Tile {
        uint16_t tile_id;      // Tile graphic ID
    } tiles[];
};
```

### Compression Detection
To determine if a packet is compressed:
```cpp
bool isCompressed(const PacketBuffer& packet, ClientType client_type) {
    switch (client_type) {
        case ClientType::RC:
            return true;  // Always compressed
            
        case ClientType::CLIENT:
        case ClientType::CLIENT3:
            // Check capability flags during handshake
            return clientSupportsCompression;
            
        case ClientType::NPCSERVER:
            // Compression for large scripts
            return packet.size() > COMPRESSION_THRESHOLD;
            
        default:
            return false;
    }
}
```

## Debugging and Analysis

### Packet Logging Format
```cpp
struct PacketLog {
    uint64_t timestamp;        // Unix timestamp
    uint32_t player_id;        // Source/destination player
    PacketDirection direction; // INCOMING/OUTGOING
    ClientType client_type;    // Client type
    uint16_t packet_type;      // Packet type ID
    uint16_t packet_length;    // Total length
    bool compressed;           // Compression flag
    uint8_t data[];           // Raw packet data
};
```

### Wireshark Dissector
For network analysis, a Wireshark dissector can decode GServer packets:
```lua
-- graal.lua - Wireshark dissector for GServer-v2
graal_protocol = Proto("Graal", "Graal Online Protocol")

function graal_protocol.dissector(buffer, pinfo, tree)
    local length = buffer:len()
    if length == 0 then return end
    
    pinfo.cols.protocol = graal_protocol.name
    
    local subtree = tree:add(graal_protocol, buffer(), "Graal Packet")
    
    -- Parse header based on port (heuristic for client type)
    local client_port = pinfo.dst_port
    if client_port == 14922 then
        -- RC client (big-endian)
        parse_rc_packet(buffer, subtree)
    else
        -- Game client (little-endian)
        parse_game_packet(buffer, subtree)
    end
end
```

### Common Debugging Issues

1. **Endianness Problems**
   ```cpp
   // Wrong: assuming little-endian on all platforms
   uint16_t length = *(uint16_t*)buffer;
   
   // Correct: explicit endianness
   uint16_t length = buffer[0] | (buffer[1] << 8);  // Little-endian
   uint16_t length = (buffer[0] << 8) | buffer[1];  // Big-endian
   ```

2. **String Length Decoding**
   ```cpp
   // Handle both normal and extended length encoding
   size_t decode_string_length(uint8_t encoded) {
       if (encoded >= 32) {
           return encoded - 32;     // Normal: 0-95
       } else {
           return encoded + 224;    // Extended: 96-255
       }
   }
   ```

3. **Compression Detection**
   ```cpp
   bool try_decompress(const std::vector<uint8_t>& data) {
       try {
           z_stream stream = {};
           if (inflateInit(&stream) != Z_OK) return false;
           // ... decompression logic
           return true;
       } catch (...) {
           return false;  // Not compressed
       }
   }
   ```

---

*This packet format specification provides the detailed binary layout needed to implement compatible clients or debugging tools. The format variations between client types reflect the different needs and capabilities of each client category.*