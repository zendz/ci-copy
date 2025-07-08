# CI Copy (Container Image Copy)

A cross-platform Rust application for copying Docker container images between AWS accounts, designed to replace PowerShell-based CI/CD workflows with a unified solution that works across Linux, macOS, and Windows.

## Overview

CI Copy (Container Image Copy) was created to address the complexity of managing source code across multiple platforms in CI/CD environments. Originally implemented as a PowerShell script (`container-copy.ps1`), this Rust rewrite provides a single, cross-platform binary that can handle container image copying between different AWS environments (dev → UAT → production).

## Features

- **Cross-platform compatibility**: Single binary works on Linux, macOS, and Windows
- **AWS ECR integration**: Seamless authentication and image management across multiple AWS accounts
- **Multi-environment support**: Copy images between development, UAT, and production environments
- **Progress tracking**: Real-time progress indicators and detailed logging
- **Image verification**: SHA256 hash comparison to ensure successful copies
- **Error handling**: Comprehensive error reporting and recovery mechanisms
- **Batch processing**: Support for copying multiple images in a single operation

## Prerequisites

- Rust 1.70+ (using 2024 edition)
- **Copy Tool Dependencies** (automatically detected):
  - **Linux/macOS**: Skopeo installed and available in PATH
  - **Windows**: Docker installed and running
- AWS CLI configured with appropriate profiles
- Valid AWS credentials for source and target accounts
- Appropriate ECR permissions for pulling and pushing images

### Installing Skopeo (Linux/macOS)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install skopeo

# macOS (Homebrew)
brew install skopeo

# RHEL/CentOS/Fedora
sudo dnf install skopeo  # or sudo yum install skopeo

# Arch Linux
sudo pacman -S skopeo
```

## Installation

### From Source

```bash
git clone <repository-url>
cd ci-copy
cargo build --release
```

The compiled binary will be available at `target/release/ci-copy`.

### Using Cargo

```bash
cargo install ci-copy  # Container Image Copy
```

## Configuration

### Default Configuration File

CI Copy uses a configuration-first approach with intelligent defaults:

- **Default behavior**: When run without parameters, CI Copy automatically looks for `ci-copy.yaml` in the current directory
- **Custom config**: You can specify a different configuration file using `-c` or `--config` parameter
- **CLI override**: Command-line parameters can override configuration file settings
- **Multiple input methods**: Support both configuration files and inline image specification

### Configuration File Specification

```bash
# These commands are equivalent and use different config file specification formats:
ci-copy -c myconfig.yaml
ci-copy -c=myconfig.yaml  
ci-copy --config myconfig.yaml
ci-copy --config=myconfig.yaml

# Use default configuration file (ci-copy.yaml)
ci-copy

# Inline image specification (no config file needed)
ci-copy --source-profile source-profile --target-profile target-profile \
        --images "app-service:v1.0.0,web-service:v1.0.1" \
        --region us-east-1
```

### AWS Profiles

Ensure you have AWS profiles configured for your environments:

```bash
# Development/UAT environment
aws configure --profile source-profile

# Production environment  
aws configure --profile target-profile
```

### ECR Repositories

The application automatically detects ECR repository URLs from your configured AWS profiles. It will construct the ECR URLs using the account ID associated with each profile:

- **Source ECR**: Automatically detected from source profile
- **Target ECR**: Automatically detected from target profile

> **Note**: Ensure your AWS profiles are properly configured with the correct account credentials. The application will automatically determine the ECR URLs based on the account IDs.

## Usage

### Usage Examples

```bash
# Check system dependencies and configuration
ci-copy check
ci-copy doctor  # alias for 'check'

# Copy a single container image from source to target environment
ci-copy --source-profile source-profile --target-profile target-profile \
        --repository your-app-service \
        --tag release-v.1.0.0 \
        --region us-east-1

# Use default configuration file (ci-copy.yaml in current directory)
ci-copy

# Specify custom configuration file using different formats
ci-copy -c example.yaml
ci-copy -c=example.yaml
ci-copy --config example.yaml
ci-copy --config=example.yaml

# Copy multiple images with inline specification
ci-copy --images "your-app-service:release-v.1.0.0,your-web-service:release-v.1.0.1"

# Force Docker method on Linux/macOS (useful for testing or specific requirements)
ci-copy --source-profile source-profile --target-profile target-profile \
        --repository your-app-service \
        --tag release-v.1.0.0 \
        --region us-east-1 \
        --force-docker
```

### Configuration File Example

Create a `ci-copy.yaml` file (default configuration file name):

```yaml
source:
  profile: source-profile
  region: us-east-1

target:
  profile: target-profile
  region: us-east-1

# Simple string format (recommended - Docker compatible)
images:
  - your-app-service:release-v.1.0.0
  - your-web-service:release-v.1.0.1
  - another-service:latest
  - microservice-api:v2.1.3

# Alternative object format (for advanced use cases with future extensibility)
# images:
#   - repository: your-app-service
#     tag: release-v.1.0.0
#   - repository: your-web-service  
#     tag: release-v.1.0.1
#     # Future: additional fields like platform, digest, etc.
```

### Input Methods Comparison

| Method | Format | Use Case | Example |
|--------|--------|----------|---------|
| **Config File (Recommended)** | `repo:tag` | Multiple images, reusable | `ci-copy` |
| **Inline Images** | `repo:tag,repo:tag` | Quick batch copy | `--images "app:v1,web:v2"` |
| **Single Image** | Separate params | One-off copy | `--repository app --tag v1` |

> **Note**: 
> - The application automatically looks for `ci-copy.yaml` in the current directory when no config file is specified
> - Use the string format (`repository:tag`) for simplicity and Docker compatibility
> - Replace `source-profile` and `target-profile` with your actual AWS profile names, and update regions as needed
> - ECR URLs will be automatically detected from the profiles

### Command Line Options

```
OPTIONS:
    --source-profile <PROFILE>    AWS profile for source ECR
    --target-profile <PROFILE>    AWS profile for target ECR
    --source-ecr <URL>           Source ECR repository URL
    --target-ecr <URL>           Target ECR repository URL
    --repository <NAME>          Repository name to copy
    --tag <TAG>                  Image tag to copy
    --config <FILE>              Configuration file path
    --verify                     Verify image integrity after copy
    --parallel <N>               Number of parallel operations (default: 1)
    --verbose                    Enable verbose logging
    --help                       Print help information
    --version                    Print version information
```

## Workflow

The application follows this process with optimized copying methods per platform:

### Linux/macOS (Skopeo Method)
1. **Authentication**: Login to both source and target ECR repositories using configured AWS profiles
2. **ECR Discovery**: Automatically detect ECR URLs from AWS account IDs associated with the profiles  
3. **Validation**: Verify source image exists and target repository is accessible
4. **Direct Copy**: Use Skopeo to copy directly from source ECR to target ECR (no local storage required)
5. **Verification**: Compare SHA256 hashes to ensure successful copy

### Windows (Docker Method)
1. **Authentication**: Login to both source and target ECR repositories using configured AWS profiles
2. **ECR Discovery**: Automatically detect ECR URLs from AWS account IDs associated with the profiles  
3. **Validation**: Verify source image exists and target repository is accessible
4. **Pull**: Download the image from the source ECR using Docker
5. **Tag**: Tag the image for the target ECR
6. **Push**: Upload the image to the target ECR using Docker
7. **Verification**: Compare SHA256 hashes to ensure successful copy
8. **Cleanup**: Remove local images to free up disk space

## System Check & Diagnostics

CI Copy includes a comprehensive system check to verify all dependencies and configurations are properly set up.

### Health Check Command

```bash
# Run system diagnostic check
ci-copy check

# Verbose diagnostic with detailed information
ci-copy check --verbose

# Check specific AWS profile
ci-copy check --profile source-profile

# Attempt to fix common issues automatically
ci-copy check --fix
```

### What Gets Checked

#### ✅ **Core Dependencies**
- **Operating System**: Linux/macOS/Windows detection
- **Copy Tools**: 
  - Skopeo availability and version (Linux/macOS)
  - Docker daemon status and version (Windows/fallback)
- **AWS CLI**: Installation and version compatibility

#### ✅ **AWS Configuration**
- **AWS Profiles**: Validation of configured profiles
- **AWS Credentials**: Access key and authentication status
- **ECR Permissions**: Read/write access to ECR repositories
- **Region Configuration**: Default region settings

#### ✅ **Application Configuration**
- **Config File**: `ci-copy.yaml` syntax and validation
- **Image References**: Repository and tag format validation
- **Network Connectivity**: ECR endpoint reachability

### Sample Check Output

```console
$ ci-copy check

🔍 CI Copy System Diagnostics
==========================

✅ Operating System: Linux (Ubuntu 22.04)
✅ Copy Method: Skopeo (v1.11.1) - Optimal for this platform
✅ AWS CLI: v2.13.25 - Compatible
✅ Configuration File: ci-copy.yaml - Valid

🔧 AWS Configuration:
✅ Profile 'source-profile': Valid credentials
✅ Profile 'target-profile': Valid credentials  
✅ ECR Access: Read/Write permissions confirmed
✅ Network: ECR endpoints reachable

📊 Ready to copy container images!

Total checks: 8/8 passed ✅
```

### Error Detection & Suggestions

```console
$ ci-copy check

🔍 CI Copy System Diagnostics
==========================

❌ Copy Method: Docker daemon not running
   💡 Fix: Start Docker Desktop or run 'sudo systemctl start docker'

⚠️  AWS Profile 'prod-profile': Credentials expired
   💡 Fix: Run 'aws configure --profile prod-profile' or refresh SSO

❌ ECR Access: Insufficient permissions for target ECR
   💡 Required: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:PutImage

Total checks: 5/8 passed ❌
Run 'ci-copy check --fix' to attempt automatic fixes.
```

CI Copy automatically selects the optimal copying method based on your operating system:

### Skopeo Method (Linux/macOS)
- **Direct registry-to-registry copying**: No intermediate local storage required
- **Faster transfers**: Eliminates the pull/tag/push cycle
- **Lower resource usage**: No Docker daemon dependency
- **Better for CI/CD environments**: Ideal for containerized build systems
- **Network efficient**: Streams data directly between registries

Example workflow:
```bash
# Single image copy using Skopeo
skopeo copy docker://source-ecr/app:v1.0.0 docker://target-ecr/app:v1.0.0
```

### Docker Method (Windows)
- **Traditional workflow**: Pull, tag, and push using Docker daemon
- **Broad compatibility**: Works with all Docker-compatible systems
- **Local verification**: Images are temporarily stored locally for verification
- **Familiar workflow**: Uses standard Docker commands under the hood

Example workflow:
```bash
# Multi-step Docker workflow
docker pull source-ecr/app:v1.0.0
docker tag source-ecr/app:v1.0.0 target-ecr/app:v1.0.0
docker push target-ecr/app:v1.0.0
```

The application automatically detects your operating system and uses the appropriate method. You can override this behavior using the `--force-docker` flag if needed.

The application includes comprehensive error handling for common scenarios:

- Invalid AWS credentials or profiles
- Network connectivity issues
- Missing source images
- Insufficient permissions
- Disk space limitations
- ECR API rate limiting

## Logging

Detailed logging is provided with different verbosity levels:

- **Info**: Basic operation status
- **Debug**: Detailed operation steps (use `--verbose`)
- **Error**: Error messages with context
- **Progress**: Real-time progress indicators

## Comparison with PowerShell Version

| Feature | PowerShell | Rust |
|---------|------------|------|
| Platform Support | Windows only | Linux, macOS, Windows |
| Copy Method | Docker only | Skopeo (Linux/macOS), Docker (Windows) |
| Dependencies | PowerShell, AWS CLI, Docker | Self-contained binary + optimal copy tool |
| Performance | Moderate | Fast (especially with Skopeo direct copy) |
| Local Storage | Always required (pull/tag/push) | Optional (Skopeo: no local storage needed) |
| Docker Daemon | Required | Not required on Linux/macOS |
| Error Handling | Basic | Comprehensive |
| Parallel Processing | Limited | Full support |
| Configuration | Hardcoded ECR URLs | Auto-detected from AWS profiles |
| ECR Discovery | Manual URL specification | Automatic from account ID |
| **System Diagnostics** | ❌ None | ✅ `ci-copy check` command |
| **Dependency Detection** | ❌ Manual troubleshooting | ✅ Automated dependency validation |
| **Health Checks** | ❌ None | ✅ Pre-flight validation |

## Development

### Building

```bash
# Debug build
cargo build

# Release build with optimizations
cargo build --release

# Run tests
cargo test

# Run system check for development
RUST_LOG=debug cargo run -- check --verbose

# Test copy functionality with logging
RUST_LOG=debug cargo run -- copy --config test-config.yaml --verbose
```

### Testing Dependencies

Before running tests, ensure your development environment has the required dependencies:

```bash
# Check development environment
cargo run -- check --verbose

# Test with different copy methods
cargo run -- copy --force-docker --repository test-repo --tag test-tag
```

### Dependencies

Key dependencies include:
- `tokio` - Async runtime
- `clap` - Command line parsing
- `serde` - Serialization/deserialization
- `aws-sdk-ecr` - AWS ECR client
- `docker-api` - Docker API client (Windows/fallback)
- Platform detection utilities for copy method selection

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Original PowerShell implementation for inspiration and workflow design
- Docker ecosystem for image reference format conventions (`repository:tag`)
- Skopeo project for efficient registry-to-registry copying capabilities
- AWS ECR documentation and best practices
- CLI tools like `brew doctor`, `flutter doctor` for diagnostic command inspiration
- Rust community for excellent tooling and libraries