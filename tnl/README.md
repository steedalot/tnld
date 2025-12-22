# tnl - Tunnel Client for IT.Box

**Single-file Python tool** für IT.Boxes um SSH-Reverse-Tunnels zum Gateway-Server aufzubauen.

---

## 🚀 Quick Start

### Installation auf IT.Box

```bash
# 1. Download tnl
curl -O https://example.com/tnl
chmod +x tnl

# 2. Install (einmalig)
sudo ./tnl install
# → Zeigt Public Key an → Kopieren!

# 3. Public Key auf Gateway registrieren
# (auf Gateway-Server ausführen)
sudo gtwy add-box box01 kibox.online '<public-key>'
gtwy get-port box01  # → z.B. 20001

# 4. Admin-Tunnel einrichten
sudo tnl setup gateway.example.com 20001
```

**Fertig!** Admin-SSH-Tunnel läuft jetzt permanent. 🎉

---

## 📋 Befehle

### `sudo tnl install`

**Einmalige Installation auf der Box**

Was passiert:
- ✅ Erstellt User `tunneluser`
- ✅ Generiert SSH-Key (`/home/tunneluser/.ssh/tunnel_key`)
- ✅ Installiert `autossh` via apt
- ✅ Kopiert sich selbst nach `/usr/local/bin/tnl`
- ✅ Zeigt Public Key an

**Output:**
```
🔧 tnl Client Installation

1. Creating tunneluser...
   ✓ User 'tunneluser' created

2. Generating SSH key...
   ✓ SSH key generated

3. Installing autossh...
   ✓ autossh installed

4. Installing tnl command...
   ✓ Copied to /usr/local/bin/tnl

============================================================
✓ Installation complete!
============================================================

📋 SSH Public Key (register this on gateway server):

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHx... tunneluser@box01

============================================================

Next steps:
  1. Copy the public key above
  2. Register this box on gateway server:
     sudo gtwy add-box <box-id> <domain> '<public-key>'

  3. Get admin SSH port from server:
     gtwy get-port <box-id>

  4. Setup tunnel on this box:
     sudo tnl setup <gateway-ip> <admin-port>
============================================================
```

---

### `sudo tnl setup <gateway-ip> <admin-port>`

**Admin-SSH-Tunnel einrichten**

Argumente:
- `gateway-ip`: IP oder Hostname des Gateway-Servers
- `admin-port`: Admin-SSH-Port (von `gtwy get-port` auf Server)

Was passiert:
- ✅ Testet SSH-Verbindung zum Gateway
- ✅ Erstellt systemd-Service (`tnl-admin.service`)
- ✅ Startet Service (autossh hält Verbindung aufrecht)

**Beispiel:**
```bash
sudo tnl setup 192.168.1.100 20001
```

**Output:**
```
🚀 tnl Tunnel Setup

Gateway IP:   192.168.1.100
Admin Port:   20001

1. Testing SSH connection...
   ✓ SSH connection successful

2. Creating systemd service...
   ✓ Service created: /etc/systemd/system/tnl-admin.service

3. Enabling service...
   ✓ Service enabled

4. Starting service...
   ✓ Service started

5. Checking status...
   ✓ Tunnel is active

============================================================
✓ Setup complete!
============================================================

Admin SSH tunnel is now running.

Useful commands:
  sudo systemctl status tnl-admin     # Check status
  sudo systemctl restart tnl-admin    # Restart tunnel
  sudo systemctl stop tnl-admin       # Stop tunnel
  sudo journalctl -u tnl-admin -f     # View logs
============================================================
```

---

### `sudo tnl status`

**Tunnel-Status prüfen**

Zeigt:
- Service-Status (active/inactive)
- Letzte Log-Einträge

**Beispiel:**
```bash
sudo tnl status
```

**Output:**
```
📊 tnl Status

✓ Admin Tunnel: ACTIVE

Recent logs:
------------------------------------------------------------
Jan 15 14:23:45 box01 systemd[1]: Started tnl Admin SSH Tunnel.
Jan 15 14:23:46 box01 autossh[12345]: starting ssh ...
```

---

### `tnl version`

**Version anzeigen**

```bash
tnl version
# → tnl v1.2.6
```

---

## 🏗️ Architektur

### Komponenten

```
┌─────────────────────────────────────┐
│          IT.Box (Client)            │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  tnl (Tunnel Client)          │ │
│  │  - tunneluser                 │ │
│  │  - autossh                    │ │
│  │  - systemd service            │ │
│  └─────────────┬─────────────────┘ │
│                │ SSH Tunnel         │
└────────────────┼───────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│       Gateway Server (gtwy)         │
│                                     │
│  Port 20001 (Admin SSH)             │
│  - SSH restricted commands          │
│  - gtwy request/release             │
└─────────────────────────────────────┘
```

### Dateien auf Box

```
/home/tunneluser/
└── .ssh/
    ├── tunnel_key         # Private key
    └── tunnel_key.pub     # Public key

/etc/systemd/system/
└── tnl-admin.service      # systemd service

/usr/local/bin/
└── tnl                    # Command (global)
```

### systemd Service

Der Service hält den Admin-SSH-Tunnel permanent aufrecht:

```ini
[Unit]
Description=tnl Admin SSH Tunnel
After=network-online.target

[Service]
Type=simple
User=tunneluser
Restart=always
RestartSec=10
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval=30" \
  -o "ServerAliveCountMax=3" \
  -i /home/tunneluser/.ssh/tunnel_key \
  -p 20001 \
  tunneluser@gateway.example.com

[Install]
WantedBy=multi-user.target
```

**Features:**
- ✅ Auto-Restart bei Verbindungsabbruch
- ✅ Server-Alive-Checks alle 30s
- ✅ Startet automatisch beim Boot
- ✅ Läuft als `tunneluser` (nicht root)

---

## 🔒 Sicherheit

### SSH-Key

- **Typ:** ED25519 (modern, sicher, klein)
- **Passphrase:** Keine (Service muss automatisch starten)
- **Zugriff:** Nur `tunneluser` (0600)
- **Zweck:** Nur für Tunnel-Verbindung

### Benutzer

- **Name:** `tunneluser`
- **Typ:** System-User (`-r`)
- **Home:** `/home/tunneluser`
- **Shell:** `/bin/bash`
- **Rechte:** Minimal (nur SSH-Tunnel)

### Gateway-Seite

Auf dem Gateway-Server ist der Key mit Command-Restriction eingetragen:

```bash
command="BOX_ID=box01 /opt/gtwy/gtwy request",restrict,port-forwarding ssh-ed25519 AAAAC3...
```

- ✅ Nur `gtwy request`/`release` Befehle möglich
- ✅ Kein Shell-Zugriff
- ✅ Nur Port-Forwarding erlaubt

---

## 🧪 Testing

### 1. Installation testen

```bash
# Auf Box
sudo tnl install

# Prüfen
which tnl              # → /usr/local/bin/tnl
id tunneluser          # → User existiert
ls -la /home/tunneluser/.ssh/  # → Keys existieren
which autossh          # → /usr/bin/autossh
```

### 2. Setup testen

```bash
# Auf Gateway: Box registrieren
sudo gtwy add-box test-box kibox.online "ssh-ed25519 AAAAC..."
gtwy get-port test-box  # → z.B. 20001

# Auf Box: Tunnel einrichten
sudo tnl setup gateway.example.com 20001

# Status prüfen
sudo tnl status
sudo systemctl status tnl-admin
```

### 3. Verbindung testen

```bash
# Logs beobachten
sudo journalctl -u tnl-admin -f

# Service neu starten (Verbindung sollte wieder aufgebaut werden)
sudo systemctl restart tnl-admin

# Nach ~5 Sekunden sollte Tunnel wieder stehen
```

---

## 🐛 Troubleshooting

### Installation schlägt fehl

```bash
# Root-Rechte prüfen
sudo tnl install

# Python-Version prüfen
python3 --version  # mind. 3.6

# autossh manuell installieren
sudo apt-get update
sudo apt-get install autossh
```

### SSH-Verbindung schlägt fehl

```bash
# Public Key auf Gateway registriert?
# Auf Gateway prüfen:
cat /home/tunneluser/.ssh/authorized_keys | grep test-box

# IP/Port korrekt?
ping gateway.example.com
telnet gateway.example.com 20001

# Manuell testen
sudo -u tunneluser ssh -i /home/tunneluser/.ssh/tunnel_key \
  -p 20001 tunneluser@gateway.example.com
```

### Service startet nicht

```bash
# Logs prüfen
sudo journalctl -u tnl-admin -n 50

# Service-Status
sudo systemctl status tnl-admin

# Manuell starten (zum Debuggen)
sudo -u tunneluser /usr/bin/autossh -M 0 -N -v \
  -i /home/tunneluser/.ssh/tunnel_key \
  -p 20001 tunneluser@gateway.example.com
```

---

## 🔄 Service-Management

### Status prüfen
```bash
sudo systemctl status tnl-admin
```

### Neu starten
```bash
sudo systemctl restart tnl-admin
```

### Stoppen
```bash
sudo systemctl stop tnl-admin
```

### Logs anschauen
```bash
# Live-Logs
sudo journalctl -u tnl-admin -f

# Letzte 50 Zeilen
sudo journalctl -u tnl-admin -n 50
```

### Service deaktivieren
```bash
sudo systemctl disable tnl-admin
sudo systemctl stop tnl-admin
```

---

## 📝 Workflow

### Komplett-Ablauf Box-zu-Gateway

**1. Auf Box:**
```bash
# Installation
curl -O https://example.com/tnl
chmod +x tnl
sudo ./tnl install

# Public Key kopieren (wird angezeigt)
```

**2. Auf Gateway:**
```bash
# Box registrieren
sudo gtwy add-box box01 kibox.online 'ssh-ed25519 AAAAC3...'

# Admin-Port abrufen
gtwy get-port box01
# → 20001
```

**3. Zurück auf Box:**
```bash
# Tunnel einrichten
sudo tnl setup gateway.example.com 20001

# Status prüfen
sudo tnl status
```

**4. Service-Tunnel hinzufügen**
```bash
# Service-Tunnel für Gitea hinzufügen
sudo tnl add gitea 3000

# Weitere Services hinzufügen
sudo tnl add theia 8080
sudo tnl add portainer 9000

# Status prüfen
sudo tnl list
```

---

## 📝 Changelog

Siehe [CHANGELOG.md](../CHANGELOG.md) im Hauptverzeichnis für vollständige Versionshistorie.

### v1.2.6 (current)
- ✅ Zero-downtime updates
- ✅ Service tunnels management (`add`, `remove`, `list`)
- ✅ Admin tunnel setup
- ✅ Automatic reconnection
- ✅ Update mechanism with configuration preservation

---

## 📄 Lizenz

MIT License

---

## 👨‍💻 Autor

Entwickelt für die KI.Box / IT.Box Infrastruktur.

---

## 🔗 Links

- [Main README](../README.md) - Project overview
- [CHANGELOG](../CHANGELOG.md) - Full version history
- [gtwy Documentation](../gtwy/README.md) - Server-side gateway manager

---

**tnl** - Simpel. Robust. Single-File. 🚀
