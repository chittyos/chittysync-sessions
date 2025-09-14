# Cross-Session Sync System

Real-time coordination framework for multiple Claude sessions working on the same project.

## Architecture Overview

The Cross-Session Sync system enables multiple Claude sessions to:
- Share real-time state and progress updates
- Coordinate task division and prevent duplicate work
- Maintain consistency across parallel work streams
- Resolve conflicts automatically
- Track session health and activity

## Core Components

### 1. Session Registry
- Unique session identification
- Session lifecycle management
- Health monitoring and heartbeats
- Active session discovery

### 2. State Synchronization
- File-based state persistence
- Atomic state updates
- Event-driven notifications
- Optimistic locking

### 3. Task Coordination
- Distributed task queue
- Task claiming and locking
- Progress tracking
- Dependency management

### 4. Conflict Resolution
- Version vectors for causality tracking
- Three-way merge for concurrent edits
- Automatic conflict detection
- Manual intervention protocols

## Quick Start

```bash
# Initialize sync system in your project
./sync-init.sh

# Register a new session
./session-register.sh "session-name"

# Start sync monitoring
./sync-monitor.sh
```

## File Structure

```
cross-session-sync/
├── README.md                   # This file
├── sync-init.sh               # Initialize sync system
├── session-register.sh        # Register new session
├── sync-monitor.sh           # Monitor sync status
├── src/
│   ├── session-manager.js    # Session lifecycle management
│   ├── state-sync.js         # State synchronization engine
│   ├── task-coordinator.js   # Task distribution logic
│   ├── conflict-resolver.js  # Conflict resolution algorithms
│   └── file-watcher.js       # File system monitoring
├── config/
│   ├── sync-config.json      # Configuration settings
│   └── session-schema.json   # Session data schema
└── state/
    ├── sessions/              # Active session records
    ├── tasks/                 # Task queue and status
    ├── locks/                 # Resource locks
    └── history/               # Change history log
```

## Session Lifecycle

1. **Registration**: Session creates unique ID and registers presence
2. **Discovery**: Session discovers other active sessions
3. **Synchronization**: Session syncs with shared state
4. **Coordination**: Session claims tasks and updates progress
5. **Heartbeat**: Session maintains heartbeat for liveness
6. **Cleanup**: Session performs cleanup on termination

## Communication Protocol

Sessions communicate through:
- **State files**: JSON files in shared directories
- **Lock files**: Atomic file operations for mutual exclusion
- **Event logs**: Append-only logs for event streaming
- **Heartbeat files**: Timestamp files for liveness detection

## Best Practices

1. **Always register sessions** before starting work
2. **Use task claiming** to prevent duplicate effort
3. **Update state frequently** for real-time coordination
4. **Handle conflicts gracefully** with automatic resolution
5. **Clean up resources** on session termination

## API Reference

See individual component documentation:
- [Session Manager API](src/session-manager.md)
- [State Sync API](src/state-sync.md)
- [Task Coordinator API](src/task-coordinator.md)
- [Conflict Resolver API](src/conflict-resolver.md)