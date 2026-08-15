import os, textwrap, zipfile, json

base = "/mnt/data/whatsapp-md-bot"
os.makedirs(base, exist_ok=True)

files = {
"package.json": """{
  "name": "whatsapp-md-group-bot",
  "version": "1.0.0",
  "private": true,
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  },
  "engines": {
    "node": ">=20"
  },
  "dependencies": {
    "@whiskeysockets/baileys": "^6.7.20",
    "pino": "^9.4.0",
    "qrcode-terminal": "^0.12.0"
  }
}
""",
".gitignore": """node_modules/
auth/
.env
*.log
""",
"README.md": """# WhatsApp MD Group Bot

A starter multi-device WhatsApp group bot using Baileys.

## Deploy

1. Install Node.js 20+.
2. Run:
   npm install
   npm start
3. A QR code will appear in the terminal.
4. On WhatsApp: Linked devices -> Link a device -> scan the QR.
5. The login session is saved in `auth/`.

## Commands

- `.menu` — show the group menu
- `.ping` — check bot response
- `.tagall` — mention all group members
- `.groupinfo` — show basic group information
- `.admin` — show group admins

Use commands only in groups where you have permission to operate the bot.

## Notes

This is a clean starter project. Do not commit the `auth/` folder because it contains the WhatsApp login session.
""",
"src/index.js": r"""const {
  default: makeWASocket,
  useMultiFileAuthState,
  DisconnectReason,
  fetchLatestBaileysVersion
} = require("@whiskeysockets/baileys");
const P = require("pino");
const qrcode = require("qrcode-terminal");

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState("./auth");
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    version,
    auth: state,
    logger: P({ level: "silent" }),
    printQRInTerminal: false,
    browser: ["MD Group Bot", "Chrome", "1.0.0"]
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", ({ connection, lastDisconnect, qr }) => {
    if (qr) {
      console.log("\nScan this QR with WhatsApp Linked Devices:\n");
      qrcode.generate(qr, { small: true });
    }

    if (connection === "open") {
      console.log("WhatsApp bot connected.");
    }

    if (connection === "close") {
      const code = lastDisconnect?.error?.output?.statusCode;
      const shouldReconnect = code !== DisconnectReason.loggedOut;
      console.log("Connection closed. Reconnect:", shouldReconnect);
      if (shouldReconnect) startBot();
    }
  });

  sock.ev.on("messages.upsert", async ({ messages }) => {
    const msg = messages[0];
    if (!msg || msg.key.fromMe || !msg.message) return;

    const jid = msg.key.remoteJid;
    const text =
      msg.message.conversation ||
      msg.message.extendedTextMessage?.text ||
      "";

    if (!text.startsWith(".")) return;

    const [command] = text.trim().toLowerCase().split(/\s+/);

    if (command === ".ping") {
      await sock.sendMessage(jid, { text: "🏓 Pong!" });
      return;
    }

    if (command === ".menu") {
      const menu = `╭━━━〔 🤖 MD GROUP BOT 〕━━━╮
┃
┃ • .ping
┃ • .groupinfo
┃ • .admin
┃ • .tagall
┃
╰━━━━━━━━━━━━━━━━━━━━━━╯`;
      await sock.sendMessage(jid, { text: menu });
      return;
    }

    if (command === ".groupinfo") {
      if (!jid.endsWith("@g.us")) {
        await sock.sendMessage(jid, { text: "এই কমান্ডটি শুধু group-এ ব্যবহার করুন।" });
        return;
      }

      const meta = await sock.groupMetadata(jid);
      await sock.sendMessage(jid, {
        text: `📋 Group Info

Name: ${meta.subject}
Members: ${meta.participants.length}`
      });
      return;
    }

    if (command === ".admin") {
      if (!jid.endsWith("@g.us")) {
        await sock.sendMessage(jid, { text: "এই কমান্ডটি শুধু group-এ ব্যবহার করুন।" });
        return;
      }

      const meta = await sock.groupMetadata(jid);
      const admins = meta.participants.filter(p => p.admin);
      const mentions = admins.map(p => p.id);

      await sock.sendMessage(jid, {
        text: `👑 Group Admins: ${admins.length}`,
        mentions
      });
      return;
    }

    if (command === ".tagall") {
      if (!jid.endsWith("@g.us")) {
        await sock.sendMessage(jid, { text: "এই কমান্ডটি শুধু group-এ ব্যবহার করুন।" });
        return;
      }

      const meta = await sock.groupMetadata(jid);
      const mentions = meta.participants.map(p => p.id);
      const lines = mentions.map(id => `@${id.split("@")[0]}`);

      await sock.sendMessage(jid, {
        text: `📢 Tag All\n\n${lines.join(" ")}`,
        mentions
      });
    }
  });
}

startBot().catch(err => {
  console.error("Bot error:", err);
  process.exit(1);
});
""",
"render.yaml": """services:
  - type: worker
    name: whatsapp-md-group-bot
    runtime: node
    buildCommand: npm install
    startCommand: npm start
"""
}

for rel, content in files.items():
    path = os.path.join(base, rel)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)

zip_path = "/mnt/data/whatsapp-md-group-bot.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for root, _, filenames in os.walk(base):
        for fn in filenames:
            path = os.path.join(root, fn)
            z.write(path, os.path.relpath(path, base))

print(f"Created: {zip_path}")
