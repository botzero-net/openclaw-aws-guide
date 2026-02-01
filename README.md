# OpenClaw AWS Installation Guide

> Complete step-by-step guide to run OpenClaw on free AWS EC2 tier

## Prerequisites

- AWS account (free tier eligible)
- SSH client (Terminal on Mac/Linux, PuTTY on Windows)
- Basic command line knowledge

---

## Phase 1: EC2 Instance Setup (15 minutes)

### Step 1: Launch Instance
1. Log into AWS Console → EC2 → Launch Instance
2. **Name:** `openclaw-server`
3. **AMI:** Ubuntu Server 24.04 LTS (free tier eligible)
4. **Instance Type:** t3.micro (or t2.micro if t3 unavailable)
5. **Key Pair:** Create new RSA key pair
   - Name: `openclaw-key`
   - Download `.pem` file to `~/.ssh/openclaw-key.pem`
   - Run: `chmod 600 ~/.ssh/openclaw-key.pem`

### Step 2: Security Group Configuration
**Create new security group:**
| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | Your IP/32 | Admin access only |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | OpenClaw gateway (optional) |

**Important:** 
- Do NOT open port 22 to 0.0.0.0/0 (restrict to your IP)
- No other ports needed for basic operation
- Gateway can run on localhost only (more secure)

### Step 3: Storage
- Keep default 8GB gp3 (free tier)
- Enable "Delete on termination" (optional)

### Step 4: Launch
- Click "Launch Instance"
- Note the public IP address assigned

---

## Phase 2: Initial Server Setup (10 minutes)

### Step 1: Connect via SSH
```bash
ssh -i ~/.ssh/openclaw-key.pem ubuntu@YOUR_EC2_IP
```

### Step 2: System Updates
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git ufw fail2ban
```

### Step 3: Create OpenClaw User
```bash
sudo useradd -m -s /bin/bash openclaw
sudo usermod -aG sudo openclaw
sudo su - openclaw
```

---

## Phase 3: Node.js and pnpm Installation (10 minutes)

### Step 1: Install Node.js 22.x
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # Should show v22.x.x
```

### Step 2: Install pnpm
```bash
curl -fsSL https://get.pnpm.io/install.sh | sh -
source ~/.bashrc
pnpm --version
```

### Step 3: Configure npm/pnpm directories
```bash
mkdir -p ~/.local/bin
pnpm config set global-bin-dir ~/.local/bin
pnpm config set global-dir ~/.local/share/pnpm
pnpm config set cache-dir ~/.cache/pnpm
```

---

## Phase 4: OpenClaw Installation (15 minutes)

### Step 1: Clone Repository
```bash
cd ~
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

### Step 2: Install Dependencies
```bash
pnpm install
```

### Step 3: Build
```bash
pnpm run build
```

### Step 4: Initial Configuration
```bash
node dist/index.js gateway onboard
```

**Follow prompts:**
- Select "local" mode
- Choose Discord as primary channel (or your preferred)
- Enter your bot token when prompted
- Set model to `opencode/kimi-k2.5-free`

---

## Phase 5: Security Hardening (15 minutes)

### Step 1: Configure UFW Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from YOUR_LOCAL_IP to any port 22
sudo ufw enable
sudo ufw status
```

### Step 2: Configure fail2ban
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

**Add/modify:**
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status
```

### Step 3: SSH Hardening (Optional but Recommended)
```bash
sudo nano /etc/ssh/sshd_config
```

**Modify:**
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
sudo systemctl restart sshd
```

---

## Phase 6: ACIP Installation (5 minutes)

### Step 1: Install ACIP Security Layer
```bash
cd ~/openclaw
node dist/index.js skills install acip
```

### Step 2: Verify Installation
```bash
ls -la workspace/SECURITY.md
ls -la workspace/SECURITY.local.md
```

### Step 3: Customize Local Rules
```bash
nano workspace/SECURITY.local.md
```

**Add your restrictions:**
```markdown
- Only respond to DMs from: [your Discord ID]
- Only respond in guild: [your server ID]
- Never reveal tokens or credentials
- Confirm before sensitive actions
```

---

## Phase 7: pm2 Process Management (10 minutes)

### Step 1: Install pm2
```bash
pnpm add -g pm2
```

### Step 2: Create Ecosystem File
```bash
cd ~/openclaw
nano ecosystem.config.js
```

**Content:**
```javascript
module.exports = {
  apps: [{
    name: 'openclaw',
    script: './dist/index.js',
    args: 'gateway start',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production',
      OPENCLAW_CONFIG_PATH: '/home/openclaw/.openclaw/openclaw.json'
    },
    log_file: '/home/openclaw/.openclaw/logs/combined.log',
    out_file: '/home/openclaw/.openclaw/logs/out.log',
    error_file: '/home/openclaw/.openclaw/logs/error.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true
  }]
};
```

### Step 3: Create Log Directory
```bash
mkdir -p ~/.openclaw/logs
```

### Step 4: Start with pm2
```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup systemd
```

**Run the command pm2 outputs for systemd integration**

### Step 5: Monitor
```bash
pm2 status
pm2 logs openclaw
pm2 monit
```

---

## Phase 8: Verification and Testing (10 minutes)

### Step 1: Check Gateway Status
```bash
curl http://localhost:8080/status
```

Should return JSON with status information.

### Step 2: Test Discord Connection
- Send message to your bot
- Check logs: `pm2 logs openclaw --lines 50`
- Should see activity in logs

### Step 3: Test Commands
```bash
# From another terminal session
ssh -i ~/.ssh/openclaw-key.pem ubuntu@YOUR_EC2_IP
sudo su - openclaw
cd ~/openclaw
node dist/index.js doctor --non-interactive
```

### Step 4: Verify Security
```bash
# Check firewall
sudo ufw status verbose

# Check fail2ban
sudo fail2ban-client status sshd

# Check no unnecessary ports open
sudo netstat -tulpn | grep LISTEN
```

---

## Phase 9: Maintenance Operations

### Daily Checks
```bash
# Check system resources
free -h
df -h
pm2 status

# Check logs for errors
pm2 logs openclaw --lines 100 | grep -i error
```

### Weekly Updates
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Update OpenClaw
cd ~/openclaw
git pull
pnpm install
pnpm run build
pm2 restart openclaw
```

### Backup Configuration
```bash
# Backup OpenClaw config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d)

# Download to local machine
scp -i ~/.ssh/openclaw-key.pem ubuntu@YOUR_EC2_IP:~/.openclaw/openclaw.json.backup.* .
```

---

## Troubleshooting

### Issue: Gateway won't start
**Check:**
```bash
pm2 logs openclaw --lines 50
# Look for:
# - Port conflicts (8080 already in use)
# - Missing config files
# - Permission errors
```

**Fix port conflict:**
```bash
# Find process using 8080
sudo lsof -i :8080
# Kill if necessary (check what it is first!)
kill -9 [PID]
```

### Issue: Discord bot not responding
**Check:**
```bash
# Verify token is correct
grep -i token ~/.openclaw/openclaw.json

# Check Discord connection
pm2 logs openclaw | grep -i discord
```

### Issue: Out of memory
**Check:**
```bash
free -h
pm2 status
```

**Fix:**
```bash
# Add swap space
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Issue: SSH connection refused
**Check:**
```bash
# From AWS Console:
# 1. Verify instance is running
# 2. Check security group allows your IP
# 3. Verify key pair is correct

# From server (via AWS Systems Manager if available):
sudo systemctl status sshd
sudo cat /var/log/auth.log | tail -20
```

---

## Security Checklist

- [ ] SSH key-only authentication (no passwords)
- [ ] UFW firewall enabled, only port 22 open to your IP
- [ ] fail2ban active on SSH
- [ ] OpenClaw gateway bound to localhost (not 0.0.0.0)
- [ ] ACIP installed and active
- [ ] No secrets in environment variables or logs
- [ ] Regular backups configured
- [ ] pm2 auto-restart enabled
- [ ] Automatic security updates enabled

---

## Cost Optimization

### Free Tier Limits (12 months)
- 750 hours/month of t2/t3.micro (enough for 24/7)
- 30GB storage
- 15GB data transfer out

### After Free Tier
- t3.micro on-demand: ~$8.50/month
- t3.micro spot: ~$2.50/month (if acceptable)
- Consider reserved instances for long-term

### Cost Monitoring
```bash
# Install AWS CLI locally
aws configure
# Check estimated charges
aws ce get-cost-forecast
```

---

## Next Steps

1. **Install skills:** `node dist/index.js skills install [skill-name]`
2. **Configure cron jobs:** See hourly-goals.md for working patterns
3. **Set up monitoring:** CloudWatch or UptimeRobot for alerts
4. **Document your setup:** Save this guide with your modifications

---

**Guide Version:** 1.0  
**Last Updated:** 2026-02-01  
**Tested On:** Ubuntu 24.04 LTS, OpenClaw 2026.1.30
