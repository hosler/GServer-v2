# GServer-v2 Architecture Overview

GServer-v2 is a sophisticated C++ MMO server designed with a modular, event-driven architecture that supports multiple client types, advanced scripting, and scalable performance.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GServer-v2                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   TServer   │  │ ScriptEngine│  │ FileSystem  │              │
│  │   (Core)    │  │    (V8)     │  │ (Virtual)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   TPlayer   │  │    TLevel   │  │    TNPC     │              │
│  │ (Clients)   │  │  (Worlds)   │  │ (Entities)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Network    │  │   Database  │  │   Logger    │              │
│  │ (Sockets)   │  │ (Accounts)  │  │  (Events)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 🧠 [TServer - Central Orchestrator](core-components.md#tserver)

**Purpose**: The heart of GServer-v2, managing all subsystems and client connections.

**Key Responsibilities**:
- **Client Connection Management**: Accept and route different client types
- **Event Loop**: Process network events, timers, and callbacks
- **Resource Coordination**: Manage levels, players, NPCs, and scripts
- **Configuration Management**: Handle server settings and multi-server setup
- **Security**: Authentication, permissions, and input validation

**Architecture Pattern**: Event-driven with thread pool for blocking operations

```cpp
class TServer {
    // Core systems
    std::unique_ptr<CScriptEngine> scriptEngine;
    std::unique_ptr<TFileSystem> fileSystem;
    std::map<std::string, TLevel*> levelList;
    std::vector<TPlayer*> playerList;
    
    // Network management
    CSocket listenerSocket;
    std::vector<CSocket> clientSockets;
    
    // Event processing
    void processEvents();
    void handleNewConnection(CSocket& client);
    void handleClientData(TPlayer* player);
    void handleTimer(int timerID);
};
```

### 👥 [TPlayer - Client Connections](core-components.md#tplayer)

**Purpose**: Represents individual client connections with their state and capabilities.

**Client Type Handling**:
```cpp
class TPlayer {
    ClientType type;            // CLIENT, RC, NPCSERVER, etc.
    PermissionLevel permissions;
    std::string account;
    
    // Game state (for game clients)
    TLevel* currentLevel;
    PlayerProps properties;     // 83+ different properties
    Inventory inventory;
    
    // Network state
    CSocket socket;
    bool authenticated;
    CompressionState compression;
    
    // Protocol handling
    void handlePacket(const PacketBuffer& packet);
    void sendPacket(PacketType type, const void* data, size_t length);
};
```

**Key Features**:
- **Multi-Protocol Support**: Handles different packet formats per client type
- **State Synchronization**: Manages player properties and game state
- **Permission System**: Role-based access control for different operations
- **Bandwidth Optimization**: Compression and selective updates

### 🌍 [TLevel - Game Worlds](core-components.md#tlevel)

**Purpose**: Represents individual game levels/maps with their content and logic.

**Level Components**:
```cpp
class TLevel {
    // Terrain and layout
    TileMap board;              // 2D tile array (64x64 default)
    std::vector<LevelLink> links; // Connections to other levels
    
    // Interactive content
    std::vector<TNPC*> npcs;    // Level NPCs
    std::vector<TLevelItem*> items; // Dropped items
    std::vector<TLevelSign*> signs; // Sign posts
    std::vector<TLevelChest*> chests; // Treasure chests
    
    // Players and events
    std::vector<TPlayer*> players; // Current occupants
    EventHandler eventSystem;   // Level-specific events
    
    // Scripting
    std::string levelScript;    // Level initialization script
    std::map<std::string, std::string> triggers; // Event triggers
};
```

**Level Management**:
- **Dynamic Loading**: Levels loaded on-demand as players enter
- **Hot Reloading**: Level changes applied without server restart
- **Multi-Layer Support**: Background, collision, and overlay layers
- **Event System**: Triggers for player actions and timed events

### 🤖 [TNPC - Interactive Entities](core-components.md#tnpc)

**Purpose**: Scripted NPCs with advanced AI and interaction capabilities.

**Three-Tier NPC System**:

1. **Level NPCs**: Basic NPCs attached to specific levels
2. **Database NPCs**: Shared NPCs with persistent state
3. **Class NPCs**: Template-based NPCs with inheritance

```cpp
class TNPC {
    // Identity and appearance
    std::string name;
    std::string image;          // Sprite filename
    NPCType type;              // LEVELNPC, DBNPC, CLASSNPC
    
    // Position and movement
    float x, y;
    Direction facing;
    MovementType movement;      // FIXED, RANDOM, SCRIPTED
    
    // Scripting
    std::string script;         // V8 JavaScript code
    v8::Persistent<v8::Context> scriptContext;
    std::map<std::string, v8::Persistent<v8::Function>> eventHandlers;
    
    // Game logic
    void update(float deltaTime);
    void handlePlayerInteraction(TPlayer* player);
    void executeScript(const std::string& code);
};
```

**Scripting Capabilities**:
- **V8 JavaScript Engine**: Modern ECMAScript support
- **Event-Driven**: onCreated, onPlayerEnters, onPlayerTouches, etc.
- **API Access**: Full server API access for advanced NPCs
- **Performance Isolation**: Script timeouts and resource limits

## Advanced Systems

### 🔧 [Scripting Engine](scripting.md)

**V8 JavaScript Integration**:
```cpp
class CScriptEngine {
    v8::Isolate* isolate;
    v8::Persistent<v8::Context> globalContext;
    
    // Script management
    std::map<std::string, CompiledScript> compiledScripts;
    ScriptCache cache;
    
    // API bindings (30+ object types)
    void bindServerAPI();       // Server, Player, Level APIs
    void bindUtilityAPI();      // Math, String, File utilities
    void bindGameAPI();         // Weapons, Items, Combat
    
    // Execution control
    void setTimeout(int ms);    // Script timeout limits
    void setMemoryLimit(size_t bytes); // Memory restrictions
};
```

**Bound Object Types**:
- **Server**: Global server control and information
- **Player**: Player manipulation and queries  
- **Level**: Level modification and content management
- **NPC**: NPC control and interaction
- **Weapon**: Combat and weapon systems
- **Database**: Persistent data storage
- **File**: Secure file system access
- **Math**: Mathematical utilities
- **String**: Text processing functions

### 🌐 [Network Layer](networking.md)

**Multi-Protocol Socket Management**:
```cpp
class NetworkManager {
    // Socket management
    std::vector<CSocket> activeSockets;
    SocketSelector selector;    // epoll/kqueue wrapper
    
    // Protocol handlers
    std::map<ClientType, PacketProcessor*> processors;
    CompressionManager compression;
    EncryptionManager encryption;
    
    // Event processing
    void processNetworkEvents();
    void handleNewConnection();
    void handleClientData(CSocket& socket);
    void handleDisconnection(CSocket& socket);
};
```

**Key Features**:
- **Asynchronous I/O**: Non-blocking socket operations
- **Protocol Multiplexing**: Handle multiple client types simultaneously
- **Compression**: Optional zlib compression per client
- **Encryption**: Multiple encryption schemes (Gen 1-5)
- **Flow Control**: Bandwidth limiting and prioritization

### 💾 [Data Management](data-management.md)

**File System Architecture**:
```cpp
class TFileSystem {
    // Virtual file system
    std::map<std::string, VirtualDirectory> directories;
    SecurityManager security;   // Sandboxed access
    
    // Content types
    void loadLevels();          // .graal level files
    void loadAccounts();        // Player account database
    void loadWeapons();         // Weapon scripts and configs
    void loadImages();          // Sprite and image assets
    
    // Hot reloading
    FileWatcher watcher;        // Detect file changes
    void reloadChangedFiles();
};
```

**Security Model**:
- **Sandboxed Access**: Scripts cannot access arbitrary files
- **Permission-Based**: Different access levels per client type
- **Audit Logging**: Track all file system operations
- **Backup Integration**: Automated backups of critical data

## Performance Characteristics

### Threading Model
```
Main Thread (Event Loop)
├── Network I/O (non-blocking)
├── Timer Processing
└── Client Message Handling

Script Thread Pool
├── V8 JavaScript Execution
├── NPC AI Processing
└── Background Tasks

File I/O Thread
├── Level Loading/Saving
├── Asset Management
└── Database Operations
```

### Memory Management
- **Object Pooling**: Reuse frequently allocated objects
- **Smart Pointers**: RAII for automatic cleanup
- **Script Isolation**: Separate memory spaces for scripts
- **Level Streaming**: Load/unload levels based on player presence

### Scalability Features
- **Multi-Server Support**: Link multiple GServer instances
- **Load Balancing**: Distribute players across servers
- **Resource Limits**: Per-client and per-script resource caps
- **Monitoring**: Real-time performance metrics

## Configuration System

### Server Options
```ini
# serveroptions.txt
name=My Graal Server
description=A custom Graal server
maxplayers=200
listserver=listserver.graal.in:14900

# Security settings
staff_account=admin
staff_password=secure_password
noexpiration=true

# Performance tuning
scriptcache=true
levelcache=32
compressionlevel=6
```

### Multi-Server Setup
```
# servers.txt
127.0.0.1:2001:Main Server
127.0.0.1:2002:Development Server
remote.server.com:2001:Production Server
```

## Development Workflow

### Building the Server
```bash
# Configure with CMake
mkdir build && cd build
cmake .. -DV8NPCSERVER=TRUE -DTESTS=ON

# Build
make -j$(nproc)

# Run tests
ctest --output-on-failure
```

### Adding New Features
1. **Define Protocol**: Add new packet types and structures
2. **Implement Client Handler**: Add processing logic in TPlayer
3. **Update Script API**: Expose new functionality to scripts
4. **Add Tests**: Unit and integration tests
5. **Document Changes**: Update protocol and API documentation

### Debugging Tools
- **Packet Logger**: Capture and analyze network traffic
- **Script Debugger**: Step through V8 JavaScript execution
- **Memory Profiler**: Track memory usage and leaks
- **Performance Monitor**: Real-time server metrics

## Extension Points

### Custom Packet Types
```cpp
// Add to PacketTypes.h
enum CustomPackets {
    PLI_CUSTOM_COMMAND = 200,
    PLO_CUSTOM_RESPONSE = 200
};

// Implement handler in TPlayer
void TPlayer::handleCustomPacket(const PacketBuffer& packet) {
    // Custom logic here
}
```

### Script API Extensions
```cpp
// Add new bindings to CScriptEngine
void CScriptEngine::bindCustomAPI() {
    v8::Local<v8::ObjectTemplate> custom = v8::ObjectTemplate::New(isolate);
    custom->Set(v8::String::NewFromUtf8(isolate, "myFunction"),
                v8::FunctionTemplate::New(isolate, MyCustomFunction));
    
    globalContext->Global()->Set(v8::String::NewFromUtf8(isolate, "Custom"), 
                                custom->NewInstance());
}
```

### Plugin System
The modular architecture allows for plugin-style extensions:
- **Packet Processors**: Custom protocol handlers
- **Script Modules**: Reusable JavaScript libraries
- **Event Handlers**: Custom server-wide event processing
- **Database Adapters**: Alternative storage backends

---

*This architecture overview provides the foundation for understanding GServer-v2's design. Each component is designed to be modular, extensible, and performant, enabling complex MMO functionality while maintaining clean separation of concerns.*