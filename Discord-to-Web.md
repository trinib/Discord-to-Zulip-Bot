# 🔥 Complete Guide: Discord → Oracle Bot → WordPress Webpage

## 1. Oracle Cloud Setup (Bot Environment)

1. Log in to Oracle Cloud and create a VM instance (Ubuntu or Debian recommended).
2. SSH into the VM.
3. Install dependencies:

```bash
sudo apt update && sudo apt install -y python3 python3-venv python3-pip git
```

4. Create a bot folder:

```bash
sudo mkdir -p /opt/discord-bot
sudo chown ubuntu:ubuntu /opt/discord-bot
cd /opt/discord-bot
```

5. Create a Python virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

6. Install required libraries:

```bash
pip install --upgrade pip
pip install discord.py python-dotenv requests
```

---

## 2. Create the Discord Bot

1. Go to [Discord Developer Portal](https://discord.com/developers/applications).
2. Create a **New Application** → name it (e.g. `DiscordWebBot`).
3. Go to **Bot** tab → *Add Bot*.

   * Enable **Message Content Intent**.
   * Copy the **Bot Token** (you’ll need this later).
4. Go to **OAuth2 → URL Generator**:

   * Check `bot` under scopes.
   * Under **Bot Permissions**, select:

     * `Read Messages/View Channels`
     * `Read Message History`
     * `Send Messages`
     * `View Channels`
   * Copy the generated URL and invite the bot to your server.

---

## 3. Bot Code (Oracle VM → WordPress Upload)

Create `bot.py` inside `~/discord-bot`:

```python
import discord
import json
import requests
import os
from dotenv import load_dotenv
from datetime import datetime, timedelta, timezone

# Load environment variables
load_dotenv()
DISCORD_TOKEN = os.getenv("DISCORD_TOKEN")
UPLOAD_URL = "https://YOURWEBSITE/save.php"  # change to your site

# Allowed channels (comma-separated IDs in .env for flexibility)
# Example in .env: ALLOWED_CHANNELS=
ALLOWED_CHANNELS = [
    int(cid.strip()) for cid in os.getenv("ALLOWED_CHANNELS", "").split(",") if cid.strip()
]

# Discord intents
intents = discord.Intents.default()
intents.messages = True
intents.message_content = True
intents.guilds = True

client = discord.Client(intents=intents)

# Store messages in memory before pushing
messages = []


async def save_to_wordpress():
    """Upload current message list to WordPress server."""
    try:
        data = {"json": json.dumps(messages, ensure_ascii=False)}
        r = requests.post(UPLOAD_URL, data=data, timeout=10)
        print("Upload response:", r.text[:200])
    except Exception as e:
        print("Error uploading:", e)


async def process_message(message: discord.Message):
    """Process a message and add to memory if valid."""
    if message.channel.id not in ALLOWED_CHANNELS:
        return
    if message.author == client.user:
        return

    entry = {
        "author": message.author.display_name,
        "content": message.content.replace("\n", "<br>"),
        "channel": message.channel.name,
        "timestamp": message.created_at.replace(tzinfo=timezone.utc).isoformat(),
        "crosspost": message.flags.crossposted
    }

    messages.append(entry)

    # Keep only last 14 days
    cutoff = datetime.now(timezone.utc) - timedelta(days=14)
    messages[:] = [
        m for m in messages
        if datetime.fromisoformat(m["timestamp"]) > cutoff
    ]

    await save_to_wordpress()


@client.event
async def on_ready():
    print(f"✅ Logged in as {client.user}")

    # Fetch backlog (last 24 hours, up to 100 messages per channel)
    cutoff = datetime.now(timezone.utc) - timedelta(days=7)
    for guild in client.guilds:
        for channel in guild.text_channels:
            if channel.id in ALLOWED_CHANNELS:
                try:
                    async for msg in channel.history(limit=100, after=cutoff, oldest_first=True):
                        await process_message(msg)
                except Exception as e:
                    print(f"⚠️ Could not fetch history for {channel}: {e}")


@client.event
async def on_message(message: discord.Message):
    await process_message(message)


client.run(DISCORD_TOKEN)
```

---

## 4. Store Bot Token Securely and Allow Channels via IDs

Create `.env` file in the same folder:

```
DISCORD_TOKEN=YOUR_DISCORD_BOT_TOKEN
ALLOWED_CHANNELS=186739025187438593,1054442870804860968
```

---

## 5. WordPress Hosting Setup

1. Inside your WordPress hosting (via File Manager or FTP), create a folder `cord` in the root or public_html folder without creating cord folder.
2. Inside `cord`, create **save.php**:

```php
<?php
if (isset($_POST['json'])) {
    $data = $_POST['json'];
    $file = __DIR__ . "/messages.json";
    file_put_contents($file, $data);
    echo "OK";
} else {
    echo "No data received";
}
?>
```

3. Make sure folder `cord/` is **writable** (permissions 755).

4. Create an empty messages file and a log placeholder:
```bash
echo "[]" > messages.json
touch bot.log
```


Now, every time the bot sends data, `messages.json` is updated.

---

## 6. Webpage (Display Messages)

In WordPress, create a custom page template or use a simple HTML block:

```html
<div id="chat-app">
  <div id="sidebar">
    <h3>Sports List</h3>
    <ul id="channel-list"></ul>
  </div>
  <div id="chat-container">
    <div id="messages"></div>
  </div>
</div>

<style>
/* Layout + theme */
#chat-app {
  display: flex;
  height: 80vh;
  width: 100%;
  max-width: 1200px;
  margin: auto;
  border: 1px solid #444;
  border-radius: 6px;
  overflow: hidden;
  background: #2f3136;
  color: #dcddde;
  font-family: "Segoe UI", sans-serif;
}
#sidebar {
  width: 220px;
  background: #202225;
  padding: 10px;
  overflow-y: auto;
}
#sidebar h3 {
  font-size: 20px;
  margin-bottom: 10px;
  color: #aaa;
  text-align: center;
}
#channel-list { list-style: none; padding: 0; margin: 0; }
#channel-list li {
  padding: 6px 12px;
  cursor: pointer;
  border-radius: 4px;
}
#channel-list li:hover, #channel-list li.active {
  background: #36393f;
  color: #fff;
}

/* Chat area */
#chat-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  background: #36393f;
  overflow-y: auto;          /* <-- container we scroll */
}
#messages {
  padding-bottom: 12px;
}
.message {
  padding: 8px 12px;
  border-bottom: 1px solid #444;
}
.meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}
.author { font-weight: bold; color: #7289da; }
.timestamp {
  font-weight: bold;
  font-size: 14px;
  color: #B5B5B5;
  text-align: center;
}

/* Mobile/tablet: horizontal channel list & timestamp right */
@media (max-width: 1024px) {
  #chat-app { flex-direction: column; }
  #sidebar { width: 100%; padding: 5px; overflow: hidden; }
  #channel-list {
    display:flex; gap:8px; overflow-x:auto; white-space:nowrap;
    -webkit-overflow-scrolling: touch; scrollbar-width: none; cursor: grab;
  }
  #channel-list::-webkit-scrollbar{ display: none; }
  #channel-list li { flex-shrink:0; background: #2b2d31; }

  .meta { justify-content: flex-start; }
  .author { margin-right: auto; }
  .timestamp { flex: unset; text-align: right; }
}
</style>

<script>
let allMessages = [];
let currentChannel = null;
let shouldJumpOnRender = true;
const MESSAGES_URL = "https://vortextreams.com/messages.json?nocache=";

function formatTimestamp(isoString) {
  const date = new Date(isoString);
  return date.toLocaleString(undefined, {
    year: "numeric", month: "short", day: "2-digit",
    hour: "2-digit", minute: "2-digit", second: "2-digit",
    hour12: false
  });
}

async function loadMessages(autoRefresh = false) {
  try {
    const resp = await fetch(MESSAGES_URL + Date.now(), { cache: "no-store" });
    if (!resp.ok) throw new Error("Fetch failed: " + resp.status);
    const data = await resp.json();
    allMessages = data || [];

    if (!currentChannel && allMessages.length > 0) {
      currentChannel = allMessages[0].channel;
      shouldJumpOnRender = true;
    }

    renderChannels();
    renderMessages(currentChannel, autoRefresh);
  } catch (err) {
    console.error("Error loading messages:", err);
  }
}

function renderChannels() {
  const channels = [...new Set(allMessages.map(m => m.channel))];
  const list = document.getElementById("channel-list");
  list.innerHTML = "";
  channels.forEach(ch => {
    const li = document.createElement("li");
    li.textContent = ch;
    if (ch === currentChannel) li.classList.add("active");
    li.onclick = () => {
      if (ch === currentChannel) return;
      currentChannel = ch;
      shouldJumpOnRender = true;
      renderMessages(ch, false);
      renderChannels();
    };
    list.appendChild(li);
  });
}

function renderMessages(channel, autoRefresh = false) {
  const messagesDiv = document.getElementById("messages");
  messagesDiv.innerHTML = "";
  if (!channel) return;

  const filtered = allMessages.filter(m => m.channel === channel);
  filtered.forEach(msg => {
    const div = document.createElement("div");
    div.className = "message";
    div.innerHTML = `
      <div class="meta">
        <span class="author">${escapeHtml(msg.author)}</span>
        <span class="timestamp">${formatTimestamp(msg.timestamp)}</span>
      </div>
      <div class="content">${msg.content || ""}</div>
    `;
    messagesDiv.appendChild(div);
  });

  if (shouldJumpOnRender && !autoRefresh) {
    shouldJumpOnRender = false;
    requestAnimationFrame(() => requestAnimationFrame(scrollToLatestTimestamp));
  }
}

function scrollToLatestTimestamp() {
  const container = document.getElementById("chat-container");
  const messagesDiv = document.getElementById("messages");
  const lastMsg = messagesDiv.lastElementChild;
  if (!lastMsg) return;

  const timestamp = lastMsg.querySelector(".timestamp");
  if (!timestamp) {
    container.scrollTo({ top: container.scrollHeight, behavior: "smooth" });
    return;
  }

  const containerRect = container.getBoundingClientRect();
  const tsRect = timestamp.getBoundingClientRect();
  const delta = tsRect.top - containerRect.top;

  // ✅ Slightly below timestamp area to reveal more of last message
  const extraOffset = 80; // adjust this value (in px) for how much of the message you want visible
  const desired = container.scrollTop + delta - (container.clientHeight - tsRect.height - extraOffset);

  const maxScroll = messagesDiv.scrollHeight - container.clientHeight;
  const final = Math.max(0, Math.min(desired, maxScroll));

  container.scrollTo({ top: final, behavior: "smooth" });
}

function escapeHtml(str) {
  if (!str) return "";
  return str.replace(/[&<>"']/g, m => ({ "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;" }[m]));
}

function enableSwipeScroll() {
  const list = document.getElementById("channel-list");
  let isDown = false, startX = 0, scrollLeft = 0, last = 0, vel = 0, rafId = null;

  function momentum() {
    if (Math.abs(vel) > 0.5) {
      list.scrollLeft -= vel;
      vel *= 0.95;
      rafId = requestAnimationFrame(momentum);
    } else {
      cancelAnimationFrame(rafId);
      rafId = null;
    }
  }

  list.addEventListener("mousedown", e => {
    isDown = true; startX = e.pageX; scrollLeft = list.scrollLeft; last = startX; vel = 0;
    cancelAnimationFrame(rafId);
  });
  list.addEventListener("mouseleave", () => { if (isDown) { isDown = false; rafId = requestAnimationFrame(momentum); }});
  list.addEventListener("mouseup", () => { if (isDown) { isDown = false; rafId = requestAnimationFrame(momentum); }});
  list.addEventListener("mousemove", e => {
    if (!isDown) return;
    e.preventDefault();
    const x = e.pageX;
    list.scrollLeft = scrollLeft - (x - startX);
    vel = x - last; last = x;
  });

  list.addEventListener("touchstart", e => {
    isDown = true; startX = e.touches[0].pageX; scrollLeft = list.scrollLeft; last = startX; vel = 0;
    cancelAnimationFrame(rafId);
  });
  list.addEventListener("touchmove", e => {
    if (!isDown) return;
    const x = e.touches[0].pageX;
    list.scrollLeft = scrollLeft - (x - startX);
    vel = x - last; last = x;
  });
  list.addEventListener("touchend", () => { isDown = false; rafId = requestAnimationFrame(momentum); });
}

loadMessages(false);
enableSwipeScroll();
setInterval(() => loadMessages(true), 10000);
</script>
```

---

## 7. Run the Bot on Oracle

1. Start the bot:

```bash
cd ~/discord-bot
source venv/bin/activate
python bot.py
```

```

## ⚙️ Auto-Start on Boot (Ubuntu with systemd)

1. Create a service file:

   ```bash
   sudo nano /etc/systemd/system/discordbot.service
   ```

2. Paste:

   ```
   [Unit]
   Description=Discord → WordPress Bot
   After=network.target

   [Service]
   Type=simple
   User=ubuntu
   WorkingDirectory=/opt/discord-web
   ExecStart=/opt/discord-web/venv/bin/python /opt/discord-web/bot.py
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

3. Enable + start:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable discordbot
   sudo systemctl start discordbot
   ```

4. Check status:

   ```bash
   sudo systemctl status discordbot
   journalctl -u discordbot -f
   ```

---

