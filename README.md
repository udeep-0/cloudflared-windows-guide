# Cloudflare Tunnel Setup on Windows — Complete Guide
> Single tunnel, multiple hostnames, running as a Windows service

This guide documents how to set up a Cloudflare Tunnel on Windows to expose multiple local apps to the internet using a single tunnel with ingress rules. It also covers how to fully wipe an existing broken setup before starting fresh.

---

## Table of Contents
1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Common Problems This Guide Solves](#2-common-problems-this-guide-solves)
3. [Prerequisites](#3-prerequisites)
4. [Phase 1 — Wipe Existing Setup](#4-phase-1--wipe-existing-setup)
5. [Phase 2 — Fresh Install](#5-phase-2--fresh-install)
6. [Phase 3 — Create Tunnel & Configure](#6-phase-3--create-tunnel--configure)
7. [Phase 4 — Install as Windows Service](#7-phase-4--install-as-windows-service)
8. [Troubleshooting](#8-troubleshooting)
9. [Maintenance](#9-maintenance)

---

## 1. Overview & Key Concepts

Cloudflare Tunnel (`cloudflared`) creates a secure outbound connection from your server to Cloudflare's network — no open ports, no firewall rules needed.

**The golden rule:**
> ✅ One tunnel per server. Multiple hostnames via ingress rules.

You do **not** need a separate tunnel for each hostname. A single tunnel handles all your apps through a `config.yml` file with ingress rules:

```
One Tunnel
├── hostname: app1.yourdomain.com  →  http://127.0.0.1:3000
├── hostname: app2.yourdomain.com  →  http://127.0.0.1:8080
└── catch-all                      →  http_status:404
```

---

## 2. Common Problems This Guide Solves

| Problem | Root Cause | Fix |
|---|---|---|
| "Service already exists" error when adding a second tunnel | Cloudflare expects one tunnel per server | Use ingress rules in a single tunnel |
| Two `cloudflared.exe` instances running | Two copies downloaded from different sources | Keep only one `.exe` in one known location |
| Tunnel shows no active connections | Service not reading the config file | Explicitly pass config path in registry `ImagePath` |
| Error 1033 on both hostnames | Wrong UUID or credentials path in `config.yml` | Use real UUID from `cloudflared tunnel list` |

---

## 3. Prerequisites

- A Cloudflare account with your domain added
- Windows machine (64-bit)
- PowerShell running **as Administrator**
- Your local apps already running on known ports

---

## 4. Phase 1 — Wipe Existing Setup

> Skip this phase if you are doing a brand new install with no prior `cloudflared` setup.

Open **PowerShell as Administrator** for all steps below.

### Step 1 — Stop and uninstall the Windows service

```powershell
net stop cloudflared
cloudflared.exe service uninstall
```

> "Service not found" or "not installed" messages are fine — just continue.

### Step 2 — Kill any running cloudflared processes

```powershell
Stop-Process -Name cloudflared -Force
```

### Step 3 — Find all cloudflared.exe files

```powershell
Get-ChildItem -Path "C:\", "C:\Program Files\", "C:\Program Files (x86)\", "$env:USERPROFILE\Downloads\" -Filter "cloudflared.exe" -Recurse -ErrorAction SilentlyContinue | Select-Object FullName
```

Delete every result and its parent folder:

```powershell
Remove-Item -Force "C:\path\to\cloudflared.exe"
Remove-Item -Recurse -Force "C:\path\to\folder"
```

### Step 4 — Delete old credentials and config

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.cloudflared"
```

> No output = success.

### Step 5 — Delete old tunnels from Cloudflare dashboard

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com)
2. Navigate to **Networks → Tunnels**
3. Click the **3-dot menu** on each tunnel → **Delete**

Also delete any leftover DNS records:
1. Go to [dash.cloudflare.com](https://dash.cloudflare.com) → Select your domain
2. Navigate to **DNS → Records**
3. Delete any `CNAME` records pointing to old tunnels (look for records ending in `.cfargotunnel.com`)
4. Leave your root `A` record untouched

---

## 5. Phase 2 — Fresh Install

### Step 1 — Create a clean directory

```powershell
New-Item -ItemType Directory -Path "C:\cloudflared"
```

### Step 2 — Download cloudflared.exe

Download the official **Windows 64-bit** binary from:
👉 https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/

- Save it to `C:\cloudflared\cloudflared.exe`
- If the downloaded file has a different name (e.g. `cloudflared-windows-amd64.exe`), rename it to `cloudflared.exe`

### Step 3 — Add to PATH

```powershell
[System.Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\cloudflared", "Machine")
```

> This **appends** to your existing PATH — it will not replace anything.

**Close and reopen PowerShell as Administrator** for the PATH change to take effect.

### Step 4 — Verify the installation

```powershell
cloudflared --version
```

Expected output:
```
cloudflared version 2026.x.x (built ...)
```

---

## 6. Phase 3 — Create Tunnel & Configure

### Step 1 — Login to Cloudflare

```powershell
cloudflared tunnel login
```

A browser window will open. Log in and select your domain. A `cert.pem` file will be saved automatically to:
```
C:\Users\<YOU>\.cloudflared\cert.pem
```

### Step 2 — Create the tunnel

```powershell
cloudflared tunnel create my-tunnel
```

You will see output like:
```
Tunnel credentials written to C:\Users\<YOU>\.cloudflared\<UUID>.json
Created tunnel my-tunnel with id <UUID>
```

> ⚠️ Copy the **UUID** — you will need it in the config file.

Verify the tunnel exists:
```powershell
cloudflared tunnel list
```

### Step 3 — Create config.yml

Open Notepad to create the config file:

```powershell
notepad $env:USERPROFILE\.cloudflared\config.yml
```

Paste the following (replace values with your own):

```yaml
tunnel: YOUR-TUNNEL-UUID
credentials-file: C:\Users\<YOUR-USERNAME>\.cloudflared\YOUR-TUNNEL-UUID.json

ingress:
  - hostname: app1.yourdomain.com
    service: http://127.0.0.1:3000

  - hostname: app2.yourdomain.com
    service: http://127.0.0.1:8080

  - service: http_status:404
```

> ⚠️ The last `http_status:404` catch-all rule is **required** — cloudflared will fail to validate without it.

Save the file with `Ctrl+S`.

### Step 4 — Route DNS for each hostname

```powershell
cloudflared tunnel route dns my-tunnel app1.yourdomain.com
cloudflared tunnel route dns my-tunnel app2.yourdomain.com
```

Expected output for each:
```
INF Added CNAME app1.yourdomain.com which will route to this tunnel tunnelID=<UUID>
```

> If you get a "record already exists" error, delete the existing CNAME record from the Cloudflare DNS dashboard first, then rerun the command.

### Step 5 — Validate the config

```powershell
cloudflared tunnel ingress validate
```

Expected output:
```
Validating rules from C:\Users\<YOU>\.cloudflared\config.yml
OK
```

---

## 7. Phase 4 — Install as Windows Service

### Step 1 — Install the service with explicit config path

```powershell
cloudflared.exe --config "C:\Users\<YOUR-USERNAME>\.cloudflared\config.yml" service install
```

> ⚠️ Always pass the `--config` flag explicitly. Without it, the service may start without reading your config, resulting in no active tunnel connections.

### Step 2 — Start the service

```powershell
net start cloudflared
```

### Step 3 — Verify it's running

```powershell
Get-Service cloudflared
```

Expected output:
```
Status   Name          DisplayName
------   ----          -----------
Running  cloudflared   Cloudflared agent
```

### Step 4 — Verify tunnel connections

```powershell
cloudflared tunnel info my-tunnel
```

Expected output:
```
NAME:     my-tunnel
ID:       <UUID>
CONNECTOR ID   CREATED   ARCHITECTURE   VERSION   ORIGIN IP   EDGE
<id>           ...       windows_amd64  2026.x.x  x.x.x.x    1xbom08, 1xbom11, 2xhyd01
```

You should see **4 edge connections** across Cloudflare's network. Your tunnel is live. ✅

---

## 8. Troubleshooting

### Error 1033 — Cloudflare unable to resolve tunnel

**Cause:** The tunnel is not connected or the config has incorrect values.

**Fix:**
1. Check `config.yml` has the real UUID (not a placeholder like `secret`)
2. Verify the `.json` credentials filename matches the UUID exactly
3. Run `cloudflared tunnel run my-tunnel` in terminal to see live errors

---

### Tunnel shows no active connections

**Cause:** The Windows service is not reading the config file.

**Fix:** Check the registry `ImagePath` value:

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Cloudflared"
```

If `ImagePath` does not include `--config`, fix it:

```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Cloudflared" -Name ImagePath -Value 'C:\cloudflared\cloudflared.exe --config "C:\Users\<YOU>\.cloudflared\config.yml" tunnel run my-tunnel'
```

Then restart the service:

```powershell
Stop-Process -Name cloudflared -Force
net stop cloudflared
net start cloudflared
```

---

### "Service already started" when running net start

**Cause:** The service auto-started after install.

**Fix:** This is not an error. Run `Get-Service cloudflared` to confirm it is running.

---

### DNS route fails with "record already exists"

**Cause:** A leftover CNAME record from a previous tunnel setup.

**Fix:**
1. Go to Cloudflare dashboard → DNS → Records
2. Delete the existing CNAME for that hostname
3. Re-run `cloudflared tunnel route dns`

---

### Two cloudflared.exe instances running

**Cause:** Two copies installed from different locations.

**Fix:**
```powershell
# Find all instances
Get-ChildItem -Path "C:\", "C:\Program Files\", "C:\Program Files (x86)\" -Filter "cloudflared.exe" -Recurse -ErrorAction SilentlyContinue | Select-Object FullName

# Kill all running instances
Stop-Process -Name cloudflared -Force

# Delete all but C:\cloudflared\cloudflared.exe
```

---

## 9. Maintenance

### Checking service status

```powershell
Get-Service cloudflared
cloudflared tunnel info my-tunnel
```

### Restarting the service

```powershell
Stop-Process -Name cloudflared -Force
net stop cloudflared
net start cloudflared
```

### Adding a new hostname later

1. Add a new ingress rule to `config.yml`:
```yaml
  - hostname: app3.yourdomain.com
    service: http://127.0.0.1:9000
```

2. Route DNS:
```powershell
cloudflared tunnel route dns my-tunnel app3.yourdomain.com
```

3. Restart the service:
```powershell
Stop-Process -Name cloudflared -Force
net stop cloudflared
net start cloudflared
```

### Updating cloudflared on Windows

> ⚠️ cloudflared does **not** auto-update on Windows. Check for updates manually every few months.

1. Stop the service: `net stop cloudflared`
2. Download the latest `.exe` from the official page
3. Replace `C:\cloudflared\cloudflared.exe` with the new file
4. Start the service: `net start cloudflared`
5. Verify: `cloudflared --version`

---

## Quick Reference

| File | Path |
|---|---|
| Executable | `C:\cloudflared\cloudflared.exe` |
| Config file | `C:\Users\<YOU>\.cloudflared\config.yml` |
| Credentials | `C:\Users\<YOU>\.cloudflared\<UUID>.json` |
| Auth certificate | `C:\Users\<YOU>\.cloudflared\cert.pem` |

| Command | Purpose |
|---|---|
| `cloudflared tunnel list` | List all tunnels and connections |
| `cloudflared tunnel info <name>` | Show active connections for a tunnel |
| `cloudflared tunnel ingress validate` | Validate config.yml |
| `cloudflared tunnel run <name>` | Run tunnel manually (for debugging) |
| `Get-Service cloudflared` | Check Windows service status |
