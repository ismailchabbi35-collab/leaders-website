# 🚀 Deploying Leaders Tech for Free on Oracle Cloud

This guide walks you through deploying your Leaders Tech website on a **free Oracle Cloud VPS** with HTTPS, auto-restart, and production-grade security.

---

## What you'll get

- ✅ A live website at `https://your-domain.com` (or Oracle's free subdomain)
- ✅ Free SSL certificate (auto-renewing)
- ✅ Auto-restart on crash or server reboot
- ✅ Persistent SQLite database + uploaded images
- ✅ **Total cost: $0/month, forever**

---

## Prerequisites

- A computer with internet access
- 30 minutes of your time
- (Optional) A domain name like `leaderstech.dz` — if you don't have one, you'll use Oracle's IP address directly

---

## Step 1 — Create your free Oracle Cloud account

1. Go to **[oracle.com/cloud/free](https://www.oracle.com/cloud/free/)**
2. Click **Start for free**
3. Sign up with your email and verify your identity (you'll need a credit card for verification, but **you will not be charged**)
4. Choose a **home region** close to Algeria (e.g., **Marseille** or **Frankfurt**)
5. Complete the signup and wait for your account to be provisioned (5–10 minutes)

> **What you get for free, forever:**
> - 1 AMD VM (1/8 OCPU, 1 GB RAM) — or up to 4 ARM VMs (24 GB RAM total!)
> - 200 GB block storage
> - 10 GB object storage
> - 10 TB/month outbound data transfer

---

## Step 2 — Create your VM instance

1. In the Oracle Cloud Console, click the hamburger menu (☰) → **Compute** → **Instances**
2. Click **Create Instance**
3. Name it: `leaders-tech`
4. Under **Image**, click **Edit** → select **Canonical Ubuntu 22.04** (not Oracle Linux)
5. Under **Shape**, click **Edit** → select **VM.Standard.E2.1.Micro** (Always Free Eligible)
6. Under **SSH keys**, click **Save private key** — this downloads a `.key` file. **Keep this safe** — you need it to log in.
7. Click **Create**

Wait 1–2 minutes for the VM to start. Once it shows "Running", note the **Public IP Address**.

---

## Step 3 — Open port 80 and 443 in the firewall

Oracle Cloud blocks all incoming traffic by default. You need to open port 80 (HTTP) and 443 (HTTPS).

### In the Oracle Console:
1. Click the hamburger menu → **Networking** → **Virtual Cloud Networks**
2. Click your VCN (named like `Default-VCN-...`)
3. Click **Security Lists** → click the default security list
4. Click **Add Ingress Rules** and add TWO rules:

| Source CIDR | IP Protocol | Destination Port | Description |
|---|---|---|---|
| `0.0.0.0/0` | TCP | `80` | HTTP |
| `0.0.0.0/0` | TCP | `443` | HTTPS |
| `0.0.0.0/0` | TCP | `3000` | Next.js (optional, for testing) |

5. Click **Add Ingress Rules**

---

## Step 4 — SSH into your VM

### On Mac / Linux:
```bash
# Make your SSH key readable only by you
chmod 400 ~/Downloads/ssh-key-*.key

# Connect (replace with your VM's public IP)
ssh -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_VM_PUBLIC_IP
```

### On Windows:
- Use **PowerShell** with the same command, or
- Download **[PuTTY](https://www.putty.org/)** and convert your `.key` file to `.ppk` using **PuTTYgen**

Type `yes` when prompted about the host key. You should now see a prompt like `ubuntu@instance:~$`.

---

## Step 5 — Install system dependencies

Run these commands on your VM (copy-paste one block at a time):

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install Node.js 20.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Bun (faster than npm)
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# Install Nginx (reverse proxy for HTTPS)
sudo apt install -y nginx

# Install Certbot (free SSL certificates)
sudo apt install -y certbot python3-certbot-nginx

# Verify installations
node --version    # should show v20.x
bun --version     # should show 1.x
nginx -v          # should show nginx version
```

---

## Step 6 — Clone your project

You need to push your project to **GitHub** first. If you haven't already:

### On your local computer:
```bash
# In your project directory
git init
git add .
git commit -m "Initial commit — Leaders Tech website"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/leaders-tech.git
git push -u origin main
```

### On your VM:
```bash
# Clone the project
git clone https://github.com/YOUR_USERNAME/leaders-tech.git
cd leaders-tech
```

---

## Step 7 — Run the deploy script

```bash
bash deploy/oracle-vps.sh
```

This script will:
1. ✅ Create the `.env` file with the correct database path
2. ✅ Install project dependencies
3. ✅ Set up the SQLite database
4. ✅ Seed sample data (members, news, activities, etc.)
5. ✅ Build the production bundle

When it finishes, you'll see a success message with next steps.

**Test it locally:**
```bash
bun start
```
Then open `http://YOUR_VM_PUBLIC_IP:3000` in your browser. You should see the website!

Press `Ctrl+C` to stop the test server.

---

## Step 8 — Set up auto-restart with systemd

This keeps your website running 24/7, even after server reboots.

```bash
# Copy the service file
sudo cp deploy/leaders-tech.service /etc/systemd/system/

# Edit it to match your paths (replace /home/ubuntu with your actual path)
sudo nano /etc/systemd/system/leaders-tech.service
# Make sure these lines match YOUR setup:
#   User=ubuntu
#   WorkingDirectory=/home/ubuntu/leaders-tech
#   ExecStart=/home/ubuntu/.bun/bin/bun .next/standalone/server.js
# Save with Ctrl+O, Enter, Ctrl+X

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable leaders-tech    # auto-start on boot
sudo systemctl start leaders-tech     # start now

# Verify it's running
sudo systemctl status leaders-tech
```

You should see `active (running)`. If there's an error, check the logs:
```bash
sudo journalctl -u leaders-tech -f
```

---

## Step 9 — Set up Nginx (reverse proxy + HTTPS)

### If you have a domain name:

```bash
# Copy the Nginx config
sudo cp deploy/nginx.conf /etc/nginx/sites-available/leaders-tech

# Edit it — replace "your-domain.com" with your actual domain
sudo nano /etc/nginx/sites-available/leaders-tech
# Replace all instances of "your-domain.com" with "leaderstech.dz" (or whatever yours is)
# Also verify the path: /home/ubuntu/leaders-tech/public/uploads/
# Save with Ctrl+O, Enter, Ctrl+X

# Enable the site
sudo ln -s /etc/nginx/sites-available/leaders-tech /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default    # remove the default page

# Test the config
sudo nginx -t
# Should show "syntax is ok" and "test is successful"

# Reload Nginx
sudo systemctl reload nginx

# Point your domain to your VM's IP
# Go to your domain registrar (Namecheap, GoDaddy, etc.)
# Add an "A record" pointing to your VM's public IP

# Get your free SSL certificate
sudo certbot --nginx -d your-domain.com

# Certbot will ask:
#   - Enter your email (for expiry reminders)
#   - Agree to terms
#   - Redirect HTTP → HTTPS? YES
```

### If you DON'T have a domain name:

You can access the site directly via IP. Create a simpler Nginx config:

```bash
sudo tee /etc/nginx/sites-available/leaders-tech << 'EOF'
server {
    listen 80;
    server_name _;
    client_max_body_size 10M;

    location /uploads/ {
        alias /home/ubuntu/leaders-tech/public/uploads/;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/leaders-tech /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Visit `http://YOUR_VM_PUBLIC_IP` — your site is live!

> For HTTPS without a domain, you can use a free **Cloudflare Tunnel** instead of Let's Encrypt. See: [cloudflare.com/cloudflare-one](https://www.cloudflare.com/cloudflare-one/)

---

## Step 10 — Change the admin password (IMPORTANT!)

Your site is now live. **Change the default admin credentials immediately:**

1. Visit `https://your-domain.com/admin/login`
2. Log in with:
   - Email: `admin@leaderstech.dz`
   - Password: `LeadersTech2024!`
3. Click **My Profile** in the sidebar
4. Change your name, email, and password
5. Click **Save profile**

---

## How to update your website

When you make changes to the code and push to GitHub:

```bash
# SSH into your VM
ssh -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_VM_PUBLIC_IP

# Pull the latest code
cd leaders-tech
git pull

# Rebuild
bun run build

# Restart the service
sudo systemctl restart leaders-tech
```

That's it — your site is updated in under a minute.

---

## Backing up your data

Your data lives in two places:
- **Database:** `db/prod.db` (SQLite file)
- **Uploaded images:** `public/uploads/`

### Manual backup (download to your computer):
```bash
# On your local computer, download the database:
scp -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_VM_PUBLIC_IP:~/leaders-tech/db/prod.db ./backup-$(date +%Y%m%d).db

# Download uploaded images:
scp -r -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_VM_PUBLIC_IP:~/leaders-tech/public/uploads ./backup-uploads-$(date +%Y%m%d)
```

### Automated daily backup (cron job on the VM):
```bash
# Create a backup script
cat > ~/backup.sh << 'EOF'
#!/bin/bash
cd ~/leaders-tech
tar -czf ~/backups/leaders-tech-$(date +%Y%m%d).tar.gz db/ public/uploads/
# Keep only the last 7 backups
ls -t ~/backups/leaders-tech-*.tar.gz | tail -n +8 | xargs rm -f 2>/dev/null
EOF

mkdir -p ~/backups
chmod +x ~/backup.sh

# Run it every night at 3 AM
(crontab -l 2>/dev/null; echo "0 3 * * * ~/backup.sh") | crontab -
```

---

## Troubleshooting

### "Connection refused" when visiting the site
```bash
# Check if the app is running
sudo systemctl status leaders-tech

# If it's not running, check the logs:
sudo journalctl -u leaders-tech -n 50

# Common fix: restart it
sudo systemctl restart leaders-tech
```

### Nginx returns 502 Bad Gateway
```bash
# The Next.js server isn't running on port 3000
sudo systemctl status leaders-tech
sudo systemctl restart leaders-tech

# Verify it's listening:
curl http://127.0.0.1:3000
```

### Can't upload images (error 413)
Your Nginx `client_max_body_size` is too small. Edit `/etc/nginx/sites-available/leaders-tech` and increase it to `10M`, then `sudo systemctl reload nginx`.

### SSL certificate expired
```bash
sudo certbot renew
sudo systemctl reload nginx
```
Certbot auto-renews, but if it fails, run this manually.

### Database is locked
This happens if two processes try to write at once. Fix:
```bash
sudo systemctl restart leaders-tech
```

### Out of memory
If you chose the small (1 GB RAM) VM, the build might fail. Use swap:
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Oracle Cloud Free Tier limits

| Resource | Free allowance |
|---|---|
| AMD VM (1/8 OCPU, 1 GB RAM) | 2 instances |
| ARM VM (4 OCPU, 24 GB RAM) | 1 instance (4 VMs) |
| Block storage | 200 GB total |
| Outbound data | 10 TB/month |
| **Cost** | **$0 forever** |

> ⚠️ Oracle may reclaim Always Free instances if they're idle for 7+ days. To prevent this, just make sure your website gets occasional traffic (it will, since it's public).

---

## Need help?

- **Oracle Cloud docs:** [docs.oracle.com/en-us/iaas/Content/FreeTier](https://docs.oracle.com/en-us/iaas/Content/FreeTier/)
- **Nginx config:** [nginx.org/docs](https://nginx.org/en/docs/)
- **Certbot (SSL):** [certbot.eff.org](https://certbot.eff.org/)

---

## Quick reference — all commands in one place

```bash
# === One-time setup ===
ssh -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_VM_IP
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs nginx certbot python3-certbot-nginx
curl -fsSL https://bun.sh/install | bash && source ~/.bashrc

git clone https://github.com/YOUR_USERNAME/leaders-tech.git
cd leaders-tech
bash deploy/oracle-vps.sh

sudo cp deploy/leaders-tech.service /etc/systemd/system/
# Edit the service file to match your paths
sudo systemctl daemon-reload
sudo systemctl enable leaders-tech
sudo systemctl start leaders-tech

sudo cp deploy/nginx.conf /etc/nginx/sites-available/leaders-tech
# Edit to replace your-domain.com
sudo ln -s /etc/nginx/sites-available/leaders-tech /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d your-domain.com

# === Daily operations ===
sudo systemctl status leaders-tech    # check status
sudo systemctl restart leaders-tech   # restart after update
sudo journalctl -u leaders-tech -f    # view logs

# === Update the site ===
cd ~/leaders-tech
git pull && bun run build && sudo systemctl restart leaders-tech
```
