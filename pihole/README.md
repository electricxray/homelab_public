# Pi-hole + Unbound on Docker - Production Setup

**Privacy-focused network-wide ad blocking with recursive DNS**

This setup provides:
- Network-wide ad blocking via Pi-hole
- Private DNS resolution via Unbound (no third-party DNS providers)
- Optimized for Raspberry Pi
- Production-ready with health checks and auto-restart
- Minimal maintenance required

## Quick Start

### 1. Prerequisites
- Docker and Docker Compose installed
- Raspberry Pi (or any ARM based Linux system)
- Ports 53 (DNS) and 8443 (Web UI) available

### 2. Configuration
Edit the `docker-compose.yml` file and change:

```yaml
TZ: "America/Chicago"  # Change to your timezone
FTLCONF_webserver_api_password: "password"  # Change this value.
```

### 3. Start the Services
Navigate to the directory containing your docker-compose.yml and start:

```bash
cd ~/pihole  # or wherever you placed the files
sudo docker compose up -d
```

Wait about 30-60 seconds for initial setup to complete.

### 4. Access the Web Interface
Open your browser and navigate to: http://<your-pi-ip>:8443/admin
Login with the password you set in step 2.

## Usage

### Managing the Containers

**View status:**
```bash
sudo docker compose ps
```

**View logs:**
```bash
sudo docker compose logs pihole      # Pi-hole logs
sudo docker compose logs unbound     # Unbound logs
sudo docker compose logs -f pihole   # Follow logs in real-time
```


**Restart services:**
```bash
sudo docker compose restart
```

**Stop services:**
```bash
sudo docker compose down
```

**Update to latest images:**
```bash
sudo docker compose pull
sudo docker compose up -d
```

### Checking DNS Status
```bash
sudo docker exec pihole pihole status
```


### Updating Blocklists
Blocklists update automatically, but you can manually trigger an update:
```bash
sudo docker exec pihole pihole -g
```

## Using Pi-hole as Your DNS Server

### On Individual Devices
Configure your device's network settings to use your Pi's IP address as the DNS server.

**Example:** If your Raspberry Pi is at 192.168.1.100, set:
- Primary DNS: `192.168.1.100`
- Secondary DNS: (leave blank or use `1.1.1.1` as fallback) # Keep in mind that secondary DNS is not typically used as a backup, it is in rotation with the primary.

### On Your Router (Recommended)
Set your router's DNS server to your Pi's IP address. This automatically applies to all devices on your network.

1. Access your router's admin interface
2. Find DHCP/DNS settings, sometimes it is in the LAN settings.
3. Set primary DNS to your Raspberry Pi's IP
4. Save, reboots can't hurt but shouldn't be necessary.

## Exposed Ports
| Port | Service | Protocol |
|------|---------|----------|
| 53 | DNS | TCP/UDP |
| 8443 | Web UI (HTTPS) | TCP |

To verify ports are exposed:
```bash
sudo netstat -plnt | grep -E ":53|:8443"
```

## Architecture
┌─────────────────────────────────────────────────────┐
│                   Your Network                      │
│                                                     │
│  ┌──────────┐                                       │
│  │ Devices  │───────┐                               │
│  └──────────┘       │                               │
│                     │ DNS Query (Port 53)           │
│                     ▼                               │
│           ┌─────────────────┐                       │
│           │    Pi-hole      │                       │
│           │  (Ad Blocking)  │◄──── Web UI:8443      │
│           └────────┬────────┘                       │
│                    │ Query Unbound                  │
│                    │ (Port 5053)                    │
│                    ▼                                │
│           ┌─────────────────┐                       │
│           │    Unbound      │                       │
│           │  (Recursive DNS)│                       │
│           └────────┬────────┘                       │
│                    │                                │
└────────────────────┼────────────────────────────────┘
                     │
                     ▼
              Root DNS Servers
         (No Google/Cloudflare tracking)

## Security Features

**Container Isolation** - Runs in isolated Docker containers
**No New Privileges** - Container security hardening enabled
**Minimal Capabilities** - Only essential Linux capabilities granted
**Resource Limits** - Memory limits prevent resource exhaustion
**Private Network** - Unbound not exposed to host or the LAN as a whole
**Root DNS** - Queries resolved directly from root servers (no third-party DNS)
**Auto-restart** - Containers automatically restart on failure

## Troubleshooting

### DNS not resolving
1. Check Pi-hole status:
   sudo docker exec pihole pihole status

2. Check if containers are running:
   ```bash
   sudo docker compose ps
   ```

3. Check logs for errors:
   ```bash
   sudo docker compose logs pihole
   ```

### Can't access web interface
1. Verify port 8443 is listening:
   ```bash
   sudo netstat -plnt | grep 8443
   ```

2. Check firewall settings:
   ```bash
   sudo ufw status  # If using UFW, adjust for your distro's firewall service
   ```

3. Try accessing via IP: `http://<your-ip>:8443/admin`

### Containers keep restarting
Check logs for errors:
```bash
sudo docker compose logs --tail=50
```

### Performance issues on Raspberry Pi
The configuration is already optimized for Raspberry Pi with:
- Database size limits
- Memory constraints
- Reduced query retention
- tmpfs for logs (reduces SD card wear)

## Maintenance

### Automatic Updates
Blocklists update automatically via cron jobs inside the Pi-hole container.

### Manual Updates

**Update Docker images:**
Keep in mind if you want to update your pihole version, it will need to be done via Docker.

```bash
cd ~/pihole  # or your pihole directory
sudo docker compose pull
sudo docker compose up -d
```


**Update blocklists:**
```bash
sudo docker exec pihole pihole -g
```

### Backup
Your Pi-hole configuration is stored in Docker volumes. To backup:

```bash
sudo docker exec pihole tar czf /tmp/pihole-backup.tar.gz /etc/pihole
sudo docker cp pihole:/tmp/pihole-backup.tar.gz ./pihole-backup-$(date +%Y%m%d).tar.gz
```

## Sharing with Friends
This setup is **fully portable** - no hardcoded paths! The configuration uses relative paths that work for any user.

To share this setup:

1. **Copy the entire directory** to any location
   ```bash
   # Example: Copy to another user's home
   cp -r ~/pihole /home/otherUser/pihole
   ```

2. **Edit the configuration** - They only need to change:
   - Timezone (`TZ`)
   - Password (`FTLCONF_webserver_api_password`)
   **No path changes needed!** All paths are relative to the docker-compose.yml location.

3. **Start the stack** from the pihole directory:
   ```bash
   cd ~/pihole  # or wherever they copied it
   sudo docker compose up -d
   ```

### Required Files for Distribution
When sharing, include these files:
```
pihole/
├── docker-compose.yml       # Main configuration
├── README.md                # This documentation
├── unbound_conf.d/          # Unbound custom config
│   └── 00-recursive.conf
└── unbound_root.hints       # Root DNS servers list
```

All paths in docker-compose.yml are relative, so the setup works anywhere!

## Portability & Testing

### Why This Setup is Portable

This configuration uses **relative paths** exclusively:
- `./unbound_conf.d`
- `./unbound_root.hints`
- Docker named volumes for Pi-hole data (automatically managed)

**Result:** Copy the directory anywhere, and it just works!

### Testing Portability

To verify the setup works from any location:

```bash
# Copy to a different location
cp -r ~/pihole ~/test-pihole
cd ~/test-pihole

# Start the stack
sudo docker compose up -d

# Verify it's working
sudo docker exec pihole pihole status

# Clean up the test:
cd ~/test-pihole
sudo docker compose down
cd ~ && rm -rf ~/test-pihole
```

## Support & Resources

- **Pi-hole Documentation:** https://docs.pi-hole.net/
- **Unbound Documentation:** https://nlnetlabs.nl/documentation/unbound/
- **Docker Compose Reference:** https://docs.docker.com/compose/

## License & Credits

- **Pi-hole:** https://github.com/pi-hole/pi-hole (EUPL-1.2)
- **Unbound:** https://github.com/NLnetLabs/unbound (BSD-3-Clause)
- **crazymax/unbound Docker image:** https://github.com/crazy-max/docker-unbound

**Last Updated:** February 25, 2026
