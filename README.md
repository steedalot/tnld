# tnld - Tunnel Daemon Infrastructure

Self-hosted SSH reverse tunnel management system for exposing services behind NAT/firewall.

## Overview

**tnld** provides two complementary tools for managing SSH reverse tunnels:

- **gtwy** - Server-side gateway manager with automated DNS, SSL, and nginx configuration
- **tnl** - Client-side tunnel manager for IT.Boxes

Perfect for self-hosted infrastructure where services need to be exposed through a central gateway server.

## Features

### gtwy (Gateway Server)

- 🔧 **Automated Setup** - One-command installation and configuration
- 🌐 **DNS Automation** - IONOS DNS API integration
- 🔒 **SSL Certificates** - Automatic Let's Encrypt certificate management
- 🔄 **nginx Integration** - Dynamic reverse proxy configuration
- 📊 **Multi-Domain** - Support for multiple domains and subdomains
- 🗄️ **SQLite Database** - Built-in state management
- 🔑 **SSH-based Auth** - Secure key-based authentication
- 🔄 **Update Mechanism** - Preserve configuration across updates

### tnl (Tunnel Client)

- 🚀 **Easy Installation** - Automated user and key setup
- 🔄 **Persistent Tunnels** - systemd + autossh for reliability
- 📡 **Admin Tunnel** - Reverse SSH access for management
- ⚡ **Auto-Reconnect** - Automatic recovery from network issues
- 🎯 **Service Tunnels** - Manage HTTP/HTTPS service tunnels dynamically
- 🔄 **Update Mechanism** - Preserve configuration across updates

## Architecture

```
┌────────────────────────────────────────────────────┐
│                  IT.Box (Client)                   │
│  ┌──────────────────────────────────────────────┐  │
│  │  Services: Gitea, Theia, Portainer, etc.     │  │
│  │  Ports: 3000, 8080, 9000, ...                │  │
│  │  tnl - Tunnel Client                         │  │
│  └────────────────┬─────────────────────────────┘  │
│                   │ SSH Reverse Tunnel             │
└───────────────────┼────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────┐
│               Gateway Server (gtwy)                │
│  ┌──────────────────────────────────────────────┐  │
│  │  gtwy - Gateway Manager                      │  │
│  │  - SSH tunnels (ports 10000-19999)           │  │
│  │  - nginx reverse proxy                       │  │
│  │  - Let's Encrypt SSL                         │  │
│  │  - IONOS DNS automation                      │  │
│  └────────────────┬─────────────────────────────┘  │
└───────────────────┼────────────────────────────────┘
                    │
                    ▼
              🌍 Internet
    https://gitea.box01.kibox.online
    https://theia.box01.kibox.online
    https://portainer.box01.kibox.online
```

## Quick Start

### Server Setup (Gateway)

```bash
# Download gtwy
curl -LO https://github.com/steedalot/tnld/releases/latest/download/gtwy
chmod +x gtwy

# Install (creates users, permissions, etc.)
sudo ./gtwy install

# Configure (interactive wizard)
sudo gtwy setup

# Add a box
sudo gtwy add-box box01 kibox.online '<ssh-public-key>'
```

### Client Setup (IT.Box)

```bash
# Download tnl
curl -LO https://github.com/steedalot/tnld/releases/latest/download/tnl
chmod +x tnl

# Install (creates user, generates SSH key)
sudo ./tnl install
# Copy the displayed public key and register on gateway

# Setup tunnel (get admin_port from: gtwy get-port box01)
sudo tnl setup gateway.example.com 20001

# Add service tunnel (e.g., for Gitea)
sudo tnl add gitea 3000
```

**Done!** The tunnels are now running. From the gateway server:

```bash
ssh -p 20001 user@localhost  # SSH to the box
```

Access services via HTTPS:
```
https://gitea.box01.kibox.online
```

## Updating

tnld supports seamless updates that preserve all configuration:

```bash
# Gateway
curl -LO https://github.com/steedalot/tnld/releases/latest/download/gtwy
chmod +x gtwy
sudo ./gtwy update

# Client
curl -LO https://github.com/steedalot/tnld/releases/latest/download/tnl
chmod +x tnl
sudo ./tnl update
```

## Documentation

- [gtwy Documentation](gtwy/README.md) - Server-side gateway manager
- [tnl Documentation](tnl/README.md) - Client-side tunnel manager
- [CHANGELOG](CHANGELOG.md) - Version history and upgrade notes
- [Project Structure](STRUCTURE.md) - Repository organization

## Requirements

### Gateway Server

- Ubuntu/Debian Linux
- Python 3.8+
- nginx
- certbot (Let's Encrypt)
- IONOS account with DNS API access

### IT.Box (Client)

- Ubuntu/Debian Linux
- Python 3.8+
- autossh
- OpenSSH client
- PyYAML (for service tunnel management)

## Use Cases

- **Self-hosted Services** - Expose Gitea, Nextcloud, etc. from behind NAT
- **IoT Devices** - Manage devices without public IPs
- **Remote Development** - Access code-server/Theia instances
- **Multi-tenant Hosting** - Separate domains per box
- **Educational Institutions** - School IT.Boxes with centralized gateway

## Security

- SSH key-based authentication only
- Command restriction in authorized_keys (no shell access)
- Automatic SSL/TLS via Let's Encrypt
- GatewayPorts disabled (tunnels only accessible from gateway)
- Minimal sudo permissions for tunnel operations
- Separation of admin tunnels (SSH) and service tunnels (HTTP/HTTPS)

## Versioning

This project uses [Semantic Versioning](https://semver.org/):

- **v1.2.2** - Async certificate workflow (current)
  - Instant tunnel creation (no waiting for DNS/certificates)
  - Background certificate requests with status tracking
  - HTTP-only → automatic HTTPS upgrade
  - Simplified update paths (only last stable → current)

- **v1.2.1** - Bugfix release
  - Fixed: v1.0.0 → v1.2.x update path
  - Fixed: Multiple permission issues

- **v1.2.0** - Update mechanism
  - gtwy/tnl: `update` command with automatic backups
  - Enhanced box setup instructions

See [CHANGELOG.md](CHANGELOG.md) for detailed version history.

## License

MIT License - See [LICENSE](LICENSE)

## Contributing

This is currently a private project for IT.Box infrastructure. Contributions, bug reports, and feature requests are welcome via GitHub issues.

## Support

- **Documentation**: See [CHANGELOG.md](CHANGELOG.md) and inline help (`gtwy --help`, `tnl --help`)
- **Issues**: [GitHub Issues](https://github.com/steedalot/tnld/issues)
- **Releases**: [GitHub Releases](https://github.com/steedalot/tnld/releases)

## Authors

Developed for the KI.Box / IT.Box infrastructure.

---

**tnld** - Simple. Robust. Self-hosted. 🚀
