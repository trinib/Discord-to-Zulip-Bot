<p align="center">
  <h1 align="center">🤖 Discord → Zulip Bridge Bot</h1>
  <h3 align="center">A production-ready bridge that automatically forwards messages from Discord channels to Zulip streams. Perfect for communities transitioning between platforms or maintaining cross-platform communication.</h3>
</p>

### ✨ Features

- **🔁 Real-time Sync**: Instantly forwards messages from Discord to Zulip
- **📊 Multi-Channel Support**: Configure unlimited channel mappings
- **💾 Message Persistence**: Tracks last processed messages to avoid duplicates
- **⚡ Production Ready**: Systemd service with auto-restart and monitoring
- **🔒 Secure**: Proper secret management and error handling
- **📝 Rich Formatting**: Preserves mentions, attachments, and embeds

---

## 📋 Table of Contents
1. [Oracle Cloud Infrastructure (OCI) Setup Guide ](#-oracle-cloud-infrastructure-oci-setup-guide)
2. [Discord & Zulip API Setup](#-discord--zulip-api-setup)
3. [Create Discord → Zulip Bridge Bot](#-create-discord--zulip-bridge-bot)

---

# 🚀 Oracle Cloud Infrastructure (OCI) Setup Guide  

## 📋 Table of Contents
1. [Create & Prepare Your Oracle Cloud Account](#-step-1--create--prepare-your-oracle-cloud-account)
2. [Setup Networking (VCN & Subnets)](#-step-2--setup-networking-vcn--subnets)
3. [Create a Compute Instance (Ubuntu VM)](#-step-3--create-a-compute-instance-ubuntu-vm)
4. [Secure SSH Access](#-step-4--secure-ssh-access)
5. [Install Software Environment](#-step-5--install-software-environment)
6. [Monitoring & Logging](#-step-6--monitoring--logging)
7. [Backups & Scaling](#-step-7--backups--scaling)
8. [Hardening & Best Practices](#-step-8--hardening--best-practices)

---

## 🔹 Step 1 — Create & Prepare Your Oracle Cloud Account

1. Go to [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/).
2. Sign up (valid email, phone, and payment card required for verification).
3. After signup, you’ll receive:
   - **Tenancy**: Your top-level OCI container for resources.
   - **Home Region**: Choose the closest region to your users for optimal latency.
   - $300 free credits (trial) + Always Free eligible resources.

👉 **Best Practice**: Use separate **Admin** and **App/Service users** (via IAM). Avoid running everything as the root tenancy admin.

---

## 🔹 Step 2 — Setup Networking (VCN & Subnets)

OCI requires a **Virtual Cloud Network (VCN)** for all compute resources.

1. **VCN Wizard** (Recommended):
   - OCI Console → Networking → Virtual Cloud Networks → **Start VCN Wizard**.
   - Choose **VCN with Internet Connectivity**.
   - Name it `app-vcn`.

2. **Subnets**:
   - **Public Subnet**: For internet-facing instances (e.g., your bot VM needs outbound access to Discord/Zulip).
   - **Private Subnet**: For databases/internal services (optional but recommended for best practices).

3. **Security Lists**:
   - By default, ingress allows SSH (port 22).
   - Add only necessary rules (e.g., no inbound HTTP required for the bot—only outbound 443).

👉 **Best Practice**: Use **Network Security Groups (NSG)** for fine-grained rules instead of relying solely on Security Lists.

---

## 🔹 Step 3 — Create a Compute Instance (Ubuntu VM)

1. Console → Compute → Instances → **Create Instance**.
2. Name: `discord-zulip-bridge`.
3. **Image**: Ubuntu 22.04 or 24.04 (LTS recommended).
4. **Shape**:
   - *Always Free*: Ampere A1 (1 OCPU, 6 GB RAM).
   - Otherwise: VM.Standard.E2.1 (1 OCPU, 8 GB RAM).
5. **Networking**: Attach to your **Public Subnet** in `app-vcn`.
6. **SSH Keys**: Upload your public SSH key (generate with `ssh-keygen -t ed25519`).
7. **Boot Volume**: 50 GB (default).

Click **Create**. Wait a few minutes—your instance will display a **Public IP**.

👉 **Best Practice**: Disable password logins; use SSH keys only.

---

## 🔹 Step 4 — Secure SSH Access

1. Connect from your local machine:
   ```bash
   ssh -i ~/.ssh/id_ed25519 ubuntu@<public_ip>
   ```

2. Update the system:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

3. Configure the firewall:
   ```bash
   sudo ufw allow OpenSSH
   sudo ufw enable
   ```
   (Only SSH is open; all other ports are blocked.)

👉 **Best Practice**: Create a non-root user for daily operations and disable root SSH access.

---

## 🔹 Step 5 — Install Software Environment

On your Ubuntu VM, install essential tools:

```bash
# Essentials
sudo apt install -y python3 python3-venv python3-pip git curl unzip

# Optional: Docker
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu
```

👉 **Tip**: Use Python virtual environments or Docker to isolate dependencies.

---

## 🔹 Step 6 — Monitoring & Logging

1. Enable **OCI Monitoring & Logging** to push syslogs to OCI for review.
2. On the VM, check logs with:
   ```bash
   sudo journalctl -u discord-zulip -f
   ```
3. Optional: Set up **Prometheus/Grafana** or use Oracle Monitoring Alarms.

---

## 🔹 Step 7 — Backups & Scaling

- **Boot Volume Backup**: Schedule automatic backups in OCI → Block Storage.
- **Scaling**: Resize the instance shape if you need more than free-tier resources.
- **High Availability**: For mission-critical apps, run a backup instance in another availability domain/region.

---

## 🔹 Step 8 — Hardening & Best Practices

- Disable root login via SSH.
- Use `fail2ban` or UFW rate limits.
- Rotate API keys regularly (Discord & Zulip).
- Run system updates weekly (`apt upgrade`).
- Limit bot permissions (e.g., avoid granting Admin unless necessary).
- Restrict the Zulip bot to only needed streams.

---

## ✅ Final Outcome

After completing these steps, you’ll have:
- A properly configured OCI tenancy and VCN.
- A secure Ubuntu VM running your application with secrets managed in OCI Vault.
- Automated startup via systemd or Docker.
- Monitoring, backups, and a scaling plan in place.

---


# 🤖 Discord & Zulip API Setup

## 📋 Table of Contents
- [Prerequisites](#1-prerequisites)
- [Setup Instructions](#2-setup-instructions)
- [Discord Bot Scopes & Permissions Guide](#-discord-bot-scopes--permissions-guide)
- [Troubleshooting](#-troubleshooting)

---

## 1. Prerequisites

- ✅ Discord server admin access
- ✅ Zulip organization admin access  
- ✅ Oracle Cloud account (free tier eligible)
- ✅ Python 3.10+ familiarity
- ✅ Basic shell/command line experience

---

## 2. Setup Instructions

### A. Create Zulip Bot & API Key

1. Log into Zulip as admin → **Settings (gear)** → **Bots** → **Add a new bot**
2. Choose **Generic bot** type
3. Copy the **bot email** and **API key** (store securely)

> **Note**: New bot users aren't automatically subscribed to streams. For private streams, ensure the bot (or its owner) has appropriate permissions.

### B. Create Discord Bot with Message Content Intent

1. **Discord Developer Portal** → **Applications** → **New Application**
2. **Bot** tab → **Add Bot** → copy **Bot Token**
3. Enable **Message Content Intent** under Privileged Gateway Intents
4. Generate OAuth2 URL with scopes: `bot`, permissions: `Read Messages/View Channels`, `Read Message History`, `Send Messages`
5. Invite bot to your server using the generated URL

**Getting Channel IDs**: Enable Developer Mode (User Settings → Advanced → Developer Mode), then right-click channels → **Copy Channel ID**

---

## 🔍 Discord Bot Scopes & Permissions Guide

## ✅ Required Scopes

Select these two scopes:

1. **`bot`** - Allows the bot to join servers and function as a bot
2. **`applications.commands`** - Optional but recommended for future slash commands

---

## ✅ Bot Permissions (After selecting `bot` scope)

Enable these specific permissions:

### Essential Permissions
- **Read Messages/View Channels** 📖
- **Send Messages** 💬  
- **Read Message History** 📚
- **Embed Links** 🔗
- **Attach Files** 📎

### Optional (Recommended)
- **Use External Emojis** - If you want emojis to display properly
- **Add Reactions** - For bot responses/interactions

---

## 🚫 What NOT to Enable

- **`messages.read`** - This is for user accounts, not bots (against Discord ToS)
- **Administrator** - Keep permissions minimal
- **Manage Messages/Roles/Channels** - Not needed for bridging
- **Voice/Video permissions** - Irrelevant for text bridging

---

## 🔗 Pre-filled Invite URL Format

Here's the exact URL format with scopes and permissions pre-filled:

```
https://discord.com/oauth2/authorize?client_id=YOUR_BOT_CLIENT_ID&permissions=75776&scope=bot%20applications.commands
```

**To use this:**
1. Replace `YOUR_BOT_CLIENT_ID` with your actual bot's client ID
2. Visit the URL to invite your bot

**Permission Breakdown (75776):**
- Read Messages/View Channels: 1024
- Send Messages: 2048  
- Read Message History: 65536
- Embed Links: 16384
- Attach Files: 32768
- **Total**: 1024 + 2048 + 65536 + 16384 + 32768 = **75776**

---

## ❓ Troubleshooting

**If the bot can't read messages:**
- Double-check "Read Messages/View Channels" permission
- Verify the bot's role position in server settings
- Ensure the bot has access to specific channels

**If messages aren't sending:**
- Confirm "Send Messages" permission is enabled
- Check channel-specific permissions

---

# 🤖 Create Discord → Zulip Bridge Bot

## 📋 Table of Contents

### 🚀 Quick Start
- [1. Update System](#1-update-system)
- [2. Install Dependencies](#2-install-dependencies)

### 📁 Project Setup
- [3. Create Project Directory](#3-create-project-directory)
- [4. Set Up Python Virtual Environment](#4-set-up-python-virtual-environment)
- [5. Install Required Packages](#5-install-required-packages)

### 🔧 Configuration Files
- [6. Environment Variables (.env)](#6-environment-variables-env)
- [7. Channel Mapping (channel_mapping.json)](#7-channel-mapping-channel_mappingjson)
- [8. Message Tracking File](#8-message-tracking-file)

### 🤖 Bot Code
- [9. Create Main Bot Script (bot.py)](#9-create-main-bot-script-botpy)

### 🔄 Systemd Service
- [10. Create Systemd Service File](#10-create-systemd-service-file)
- [11. Enable and Start Service](#11-enable-and-start-service)
- [12. Check Service Status](#12-check-service-status)

### 🧪 Testing
- [13. Manual Test Run](#13-manual-test-run)
- [14. Test Commands in Discord](#14-test-commands-in-discord)

### 📊 Monitoring and Maintenance
- [View Logs](#view-logs)
- [Update Bot Code](#update-bot-code)

### 🛠️ Troubleshooting
- [Common Issues and Solutions](#common-issues-and-solutions)

---

### 1. Update System
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Dependencies
```bash
sudo apt install -y python3 python3-venv python3-pip git curl unzip
```

---

## 📁 Project Setup

### 3. Create Project Directory
```bash
mkdir ~/discord-zulip-bot
cd ~/discord-zulip-bot
```

### 4. Set Up Python Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
```

### 5. Install Required Packages
```bash
pip install discord.py zulip python-dotenv
```

---

## 🔧 Configuration Files

### 6. Environment Variables (`.env`)
Create `.env` file:
```bash
nano .env
```

Add your credentials:
```env
DISCORD_TOKEN=your_discord_bot_token_here
ZULIP_EMAIL=your-bot@your-domain.zulipchat.com
ZULIP_API_KEY=your_zulip_api_key_here
ZULIP_SITE=https://your-domain.zulipchat.com
```

### 7. Channel Mapping (`channel_mapping.json`)
Create channel mapping:
```bash
nano channel_mapping.json
```

```json
{
  "123456789012345678": {
    "zulip_stream": "discord-mirror",
    "zulip_topic": "general-chat",
    "discord_name": "general"
  },
  "234567890123456789": {
    "zulip_stream": "discord-mirror", 
    "zulip_topic": "announcements",
    "discord_name": "announcements"
  }
}
```

### 8. Message Tracking File
```bash
echo "{}" > last_message_ids.json
```

---

## 🤖 Bot Code

### 9. Create Main Bot Script (`bot.py`)
```bash
nano bot.py
```

```python
import discord
from discord.ext import commands, tasks
import zulip
import json
import os
import logging
from datetime import datetime, timedelta, timezone
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("bot.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Load configuration
DISCORD_TOKEN = os.getenv("DISCORD_TOKEN")
ZULIP_EMAIL = os.getenv("ZULIP_EMAIL")
ZULIP_API_KEY = os.getenv("ZULIP_API_KEY")
ZULIP_SITE = os.getenv("ZULIP_SITE")

# Load channel mapping
with open("channel_mapping.json", "r") as f:
    CHANNEL_MAPPING = json.load(f)

# Load last seen message IDs (one per channel)
LAST_MSG_FILE = "last_message_ids.json"
if os.path.exists(LAST_MSG_FILE):
    with open(LAST_MSG_FILE, "r") as f:
        LAST_MESSAGE_IDS = json.load(f)
else:
    LAST_MESSAGE_IDS = {}

# Initialize Zulip client
zulip_client = zulip.Client(
    email=ZULIP_EMAIL,
    api_key=ZULIP_API_KEY,
    site=ZULIP_SITE
)

# Configure Discord intents
intents = discord.Intents.default()
intents.message_content = True
intents.guilds = True
intents.guild_messages = True

# Initialize Discord bot
bot = commands.Bot(command_prefix="!", intents=intents)


def save_last_message_ids():
    """Save updated last message IDs to disk."""
    with open(LAST_MSG_FILE, "w") as f:
        json.dump(LAST_MESSAGE_IDS, f, indent=2)


def format_message_for_zulip(message, mapping):
    """Format Discord message for Zulip."""
    author = message.author.display_name if message.author else "System"
    timestamp = message.created_at.strftime("%Y-%m-%d %H:%M:%S UTC")
    content = message.content or ""

    # Mentions
    if message.mentions:
        for mention in message.mentions:
            content = content.replace(f"<@{mention.id}>", f"@{mention.display_name}")

    if message.channel_mentions:
        for channel in message.channel_mentions:
            content = content.replace(f"<#{channel.id}>", f"#{channel.name}")

    if message.role_mentions:
        for role in message.role_mentions:
            content = content.replace(f"<@&{role.id}>", f"@{role.name}")

    formatted_message = f"**{author}** ({timestamp}):\n{content}"

    # Attachments
    if message.attachments:
        formatted_message += "\n\n**Attachments:**"
        for attachment in message.attachments:
            formatted_message += f"\n- [{attachment.filename}]({attachment.url})"

    # Embeds
    if message.embeds:
        formatted_message += "\n\n**Embeds:**"
        for embed in message.embeds:
            if embed.title:
                formatted_message += f"\n**{embed.title}**"
            if embed.description:
                formatted_message += f"\n{embed.description}"
            if embed.url:
                formatted_message += f"\n[Link]({embed.url})"

    # Special flag for crossposts
    if message.flags.crossposted:
        formatted_message = f"📢 **Crosspost Announcement**\n{formatted_message}"

    return formatted_message


async def forward_to_zulip(message, mapping):
    """Forward a message to Zulip and update last seen ID."""
    content = format_message_for_zulip(message, mapping)

    result = zulip_client.send_message({
        "type": "stream",
        "to": mapping["zulip_stream"],
        "topic": mapping["zulip_topic"],
        "content": content
    })

    if result["result"] == "success":
        logger.info(
            f"Forwarded from #{mapping['discord_name']} "
            f"to {mapping['zulip_stream']}/{mapping['zulip_topic']}"
        )
        LAST_MESSAGE_IDS[str(message.channel.id)] = str(message.id)
        save_last_message_ids()
    else:
        logger.error(f"Failed to send to Zulip: {result}")


@bot.event
async def on_ready():
    logger.info(f"{bot.user} connected to Discord!")
    logger.info(f"Monitoring {len(CHANNEL_MAPPING)} channels")

    # Test Zulip connection
    try:
        result = zulip_client.get_profile()
        if result["result"] == "success":
            logger.info(f"Connected to Zulip as {result['email']}")
        else:
            logger.error(f"Zulip connection failed: {result}")
    except Exception as e:
        logger.error(f"Zulip connection error: {e}")

    # Catch up missed messages (last 24 hours)
    cutoff_time = datetime.now(timezone.utc) - timedelta(hours=24)
    for channel_id, mapping in CHANNEL_MAPPING.items():
        try:
            channel = bot.get_channel(int(channel_id))
            if not channel:
                logger.warning(f"Channel {channel_id} not found")
                continue

            last_id = LAST_MESSAGE_IDS.get(channel_id)
            history_kwargs = {"after": discord.Object(id=last_id)} if last_id else {}

            async for msg in channel.history(limit=200, oldest_first=True, **history_kwargs):
                if msg.created_at >= cutoff_time:
                    await forward_to_zulip(msg, mapping)

        except Exception as e:
            logger.error(f"Error during backlog sync for {channel_id}: {e}")


@bot.event
async def on_message(message):
    """Forward normal messages and announcement crossposts."""
    channel_id = str(message.channel.id)
    if channel_id not in CHANNEL_MAPPING:
        return

    mapping = CHANNEL_MAPPING[channel_id]
    await forward_to_zulip(message, mapping)


@bot.event
async def on_message_edit(before, after):
    """Forward edits of crossposted announcements too."""
    channel_id = str(after.channel.id)
    if channel_id in CHANNEL_MAPPING:
        mapping = CHANNEL_MAPPING[channel_id]
        await forward_to_zulip(after, mapping)


@bot.command(name="status")
async def status_command(ctx):
    if ctx.author.guild_permissions.administrator:
        embed = discord.Embed(
            title="Discord-Zulip Bridge Status",
            color=0x00ff00,
            timestamp=datetime.utcnow()
        )
        embed.add_field(
            name="Monitored Channels", value=len(CHANNEL_MAPPING), inline=True
        )
        embed.add_field(name="Zulip Connection", value="✅ Connected", inline=True)
        await ctx.send(embed=embed)


# -------------------------------
# Maintenance: Clean up bot.log
# -------------------------------
@tasks.loop(hours=24)
async def cleanup_logs():
    """Clean up bot.log if older than 7 days."""
    log_file = "bot.log"
    if not os.path.exists(log_file):
        return
    mtime = datetime.fromtimestamp(os.path.getmtime(log_file))
    if datetime.now() - mtime > timedelta(days=7):
        open(log_file, "w").close()
        logger.info("Cleaned up bot.log after 7 days.")


@bot.event
async def setup_hook():
    cleanup_logs.start()


if __name__ == "__main__":
    bot.run(DISCORD_TOKEN)
```

---

## 🔄 Systemd Service

### 10. Create Systemd Service File
```bash
sudo nano /etc/systemd/system/discord-zulip-bot.service
```

```ini
[Unit]
Description=Discord to Zulip Bridge Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/discord-zulip
ExecStart=/opt/discord-zulip/venv/bin/python /opt/discord-zulip/bot.py
Restart=always
Environment="PATH=/opt/discord-zulip/venv/bin"

[Install]
WantedBy=multi-user.target
```

### 11. Enable and Start Service
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart discord-zulip-bot
sudo systemctl status discord-zulip-bot
```

### 12. Check Service Status
```bash
# View logs in real-time
sudo journalctl -u discord-zulip-bot -f

# Check service status
sudo systemctl status discord-zulip-bot

# Restart service if needed
sudo systemctl restart discord-zulip-bot
```

---

## 🧪 Testing

### 13. Manual Test Run
```bash
cd ~/discord-zulip-bot
source venv/bin/activate
python bot.py
```

### 14. Test Commands in Discord
```bash
!status    # Check bot status
!test      # Send test message to Zulip
```

---

## 📊 Monitoring and Maintenance

### View Logs
```bash
# Real-time logs
sudo journalctl -u discord-zulip-bot -f

# Last 100 lines
sudo journalctl -u discord-zulip-bot -n 100

# Logs since boot
sudo journalctl -u discord-zulip-bot --since boot
```

### Update Bot Code
```bash
cd ~/discord-zulip-bot
sudo systemctl stop discord-zulip-bot
git pull  # if using version control
sudo systemctl start discord-zulip-bot
```

---

## 🛠️ Troubleshooting

### Common Issues and Solutions

**Bot won't start:**
```bash
# Check syntax errors
python -m py_compile bot.py

# Test environment variables
source venv/bin/activate
python -c "from dotenv import load_dotenv; load_dotenv(); import os; print('DISCORD_TOKEN:', bool(os.getenv('DISCORD_TOKEN')))"
```

**Permissions issues:**
```bash
sudo chown -R ubuntu:ubuntu ~/discord-zulip-bot
sudo chown -R ubuntu:ubuntu /opt/discord-zulip
chmod +x /opt/discord-zulip/bot.py
```

**Service not starting:**
```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed discord-zulip-bot
sudo systemctl start discord-zulip-bot
```

---

## 🎉 Success!
Your Discord → Zulip bridge is now running as a production service! The bot will:
- Automatically start on system boot
- Survive crashes with auto-restart
- On startup, does a 24-hour backlog sync to catch up up to 200 missed messages and post per channel.
- Track message history to avoid duplicates
- Ignores anything older than 24h.
- Provide status commands for monitoring
- Log all activity for debugging


