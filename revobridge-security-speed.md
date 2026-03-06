# Revobridge — Security & Page Speed Optimization

Run these on your VPS AFTER the homepage is deployed and working.

---

## PART 1: PAGE SPEED

### 1.1 Update Nginx Gzip (replace the current gzip.conf)

```bash
nano /etc/nginx/conf.d/gzip.conf
```

Replace with:

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_min_length 256;
gzip_types
  text/plain
  text/css
  text/javascript
  application/javascript
  application/json
  application/xml
  image/svg+xml
  font/woff2;
```

Save: Ctrl+O → Enter → Ctrl+X

---

### 1.2 Update cache config (replace the current cache.conf)

```bash
nano /etc/nginx/conf.d/cache.conf
```

Replace with:

```nginx
# Browser caching handled in revobridgeai.conf location blocks
# This file intentionally left minimal to avoid conflicts
```

Save: Ctrl+O → Enter → Ctrl+X

---

### 1.3 Update Revobridge Nginx config with full optimizations

```bash
nano /etc/nginx/conf.d/revobridgeai.conf
```

Replace EVERYTHING (Certbot will re-add its SSL lines after):

```nginx
server {
    listen 80;
    server_name revobridgeai.com www.revobridgeai.com;
    root /var/www/revobridge-academy;
    index index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # Homepage
    location / {
        try_files $uri $uri/ =404;
    }

    # Academy / Training page
    location /training {
        try_files /training.html =404;
    }

    # Cache static assets aggressively
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public";
        access_log off;
    }

    location ~* \.(woff|woff2|ttf|otf|eot)$ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Block access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Block access to sensitive files
    location ~* \.(git|gitignore|md|json|lock|yml|yaml|sh|env)$ {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Custom error pages (optional — create these files later)
    # error_page 404 /404.html;
    # error_page 500 502 503 504 /50x.html;
}
```

Save: Ctrl+O → Enter → Ctrl+X

---

### 1.4 Re-add SSL after config change

```bash
nginx -t
systemctl reload nginx
certbot --nginx -d revobridgeai.com -d www.revobridgeai.com
```

Select "reinstall existing certificate" if prompted.
Choose "redirect HTTP to HTTPS" when asked.

---

### 1.5 Enable HTTP/2 (faster parallel loading)

After Certbot runs, open the config again:

```bash
nano /etc/nginx/conf.d/revobridgeai.conf
```

Find the line that says:
```
listen 443 ssl;
```

Change it to:
```
listen 443 ssl http2;
```

Save and reload:
```bash
nginx -t
systemctl reload nginx
```

---

## PART 2: SECURITY

### 2.1 Auto-renew SSL certificates

Certbot usually sets this up automatically. Verify:

```bash
systemctl status certbot-renew.timer
```

If it shows "active", you're good. If not:

```bash
echo "0 3 * * * root certbot renew --quiet --post-hook 'systemctl reload nginx'" >> /etc/crontab
```

Test renewal:
```bash
certbot renew --dry-run
```

---

### 2.2 Secure SSH (prevent brute force)

You had 25,570 failed login attempts last time. Let's fix that.

**Install fail2ban:**

```bash
dnf install epel-release -y
dnf install fail2ban -y
```

**Configure it:**

```bash
nano /etc/fail2ban/jail.local
```

Paste:

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 86400
```

This bans any IP that fails SSH login 3 times for 24 hours.

Save: Ctrl+O → Enter → Ctrl+X

**Start fail2ban:**

```bash
systemctl enable fail2ban
systemctl start fail2ban
```

**Verify it's running:**
```bash
fail2ban-client status sshd
```

---

### 2.3 Change SSH port (optional but recommended)

Default port 22 gets hammered by bots. Changing it stops 99% of automated attacks.

```bash
nano /etc/ssh/sshd_config
```

Find the line:
```
#Port 22
```

Change to:
```
Port 2222
```

Open the new port in firewall BEFORE restarting SSH:

```bash
firewall-cmd --permanent --add-port=2222/tcp
firewall-cmd --reload
```

Then restart SSH:
```bash
systemctl restart sshd
```

**IMPORTANT:** From now on, connect with:
```bash
ssh -p 2222 root@198.177.123.153
```

If you do this, also update fail2ban:
```bash
nano /etc/fail2ban/jail.local
```
Change `port = ssh` to `port = 2222`

Then:
```bash
systemctl restart fail2ban
```

---

### 2.4 Set up SSH key authentication (recommended)

On your LOCAL Windows machine (PowerShell):

```powershell
ssh-keygen -t ed25519 -C "jazz@revobridge"
```

Press Enter for defaults. Then copy the key to your VPS:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@198.177.123.153 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

(If you changed the port, add `-p 2222` to the ssh command)

Test that key login works:
```bash
ssh root@198.177.123.153
```

It should log in without asking for a password. Once confirmed, disable password login:

```bash
nano /etc/ssh/sshd_config
```

Find and change:
```
PasswordAuthentication yes
```
to:
```
PasswordAuthentication no
```

```bash
systemctl restart sshd
```

---

### 2.5 Keep the system updated

```bash
dnf update -y
```

Set up automatic security updates:

```bash
dnf install dnf-automatic -y
nano /etc/dnf/automatic.conf
```

Find `apply_updates` and set:
```
apply_updates = yes
```

Enable:
```bash
systemctl enable --now dnf-automatic.timer
```

---

### 2.6 Firewall audit

Check what's currently open:
```bash
firewall-cmd --list-all
```

You should ONLY see: ssh (or 2222/tcp), http, https, and cockpit (if you use it).

Remove anything unnecessary:
```bash
firewall-cmd --permanent --remove-service=cockpit
firewall-cmd --reload
```

---

## PART 3: MONITORING (optional but useful)

### 3.1 Check disk space
```bash
df -h
```

### 3.2 Check memory
```bash
free -h
```

### 3.3 Check what's using resources
```bash
top
```

### 3.4 Check Nginx error logs
```bash
tail -50 /var/log/nginx/error.log
```

### 3.5 Check access logs
```bash
tail -50 /var/log/nginx/access.log
```

---

## CHECKLIST

Run through this after completing the above:

- [ ] Site loads over HTTPS with no mixed content warnings
- [ ] HTTP redirects to HTTPS automatically
- [ ] Gzip is working (check: curl -H "Accept-Encoding: gzip" -I https://revobridgeai.com)
- [ ] Security headers present (check: curl -I https://revobridgeai.com)
- [ ] .git and .gitignore not accessible (visit https://revobridgeai.com/.git — should show 403)
- [ ] fail2ban running and protecting SSH
- [ ] SSL auto-renewal working (certbot renew --dry-run)
- [ ] System updates configured
- [ ] Google PageSpeed score above 90 (test: pagespeed.web.dev)

---

## ORDER OF OPERATIONS

1. Deploy the homepage first (get it live and working)
2. Run Part 1 (page speed) — takes 5 minutes
3. Run Part 2.1-2.2 (SSL renewal + fail2ban) — takes 5 minutes
4. Part 2.3-2.4 (SSH port + keys) — do this when you have 15 uninterrupted minutes
5. Part 2.5-2.6 (updates + firewall) — takes 2 minutes
6. Run the checklist
