# Manjaro Package Builder & Repository Management System

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Workflow Execution](#workflow-execution)
5. [Repository Management](#repository-management)
6. [Package Building Process](#package-building-process)
7. [Configuration Guide](#configuration-guide)
8. [Troubleshooting](#troubleshooting)
9. [Security Considerations](#security-considerations)
10. [Advanced Topics](#advanced-topics)

## Overview

This system automates building, managing, and distributing Arch/Manjaro Linux packages through GitHub Actions. It serves as a complete CI/CD pipeline for custom package repositories.

### Key Features
- **Automated Package Building**: Builds both AUR packages and custom local packages
- **Repository Management**: Creates and maintains Arch Linux package repositories
- **Version Synchronization**: Automatically updates PKGBUILDs to match built versions
- **Network Diagnostics**: Comprehensive connectivity testing and troubleshooting
- **Retry Logic**: Robust error handling with automatic retries for uploads
- **Artifact Management**: Built packages stored as GitHub artifacts

### Target Audience
- Package maintainers who want to automate their build process
- System administrators managing custom repositories
- Developers distributing software for Arch/Manjaro
- CI/CD practitioners interested in Linux package automation

## Architecture

### System Diagram
```
GitHub Repository (PKGBUILDs) 
       ↓
GitHub Actions Workflow (Trigger)
       ↓
Arch Linux Container (Build Environment)
       ↓
├─ AUR Packages (Cloned from AUR)
├─ Local Packages (From repository)
       ↓
Package Building (makepkg)
       ↓
Repository Database Generation (repo-add)
       ↓
Remote VPS Repository (Storage & Distribution)
       ↓
Pacman Clients (Consumers)
```

### Data Flow
1. **Source**: GitHub repository with PKGBUILDs and configuration
2. **Build**: Arch Linux container executes builder.py
3. **Dependencies**: yay for AUR, pacman for system packages
4. **Storage**: VPS hosts the repository via HTTP/HTTPS
5. **Distribution**: Clients access via pacman with configured repository

### Technology Stack
- **GitHub Actions**: CI/CD orchestration
- **Arch Linux**: Build environment (containerized)
- **Python**: Builder script logic
- **Bash**: Environment setup and diagnostics
- **SSH/rsync**: Secure file transfer to VPS
- **Pacman/Repo-add**: Package management and repository tools

## Core Components

### 1. GitHub Actions Workflow (`.github/workflows/workflow.yaml`)

The main orchestration file that defines the CI/CD pipeline:

```yaml
name: MPB - with diagnostic
on:
  workflow_dispatch:    # Manual trigger
  schedule:             # Daily automatic run
    - cron: '0 4 * * *'  # 4 AM UTC daily
```

#### Key Steps:
1. **Environment Setup**: Configures Arch Linux container with build tools
2. **PKGBUILD Fixes**: Handles versioning issues (epoch removal)
3. **SSH Configuration**: Sets up secure connection to VPS
4. **Package Building**: Executes builder.py for actual build process
5. **Diagnostics**: Network and connectivity testing
6. **Artifact Upload**: Stores logs and packages for debugging

### 2. Builder Script (`builder.py`)

The Python orchestrator that manages the entire build process:

#### Main Responsibilities:
- **Phase Management**: Coordinates build phases in correct order
- **Repository State**: Checks remote repository status before building
- **Package Building**: Builds AUR and local packages
- **Database Generation**: Creates repository database files
- **Upload Management**: Handles file transfer with retry logic
- **PKGBUILD Synchronization**: Updates version numbers in source

#### Critical Execution Order:
1. Disable repository in pacman.conf
2. Mirror existing packages from remote
3. Build new packages
4. Generate complete database
5. Upload everything to remote
6. Enable repository and sync pacman
7. Update PKGBUILDs in git repository

### 3. Configuration Files

#### `config.py` - Build Configuration
```python
REPO_DB_NAME = "manjaro-awesome"  # Repository name
SPECIAL_DEPENDENCIES = {           # Extra dependencies
    "gtk2": ["gtk-doc", "docbook-xsl"],
    "awesome-git": ["lua", "lgi", "imagemagick", "asciidoc"]
}
```

#### `packages.py` - Package Definitions
```python
LOCAL_PACKAGES = [          # Custom packages in this repo
    "awesome-git",
    "gtk2",
    "awesome-freedesktop-git"
]

AUR_PACKAGES = [            # Packages from Arch User Repository
    "libinput-gestures",
    "betterlockscreen",
    "simplescreenrecorder"
]
```

## Workflow Execution

### Trigger Conditions
- **Manual**: Via GitHub Actions UI (workflow_dispatch)
- **Scheduled**: Daily at 4 AM UTC (avoids peak hours)
- **Push Events**: (Optional) Can be configured for automatic builds

### Environment Preparation
1. **Container Setup**: Starts Arch Linux base-devel container
2. **System Update**: Updates container without accessing custom repository
3. **Build Tools Installation**: Installs compilers, build systems, and utilities
4. **User Configuration**: Creates 'builder' user with appropriate permissions
5. **SSH Setup**: Configures SSH keys for VPS access

### Network Diagnostics
The workflow includes comprehensive network testing:

```bash
# Tests performed:
1. Internet connectivity (ping 8.8.8.8)
2. VPS SSH port accessibility (port 22)
3. SSH connection with actual credentials
4. Repository server reachability
5. IP information collection (for fail2ban whitelisting)
6. Network interface and DNS configuration
```

### Error Handling
- **Graceful Failure**: Non-critical errors don't stop the entire build
- **Retry Logic**: Upload failures trigger a second attempt with different SSH options
- **Artifact Preservation**: Logs and packages are saved even on failure
- **Clean Exit**: Proper cleanup of temporary files

## Repository Management

### Repository Structure
```
/var/www/repo/ (on VPS)
├── manjaro-awesome.db.tar.gz      # Package metadata database
├── manjaro-awesome.files.tar.gz   # File lists database
├── awesome-git-4.0.0.r1234.gabcdef-1-x86_64.pkg.tar.zst
├── gtk2-2.24.33-3-x86_64.pkg.tar.zst
└── ... (other packages)
```

### Database Generation
The repository database is critical for pacman functionality:

```bash
# Process:
1. Collect ALL package files (mirrored + newly built)
2. Verify all packages exist locally
3. Run: repo-add manjaro-awesome.db.tar.gz *.pkg.tar.zst
4. Generate companion files.tar.gz database
```

**Important**: All packages must be present locally before database generation. This is why mirroring remote packages is mandatory.

### Version Management
- **Keep Last 3**: Only the last 3 versions of each package are kept on server
- **Automatic Cleanup**: Old versions removed after successful upload
- **PKGBUILD Sync**: Version numbers in PKGBUILDs are updated to match built versions

### Pacman Integration
```bash
# Client configuration (/etc/pacman.conf):
[manjaro-awesome]
Server = https://your-vps.example.com/repo
SigLevel = Optional TrustAll
```

## Package Building Process

### Local Packages vs AUR Packages

#### Local Packages
- **Source**: PKGBUILDs in repository subdirectories
- **Build**: Uses local sources and patches
- **Version Control**: PKGBUILDs are tracked in git
- **Synchronization**: Built versions update PKGBUILDs

#### AUR Packages
- **Source**: Cloned from Arch User Repository
- **Build**: Standard makepkg process
- **Ephemeral**: Cloned, built, then deleted
- **No Synchronization**: PKGBUILDs not tracked in this repository

### Dependency Resolution
Three-tier dependency installation strategy:

1. **System Packages First**: Use pacman for official repositories
2. **AUR Fallback**: Use yay for AUR dependencies
3. **Special Dependencies**: Extra packages defined in config.py
4. **Non-Fatal**: Failed dependencies don't stop the build (logged as warning)

### Build Optimization
```bash
# makepkg optimizations in /etc/makepkg.conf:
OPTIONS=(!debug)                    # Skip debug symbols
COMPRESSZST=(zstd -c -T0 --ultra -20 -)  # Better compression
```

### Special Cases
- **GTK2**: Long test suite disabled (--nocheck flag)
- **Epoch Handling**: Epoch values removed from PKGBUILDs to avoid version issues
- **Large Packages**: Extended timeouts for complex builds

## Configuration Guide

### Required GitHub Secrets
| Secret Name | Purpose | Format |
|-------------|---------|--------|
| `VPS_USER` | SSH username for VPS | Plain text (e.g., `root`) |
| `VPS_HOST` | VPS hostname or IP | Plain text (e.g., `vps.example.com`) |
| `VPS_SSH_KEY` | Private SSH key | Base64 or plain text |
| `REPO_SERVER_URL` | Repository URL | HTTP/HTTPS URL |
| `REMOTE_DIR` | Remote directory path | Absolute path |
| `REPO_NAME` | Repository name | Alphanumeric (optional) |

### Setting Up SSH Keys
```bash
# On your VPS:
ssh-keygen -t ed25519 -C "github-actions"
# Copy private key to GitHub Secrets
cat ~/.ssh/id_ed25519 | base64 -w0

# On GitHub:
1. Go to Repository → Settings → Secrets → Actions
2. Add new secret: VPS_SSH_KEY = (base64 encoded key)
```

### Repository Server Setup
```bash
# On VPS (example with nginx):
mkdir -p /var/www/repo
chown -R $USER:www-data /var/www/repo
chmod 755 /var/www/repo

# nginx configuration (/etc/nginx/sites-available/repo):
server {
    listen 80;
    server_name repo.example.com;
    root /var/www/repo;
    
    location / {
        autoindex on;
        try_files $uri $uri/ =404;
    }
}

# Enable site:
ln -s /etc/nginx/sites-available/repo /etc/nginx/sites-enabled/
systemctl restart nginx
```

### Pacman Client Configuration
```bash
# On client machines:
echo '[manjaro-awesome]
Server = https://repo.example.com
SigLevel = Optional TrustAll' | sudo tee -a /etc/pacman.conf

sudo pacman -Sy  # Sync databases
sudo pacman -S awesome-git  # Install from your repository
```

## Troubleshooting

### Common Issues

#### 1. SSH Connection Failures
**Symptoms**: Build fails at SSH test or rsync upload
**Solutions**:
```bash
# On VPS:
# Check SSH service
systemctl status sshd

# Check fail2ban (common culprit)
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unban <IP>

# Whitelist GitHub Actions IP
sudo fail2ban-client set sshd addignoreip <IP>

# Verify SSH key permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

#### 2. Package Build Failures
**Symptoms**: Specific packages fail to build
**Debugging**:
1. Check build logs in GitHub Actions artifacts
2. Examine builder.log for specific error messages
3. Test build manually in clean Arch container
4. Check dependency availability

#### 3. Repository Database Issues
**Symptoms**: Clients get "database is invalid" errors
**Solutions**:
```bash
# On VPS:
cd /var/www/repo
rm manjaro-awesome.*  # Remove old databases
# Rerun workflow to regenerate databases
```

#### 4. Network Diagnostics
The workflow includes Hungarian-language diagnostics that provide:
- Public IP addresses (for fail2ban whitelisting)
- Network connectivity tests
- DNS configuration checks
- SSH connection verification

### Debugging Workflow
1. **Check Artifacts**: Download build-results artifact
2. **Examine Logs**: builder.log and build_output.log
3. **Manual Testing**: Run commands from workflow in local Arch container
4. **SSH Debug**: Test SSH connection manually with same options

### Error Codes
| Code | Meaning | Action |
|------|---------|--------|
| 0 | Success | None needed |
| 1 | Build failure | Check logs for specific error |
| 2 | Configuration error | Verify GitHub Secrets |
| 3 | Network error | Check SSH/VPS connectivity |
| 4 | Package error | Individual package build failed |

## Security Considerations

### SSH Key Management
- **GitHub Secrets**: Keys stored encrypted in GitHub
- **Limited Access**: SSH keys should have minimal necessary permissions
- **Regular Rotation**: Consider periodic key rotation
- **No Password**: Keys should be passphrase-less for automation

### Repository Security
- **HTTP vs HTTPS**: Use HTTPS for production repositories
- **Signature Verification**: Consider enabling package signing
- **Access Control**: Restrict repository access if needed
- **Backup Strategy**: Regular backups of repository directory

### Container Security
- **Ephemeral Environment**: Fresh container for each build
- **User Isolation**: Build runs as non-root 'builder' user
- **Limited Scope**: Container has minimal installed packages
- **Network Restrictions**: Only required network access

### Best Practices
1. **Secret Management**: Never hardcode credentials
2. **Audit Logging**: Keep build logs for security review
3. **Dependency Scanning**: Monitor for vulnerable dependencies
4. **Access Reviews**: Regularly review who has access to GitHub repository

## Advanced Topics

### Customizing Build Process

#### Adding New Packages
1. **Local Packages**:
   ```bash
   mkdir new-package
   # Create PKGBUILD and other files
   # Add to LOCAL_PACKAGES in packages.py
   ```

2. **AUR Packages**:
   ```python
   # Add package name to AUR_PACKAGES in packages.py
   AUR_PACKAGES.append("new-aur-package")
   ```

#### Special Dependencies
```python
# In config.py:
SPECIAL_DEPENDENCIES = {
    "package-name": ["extra-dep1", "extra-dep2"],
    # Built-in conversions (jack → jack2)
    "simplescreenrecorder": ["jack2"],
}
```

#### Build Timeouts
```python
# In config.py:
MAKEPKG_TIMEOUT = {
    "default": 3600,          # 1 hour
    "large_packages": 7200,   # 2 hours
    "specific-package": 5400, # Custom timeout
}
```

### Extending the System

#### Additional Package Sources
The system can be extended to support:
1. **Git Repositories**: Clone and build from git
2. **Local Sources**: Pre-downloaded source tarballs
3. **Multiple AUR Helpers**: Support for paru, aura, etc.

#### Parallel Building
Potential enhancement for faster builds:
```python
# Concept for parallel execution:
import concurrent.futures

with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(self._build_single_package, pkg, is_aur): pkg 
               for pkg in package_list}
```

#### Notifications
Add notification integrations:
- **Slack/Teams**: Build status notifications
- **Email**: Failure alerts
- **Webhook**: Custom integrations

### Performance Optimization

#### Caching Strategies
```yaml
# In workflow.yaml:
- uses: actions/cache@v3
  with:
    path: ~/.cache/yay
    key: ${{ runner.os }}-yay-${{ hashFiles('packages.py') }}
```

#### Build Matrix
Build for multiple architectures or configurations:
```yaml
# Multi-architecture support:
strategy:
  matrix:
    arch: [x86_64, aarch64]
```

#### Dependency Caching
Cache downloaded sources and built packages between runs.

### Monitoring and Metrics

#### Build Statistics
Track metrics over time:
- Build success/failure rates
- Package build times
- Repository growth
- Network transfer statistics

#### Health Checks
Automated repository validation:
```bash
# Verify repository integrity:
paccache -r -k 3  # Clean old packages
repoctl status     # Check repository health
pacman -Syy        # Test repository access
```

## Contributing

### Development Workflow
1. **Fork Repository**: Create your own copy
2. **Create Branch**: Feature or bugfix branch
3. **Make Changes**: With appropriate testing
4. **Submit PR**: With description of changes
5. **Review Process**: Code review and testing

### Testing Guidelines
1. **Local Testing**: Test in Arch Linux container
2. **Integration Testing**: Full workflow test
3. **Backward Compatibility**: Don't break existing functionality
4. **Documentation**: Update README for new features

### Code Style
- **Python**: PEP 8 compliance
- **Bash**: ShellCheck compliance
- **YAML**: Proper indentation and structure
- **Comments**: Clear, concise, and helpful

## Support

### Getting Help
1. **GitHub Issues**: Report bugs or request features
2. **Discussions**: General questions and community support
3. **Documentation**: This README and code comments

### Community Resources
- **Arch Wiki**: Comprehensive Arch Linux documentation
- **AUR Documentation**: Package submission guidelines
- **GitHub Actions Docs**: Workflow syntax and features

### Maintenance
- **Regular Updates**: Keep dependencies current
- **Security Updates**: Prompt response to vulnerabilities
- **Compatibility**: Support new Arch/Manjaro releases

## License

This project is open source and available under the MIT License. See LICENSE file for details.

## Acknowledgments

- **Arch Linux Team**: For the excellent package system
- **GitHub**: For providing free CI/CD for open source
- **AUR Maintainers**: For the vast collection of user packages
- **Community Contributors**: For improvements and bug reports

---

*This README is a living document. Please update it when making significant changes to the system.*
```

This comprehensive README.md provides:

1. **Complete Overview**: What the system does and who it's for
2. **Technical Architecture**: How components interact
3. **Step-by-Step Guides**: Configuration and setup instructions
4. **Troubleshooting**: Common issues and solutions
5. **Security Best Practices**: Important considerations for production use
6. **Advanced Topics**: Extensibility and customization options
7. **Contributing Guidelines**: For community involvement
8. **Reference Material**: Commands, configurations, and examples

The documentation is structured to be useful for:
- **New Users**: Getting started with basic setup
- **Maintainers**: Day-to-day operations and troubleshooting
- **Developers**: Extending and customizing the system
- **Administrators**: Security and deployment considerations
