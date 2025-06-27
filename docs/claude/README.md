# GServer-v2 Documentation

**GServer-v2** is a comprehensive C++ MMO game server implementation compatible with Graal Online. This documentation provides detailed information about the server architecture, protocol specifications, client types, and development guidelines.

## Table of Contents

### 📚 **Core Documentation**
- [🏗️ Architecture Overview](architecture/README.md) - Server design and component relationships
- [🔌 Protocol Specification](protocol/README.md) - Complete packet format and communication protocols
- [👥 Client Types](client-types/README.md) - Different client implementations and their behaviors
- [⚙️ Development Guide](development/README.md) - Building, testing, and extending the server

### 📋 **Protocol Details**
- [📦 Packet Format](protocol/packet-format.md) - Binary packet structure and encoding
- [🔐 Authentication](claude/protocol/authentication.md) - Login flows for different client types
- [🎮 Game Protocol](claude/protocol/game-protocol.md) - Player movement, chat, level management
- [🛠️ RC Protocol](protocol/rc-protocol.md) - Remote Control administration interface
- [🤖 NPC Protocol](claude/protocol/npc-protocol.md) - NPC server communication

### 👤 **Client Types**
- [🎮 Game Clients](claude/client-types/game-clients.md) - Regular players (CLIENT, CLIENT3, WEB)
- [🛠️ Remote Control](claude/client-types/remote-control.md) - Administration and monitoring (RC)
- [🤖 NPC Server](claude/client-types/npc-server.md) - Scripted NPCs and automation (NPCSERVER)
- [💻 NC Clients](claude/client-types/nc-clients.md) - Level editing and development tools (NC)

### 🏗️ **Architecture**
- [🧠 Core Components](claude/architecture/core-components.md) - TServer, TPlayer, TLevel, TNPC
- [📜 Scripting Engine](claude/architecture/scripting.md) - V8 JavaScript and GraalScript integration
- [🌐 Network Layer](claude/architecture/networking.md) - Socket handling and packet processing
- [💾 Data Management](claude/architecture/data-management.md) - Levels, accounts, and file systems

### 🛠️ **Development**
- [🔧 Build System](development/building.md) - CMake configuration and dependencies
- [🧪 Testing](development/testing.md) - Unit tests and integration testing
- [📝 Contributing](development/contributing.md) - Coding standards and pull request guidelines
- [🐛 Debugging](development/debugging.md) - Debugging tools and techniques

### 📖 **Examples**
- [🚀 Quick Start](examples/quick-start.md) - Getting a server running
- [👥 Multi-Client Setup](claude/examples/multi-client.md) - Supporting different client types
- [🤖 NPC Development](claude/examples/npc-development.md) - Creating scripted NPCs
- [🛠️ RC Administration](claude/examples/rc-administration.md) - Server management via RC

### 🔧 **Troubleshooting**
- [❓ Common Issues](troubleshooting/common-issues.md) - Frequently encountered problems
- [🔍 Protocol Debugging](claude/troubleshooting/protocol-debugging.md) - Analyzing network traffic
- [⚡ Performance](claude/troubleshooting/performance.md) - Optimization and profiling

## Key Features

### 🎯 **Multi-Client Support**
GServer-v2 supports multiple client types simultaneously:
- **Game Clients**: Regular players connecting to play
- **Remote Control**: Administrators monitoring and managing the server
- **NPC Server**: Scripted NPCs providing automated gameplay
- **NC Clients**: Level editors and development tools

### 🔌 **Protocol Flexibility**
- **Binary Protocol**: Efficient packet-based communication
- **Compression**: Optional zlib compression for bandwidth optimization
- **Encryption**: Support for multiple encryption schemes (Gen 1-5)
- **Version Compatibility**: Handles clients from v1.41.1 to v6.037

### 🤖 **Advanced Scripting**
- **V8 JavaScript Engine**: Modern ECMAScript support for NPCs and weapons
- **GraalScript 2.0**: Legacy scripting compatibility with bytecode caching
- **Sandboxed Execution**: Secure script isolation with resource limits
- **Hot Reload**: Dynamic script updates without server restart

### 🌐 **Scalable Architecture**
- **Multi-threaded**: Concurrent handling of multiple clients
- **Event-driven**: Efficient packet processing and script execution
- **Modular Design**: Pluggable components for easy extension
- **Resource Management**: Memory and CPU optimization

## Getting Started

1. **[Build the Server](development/building.md)** - Compile GServer-v2 with dependencies
2. **[Configure Settings](examples/quick-start.md)** - Set up basic server configuration
3. **[Understand Protocols](protocol/README.md)** - Learn how clients communicate
4. **[Add NPCs](claude/examples/npc-development.md)** - Create interactive game content
5. **[Monitor with RC](claude/examples/rc-administration.md)** - Manage your server

## Protocol Overview

GServer-v2 uses a sophisticated binary protocol that adapts based on client type:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Game Client   │    │  Remote Control │    │   NPC Server    │
│                 │    │                 │    │                 │
│  PLI_LOGIN      │    │  RC Login       │    │  NPCSERVER      │
│  PLI_MOVE       │    │  RC Commands    │    │  NPC Scripts    │
│  PLI_CHAT       │    │  RC Monitoring  │    │  Event Hooks    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   GServer-v2    │
                    │                 │
                    │  • TServer      │
                    │  • TPlayer      │
                    │  • TLevel       │
                    │  • TNPC         │
                    │  • V8 Engine    │
                    └─────────────────┘
```

## Key Concepts

### 📦 **Packets**
All communication uses length-prefixed binary packets:
- **Header**: 2-byte length (big-endian for most clients)
- **Type**: 1-byte packet type identifier  
- **Data**: Variable-length payload (often zlib compressed)

### 🔐 **Authentication**
Different flows based on client type:
- **Game Clients**: Account/password + protocol version
- **RC Clients**: Admin credentials + compression
- **NPC Server**: Server-to-server authentication

### 🎮 **Game State**
- **Players**: Position, health, inventory, appearance
- **Levels**: Tiles, NPCs, items, links between areas
- **NPCs**: Scripted entities with V8 JavaScript
- **Weapons**: Player and NPC combat systems

## Advanced Topics

- **[Multi-Server Architecture](claude/architecture/multi-server.md)** - Connecting multiple GServer instances
- **[Custom Protocol Extensions](claude/protocol/extensions.md)** - Adding new packet types
- **[Performance Optimization](claude/troubleshooting/performance.md)** - Scaling for large player counts
- **[Security Considerations](development/security.md)** - Protecting against exploits

## Community & Support

- **Issues**: Report bugs and feature requests on GitHub
- **Development**: See [Contributing Guide](development/contributing.md)
- **Protocol**: Based on original Graal Online specifications
- **License**: Open source under MIT/GPL license

---

*This documentation covers the complete GServer-v2 architecture, from low-level protocol details to high-level server administration. Whether you're developing clients, creating NPCs, or running a server, you'll find detailed information for every aspect of the system.*