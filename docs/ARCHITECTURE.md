# yoctoctl Architecture Overview

This document describes the technical architecture and implementation details of yoctoctl, a container orchestration tool for Yocto development environments on Apple Silicon macOS.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [System Components](#system-components)
- [Technical Architecture](#technical-architecture)
- [Implementation Details](#implementation-details)
- [Network Architecture](#network-architecture)
- [Storage Architecture](#storage-architecture)
- [Error Handling Strategy](#error-handling-strategy)
- [UI/UX Design](#uiux-design)

## Design Philosophy

yoctoctl follows several core design principles:

### Single Responsibility
The tool focuses exclusively on container orchestration for Yocto development, avoiding feature creep and maintaining clarity of purpose.

### Zero Configuration
Users should be able to start development with a single command. All complex configuration is handled internally with sensible defaults.

### Graceful Degradation
The system continues to function even when some components fail, providing clear feedback about degraded functionality.

### Native Integration
Leverages macOS native capabilities (SMB, Finder) rather than requiring additional tools or modifications.

## System Components

### Core Script Structure

```
yoctoctl
├── Global Configuration
│   ├── Script metadata (name, version, author)
│   ├── Container definitions
│   ├── Network configuration
│   └── Color palette definitions
│
├── Error Handling Layer
│   ├── Trap handlers for ERR and EXIT
│   ├── Contextual error reporting
│   └── Terminal state restoration
│
├── Utility Functions
│   ├── Docker verification
│   ├── Dependency checks
│   └── Banner display
│
├── Network Management
│   ├── Loopback alias configuration
│   ├── Port accessibility checks
│   └── SMB endpoint management
│
├── Container Operations
│   ├── Status monitoring
│   ├── Lifecycle management
│   └── Health verification
│
├── Storage Analysis
│   ├── Volume inspection
│   ├── Usage metrics
│   └── Content statistics
│
├── Command Implementations
│   ├── status, start, shell
│   ├── shutdown, help, version
│   └── interactive menu
│
└── Main Entry Point
    └── Command dispatcher
```

### External Dependencies

1. **Docker Engine**
   - Container runtime for all operations
   - Volume management for persistent storage
   - Network isolation and port mapping

2. **Charm gum**
   - Professional TUI components
   - Spinner animations during operations
   - Styled text output and formatting
   - Interactive menu selections

3. **macOS System Utilities**
   - `ifconfig` for network alias management
   - `nc` (netcat) for port connectivity testing
   - Native SMB client for file access

## Technical Architecture

### Container Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│                         macOS Host                          │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐    │
│  │   yoctoctl   │  │    Finder    │  │   User's IDE    │    │
│  │  (CLI Tool)  │  │ (SMB Client) │  │   (VS Code)     │    │
│  └──────┬───────┘  └───────┬──────┘  └────────┬────────┘    │
│         │                  │                  │             │
│         │                  │                  │             │
├─────────┼──────────────────┼──────────────────┼─────────────┤
│         │              SMB │ Protocol         │             │
│         │           (127.0.0.2:445)           │             │
│         │                  │                  │             │
│  ┌──────▼──────────────────▼──────────────────▼───────────┐ │
│  │                    Docker Engine                       │ │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐  │ │
│  │  │ yocto-fileserver│  │    yocto-builder            │  │ │
│  │  │                 │  │                             │  │ │
│  │  │ crops/samba     │  │ crops/poky (Ubuntu 22.04)   │  │ │
│  │  │ Port: 445 ──────┼──┤ Yocto build environment     │  │ │
│  │  │ /workdir ◄──────┼──┤ /workdir                    │  │ │
│  │  └─────────────────┘  └─────────────────────────────┘  │ │
│  │         ▲                          ▲                   │ │
│  │         └──────────┬───────────────┘                   │ │
│  │                    │                                   │ │
│  │         ┌──────────▼─────────────┐                     │ │
│  │         │   yocto-workspace      │                     │ │
│  │         │   (Docker Volume)      │                     │ │
│  │         │   Persistent Storage   │                     │ │
│  │         └────────────────────────┘                     │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Process Flow

1. **Initialization Phase**
   ```bash
   main() → check_gum() → check_docker() → command_dispatch()
   ```

2. **Start Command Flow**
   ```bash
   cmd_start() → configure_network() → start_containers() → wait_for_fileserver()
   ```

3. **Status Command Flow**
   ```bash
   cmd_status() → check_network_status() → get_container_status() → analyze_storage()
   ```

## Implementation Details

### Network Configuration

The tool configures a loopback alias to enable SMB access without modifying system-wide network settings:

```bash
# Configure loopback alias (non-persistent by design)
sudo ifconfig lo0 127.0.0.2 alias up

# Why 127.0.0.2?
# - Avoids conflict with standard localhost (127.0.0.1)
# - Remains within loopback range for security
# - Predictable and memorable for users
```

### Container Lifecycle Management

```bash
# Container states and transitions
┌─────────┐  docker start   ┌─────────┐  docker exec    ┌─────────┐
│ missing │ ──────────────► │ stopped │ ──────────────► │ running │
└─────────┘                 └─────────┘                 └─────────┘
                                  ▲                           │
                                  └───────────────────────────┘
                                       docker stop
```

### Storage Architecture

The shared volume architecture enables efficient multi-project development:

```
yocto-workspace/
├── shared/
│   ├── downloads/      # DL_DIR - Source archives
│   ├── sstate-cache/   # SSTATE_DIR - Build artifacts
│   └── ccache/         # CCACHE_DIR - Compiler cache
├── documentation/      # Project documentation
└── [projects]/         # Individual Yocto projects
```

### Service Verification

The file server readiness check monitors Samba daemon logs for specific patterns:

```bash
# Pattern matching for service readiness
"STATUS=daemon 'smbd' finished starting up and ready to serve connections"

# Timeout mechanism prevents indefinite waiting
timeout=30 seconds with 1-second polling interval
```

## Network Architecture

### SMB Service Configuration

The yocto-fileserver container exposes SMB services through:

1. **Port Mapping**: Container port 445 → Host port 445
2. **Loopback Alias**: 127.0.0.2 configured on lo0 interface
3. **Volume Mount**: /workdir in container → yocto-workspace volume

### Network Isolation

All network communication remains within the Docker bridge network and loopback interface, ensuring:
- No exposure to external networks
- Predictable network behavior
- Security through isolation

## Storage Architecture

### Volume Management Strategy

```bash
# Single shared volume approach
docker volume create yocto-workspace

# Benefits:
# - Data persistence across container restarts
# - Shared access between fileserver and builder
# - Atomic operations for data consistency
# - Easy backup and migration
```

### Cache Optimization

The storage layout optimizes build performance through strategic caching:

1. **Download Cache** (`shared/downloads/`)
   - Prevents redundant source downloads
   - Shared across all projects and builds

2. **State Cache** (`shared/sstate-cache/`)
   - Stores intermediate build artifacts
   - Dramatically reduces rebuild times

3. **Compiler Cache** (`shared/ccache/`)
   - Caches compilation results
   - Speeds up C/C++ compilation

## Error Handling Strategy

### Comprehensive Error Trapping

```bash
# Error handler with context
handle_error() {
    local exit_code=$?
    local line_number="${1:-unknown}"
    local function_name="${2:-main}"
    
    # Contextual error reporting via gum
    gum log --level error "Operation failed" \
        function "$function_name" \
        line "$line_number" \
        code "$exit_code"
}

# Trap installation
trap 'handle_error ${LINENO} ${FUNCNAME:-main}' ERR
```

### Terminal State Protection

```bash
# Ensure terminal restoration on any exit
trap 'printf "\033[?25h\033[0m"' EXIT

# Restores:
# - Cursor visibility (\033[?25h)
# - Color/formatting reset (\033[0m)
```

### Graceful Degradation

The tool continues operation when non-critical components fail:

- Missing containers are reported but don't halt execution
- Network configuration failures provide fallback instructions
- Service timeouts issue warnings but allow continuation

## UI/UX Design

### Color Palette Strategy

The muted enterprise theme provides professional appearance while maintaining readability:

```bash
COLOR_PRIMARY="#6B8CAE"   # Muted Steel Blue - Primary actions
COLOR_ACCENT="#7EA885"    # Sage Green - Success states
COLOR_WARNING="#D4A574"   # Muted Gold - Warning conditions
COLOR_ERROR="#B87B7B"     # Dusty Rose - Error states
COLOR_INFO="#8FA4B8"      # Powder Blue - Information
COLOR_MUTED="#9B9B9B"     # Neutral Gray - Secondary text
COLOR_DARK="#4A5F7A"      # Deep Blue Gray - Headers
```

### Interactive Menu Design

The menu system provides intuitive navigation with visual status indicators:

```
┌─────────────────────────────┐
│        ASCII Banner         │
│   Status: Operational ✓     │
│                             │
│   > View Status             │
│     Start Environment       │
│     Development Shell       │
│     Shutdown Environment    │
│                             │
└─────────────────────────────┘
```

### Command Output Formatting

Consistent formatting patterns enhance readability:

1. **Status Reports**: Tabular layout with aligned columns
2. **Progress Indicators**: Spinner animations with descriptive text
3. **Confirmation Prompts**: Clear yes/no options with default selections
4. **Error Messages**: Contextual information with suggested actions

## Security Considerations

### Principle of Least Privilege

- Only requests sudo for network configuration
- Containers run with minimal required permissions
- No persistent system modifications

### Network Security

- SMB access restricted to loopback interface
- No external network exposure
- Port 445 only accessible locally

### Data Protection

- All data remains within Docker-managed volumes
- No temporary files in system directories
- Clean separation from host filesystem

## Performance Optimizations

### Startup Performance

- Parallel container startup when possible
- Minimal dependency checking
- Cached status information

### Runtime Efficiency

- Direct Docker API usage (no wrapper overhead)
- Efficient string parsing with bash built-ins
- Minimal external command invocations

### Resource Usage

- No background processes or daemons
- Clean exit with proper cleanup
- Memory-efficient status checking

## Future Architecture Considerations

### Potential Enhancements

1. **Plugin Architecture**: Support for custom commands
2. **Remote Operations**: Docker context support for remote hosts
3. **Multi-Architecture**: Support for x86_64 alongside ARM64
4. **Advanced Monitoring**: Real-time build progress tracking

### Maintaining Simplicity

Any future enhancements should preserve the core design principles:
- Single-file distribution
- Zero configuration philosophy
- Native macOS integration
