# Yocto Development Environment for Apple Silicon

This comprehensive guide establishes a production-ready, persistent Yocto Project development environment optimized for Apple Silicon Macs.

## Table of Contents

- [System Architecture](#system-architecture)
- [Infrastructure Prerequisites](#infrastructure-prerequisites)
- [Container Infrastructure Deployment](#container-infrastructure-deployment)
- [Development Environment Access](#development-environment-access-and-integration)
- [Optimized Build Configuration](#optimized-build-configuration)
- [Development Workflow](#development-workflow)
- [Workspace Usage Guide](#workspace-usage-guide)
- [Performance Optimization](#performance-optimization-tips)

## System Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  macOS Host     │◄──►│  File Server     │◄──►│  Build Engine   │
│  IDE/Finder     │    │  SMB Protocol    │    │  Yocto Runtime  │
│  yoctoctl CLI   │    │  crops/samba     │    │  crops/poky     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                ▲                        ▲
                                │                        │
                                ▼                        ▼
                        ┌─────────────────────────────────────┐
                        │        Docker Volume                │
                        │    Persistent Storage Layer         │
                        │    Downloads │ Cache │ Projects     │
                        └─────────────────────────────────────┘
```

## Infrastructure Prerequisites

### Docker Desktop Configuration

Configure Docker Desktop for optimal Yocto development performance:

```bash
# Verify Docker installation and ARM64 compatibility
docker --version
docker system info

# Required Docker Desktop Settings:
# Settings → Resources → Advanced:
# - Memory: 16GB (minimum 8GB)
# - CPUs: 6 cores (minimum 4 cores) 
# - Disk: 200GB (minimum 100GB)
```

### System Requirements

- macOS on Apple Silicon (M1/M2/M3/M4)
- Docker Desktop 4.0+ with Rosetta enabled
- Homebrew package manager
- Minimum 16GB system RAM
- Minimum 200GB available storage

## Container Infrastructure Deployment

### ARM64-Compatible Image Building

Create the required container images for Apple Silicon architecture:

```bash
# Create workspace for container building
mkdir -p ~/yocto-project && cd ~/yocto-project

# Clone ARM64-compatible repositories
git clone https://github.com/ejaaskel/yocto-dockerfiles.git
git clone https://github.com/crops/samba.git  
git clone https://github.com/crops/poky-container.git

# Build ARM64-compatible Yocto base images
cd yocto-dockerfiles
export REPO=ejaaskel/yocto
export DISTRO_TO_BUILD=ubuntu-22.04
./build_container.sh

# Build Samba container
cd ../samba
docker build -t crops/samba .

# Configure and build Poky container
cd ../poky-container
sed -i '' 's/ARG BASE_DISTRO=SPECIFY_ME/ARG BASE_DISTRO=ubuntu-22.04/' Dockerfile
sed -i '' 's/FROM crops\/yocto:$BASE_DISTRO-base/FROM ejaaskel\/yocto:$BASE_DISTRO-base/' Dockerfile
docker build -t crops/poky .

cd ~/yocto-project
```

### Storage Infrastructure

Create and initialize the persistent storage volume:

```bash
# Create persistent volume for Yocto development
docker volume create yocto-workspace

# Initialize workspace with directory structure
docker run --rm -v yocto-workspace:/workspace ubuntu:22.04 bash -c "
    chown -R 1000:1000 /workspace
    mkdir -p /workspace/{shared/{downloads,sstate-cache,ccache},documentation}
    
    cat > /workspace/README.md << 'WORKSPACE_DOC'
# Yocto Workspace

## Directory Structure
- shared/downloads/   : Source package download cache
- shared/sstate-cache/: Shared state cache for accelerated builds  
- shared/ccache/      : Compiler cache for faster compilation
- documentation/      : Development notes and project documentation

## Development Access
- File System: smb://127.0.0.2/workdir
- Build Environment: yoctoctl shell
WORKSPACE_DOC
"
```

### Container Deployment

Deploy the persistent container infrastructure:

```bash
# Deploy persistent file server
docker run -d \
    --name yocto-fileserver \
    --hostname yocto-fs \
    -p 445:445 \
    -v yocto-workspace:/workdir \
    --restart unless-stopped \
    crops/samba

# Deploy persistent build environment
docker run -d \
    --name yocto-builder \
    --hostname yocto-dev \
    -v yocto-workspace:/workdir \
    --volumes-from yocto-fileserver \
    --restart unless-stopped \
    --env TERM=xterm-256color \
    --env LANG=en_US.UTF-8 \
    --env CCACHE_DIR=/workdir/shared/ccache \
    crops/poky \
    --workdir=/workdir \
    tail -f /dev/null

# Verify infrastructure deployment
docker ps --filter "name=yocto-" --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

## Development Environment Access and Integration

### Network Configuration

yoctoctl automatically handles network configuration, but manual setup is:

```bash
# Configure loopback alias for SMB access
sudo ifconfig lo0 127.0.0.2 alias up

# Verify configuration
ifconfig lo0 | grep 127.0.0.2
```

### File System Integration

```bash
# Initialize complete environment with automated configuration
yoctoctl start

# Connect macOS Finder to development workspace
open "smb://127.0.0.2/workdir"

# Alternative manual connection:
# Finder → ⌘+K → smb://127.0.0.2/workdir
```

### IDE Integration

Create VS Code workspace for Yocto development:

```bash
# Create VS Code workspace configuration
cat > ~/yocto-development.code-workspace << 'EOF'
{
    "folders": [
        {
            "name": "Yocto Workspace", 
            "path": "/Volumes/workdir"
        }
    ],
    "settings": {
        "files.watcherExclude": {
            "**/shared/downloads/**": true,
            "**/shared/sstate-cache/**": true,
            "**/shared/ccache/**": true,
            "**/tmp/**": true,
            "**/build/tmp/**": true
        },
        "search.exclude": {
            "**/shared/downloads/**": true,
            "**/shared/sstate-cache/**": true,
            "**/shared/ccache/**": true,
            "**/tmp/**": true,
            "**/build/tmp/**": true
        },
        "files.associations": {
            "*.bb": "shellscript",
            "*.bbappend": "shellscript",
            "*.bbclass": "shellscript",
            "*.inc": "shellscript"
        },
        "editor.rulers": [80, 120],
        "files.trimTrailingWhitespace": true,
        "editor.detectIndentation": false,
        "editor.tabSize": 4,
        "editor.insertSpaces": true,
        "editor.wordWrap": "wordWrapColumn",
        "editor.wordWrapColumn": 120
    },
    "extensions": {
        "recommendations": [
            "ms-vscode.cpptools",
            "ms-python.python", 
            "redhat.vscode-yaml",
            "ms-vscode.cmake-tools",
            "ms-vscode.makefile-tools",
            "yocto-project.yocto-bitbake"
        ]
    }
}
EOF

# Open workspace
code ~/yocto-development.code-workspace
```

## Optimized Build Configuration

Create production-optimized configuration template:

```bash
# Production-optimized local.conf template for Apple Silicon
cat << 'PRODUCTION_CONFIG' > ~/production-local.conf
# =============================================================================
# Yocto Production Build Configuration
# Optimized for Apple Silicon containerized development environments
# =============================================================================

# Performance Optimization Configuration
BB_NUMBER_THREADS = "6"
PARALLEL_MAKE = "-j 6"
BB_TASK_NICE_LEVEL = "10"
BB_TASK_IONICE_LEVEL = "2.7"

# Shared Resource Configuration for Maximum Efficiency
DL_DIR = "/workdir/shared/downloads"
SSTATE_DIR = "/workdir/shared/sstate-cache"

# Compiler Cache Optimization for Faster Builds
CCACHE_DIR = "/workdir/shared/ccache"
INHERIT += "ccache"

# Build Performance Monitoring and History
INHERIT += "buildhistory"
BUILDHISTORY_COMMIT = "1"
BUILDHISTORY_DIR = "${TOPDIR}/buildhistory"

# Development and Debugging Features
EXTRA_IMAGE_FEATURES += "debug-tweaks tools-debug tools-profile"

# Package Management Optimization
PACKAGE_CLASSES ?= "package_rpm"

# SDK Configuration for Cross-Development
SDKMACHINE ?= "x86_64"
INHERIT += "populate_sdk_ext"

# Resource Protection and Monitoring
BB_DISKMON_DIRS = "\
    STOPTASKS,${TMPDIR},3G,1G \
    STOPTASKS,${DL_DIR},2G,1G \
    STOPTASKS,${SSTATE_DIR},2G,1G \
    ABORT,${TMPDIR},1G,100M \
    ABORT,${DL_DIR},1G,100M \
    ABORT,${SSTATE_DIR},1G,100M"

# Network and Download Optimization
BB_NUMBER_PARSE_THREADS = "4"
BB_FETCH_PREMIRRORONLY = "0"
BB_NO_NETWORK = "0"

# Advanced Build Configuration
INHERIT += "rm_work"
RM_WORK_EXCLUDE += "linux-yocto"

# Image and Filesystem Optimization
IMAGE_FSTYPES += "tar.bz2"
IMAGE_ROOTFS_EXTRA_SPACE = "1048576"
PRODUCTION_CONFIG

echo "Production build configuration saved to: ~/production-local.conf"
```

## Development Workflow

### Environment Initialization

```bash
# Complete environment startup with comprehensive validation
yoctoctl start

# Verify all systems operational with detailed health assessment
yoctoctl status

# Access development environment
yoctoctl shell
```

### Project Setup Workflow

Within the development container:

```bash
# 1. Navigate to workspace
cd /workdir

# 2. Setup your Yocto project
git clone <your-yocto-project-repository>
cd <project-directory>

# 3. Initialize build environment with optimized configuration
source oe-init-build-env build

# 4. Apply production-optimized configuration
cp /tmp/production-local.conf conf/local.conf

# 5. Execute builds with shared cache benefits
bitbake core-image-minimal

# 6. Generate SDK for cross-development
bitbake -c populate_sdk core-image-minimal
```

## Workspace Usage Guide

### Workspace Structure

```
/workdir/
├── shared/
│   ├── downloads/     # Source package cache (shared across all projects)
│   ├── sstate-cache/  # Build state cache for faster incremental builds  
│   └── ccache/        # Compiler cache for faster C/C++ compilation
├── documentation/     # Your project documentation and notes
└── [your-projects]/   # Your individual Yocto projects
```

### Working with Multiple Projects

The shared cache system enables efficient multi-project development:

```bash
# Project A
cd /workdir/project-a
source oe-init-build-env build-a
bitbake my-image-a

# Project B (benefits from Project A's cached downloads and builds)
cd /workdir/project-b  
source oe-init-build-env build-b
bitbake my-image-b
```

### File Access Methods

**SMB/Finder Access:**
```bash
# Open in Finder
open "smb://127.0.0.2/workdir"
```

**Direct File Transfer:**
```bash
# From container to macOS
docker cp yocto-builder:/workdir/project/build/tmp/deploy/images/ ~/Desktop/

# From macOS to container  
docker cp ~/Desktop/my-layer/ yocto-builder:/workdir/project/
```

## Performance Optimization Tips

### Monitor Resource Usage

```bash
# Check workspace utilization
yoctoctl status

# Monitor build performance
docker stats yocto-builder

# Inside container - check cache efficiency
du -sh /workdir/shared/*
ccache -s  # Show compiler cache statistics
```

### Cache Management

```bash
# Clean old downloads (if space needed)
find /workdir/shared/downloads -mtime +30 -delete

# Clean sstate cache selectively
bitbake -c cleanall [recipe-name]

# Clear ccache if needed
ccache -C
```

### Build Optimization

The provided local.conf template includes:
- `BB_NUMBER_THREADS = "6"` - Parallel bitbake task execution
- `PARALLEL_MAKE = "-j 6"` - Parallel compilation
- `INHERIT += "ccache"` - Compiler cache enabled
- `INHERIT += "rm_work"` - Automatic work directory cleanup

## Troubleshooting

### Common Issues

**Container not starting:**
```bash
# Check Docker logs
docker logs yocto-builder
docker logs yocto-fileserver
```

**SMB connection issues:**
```bash
# Verify network alias
ifconfig lo0 | grep 127.0.0.2

# Check SMB service
nc -z 127.0.0.2 445
```

**Build failures:**
```bash
# Check available disk space
df -h /workdir

# Verify resource limits
docker exec yocto-builder ulimit -a
```

## Maintenance

### Regular Cleanup

```bash
# Remove old build artifacts
find /workdir -name "tmp" -type d -exec rm -rf {} + 2>/dev/null

# Prune Docker system
docker system prune -a --volumes
```

### Backup Important Data

```bash
# Backup project configurations
tar -czf ~/yocto-backup-$(date +%Y%m%d).tar.gz \
    /Volumes/workdir/*/conf \
    /Volumes/workdir/documentation
```
