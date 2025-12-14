# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.1] - 2025-12-14

### Fixed

#### tnl (Tunnel Client)
- **v1.0.0 → v1.2.x update path** - Fixed update failure for v1.0.0 installations
  - v1.0.0 didn't have `/etc/tnl/config.yml` (introduced in v1.1.0)
  - Update now checks for systemd service OR SSH key instead of just config file
  - Automatic config migration: parses systemd service to create `config.yml`
  - Extracts gateway IP, admin_port, ssh_port from existing service file
  - Seamless upgrade from v1.0.0 without manual intervention

- **Missing shutil import** - Fixed import error in tnl update
  - `shutil` was imported locally in functions instead of globally
  - Caused "cannot access local variable 'shutil'" error during update
  - Now imported at module level

#### gtwy (Gateway Manager)
- **Configuration validation** - Changed from errors to warnings
  - Old configs with different structure no longer fail validation
  - Missing recommended keys now show as warnings, not errors
  - Non-critical warnings don't prevent successful update
  - Clear messaging: "These are non-critical, gtwy should work normally"

- **SSH command restriction** - Fixed authorized_keys for service tunnels
  - Changed from hardcoded `gtwy request` to `gtwy $SSH_ORIGINAL_COMMAND`
  - Allows boxes to call both `request` and `release` with arguments
  - Path updated from `/opt/gtwy/gtwy` to `/usr/local/bin/gtwy`
  - **BREAKING**: Existing boxes need authorized_keys entry manually updated

- **Logging permissions** - Fixed permission error for SSH commands
  - SSH commands (via tunneluser) now use console-only logging
  - No longer tries to write to `/opt/gtwy/gtwy.log` (permission denied)
  - Regular admin commands still use file logging

### Technical
- Added v1.2.1 to version sequences in migration runners
- Improved error handling for config parsing edge cases
- Better user feedback during update process
- Console-only logging for SSH-invoked commands (BOX_ID env var detection)

### Migration Notes (v1.0.0 → v1.2.1)
**This release specifically fixes the v1.0.0 → v1.2.x upgrade path.**

If you got this error on v1.2.0:
```
❌ tnl not configured. Run 'sudo tnl setup' first.
```

Now with v1.2.1:
```bash
sudo ./tnl update
# Works! Automatically creates config.yml from systemd service
```

**IMPORTANT - Manual Step for Existing Boxes:**

If you registered boxes before v1.2.1, you need to update their authorized_keys entry on the gateway:

```bash
# On gateway server:
sudo nano /home/tunneluser/.ssh/authorized_keys

# Find the line for your box:
# OLD (broken):
command="BOX_ID=xyz /opt/gtwy/gtwy request",restrict,port-forwarding ssh-ed25519 AAAA...

# Change to:
# NEW (working):
command="BOX_ID=xyz /usr/local/bin/gtwy $SSH_ORIGINAL_COMMAND",restrict,port-forwarding ssh-ed25519 AAAA...
```

**New boxes** (added with v1.2.1) get the correct entry automatically.

## [1.2.0] - 2025-12-14

### Added

#### gtwy (Gateway Manager)
- **`update` command** - Update gtwy to new version while preserving configuration
  - Automatic backup creation (`/opt/gtwy/backup-YYYYMMDD-HHMMSS/`)
  - Preserves `config.yml` (IONOS API keys, gateway settings)
  - Preserves `tunnels.db` (boxes and tunnels)
  - Database schema migration system
  - Version detection from installed binary or database
  - Validates configuration after update
  - `--force` flag to force update even if same version

- **Enhanced `add-box` output** - Copy-paste ready setup instructions
  - Displays complete `tnl setup` command with correct IP and ports
  - Clear step-by-step instructions for box setup
  - Formatted box with visual separators

#### tnl (Tunnel Client)
- **`update` command** - Update tnl to new version while preserving configuration
  - Automatic backup creation (`/etc/tnl/backup-YYYYMMDD-HHMMSS/`)
  - Preserves `config.yml` (gateway details, service tunnels)
  - Preserves SSH keys (`/home/tunneluser/.ssh/tunnel_key*`)
  - Graceful service restart (stops all tnl services, updates, restarts)
  - Config migration system for future upgrades
  - Real-time service status after restart
  - `--force` flag to force update even if same version

### Changed
- Project repository renamed to `tnld` (Tunnel Daemon Infrastructure)
  - Repository: https://github.com/steedalot/tnld
  - Tool names unchanged: `gtwy` and `tnl` remain the same

### Technical

#### Update Mechanism
- **Version detection**: Queries installed binary `version` command or fallback to metadata
- **Migration framework**: Sequential migration runner for future schema/config changes
- **Backup system**: Timestamped backups before any changes
- **Rollback capability**: Manual rollback by restoring from backup directory

#### Database Migrations (gtwy)
- New `metadata` table for schema version tracking
- Migration runner: `run_database_migrations(conn, from_version, to_version)`
- Extensible for future versions (v1.3.0+)

#### Config Migrations (tnl)
- Config metadata: `_meta.schema_version` in `config.yml`
- Migration runner: `run_config_migrations(config, from_version, to_version)`
- Extensible for future config format changes

### Migration Notes (v1.1.0 → v1.2.0)
- **No breaking changes** - v1.2.0 is fully backward compatible with v1.1.0
- **Recommended upgrade order**: Update gtwy first, then tnl on boxes
- **Update command**:
  ```bash
  # Gateway
  curl -LO https://github.com/steedalot/tnld/releases/latest/download/gtwy
  chmod +x gtwy
  sudo ./gtwy update

  # Box
  curl -LO https://github.com/steedalot/tnld/releases/latest/download/tnl
  chmod +x tnl
  sudo ./tnl update
  ```
- **Rollback** (if needed):
  ```bash
  # gtwy
  sudo cp /opt/gtwy/backup-YYYYMMDD-HHMMSS/gtwy.old /usr/local/bin/gtwy

  # tnl
  sudo cp /etc/tnl/backup-YYYYMMDD-HHMMSS/tnl.old /usr/local/bin/tnl
  sudo systemctl restart tnl-admin.service
  ```

## [1.1.0] - 2025-12-14

### Added

#### gtwy (Gateway Manager)
- JSON response format for `request` command
  - Returns both `server_port` and `subdomain` as JSON
  - Enables tnl to display full URLs to users
  - More extensible for future enhancements

#### tnl (Tunnel Client)
- **`add <service> <port>` command** - Add service tunnel
  - Requests tunnel from gateway via SSH
  - Creates systemd service for persistent tunnel (tnl-<service>.service)
  - Displays full subdomain URL
  - Automatic rollback on failure (atomic operation)
  - One systemd service per tunnel for easy management

- **`remove <service>` command** - Remove service tunnel
  - Stops and disables systemd service
  - Releases tunnel on gateway (DNS, nginx, SSL cleanup)
  - Removes from local configuration
  - Clean teardown of all resources

- **`list` command** - List all service tunnels
  - Shows service name, local port, server port, URL, and status
  - Real-time systemd status check (active/inactive)
  - Formatted table output with totals

- **Central configuration file: `/etc/tnl/config.yml`**
  - Gateway connection details (IP, ports)
  - Service tunnel tracking
  - YAML format (human-readable, supports comments)
  - Permissions: root:root 644

### Changed
- `tnl setup` now creates `/etc/tnl/config.yml` with gateway connection details
- `gtwy request` output format: plain text → JSON (**BREAKING CHANGE**)
  - Old: `10001`
  - New: `{"server_port": 10001, "subdomain": "gitea.box01.kibox.online"}`
- Updated `tnl status` help text to clarify it shows admin tunnel status

### Technical
- Dependencies: Added PyYAML to tnl (already used in gtwy)
- Config location: `/etc/tnl/config.yml` (root:root 644)
- One systemd service per tunnel: `/etc/systemd/system/tnl-<service>.service`
- Service tunnels depend on admin tunnel (systemd `Requires=tnl-admin.service`)
- Atomic operations with rollback on failure
- Input validation: service names (lowercase, numbers, hyphens), ports (1-65535)

### Migration Notes (v1.0.0 → v1.1.0)
- **gtwy**: Update to v1.1.0 first (breaking change in `request` output)
- **tnl**: Update to v1.1.0 after gtwy is updated
- Existing v1.0.0 installations: No migration needed, config created on first `tnl add`
- New installations: Use `tnl setup` as before, config created automatically

## [1.0.0] - 2025-12-13

### Added

#### gtwy (Gateway Manager)
- `install` command - One-time system setup
  - Creates `/opt/gtwy` directory structure
  - Creates `tunneluser` and `gtwy-admin` group
  - Sets up SSH directory and permissions
  - Creates sudoers rules for nginx and certbot
  - Installs script globally to `/usr/local/bin/gtwy`
  - Embeds nginx template as constant (single-file tool)

- `setup` command - Interactive configuration wizard
  - IONOS DNS API configuration
  - Server IP and hostname detection
  - Domain whitelist configuration
  - Port range allocation (services and admin SSH)
  - Certbot email and staging mode
  - Creates `config.yml` and initializes SQLite database

- `add-box` command - Register new IT.Box
  - SSH public key validation and fingerprinting
  - Per-box domain assignment
  - Automatic admin SSH port allocation (20000-20999 range)
  - authorized_keys management with command restriction

- `remove-box` command - Remove box and all tunnels
  - Cascading tunnel deletion
  - DNS cleanup
  - Certificate revocation
  - authorized_keys cleanup

- `list-boxes` command - Show registered boxes
  - Display box ID, domain, SSH key info, admin port
  - Show tunnel count per box
  - Last seen timestamp

- `get-port` command - Retrieve admin SSH port for box

- `request` command - Create service tunnel (called by box via SSH)
  - Automatic server port allocation (10000-19999 range)
  - nginx configuration generation
  - DNS A record creation (IONOS API)
  - SSL certificate scheduling
  - Subdomain pattern: `<service>.<box-id>.<domain>`

- `release` command - Remove service tunnel (called by box via SSH)
  - nginx cleanup
  - DNS record deletion
  - Certificate revocation

- `list` command - Show all active tunnels
- `info` command - Detailed tunnel information
- `version` command - Show version

#### tnl (Tunnel Client)
- `install` command - One-time client setup
  - Creates `tunneluser` with home directory
  - Generates ED25519 SSH key pair
  - Installs autossh via apt
  - Copies script to `/usr/local/bin/tnl`
  - Displays public key for gateway registration
  - Handles existing users gracefully

- `setup` command - Configure admin SSH tunnel
  - Tests SSH connection to gateway
  - Creates systemd service for persistent tunnel
  - Configures autossh with proper reconnection settings
  - Reverse tunnel: `Gateway:{admin_port} → Box:22`
  - Auto-starts on boot
  - Supports custom SSH ports

- `status` command - Show tunnel status
  - systemd service status
  - Recent journal logs

- `version` command - Show version

### Security
- SSH key-based authentication only
- Command restriction in authorized_keys (no shell access)
- `restrict` flag prevents shell access, only port-forwarding allowed
- GatewayPorts disabled (admin tunnels only accessible from localhost)
- Minimal sudo permissions (only nginx reload and certbot)
- Separate port ranges for admin SSH and service tunnels

### Documentation
- Complete README for gtwy with command reference
- Complete README for tnl with installation guide
- TESTING.md with 4-phase test plan (local, server setup, staging, production)
- CLAUDE.md technical specification
- Architecture diagrams
- Troubleshooting guides

### Technical
- Single-file Python scripts (gtwy ~2600 lines, tnl ~400 lines)
- Embedded nginx template in gtwy (no external files needed)
- SQLite database for state management
- Python 3.8+ compatible
- Dependencies: PyYAML, requests (gtwy only)
- Idempotent installation (can run install multiple times)

### Fixed
- Home directory creation when tunneluser exists without home
- SSH connection test now correctly handles command restriction
- Port confusion between admin_port (reverse tunnel) and ssh_port (connection)
- Proper separation of admin tunnels (static) and service tunnels (dynamic)

## [Unreleased]

### Planned for v1.1.0
- `tnl add <service> <port>` - Add service tunnel via admin tunnel
- `tnl remove <service>` - Remove service tunnel
- `tnl list` - List all service tunnels for this box
- Communication with gateway via admin SSH tunnel for tunnel requests

### Planned for v1.2.0
- `gtwy stats` - Gateway statistics
- `gtwy health` - System health check
- `gtwy test-tunnel` - End-to-end tunnel testing
- `gtwy sync-nginx` - Rebuild nginx from database
- `gtwy sync-dns` - Sync DNS records
- `gtwy sync-certs` - Request pending certificates
- `gtwy cleanup` - Remove orphaned resources

### Future Considerations
- Web dashboard
- Prometheus metrics export
- Automatic tunnel health monitoring
- Multi-gateway support (HA)
- Backup/restore functionality
- Rate limiting per box
- Webhook notifications

---

[1.0.0]: https://github.com/yourusername/tunnel-gateway/releases/tag/v1.0.0
