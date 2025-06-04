# yoctoctl - Yocto Development Environment Manager

A comprehensive environment management tool for Poky CROPS development on Apple Silicon macOS, providing automated container orchestration, network configuration and seamless file system integration.

## Overview

`yoctoctl` provides a streamlined interface for managing containerized Yocto Project development environments, specifically optimized for Apple Silicon Macs. It automates the complexity of container lifecycle management while providing seamless integration with macOS native tools.

## Key Features

- **Automated Container Lifecycle Management** - Intelligent orchestration of file server and build containers
- **Network Infrastructure Configuration** - Automatic SMB endpoint setup for native macOS file access
- **Interactive Command Interface** - Professional TUI powered by Charm's gum for intuitive operations
- **Real-time Environment Monitoring** - Comprehensive health checks and status reporting
- **Persistent Storage Management** - Shared volume architecture for efficient builds across projects

## System Requirements

- macOS on Apple Silicon (M1/M2/M3/M4)
- Docker Desktop 4.0+ with Rosetta enabled
- Homebrew package manager
- Minimum 16GB RAM (8GB usable for Docker)
- Minimum 100GB available disk space

## Installation

### Prerequisites

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install gum for UI components
brew install gum

# Verify Docker Desktop is installed and running
docker --version
```

### Installing yoctoctl

```bash
# Download the latest release
sudo curl -L https://raw.githubusercontent.com/anaskalt/yoctoctl/main/yoctoctl -o /usr/local/bin/yoctoctl

# Make executable
sudo chmod +x /usr/local/bin/yoctoctl

# Verify installation
yoctoctl version
```

## Quick Start

> **Note**: First-time users must build the container infrastructure. See the [Yocto Development Environment for Apple Silicon](docs/SETUP.md) guide for detailed instructions.

```bash
# Initialize and start the development environment
yoctoctl start

# Check environment status
yoctoctl status

# Connect to the development shell
yoctoctl shell

# Access files via Finder
open "smb://127.0.0.2/workdir"
```

## Usage

### Interactive Mode

Launch without arguments for the interactive menu:

```bash
yoctoctl
```

### Command Line Mode

```bash
yoctoctl [command]

Commands:
  status    Display comprehensive environment status
  start     Initialize and start Yocto containers
  shell     Connect to Poky development container
  shutdown  Stop all containers gracefully
  version   Display version information
  help      Show help message
```

## Container Architecture

The system manages two primary containers:

1. **yocto-fileserver** - Samba server providing SMB access to the workspace
2. **yocto-builder** - Poky CROPS environment for Yocto builds

Both containers share a persistent Docker volume (`yocto-workspace`) ensuring data preservation across sessions.

## Documentation

- [Yocto Development Environment for Apple Silicon](docs/SETUP.md) - Comprehensive guide for environment setup and deployment
- [Architecture Overview](docs/ARCHITECTURE.md) - Technical implementation details

## Troubleshooting

### Common Issues

**Docker daemon not running**
```bash
# Start Docker Desktop application
open -a Docker
```

**Network alias already exists**
```bash
# The tool handles this automatically, but you can manually clear:
sudo ifconfig lo0 -alias 127.0.0.2
```

**Permission denied on script execution**
```bash
# Ensure proper permissions
sudo chmod +x /usr/local/bin/yoctoctl
```

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## Author

**Anastasios Kaltakis**  
GitHub: [@anaskalt](https://github.com/anaskalt)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Charm](https://charm.sh/) for the excellent gum TUI framework
- [CROPS](https://github.com/crops/) for Poky container images
- [ejaaskel](https://github.com/ejaaskel/yocto-dockerfiles) for exceptional work supporting Yocto on Apple Silicon Macs
- The Yocto Project community