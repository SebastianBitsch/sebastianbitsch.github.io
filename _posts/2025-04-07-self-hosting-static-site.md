---
title: 'Self-Hosting a Static Website'
date: 2025-04-07
permalink: /posts/2025/04/self-hosting-static-website/
tags:
  - website
  - hosting
  - server
  - code
---

The simplest way of hosting your project from a home server

# Background

During a recent robotics project, I found myself searching for a tool that could generate printable ArUco markers of precise dimensions. ArUco markers are QR-code-looking markers used in robotics and computer vision applications for pose estimation and camera calibration - and it is super important they are the size you expect. Despite their popularity, my search for a good tool to generate them came up empty.

The only good option seemed to be writing writing my own. However, ensuring the markers maintained their exact dimensions when printed proved non-trivial. Converting from pixel dimensions to physical measurements and accounting for printer settings, DPI configurations, and scaling issues was just a mess.

After solving these problems through trial and error mostly, I figured I would save someone else the trouble, and host the tool as a small static website rather than letting the code gather dust.

![Example of a image the site can generate](/images/arucogrid.png)

There was however no way i was going to pay for a server somewhere, and already having a home server setup (an old Chromebook laptop), I figured this was a good opportunity to try self-hosting a website.

The development process was pretty straightforward: I ported the `OpenCV` `python` code to `pyscript`, which turned out to be a pleasant experinence - and wrote some basic `HTML` and `Javascript` to complete the site.

Now I just needed to make it publicly accessible. The documentation that follows outlines my approach to setting up a home server for hosting static websites. I wrote it for my self to remember steps - but am sharing it here in the hope it could prove useful to anyone wanting to do something similar.

The tool is by the way available at [arucogrid.com](arucogrid.com) 


--- 

# Self-Hosting a Static Website: Complete Guide

This guide details how to set up, configure, and maintain a static website on your own hardware using Ubuntu, Nginx, and Cloudflare Tunnel to make it accessible on the public internet.

## System Overview

- **Domain**: Your registered domain (e.g., example.com). Bought mine from GoDaddy
- **Server**: Ubuntu PC/server on your local network. Doesn't need to be anything special, I used a Chromebook.
- **Web Server**: Nginx
- **Connection Method**: Cloudflare Tunnel (useful for bypassing CGNAT or avoiding port forwarding)
- **Analytics**: GoAccess for basic usage statistics

## Installation Process

### 1. Server Setup

```bash
# Update the system
sudo apt update
sudo apt upgrade

# Install Nginx web server
sudo apt install nginx

# Enable and start Nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 2. Website Configuration

```bash
# Create directory for website files
sudo mkdir -p /var/www/example.com

# Set appropriate permissions
sudo chown -R $USER:$USER /var/www/example.com
sudo chmod -R 755 /var/www/example.com

# Copy website files to the directory
# (Example command if files were in your home directory)
# cp -r ~/mywebsite/* /var/www/example.com/
```

### 3. Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Use this configuration (replace `example.com` with your domain):

```nginx
server {
    listen 80;
    listen [::]:80;
    
    root /var/www/example.com;
    index index.html;
    
    server_name example.com www.example.com;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Basic security headers
    add_header X-Content-Type-Options "nosniff";
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # Remove default site
sudo nginx -t  # Test configuration
sudo systemctl reload nginx
```

### 4. Firewall Configuration

```bash
# Install and configure UFW firewall
sudo apt install ufw

# Allow SSH and Nginx
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### 5. Domain and DNS Setup

1. **Register a domain** with a domain registrar of your choice (GoDaddy, Namecheap, etc.)
2. **Create a Cloudflare account** and add your domain
3. **Update nameservers** with your registrar to use Cloudflare's nameservers. Follow the instructions on Cloudflare and it should work out. 
4. **Enable HHTPS traffic only**

#### Option A: Traditional Port Forwarding (If Not Using Cloudflare Tunnel)

If you don't need Cloudflare Tunnel (e.g., you have a static IP or can port forward. If your WiFi is cellular and not cabled you will need a tunnel):

1. Access your router's admin panel
2. Find the "Port Forwarding" section
3. Create rules to forward ports 80 and 443 to your server's internal IP address. Make sure you set the static IP outside the range of where your router assigns. e.g. if your router assigns in the range 100 to 200. Give it .50

Example HTTP rule:
- Name: HTTP Web Server
- Protocol: TCP
- External ports: 80
- Internal IP: Your server's local IP (e.g., 192.168.1.50)
- Internal port: 80

Example HTTPS rule:
- Name: HTTPS Web Server
- Protocol: TCP
- External ports: 443
- Internal IP: Your server's local IP (e.g., 192.168.1.50)
- Internal port: 443

#### Option B: Cloudflare Tunnel Setup (For CGNAT or No Port Forwarding)

If you're behind CGNAT or can't use port forwarding (because your router is using cellular for instance):

```bash
# Install cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# Authenticate with Cloudflare
cloudflared tunnel login

# Create a tunnel
cloudflared tunnel create my-website-tunnel

# Configure the tunnel
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

Add this configuration (replace `YOUR_TUNNEL_ID` with the actual ID from the commands above):

```yaml
tunnel: YOUR_TUNNEL_ID
credentials-file: /home/username/.cloudflared/YOUR_TUNNEL_ID.json

ingress:
  - hostname: example.com
    service: http://localhost:80
  - hostname: www.example.com
    service: http://localhost:80
  - service: http_status:404
```

Route domain to tunnel:

```bash
cloudflared tunnel route dns my-website-tunnel example.com
cloudflared tunnel route dns my-website-tunnel www.example.com
```

Install as a service:

```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

### 6. Analytics Setup (Not needed)

```bash
# Install GoAccess for simple analytics
sudo apt install goaccess

# Create script to generate statistics
sudo nano /usr/local/bin/generate_stats.sh
```

Add this content:

```bash
#!/bin/bash
LOGFILE="/var/log/nginx/access.log"
OUTDIR="/var/www/example.com/stats"
mkdir -p $OUTDIR
goaccess $LOGFILE -o $OUTDIR/report.html --log-format=COMBINED
```

Make it executable and set up a cron job:

```bash
sudo chmod +x /usr/local/bin/generate_stats.sh
sudo crontab -e
```

Add this line for daily stats generation:
```
0 0 * * * /usr/local/bin/generate_stats.sh
```

## Server Configuration for Laptops

If you're using a laptop as your server, configure it to keep running when the lid is closed:

```bash
# Edit the logind.conf file
sudo nano /etc/systemd/logind.conf

# Find and change the lines
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore

# Restart the systemd-logind service
sudo systemctl restart systemd-logind

# Disable sleep and hibernation for extra security
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## Post-Restart Procedures

If your system restarts for any reason, verify that services started properly:

1. **Check service status**:
   ```bash
   sudo systemctl status nginx
   sudo systemctl status cloudflared  # If using Cloudflare Tunnel
   ```

2. **Start services if needed**:
   ```bash
   sudo systemctl start nginx
   sudo systemctl start cloudflared  # If using Cloudflare Tunnel
   ```

3. **Verify website accessibility**:
   - Try accessing your website in a browser
   - Check logs if there are issues: `sudo journalctl -u cloudflared -f`

4. **Re-enable services to start on boot if needed**:
   ```bash
   sudo systemctl enable nginx
   sudo systemctl enable cloudflared  # If using Cloudflare Tunnel
   ```

## Maintenance Guide

### Server Status Commands

```bash
# Check Nginx status
sudo systemctl status nginx

# Check Cloudflare Tunnel status (if using)
sudo systemctl status cloudflared

# Check Nginx error logs
sudo tail -f /var/log/nginx/error.log

# Check access logs
sudo tail -f /var/log/nginx/access.log

# Check Cloudflare Tunnel logs (if using)
sudo journalctl -u cloudflared -f
```

### Restart Procedures

#### Restart Nginx
```bash
sudo systemctl restart nginx
```

#### Restart Cloudflare Tunnel (if using)
```bash
sudo systemctl restart cloudflared
```

#### Full Server Restart
```bash
# Graceful restart of services
sudo systemctl restart nginx
sudo systemctl restart cloudflared  # If using Cloudflare Tunnel

# System reboot (if needed)
sudo reboot
```

### Update Procedures

#### Update Website Content
```bash
# Copy new files to web directory
cp -r ~/new-content/* /var/www/example.com/

# Check permissions if needed
sudo chown -R $USER:$USER /var/www/example.com
sudo chmod -R 755 /var/www/example.com
```

#### Update Server Software
```bash
# Update system packages
sudo apt update
sudo apt upgrade

# Restart services after major updates
sudo systemctl restart nginx
sudo systemctl restart cloudflared  # If using Cloudflare Tunnel
```

#### Update Cloudflare Tunnel Client (if using)
```bash
# Download latest version
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Install update
sudo dpkg -i cloudflared.deb

# Restart service
sudo systemctl restart cloudflared
```

### Backup Procedures

#### Backup Website Files
```bash
# Create a backup directory
mkdir -p ~/backups

# Backup the website files
sudo tar -czf ~/backups/website-$(date +%Y%m%d).tar.gz /var/www/example.com
```

#### Backup Configuration Files
```bash
# Backup Nginx configuration
sudo cp /etc/nginx/sites-available/example.com ~/backups/nginx-config-$(date +%Y%m%d)

# Backup Cloudflare Tunnel configuration (if using)
sudo cp /etc/cloudflared/config.yml ~/backups/cloudflared-config-$(date +%Y%m%d)

# Backup Cloudflare Tunnel credentials (if using)
sudo cp ~/.cloudflared/*.json ~/backups/cloudflared-creds-$(date +%Y%m%d).json
```

### Monitoring and Troubleshooting

#### Basic Health Checks
Create a health check script:

```bash
sudo nano /usr/local/bin/health_check.sh
```

Add this content:

```bash
#!/bin/bash

# Check if Nginx is running
if systemctl is-active --quiet nginx; then
    echo "Nginx: Running"
else
    echo "Nginx: NOT running" | mail -s "Website Server Alert" your@email.com
    systemctl start nginx
fi

# Check if Cloudflare Tunnel is running (if using)
if systemctl is-active --quiet cloudflared; then
    echo "Cloudflare Tunnel: Running"
else
    echo "Cloudflare Tunnel: NOT running" | mail -s "Website Server Alert" your@email.com
    systemctl start cloudflared
fi

# Check disk space
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$DISK_USAGE" -gt 80 ]; then
    echo "Disk space critical: $DISK_USAGE%" | mail -s "Website Server Alert" your@email.com
fi
```

Make it executable and set up a cron job:

```bash
sudo chmod +x /usr/local/bin/health_check.sh
sudo crontab -e
```

Add this line to run hourly:

```
0 * * * * /usr/local/bin/health_check.sh
```

#### Common Issues and Solutions

1. **Website Not Loading**
   - Check if Nginx is running: `sudo systemctl status nginx`
   - Check if Cloudflare Tunnel is running (if using): `sudo systemctl status cloudflared`
   - Check Nginx error logs: `sudo tail -f /var/log/nginx/error.log`
   - Check Cloudflare Tunnel logs (if using): `sudo journalctl -u cloudflared -f`

2. **Cloudflare Tunnel Disconnects** (if using)
   - Restart the tunnel: `sudo systemctl restart cloudflared`
   - Check for network issues: `ping -c 4 google.com`
   - Verify tunnel status in Cloudflare dashboard

3. **SSL/TLS Issues**
   - Check Cloudflare SSL/TLS settings in dashboard
   - Ensure "Full" or "Full (strict)" is selected

4. **Performance Issues**
   - Check system load: `top`
   - Check disk space: `df -h`
   - Review Nginx access logs for unusual traffic patterns

### Security Maintenance

#### Regular Security Updates
```bash
# Update and upgrade system
sudo apt update
sudo apt upgrade

# Install unattended upgrades for automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

#### Firewall Management
```bash
# Check firewall status
sudo ufw status

# Allow new service if needed
sudo ufw allow [port]/tcp

# Remove rule if no longer needed
sudo ufw delete allow [port]/tcp
```

#### Log Rotation
Ensure logs are properly rotated to prevent disk space issues:

```bash
# Check log rotation configuration
sudo nano /etc/logrotate.d/nginx
```

#### Security Audit
Perform a regular security audit:

```bash
# Install security audit tool
sudo apt install lynis

# Run security audit
sudo lynis audit system
```

## Additional Resources

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [GoAccess Documentation](https://goaccess.io/man)
- [DigitalOcean's Nginx guides](https://www.digitalocean.com/community/tutorials?q=nginx)
- [Mozilla's Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)