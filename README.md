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

# Local development using AWS profiles
ci-copy --source-profile source-profile --target-profile target-profile \
        --repository your-app-service \
        --tag release-v.1.0.0 \
        --region us-east-1

# CI/CD using IAM roles (recommended for automated environments)
ci-copy --source-role arn:aws:iam::123456789012:role/ECRSourceRole \
        --target-role arn:aws:iam::987654321098:role/ECRTargetRole \
        --images "app-service:v1.0.0,web-service:v1.0.1" \
        --ci-mode --output json

# Use default configuration file (ci-copy.yaml in current directory)
ci-copy

# Environment-specific configuration files
ci-copy -c ci-copy-dev.yaml    # Development
ci-copy -c ci-copy-prod.yaml   # Production

# Copy multiple images with inline specification
ci-copy --images "your-app-service:release-v.1.0.0,your-web-service:release-v.1.0.1"

# CI/CD optimized: parallel, timeout, retry configuration
ci-copy --ci-mode --parallel 4 --timeout 1800 --retry 3 \
        --images "app1:v1,app2:v1,app3:v1" \
        --output json

# Force Docker method on Linux/macOS (useful for testing)
ci-copy --source-profile source-profile --target-profile target-profile \
        --repository your-app-service \
        --tag release-v.1.0.0 \
        --force-docker

# Fast mode for CI with minimal verification
ci-copy --ci-mode --fast --no-verify \
        --images "quick-deploy:latest" \
        --quiet
```

### Configuration File Example

Create a `ci-copy.yaml` file (default configuration file name):

```yaml
# Development/Local configuration (using profiles)
source:
  profile: source-profile
  region: us-east-1

target:
  profile: target-profile
  region: us-east-1

# CI/CD configuration (using IAM roles)
# source:
#   role: arn:aws:iam::123456789012:role/ECRSourceRole
#   region: us-east-1
#   session_name: ci-copy-dev-session
#   duration: 3600

# target:
#   role: arn:aws:iam::987654321098:role/ECRTargetRole
#   region: us-east-1
#   session_name: ci-copy-prod-session
#   duration: 3600

# Simple string format (recommended - Docker compatible)
images:
  - your-app-service:release-v.1.0.0
  - your-web-service:release-v.1.0.1
  - another-service:latest
  - microservice-api:v2.1.3

# CI/CD specific settings
settings:
  parallel: 4
  timeout: 1800  # 30 minutes
  retry_count: 3
  retry_delay: 30
  verify: true
  ci_mode: false  # Set to true for CI/CD environments
  output_format: text  # text or json
  log_level: info  # debug, info, warn, error

# Alternative object format (for advanced use cases with future extensibility)
# images:
#   - repository: your-app-service
#     tag: release-v.1.0.0
#   - repository: your-web-service  
#     tag: release-v.1.0.1
#     # Future: additional fields like platform, digest, etc.
```

### Environment-Specific Configuration Files

```yaml
# ci-copy-dev.yaml (Development environment)
source:
  profile: dev-profile
  region: us-east-1

target:
  profile: staging-profile
  region: us-east-1

settings:
  parallel: 2
  verify: true
  log_level: debug

images:
  - app-service:dev-latest
  - web-service:dev-latest
```

```yaml
# ci-copy-prod.yaml (Production CI/CD)
source:
  role: arn:aws:iam::123456789012:role/ECRSourceRole
  region: us-east-1
  session_name: prod-deployment

target:
  role: arn:aws:iam::987654321098:role/ECRTargetRole
  region: us-east-1
  session_name: prod-deployment

settings:
  parallel: 4
  timeout: 3600
  retry_count: 5
  verify: true
  ci_mode: true
  output_format: json
  fail_fast: false

images:
  - app-service:${IMAGE_TAG}
  - web-service:${IMAGE_TAG}
  - api-service:${IMAGE_TAG}
```

### Input Methods Comparison

| Method | Format | Use Case | Example |
|--------|--------|----------|---------|
| **Config File (Local)** | `profile` | Development, manual ops | `ci-copy -c dev.yaml` |
| **Config File (CI/CD)** | `role` | Automated deployments | `ci-copy -c prod.yaml` |
| **Inline Images** | `repo:tag,repo:tag` | Quick batch copy | `--images "app:v1,web:v2"` |
| **Single Image** | Separate params | One-off copy | `--repository app --tag v1` |
| **Environment Variables** | `AWS_*` vars | CI/CD with dynamic values | `AWS_ROLE_ARN=... ci-copy` |

> **Note**: 
> - The application automatically looks for `ci-copy.yaml` in the current directory when no config file is specified
> - Use the string format (`repository:tag`) for simplicity and Docker compatibility
> - For CI/CD: Use IAM roles instead of profiles for better security and credential management
> - Environment variable substitution is supported in config files (e.g., `${IMAGE_TAG}`)
> - Replace example ARNs and profiles with your actual AWS resources

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

## CI/CD Pipeline Integration

CI Copy is designed to work seamlessly in automated CI/CD environments with support for assume roles, environment variables, and non-interactive operations.

### IAM Role Support

```bash
# Using assume role (recommended for CI/CD)
ci-copy --source-role arn:aws:iam::123456789012:role/ECRSourceRole \
        --target-role arn:aws:iam::987654321098:role/ECRTargetRole \
        --images "app-service:v1.0.0,web-service:v1.0.1" \
        --region us-east-1

# Using environment variables (common in CI/CD)
export AWS_ROLE_ARN="arn:aws:iam::123456789012:role/ECRRole"
export AWS_ROLE_SESSION_NAME="ci-copy-session"
ci-copy --images "app:v1.0.0" --region us-east-1

# Mixed mode: different roles for source and target
ci-copy --source-role arn:aws:iam::123456789012:role/DevRole \
        --target-role arn:aws:iam::987654321098:role/ProdRole \
        --config ci-copy.yaml
```

### Environment Variable Configuration

CI Copy supports standard AWS environment variables and CI-specific configurations:

```bash
# AWS Credentials (standard)
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
export AWS_DEFAULT_REGION="us-east-1"

# Assume Role Configuration
export AWS_ROLE_ARN="arn:aws:iam::ACCOUNT:role/ROLE_NAME"
export AWS_ROLE_SESSION_NAME="ci-copy-session-$(date +%s)"
export AWS_ROLE_DURATION_SECONDS="3600"

# CI Copy Specific
export CI_COPY_CONFIG="production.yaml"
export CI_COPY_PARALLEL="4"
export CI_COPY_TIMEOUT="1800"  # 30 minutes
export CI_COPY_RETRY_COUNT="3"
export CI_COPY_LOG_LEVEL="info"  # info, warn, error, debug
```

### CI-Optimized Features

#### Non-Interactive Mode
```bash
# Fully automated operation with no user prompts
ci-copy --ci-mode --images "app:v1.0.0" --region us-east-1

# Machine-readable JSON output for CI parsing
ci-copy --output json --images "app:v1.0.0" > copy-results.json

# Quiet mode (errors only)
ci-copy --quiet --images "app:v1.0.0"
```

#### Timeout and Retry Configuration
```bash
# Configure timeouts and retries for CI stability
ci-copy --timeout 1800 --retry 3 --retry-delay 30 \
        --images "large-app:v1.0.0"

# Fail fast mode for quick CI feedback
ci-copy --fail-fast --images "app1:v1,app2:v2,app3:v3"
```

### Sample CI Pipeline Configurations

#### GitHub Actions
```yaml
name: Copy Container Images
on:
  push:
    tags: ['v*']

jobs:
  copy-images:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
        aws-region: us-east-1
    
    - name: Download CI Copy
      run: |
        curl -L https://github.com/your-org/ci-copy/releases/latest/download/ci-copy-linux > ci-copy
        chmod +x ci-copy
    
    - name: Check dependencies
      run: ./ci-copy check --ci-mode
    
    - name: Copy images to production
      run: |
        ./ci-copy --ci-mode --output json \
          --source-role arn:aws:iam::123456789012:role/DevECRRole \
          --target-role arn:aws:iam::987654321098:role/ProdECRRole \
          --images "myapp:${{ github.ref_name }}" \
          --timeout 1800 --retry 3
```

#### GitLab CI
```yaml
stages:
  - build
  - deploy

copy-to-production:
  stage: deploy
  image: alpine:latest
  variables:
    AWS_DEFAULT_REGION: us-east-1
    CI_COPY_LOG_LEVEL: info
  before_script:
    - apk add --no-cache curl
    - curl -L https://github.com/your-org/ci-copy/releases/latest/download/ci-copy-linux > ci-copy
    - chmod +x ci-copy
  script:
    - ./ci-copy check --ci-mode
    - |
      ./ci-copy --ci-mode --quiet \
        --source-role $SOURCE_ECR_ROLE \
        --target-role $TARGET_ECR_ROLE \
        --images "$CI_PROJECT_NAME:$CI_COMMIT_TAG" \
        --region $AWS_DEFAULT_REGION
  only:
    - tags
```

#### Jenkins Pipeline
```groovy
pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        CI_COPY_TIMEOUT = '1800'
        CI_COPY_RETRY_COUNT = '3'
    }
    
    stages {
        stage('Setup') {
            steps {
                sh '''
                    curl -L https://github.com/your-org/ci-copy/releases/latest/download/ci-copy-linux > ci-copy
                    chmod +x ci-copy
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                sh './ci-copy check --ci-mode --output json > health-check.json'
                archiveArtifacts artifacts: 'health-check.json'
            }
        }
        
        stage('Copy Images') {
            steps {
                withAWS(role: 'arn:aws:iam::123456789012:role/JenkinsECRRole') {
                    sh '''
                        ./ci-copy --ci-mode --output json \
                          --images "${IMAGE_NAME}:${BUILD_NUMBER}" \
                          --timeout ${CI_COPY_TIMEOUT} \
                          --retry ${CI_COPY_RETRY_COUNT} > copy-results.json
                    '''
                }
                archiveArtifacts artifacts: 'copy-results.json'
            }
        }
    }
}
```

### Security Best Practices for CI/CD

#### Credential Management
```bash
# Use temporary credentials with limited scope
export AWS_ROLE_ARN="arn:aws:iam::ACCOUNT:role/ECRCopyRole"
export AWS_ROLE_SESSION_NAME="ci-copy-$(uuidgen)"

# Limit role duration
export AWS_ROLE_DURATION_SECONDS="3600"  # 1 hour max

# Use specific ECR permissions only
# ECR_COPY_POLICY: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:PutImage
```

#### Audit and Monitoring
```bash
# Enable detailed logging for audit trails
ci-copy --ci-mode --audit-log audit.log \
        --log-level debug \
        --output json

# Include request IDs and timing for monitoring
ci-copy --ci-mode --include-metrics \
        --output json > metrics.json
```

### Exit Codes for CI Integration

CI Copy returns specific exit codes for different scenarios:

| Exit Code | Description | CI Action |
|-----------|-------------|-----------|
| `0` | Success | Continue pipeline |
| `1` | General error | Fail pipeline |
| `2` | Configuration error | Fix config and retry |
| `3` | Authentication error | Check credentials |
| `4` | Permission error | Review IAM policies |
| `5` | Network/timeout error | Retry with backoff |
| `6` | Source image not found | Check image exists |
| `7` | Target repository error | Verify target setup |

### Performance Optimization for CI

```bash
# Parallel processing for multiple images
ci-copy --ci-mode --parallel 4 \
        --images "app1:v1,app2:v1,app3:v1,app4:v1"

# Skip verification for faster CI (when speed > safety)
ci-copy --ci-mode --no-verify --fast \
        --images "app:v1.0.0"

# Use compressed output for large logs
ci-copy --ci-mode --compress-logs \
        --output json | gzip > results.json.gz
```

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
| **CI/CD Integration** | ❌ Limited | ✅ Full CI/CD support |
| **IAM Role Support** | ❌ Profile-based only | ✅ Assume role for CI/CD |
| **Non-Interactive Mode** | ❌ None | ✅ `--ci-mode` flag |
| **JSON Output** | ❌ Text only | ✅ Machine-readable output |
| **Timeout/Retry** | ❌ Basic | ✅ Configurable timeout/retry |
| **Environment Variables** | ❌ Limited | ✅ Full environment support |
| **Exit Codes** | ❌ Generic | ✅ Specific error codes |
| **Audit Logging** | ❌ None | ✅ Detailed audit trails |
| **Security** | ❌ Profile credentials | ✅ Temporary credentials, IAM roles |

## Development

### Building

```bash
# Debug build
cargo build

# Release build with optimizations
cargo build --release

# Run tests
cargo test

# Test different environments
cargo test -- --test-threads=1  # Sequential tests for AWS integration

# Run system check for development
RUST_LOG=debug cargo run -- check --verbose

# Test CI/CD mode locally
RUST_LOG=debug cargo run -- check --ci-mode --output json

# Test copy functionality with logging
RUST_LOG=debug cargo run -- copy --config test-config.yaml --verbose

# Test IAM role integration (requires AWS setup)
export AWS_ROLE_ARN="arn:aws:iam::123456789012:role/TestRole"
cargo run -- check --role $AWS_ROLE_ARN --verbose
```

### Testing Dependencies

Before running tests, ensure your development environment has the required dependencies:

```bash
# Check development environment
cargo run -- check --verbose

# Test with different copy methods
cargo run -- copy --force-docker --repository test-repo --tag test-tag

# CI/CD integration testing
cargo run -- --ci-mode --output json --timeout 60 --retry 1 \
             --images "test-app:latest"

# Performance testing with parallel operations
cargo run -- copy --parallel 4 --images "app1:v1,app2:v1,app3:v1,app4:v1"
```

### CI/CD Testing

```bash
# Test assume role functionality (requires proper IAM setup)
export AWS_ROLE_ARN="arn:aws:iam::ACCOUNT:role/TestRole"
export AWS_ROLE_SESSION_NAME="test-session"
cargo run -- check --ci-mode

# Test environment variable configuration
export CI_COPY_CONFIG="test-config.yaml"
export CI_COPY_PARALLEL="2"
export CI_COPY_TIMEOUT="300"
cargo run -- copy --ci-mode

# Test JSON output parsing
cargo run -- check --ci-mode --output json | jq '.summary.ready'
```

### Dependencies

Key dependencies include:
- `tokio` - Async runtime
- `clap` - Command line parsing with subcommands
- `serde` - Serialization/deserialization
- `aws-sdk-ecr` - AWS ECR client
- `aws-sdk-sts` - AWS STS for role assumption
- `docker-api` - Docker API client (Windows/fallback)
- Platform detection utilities for copy method selection
- JSON output formatting for CI/CD integration

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