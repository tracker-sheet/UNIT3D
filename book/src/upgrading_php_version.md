# UNIT3D Announce Setup Tutorial

[![UNIT3D Announce](https://img.shields.io/badge/UNIT3D-Announce%20Setup-blueviolet)](https://github.com/HDInnovations/UNIT3D-Announce)

<p align="center">
  <img src="https://ptpimg.me/6o8x8j.png" alt="UNIT3D Logo" style="width: 12%;">
</p>

_Set up the UNIT3D Announce service on your existing UNIT3D installation using this step-by-step guide._

---

> [!IMPORTANT]
> **Before starting, ensure you have created and securely stored a backup of your current installation.**

## Table of Contents

- [Prerequisites](#prerequisites)  
- [Installation Steps](#installation-steps)  
  - [1. Prepare the Environment](#1-prepare-the-environment)  
  - [2. Clone & Configure UNIT3D-Announce](#2-clone--configure-unit3d-announce)  
  - [3. Build the Tracker](#3-build-the-tracker)  
  - [4. Update UNIT3D Environment](#4-update-unit3d-environment)  
  - [5. Configure Nginx & Supervisor](#5-configure-nginx--supervisor)  
  - [6. Finalize & Verify Setup](#6-finalize--verify-setup)  
- [Troubleshooting](#troubleshooting)  
- [Acknowledgements](#acknowledgements)  

---

## Prerequisites

- An existing UNIT3D installation (typically at `/var/www/html`).  
- Sudo privileges for system configuration.  
- Basic knowledge of terminal commands and text editing.  
- Your database credentials (`DB_*`) from `/var/www/html/.env`.  
- Rust compiler and package manager (`cargo`) installed.

---

## Installation Steps

### 1. Prepare the Environment

Navigate into your UNIT3D base directory:

```bash
cd /var/www/html
```

### 2. Clone & Configure UNIT3D-Announce

1. Clone the Announce repository and enter it:

   ```bash
   git clone -b v0.1 https://github.com/HDInnovations/UNIT3D-Announce unit3d-announce
   cd unit3d-announce
   ```

2. Copy and edit the example environment:

   ```bash
   cp .env.example .env
   sudo nano .env
   ```

   - Ensure `DB_*` values match your main UNIT3D `.env`.  
   - Remove any trailing comma after `ANNOUNCE_MIN_ENFORCED` (e.g. `ANNOUNCE_MIN_ENFORCED=1740`).  
   - Uncomment `REVERSE_PROXY_CLIENT_IP_HEADER_NAME="X-Real-IP"` if using a reverse proxy.

### 3. Build the Tracker

Install Rust’s package manager and compile:

```bash
sudo apt update
sudo apt -y install cargo
cargo build --release
```

> [!NOTE]
> **The compiled binary will be located at** `target/release/unit3d-announce`.

### 4. Update UNIT3D Environment

Return to your UNIT3D directory and add tracker variables:

```bash
cd /var/www/html
sudo nano .env
```

Add (or update) these variables to match your Announce `.env`:

```env
TRACKER_HOST=127.0.0.1
TRACKER_PORT=3000
TRACKER_KEY=your_32_characters_min_api_key
```

> [!IMPORTANT]
> **`TRACKER_KEY` must be at least 32 characters long.**

### 5. Configure Nginx & Supervisor

#### Nginx

Edit your site’s Nginx config (e.g. `/etc/nginx/sites-enabled/default`):

```nginx
location /announce/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_pass http://127.0.0.1:3000$request_uri;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    set_real_ip_from 0.0.0.0/0;
}
```

- Adjust `proxy_pass` IP:Port to your Announce listener.  
- Change `set_real_ip_from` to your proxy’s IP range.

#### Supervisor

Create or update `/etc/supervisor/conf.d/unit3d-announce.conf`:

```ini
[program:unit3d-announce]
process_name=%(program_name)s_%(process_num)02d
command=/var/www/html/unit3d-announce/target/release/unit3d-announce
directory=/var/www/html/unit3d-announce
autostart=true
autorestart=false
user=root
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/announce.log
```

Reload Supervisor:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

> [!NOTE]
> **Supervisor log:** `/var/www/html/storage/logs/announce.log`

#### Enable External Tracker in UNIT3D

Edit `config/announce.php`:

```php
<?php

return [
    'external' => true,
    'host'     => env('TRACKER_HOST', '127.0.0.1'),
    'port'     => env('TRACKER_PORT', 3000),
    'key'      => env('TRACKER_KEY'),
];
```

### 6. Finalize & Verify Setup

Restart services and clear caches:

```bash
cd /var/www/html
sudo systemctl restart nginx
sudo supervisorctl reload
sudo php artisan set:all_cache
sudo systemctl restart php8.4-fpm
sudo php artisan queue:restart
```

Check Announce process status:

```bash
sudo supervisorctl status unit3d-announce:*
```

For logs:

```bash
supervisorctl tail -100 unit3d-announce:unit3d-announce_00
```

---

## Troubleshooting

- **`.env` typos**: Verify all keys (`TRACKER_*`, `DB_*`, `ANNOUNCE_*`).  
- **Nginx syntax**: Run `sudo nginx -t` after edits.  
- **Supervisor errors**: Inspect `/var/www/html/storage/logs/announce.log`.  
- **API key issues**: Ensure `TRACKER_KEY` is ≥32 characters.

---

## Acknowledgements

Made possible by airclay ([@ericlay](https://github.com/ericlay)).

---

_If you have questions or suggestions, please open an issue or submit a pull request._
