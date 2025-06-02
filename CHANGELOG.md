# Changelog

All notable changes to yoctoctl will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-06-03

### Added
- Initial release of yoctoctl - Yocto Container Manager
- Automated container lifecycle management for Poky CROPS environments
- Network infrastructure configuration with automatic SMB endpoint setup
- Interactive menu system powered by Charm's gum
- Command-line interface for direct operations
- Comprehensive environment status reporting with health monitoring
- Development shell with customized environment
- Persistent storage volume management for shared build resources
- Graceful shutdown with data preservation
- Terminal state restoration on exit
- Error handling with contextual debugging information

### Technical Features
- ARM64 compatibility for Apple Silicon processors
- Docker container orchestration for yocto-fileserver and yocto-builder
- SMB network alias configuration (127.0.0.2)
- Real-time service verification and health checks
- Storage analysis with usage metrics and file counts
- Professional color-coded UI with muted enterprise theme

### Commands
- `status` - Display comprehensive environment status
- `start` - Initialize and start Yocto containers
- `shell` - Connect to Poky development container
- `shutdown` - Stop all containers gracefully
- `version` - Display version information
- `help` - Show command reference

[1.0.0]: https://github.com/anaskalt/yoctoctl/releases/tag/v1.0.0