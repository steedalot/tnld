# gtwy - Tunnel Gateway Manager

**Single-file Python tool** für zentrale Verwaltung von SSH-Reverse-Tunnels von IT.Boxes zu einem nginx-Gateway-Server.

---

## ✨ Features

- 🔧 **Single-File Tool** - Nur eine Datei, keine externe Config nötig
- 🚀 **Selbstinstallierend** - `sudo gtwy install` richtet alles automatisch ein
- 🌐 **Multi-Domain** - Unterstützt mehrere Domains gleichzeitig
- 🔒 **Automatisches SSL** - Let's Encrypt Zertifikate via certbot
- 📡 **DNS-Automation** - IONOS DNS API Integration
- 🔑 **SSH-basiert** - Sichere Authentifizierung über SSH Public Keys
- 📊 **SQLite-basiert** - Keine externe Datenbank nötig
- 🔄 **Port-Management** - Automatische Port-Allokation

---

## 🚀 Quick Start

### Installation auf Gateway-Server

```bash
# 1. Download gtwy
curl -O https://example.com/gtwy
chmod +x gtwy

# 2. Install (einmalig)
sudo ./gtwy install

# 3. Setup (konfigurieren)
sudo gtwy setup
```

### Box registrieren

```bash
# SSH-Key auf Box generieren
ssh-keygen -t ed25519 -f ~/.ssh/tunnel_key

# Box auf Gateway registrieren
sudo gtwy add-box box01 kibox.online ~/.ssh/tunnel_key.pub
```

### Tunnel von Box anfordern

```bash
# Von der Box aus
PORT=$(ssh -i ~/.ssh/tunnel_key tunneluser@gateway.example.com "request gitea 3000")

# Tunnel aufbauen
autossh -M 0 -f -N -R $PORT:localhost:3000 \
  -i ~/.ssh/tunnel_key tunneluser@gateway.example.com
```

**Ergebnis:** `gitea.box01.kibox.online` ist jetzt erreichbar! 🎉

---

## 📋 Befehle

### Installation & Setup
```bash
gtwy install          # Einmalige Installation (User, Gruppen, Permissions)
gtwy setup            # Interaktiver Konfigurations-Wizard
gtwy version          # Version anzeigen
```

### Box-Verwaltung
```bash
gtwy add-box <id> <domain> <key>   # Box registrieren
gtwy remove-box <id>                # Box entfernen
gtwy list-boxes                     # Alle Boxen anzeigen
gtwy get-port <id>                  # Admin-SSH-Port abrufen
```

### Tunnel-Verwaltung
```bash
gtwy request <service> <port>       # Tunnel anfordern (von Box)
gtwy release <service>              # Tunnel freigeben (von Box)
gtwy list                           # Alle Tunnels anzeigen
gtwy info <subdomain>               # Tunnel-Details
gtwy remove <subdomain>             # Tunnel entfernen (Admin)
```

### Monitoring
```bash
gtwy status                         # Tunnel-Health-Check
gtwy stats                          # Statistiken
gtwy health                         # System-Health-Check
gtwy test-tunnel <subdomain>        # End-to-End Test
```

### Wartung
```bash
gtwy sync-nginx                     # nginx-Config neu generieren
gtwy sync-dns                       # DNS-Records synchronisieren
gtwy sync-certs                     # Pending-Zertifikate anfordern
gtwy cleanup                        # Verwaiste Ressourcen aufräumen
gtwy rebuild                        # Kompletter Rebuild
gtwy backup                         # Backup erstellen
gtwy restore <file>                 # Backup wiederherstellen
```

---

## 🏗️ Architektur

### Komponenten

```
┌─────────────────────────────────────────────────────┐
│                   IT.Box (Client)                   │
│  ┌──────────────────────────────────────────────┐  │
│  │  Gitea, Theia, Portainer, etc.               │  │
│  │  Port 3000, 8080, 9000, ...                  │  │
│  └────────────────┬─────────────────────────────┘  │
│                   │ SSH Reverse Tunnel             │
└───────────────────┼─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│              Gateway Server (gtwy)                  │
│  ┌──────────────────────────────────────────────┐  │
│  │  tunneluser@server (SSH)                     │  │
│  │  Port 10001, 10002, 10003, ...               │  │
│  └────────────────┬─────────────────────────────┘  │
│                   │                                 │
│  ┌────────────────▼─────────────────────────────┐  │
│  │  nginx (Reverse Proxy)                       │  │
│  │  + SSL/TLS (Let's Encrypt)                   │  │
│  └────────────────┬─────────────────────────────┘  │
└───────────────────┼─────────────────────────────────┘
                    │
                    ▼
              🌍 Internet
    gitea.box01.kibox.online
    theia.box01.kibox.online
    portainer.box01.kibox.online
```

### Subdomain-Pattern

```
<service>.<box-id>.<domain>

Beispiele:
  gitea.box01.kibox.online
  theia.school-abc.itbox.niedersachsen.de
  portainer.mybox.private.example.com
```

### Verzeichnisstruktur

```
/opt/gtwy/
├── gtwy                    # Haupt-Script (single file!)
├── config.yml             # Konfiguration
├── tunnels.db             # SQLite-Datenbank
├── nginx-template         # nginx Server-Block Template
├── gtwy.log               # Log-Datei
└── backups/               # Backups

/etc/nginx/sites-enabled/
└── tunnels-autogen.conf   # Auto-generiert (nicht manuell editieren!)

/home/tunneluser/.ssh/
└── authorized_keys        # Verwaltet von gtwy
```

---

## 🔧 Konfiguration

### config.yml

```yaml
# IONOS DNS API
ionos:
  public_prefix: abc123_
  secret: xyz789secret

# Gateway Server
gateway:
  public_ip: 1.2.3.4
  hostname: gateway.kibox.online  # optional

# Erlaubte Domains (leer = alle erlaubt)
domains:
  - kibox.online
  - itbox.niedersachsen.de

# Port-Bereiche
port_range:
  services:
    start: 10000
    end: 19999
  admin_ssh:
    start: 20000
    end: 20999
    allocation: sequential  # oder: lowest

# SSL-Zertifikate
certbot:
  email: admin@kibox.online
  staging: false  # true für Tests!

# Limits
max_tunnels_per_box: 10

# Pfade (Standard)
nginx:
  config_path: /etc/nginx/sites-enabled/tunnels-autogen.conf
  template_path: /opt/gtwy/nginx-template

ssh:
  authorized_keys_path: /home/tunneluser/.ssh/authorized_keys

# Logging
logging:
  level: INFO
  file: /opt/gtwy/gtwy.log
  max_size_mb: 10
  backup_count: 5
```

---

## 🔒 Sicherheit

### SSH-Authentifizierung

Jede Box hat ihren eigenen SSH-Key. In `authorized_keys`:

```bash
command="BOX_ID=box01 /opt/gtwy/gtwy request",restrict,port-forwarding ssh-ed25519 AAAAC3...
```

- ✅ **Command-Restriction**: Box kann nur `gtwy request` ausführen
- ✅ **No Shell Access**: `restrict` verhindert Shell-Zugriff
- ✅ **Port-Forwarding Only**: Nur Tunnel-Forwarding erlaubt

### Permissions

```
/opt/gtwy/gtwy           → root:root 755
/opt/gtwy/config.yml     → root:gtwy-admin 640  (enthält API-Keys!)
/opt/gtwy/tunnels.db     → tunneluser:gtwy-admin 664
/opt/gtwy/nginx-template → root:root 644
```

### Sudo-Regeln

`tunneluser` darf **nur**:
- `nginx -t` (Config testen)
- `nginx -s reload` (Reload)
- `certbot certonly` (Zertifikate anfordern)
- `certbot delete` (Zertifikate löschen)
- `certbot certificates` (Zertifikate auflisten)

Keine Shell, keine anderen Befehle!

---

## 🧪 Testing

### Lokale Syntax-Tests
```bash
python3 -m py_compile gtwy
```

### Installation testen
```bash
sudo ./gtwy install
sudo gtwy setup
gtwy health
```

### Mit Staging-Modus testen
```yaml
# config.yml
certbot:
  staging: true  # Für Tests!
```

---

## 📚 Technische Details

### Technologie-Stack

- **Python 3.8+** - Haupt-Script
- **SQLite** - Datenbank
- **nginx** - Reverse Proxy
- **certbot** - Let's Encrypt SSL
- **IONOS DNS API** - Automatische DNS-Verwaltung
- **SSH** - Authentifizierung & Tunneling

### Dependencies

```
PyYAML>=6.0
requests>=2.31.0
```

Standard-Library: `sqlite3`, `subprocess`, `argparse`, `logging`, etc.

### Datenbank-Schema

```sql
CREATE TABLE boxes (
    box_id TEXT PRIMARY KEY,
    domain TEXT NOT NULL,
    admin_ssh_port INTEGER UNIQUE NOT NULL,
    ssh_key_type TEXT NOT NULL,
    ssh_key_fingerprint TEXT NOT NULL,
    ssh_public_key TEXT NOT NULL,
    added_date TEXT NOT NULL,
    last_seen TEXT,
    notes TEXT
);

CREATE TABLE tunnels (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    box_id TEXT NOT NULL,
    service TEXT NOT NULL,
    local_port INTEGER NOT NULL,
    server_port INTEGER UNIQUE NOT NULL,
    subdomain TEXT UNIQUE NOT NULL,
    status TEXT DEFAULT 'active',
    created TEXT NOT NULL,
    last_checked TEXT,
    error_message TEXT,
    FOREIGN KEY (box_id) REFERENCES boxes(box_id) ON DELETE CASCADE,
    UNIQUE(box_id, service)
);
```

---

## 🐛 Troubleshooting

### Installation schlägt fehl
```bash
# Prüfe Root-Rechte
sudo gtwy install

# Prüfe Python-Version
python3 --version  # mind. 3.8

# Installiere Dependencies
pip3 install PyYAML requests
```

### Tunnel nicht erreichbar
```bash
# 1. Tunnel-Status prüfen
gtwy info <subdomain>

# 2. DNS-Propagation checken
dig <subdomain>

# 3. Port-Listening prüfen
ss -tlnp | grep <port>

# 4. nginx-Config testen
sudo nginx -t

# 5. Logs prüfen
tail -f /opt/gtwy/gtwy.log
```

### IONOS API Fehler
```bash
# API-Keys prüfen
cat /opt/gtwy/config.yml | grep -A3 ionos

# API-Test
curl -H "X-API-Key: PREFIX.SECRET" https://api.hosting.ionos.com/dns/v1/zones
```

---

## 📝 Changelog

Siehe [CHANGELOG.md](../CHANGELOG.md) im Hauptverzeichnis für vollständige Versionshistorie.

### v1.2.6 (current)
- Fixed automatic SSL certificate provisioning
- Fixed DNS record cleanup
- Zero-downtime updates

---

## 📄 Lizenz

MIT License - siehe LICENSE Datei.

---

## 👨‍💻 Autor

Entwickelt für die KI.Box / IT.Box Infrastruktur.

---

## 🔗 Links

- [Main README](../README.md) - Project overview
- [CHANGELOG](../CHANGELOG.md) - Full version history
- [tnl Documentation](../tnl/README.md) - Client-side tunnel manager
