# UNIT3D-Announce Setup Tutorial

This guide will walk you through setting up **UNIT3D-Announce**, the external BitTorrent tracker for UNIT3D. Using UNIT3D-Announce (written in Rust) in place of UNIT3D's built-in PHP announce can dramatically improve performance and scalability for high-traffic trackers.

> [!NOTE]  
> The built-in PHP announce handler in UNIT3D can handle around **250 requests per second per core**. By contrast, the Rust-based **UNIT3D-Announce** can handle **~50,000 requests/sec per core** (or ~10,000/sec behind an Nginx TLS proxy). If you anticipate heavy load, the external tracker is highly recommended.

> [!IMPORTANT]  
> **Compatibility:** Make sure to use the correct **UNIT3D-Announce** version for your UNIT3D installation:  
> - For **UNIT3D v8.3.4 – v9.0.4**, use **UNIT3D-Announce v0.1** (branch `v0.1`).  
> - For **UNIT3D v9.0.5 and above**, use **UNIT3D-Announce v0.2** (branch `v0.2`).

## Table of Contents

- [Prerequisites](#prerequisites)  
- [Installation](#installation)  
- [Cloning and Building UNIT3D-Announce](#cloning-and-building-unit3d-announce)  
- [Configuring UNIT3D for the External Tracker](#configuring-unit3d-for-the-external-tracker)  
- [Reverse Proxy Configuration (Nginx)](#reverse-proxy-configuration-nginx)  
- [Supervisor Setup (Auto-Start)](#supervisor-setup-auto-start)  
- [Supervisor Configuration](#supervisor-configuration)  
- [Starting and Reloading the Tracker](#starting-and-reloading-the-tracker)  
- [Stopping (Gracefully Shutting Down) the Tracker](#stopping-gracefully-shutting-down-the-tracker)  
- [Global Freeleech and Double Upload Events](#global-freeleech-and-double-upload-events)  
- [Updating UNIT3D-Announce](#updating-unit3d-announce)  
- [Uninstalling UNIT3D-Announce](#uninstalling-unit3d-announce)

## Prerequisites

- **Running UNIT3D instance:** You should have UNIT3D installed and working (with the appropriate version as noted above).  
- **Server access:** Root or sudo privileges on the server where UNIT3D is installed.  
- **Rust toolchain:** Install Rust (which includes the `cargo` build tool) on the server. This is required to compile the Rust-based tracker. For Ubuntu/Debian, you can install via `apt` (e.g. `sudo apt install cargo`) or use Rust’s official installer.  
- **Nginx:** This guide assumes you are using Nginx as your web server (as in the official installation). If you use another web server or a separate domain for the tracker, adapt the proxy instructions accordingly.  
- **Supervisor:** We use Supervisor to run the tracker as a background service. Ensure **Supervisor** is installed (`sudo apt install supervisor` on Debian/Ubuntu), or use an alternative service manager (like systemd) if preferred.

## Installation

### Cloning and Building UNIT3D-Announce

First, download and compile the external tracker code:

```bash
# Navigate to the UNIT3D installation directory (e.g., /var/www/html)
$ cd /var/www/html

# Clone the UNIT3D-Announce repository (using the branch matching your UNIT3D version)
$ git clone -b v0.2 https://github.com/HDInnovations/UNIT3D-Announce.git unit3d-announce

# Enter the cloned repository directory
$ cd unit3d-announce

# Copy the example configuration to .env
$ cp .env.example .env

# (Optional) Edit the .env configuration as needed
$ nano .env

# Build the tracker in release mode (this compiles the Rust code)
$ cargo build --release
```

> [!NOTE]  
> The `.env` file contains configuration for the tracker (listen address/port, announce interval limits, etc). Ensure the critical values like `LISTENING_IP_ADDRESS`, `LISTENING_PORT`, and `APIKEY` are set appropriately. The default values are usually sufficient, but you may adjust settings (e.g., announce interval limits like `ANNOUNCE_MIN_INTERVAL` and `ANNOUNCE_MAX_INTERVAL`) if needed.

### Configuring UNIT3D for the External Tracker

Next, integrate the external tracker with your UNIT3D web application:

```bash
# Return to UNIT3D's base directory
$ cd /var/www/html

# Open UNIT3D's .env file to add the external tracker variables
$ nano .env
```

Add the following entries to UNIT3D's `.env` (or update them if they exist):

```ini
TRACKER_HOST=127.0.0.1              # IP address where UNIT3D-Announce is listening (from LISTENING_IP_ADDRESS)
TRACKER_PORT=34345                  # Port where UNIT3D-Announce is listening (from LISTENING_PORT)
TRACKER_KEY=<your_tracker_api_key>  # The APIKEY value set in UNIT3D-Announce's .env
```

Save the changes. These environment variables inform UNIT3D about the external tracker's location and authentication key.

Now, enable the external tracker in UNIT3D's configuration:

```bash
# Open UNIT3D's announce configuration file
$ nano config/announce.php
```

In this file, make sure the external announce tracker is enabled. Set the configuration option to use the external tracker (if there's a specific flag or setting, enable it). Typically, this involves ensuring that UNIT3D will use the `TRACKER_HOST` and related variables instead of the built-in announce. Save the file after making the change.

## Reverse Proxy Configuration (Nginx)

If the UNIT3D-Announce tracker runs on the **same server and domain** as your UNIT3D frontend, you should proxy the `/announce` URL to the tracker. This allows peers to announce to the same domain as your site, while Nginx forwards those requests to the Rust tracker.

Open your Nginx site configuration (e.g., `/etc/nginx/sites-enabled/default` or your site’s config file) and add the following `location` block inside the `server` block that handles your site:

```nginx
location /announce/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_pass http://aaa.bbb.ccc.ddd:eeee$request_uri;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    set_real_ip_from fff.ggg.hhh.iii;
}
```

In the above snippet:  
- Replace `aaa.bbb.ccc.ddd:eeee` with the **listening IP and port** of the UNIT3D-Announce service. Use the values of `LISTENING_IP_ADDRESS` and `LISTENING_PORT` from the tracker's .env.  
- Replace `fff.ggg.hhh.iii` with the **public IP address** of your Nginx proxy (i.e., the server’s public IP that users connect to). If you have additional proxy layers, you can add multiple `set_real_ip_from <proxy_ip>;` lines for each, as long as the proxies correctly add `X-Forwarded-For` headers.

After updating the Nginx configuration, edit the UNIT3D-Announce `.env` file and set `REVERSE_PROXY_CLIENT_IP_HEADER_NAME="X-Real-IP"` (uncomment it if it was commented out). This ensures the tracker uses the correct client IP address provided by Nginx.

Finally, **reload Nginx** to apply the changes:

```bash
$ sudo service nginx reload
```

## Supervisor Setup (Auto-Start)

To run the tracker in the background and have it start on boot, we recommend using **Supervisor** (a process control system). The following steps will configure Supervisor to manage the UNIT3D-Announce process.

### Supervisor Configuration

Create a Supervisor configuration file for the tracker (e.g., `/etc/supervisor/conf.d/unit3d-announce.conf`):

```bash
$ sudo nano /etc/supervisor/conf.d/unit3d-announce.conf
```

Paste the following content into the file:

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

Save and close the file. This tells Supervisor how to start the tracker (using the compiled binary at the specified path) and where to log its output.

### Starting and Reloading the Tracker

After adding the configuration, instruct Supervisor to recognize the new program and start it:

```bash
# Reload Supervisor configurations and start the unit3d-announce program
$ sudo supervisorctl reread
$ sudo supervisorctl update
$ sudo supervisorctl start unit3d-announce:*
```

This will launch the UNIT3D-Announce tracker in the background. It will also start automatically on system boot (since `autostart=true` is set).

If you make changes to the tracker's configuration and rebuild it, you can **reload** the running tracker without restarting the whole service. To apply a new configuration on the fly, run:

```bash
$ curl -X POST "http://<LISTENING_IP_ADDRESS>:<LISTENING_PORT>/announce/<APIKEY>/config/reload"
```

Replace `<LISTENING_IP_ADDRESS>`, `<LISTENING_PORT>`, and `<APIKEY>` with your tracker’s actual values from the .env. This `/config/reload` endpoint will make the tracker re-read its .env configuration without dropping active connections.

### Stopping (Gracefully Shutting Down) the Tracker

To stop the tracker (for example, before updating or uninstalling it), use Supervisor to gracefully shut it down:

```bash
$ sudo supervisorctl stop unit3d-announce:unit3d-announce_00
```

This stops the tracker process. (The `unit3d-announce_00` part is the process name as defined by our Supervisor config.)

## Global Freeleech and Double Upload Events

When using the external tracker, **global freeleech** and **double upload** events require configuration in two places: both UNIT3D *and* the external tracker's environment.

> [!IMPORTANT]  
> Enabling a global freeleech or double upload event **only in UNIT3D** is not enough when using UNIT3D-Announce. You **must also enable it in the UNIT3D-Announce tracker’s .env**, so that the tracker knows to adjust user stats. UNIT3D controls the UI (e.g. showing the freeleech timer), while the external tracker controls the actual accounting of upload/download.

To activate a global event, edit the following in the UNIT3D-Announce `.env` and then restart the tracker:

```ini
# The upload multiplier (in percent). For example, 200 means 2x upload (double upload).
# Default: 100
UPLOAD_FACTOR=200

# The download multiplier (in percent). For example, 0 means freeleech (no download counted).
# Default: 100
DOWNLOAD_FACTOR=0
```

- To enable **global double upload**, set `UPLOAD_FACTOR=200` (200% upload counted, which doubles all upload credit).  
- To enable **global freeleech**, set `DOWNLOAD_FACTOR=0` (0% download counted, so download doesn’t count against users).  
- You can adjust these values as needed (e.g., `150` for 1.5× upload, etc.). Setting them back to `100` returns to normal (no global modifier).

Remember to **restart the tracker** (via Supervisor) after changing these values so the new factors take effect.

## Updating UNIT3D-Announce

To update the UNIT3D-Announce tracker to a newer version, follow these steps:

```bash
# Stop the running tracker (if it's active)
$ sudo supervisorctl stop unit3d-announce:*

# Go to the UNIT3D-Announce install directory
$ cd /var/www/html/unit3d-announce

# Fetch the latest code from the repository (on the appropriate branch)
$ git pull origin v0.2

# Review any changes to the example .env file compared to your current .env
$ diff -u .env .env.example
```

Open the `.env` in an editor and **add or update any new configuration keys** introduced by the update (based on the diff output). Preserve your existing values for keys that remain the same.

```bash
# Re-build the tracker with the new code
$ cargo build --release
```

Once rebuilt, start the tracker again via Supervisor:

```bash
$ sudo supervisorctl start unit3d-announce:*
```

> [!NOTE]  
> If you previously had the tracker running, you can also use `sudo supervisorctl restart unit3d-announce:*` to stop and start it in one go. Always ensure the tracker is stopped during the build to avoid conflicts (though building while it's running is usually fine, you still need to restart to use the new binary).  
> After updating, it's a good idea to monitor the `announce.log` (in `storage/logs/announce.log`) for any errors on startup.

## Uninstalling UNIT3D-Announce

If you decide to remove the external tracker and revert to using UNIT3D's internal announce, follow these steps to fully uninstall UNIT3D-Announce:

```bash
# Stop the external tracker if it's running
$ sudo supervisorctl stop unit3d-announce:*

# Disable the external tracker in UNIT3D's config (revert any changes in config/announce.php)
$ sudo nano /var/www/html/config/announce.php

# Remove or comment out the /announce location block from Nginx config
$ sudo nano /etc/nginx/sites-enabled/default

# Remove the Supervisor configuration for UNIT3D-Announce
$ sudo rm -f /etc/supervisor/conf.d/unit3d-announce.conf

# Delete the UNIT3D-Announce files/directories
$ sudo rm -rf /var/www/html/unit3d-announce

# Remove the tracker environment variables from UNIT3D's .env
$ sudo nano /var/www/html/.env
```

After performing the above steps, reload/restart the related services and settings:

- **Nginx:** Reload Nginx to apply the config changes (`sudo service nginx reload`).  
- **Supervisor:** Reread and update Supervisor to unregister the tracker program (`sudo supervisorctl reread && sudo supervisorctl update`).  
- **UNIT3D config:** In UNIT3D’s `.env`, ensure `TRACKER_HOST`, `TRACKER_PORT`, and `TRACKER_KEY` are removed or left blank. This tells UNIT3D to fall back to its internal PHP announce handler. Also, double-check `config/announce.php` to confirm the internal tracker is enabled as default.

Once these steps are completed, UNIT3D will resume using its built-in announce tracker and the external UNIT3D-Announce will be fully removed from your system.
