# Revobridge Homepage — Deployment Guide

## STEP 1: Build with Claude Code

Feed the `revobridge-homepage-claude-code-prompt.md` to Claude Code.
Also give it the `revobridge-homepage-wireframe.html` file as reference.

Claude Code will create `index.html` in your REVOBRIDGE project folder.

---

## STEP 2: Test locally

Open `index.html` in Chrome. Check:
- [ ] All sections render correctly
- [ ] Mobile responsive (use Chrome DevTools → toggle device toolbar → 375px)
- [ ] Navbar links work (Academy → /training)
- [ ] Horizontal scroll works on case studies
- [ ] Animations fire on scroll
- [ ] No console errors (F12 → Console)

---

## STEP 3: Push to GitHub

```bash
git add .
git commit -m "added homepage"
git push
```

---

## STEP 4: Pull on VPS

```bash
ssh root@198.177.123.153
cd /var/www/revobridge-academy && git pull
```

---

## STEP 5: Update Nginx config

The current config serves training.html as the default. We need index.html as the homepage
and training.html at /training.

```bash
nano /etc/nginx/conf.d/revobridgeai.conf
```

Replace the contents with:

```nginx
server {
    listen 80;
    server_name revobridgeai.com www.revobridgeai.com;
    root /var/www/revobridge-academy;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /training {
        try_files /training.html =404;
    }

    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|svg|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

Save: Ctrl+O → Enter → Ctrl+X

Then:
```bash
nginx -t
systemctl reload nginx
```

Certbot will likely need to re-add its SSL lines. Run:
```bash
certbot --nginx -d revobridgeai.com -d www.revobridgeai.com
```

Select "reinstall existing certificate" if prompted.

---

## STEP 6: Test live

- https://revobridgeai.com → should show the homepage
- https://revobridgeai.com/training → should show the training/academy page
- Both should load over HTTPS
- Test on mobile

---

## STEP 7: Verify Meta Pixel

Install "Meta Pixel Helper" Chrome extension.
Visit both pages and confirm:
- Homepage: PageView fires
- Training page: PageView fires, Lead fires on form submit, InitiateCheckout fires on Enroll Now

---

## SITE STRUCTURE (current)

```
revobridgeai.com/              → index.html (Homepage — AI Agency)
revobridgeai.com/training      → training.html (Academy — Course landing page)
```

## SITE STRUCTURE (future — when you build service pages)

```
revobridgeai.com/                      → Homepage
revobridgeai.com/training              → Academy
revobridgeai.com/services/chatbots     → AI Chatbots service page
revobridgeai.com/services/automation   → Workflow Automation service page
revobridgeai.com/services/sales        → AI Sales Systems service page
revobridgeai.com/services/agents       → Custom AI Agents service page
revobridgeai.com/services/consulting   → AI Strategy & Consulting service page
revobridgeai.com/solutions/real-estate → Real Estate solutions page
revobridgeai.com/solutions/logistics   → Logistics solutions page
revobridgeai.com/solutions/ecommerce   → E-Commerce solutions page
revobridgeai.com/solutions/services    → Professional Services solutions page
revobridgeai.com/about                 → About / Team page
revobridgeai.com/contact               → Contact page
revobridgeai.com/blog                  → Blog (when ready)
```

For each new page, create the HTML file and add a location block in Nginx:

```nginx
location /services/chatbots {
    try_files /services-chatbots.html =404;
}
```

Or better: organize files in folders matching the URL structure.
