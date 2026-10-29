<div align="center">

# Jubbio Developer Documentation for LLMs & AI Agents

[![Jubbio API](https://img.shields.io/badge/Jubbio_API-v1-blue.svg)](https://jubbio.com/dev/docs)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)](https://nodejs.org/)
[![llms.txt](https://img.shields.io/badge/llms.txt-compliant-orange.svg)](https://llmstxt.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**A comprehensive technical reference repository optimized for Large Language Models (LLMs), AI coding assistants, and bot developers building on the Jubbio ecosystem.**

[Technical Reference Guide (JUBBIO_DEV_REFERENCE.md)](JUBBIO_DEV_REFERENCE.md) • [llms.txt Standard](llms.txt) • [Full Plaintext (llms-full.txt)](llms-full.txt) • [Official Documentation](https://jubbio.com/dev/docs)

</div>

---

## 📌 About This Repository

**Jubbio** is a modern real-time messaging, community, and voice platform. Its developer ecosystem provides a rich WebSocket Gateway protocol, REST APIs, and official TypeScript/JavaScript client SDKs (`@jubbio/core`, `@jubbio/voice`, `@jubbio/auth`).

Because Jubbio is a new platform, general pre-trained LLMs do not possess native knowledge of Jubbio's endpoints, payload schemas, and method signatures in their base training cutoffs. This frequently causes models to hallucinate or misapply Discord or Slack conventions.

**This repository bridges that gap:** It provides an authoritative, machine-readable, verified technical reference extracted directly from the live Jubbio Developer Portal.

> [!NOTE]
> All core technical reference files ([`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md), [`llms.txt`](llms.txt), and [`llms-full.txt`](llms-full.txt)) follow strict technical English formatting to guarantee zero syntax and parameter interpretation errors across all AI engines.

---

## 🤖 Usage with AI Coding Agents & Tools

### 1. Cursor IDE / Windsurf
Add this repository or `llms.txt` directly to your `@Docs` or project rules:
- Add the remote documentation index: `https://raw.githubusercontent.com/saintberat/jubbio-docs-for-llms/main/llms.txt`
- Or add [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) to your workspace or `.cursorrules` as context.

### 2. Claude Projects / ChatGPT Custom GPTs
Upload [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) (or `llms-full.txt`) into your Claude Project or Custom GPT **Knowledge Base**.

### 3. API & System Prompts
When prompting an AI to generate Jubbio bots or integrations, append the contents of [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) into your system prompt context.

---

## ⚡ Quick Platform Summary

| Component | Value / Endpoint |
| :--- | :--- |
| **REST API Base URL** | `https://gateway.jubbio.com/api/v1` |
| **WebSocket Gateway** | `wss://gateway.jubbio.com/` |
| **OAuth2 Authorize URL** | `https://jubbio.com/oauth2/authorize` |
| **OAuth2 Token URL** | `https://gateway.jubbio.com/api/v1/oauth2/token` |
| **Bot Authorization Header** | `Authorization: Bot <BOT_TOKEN>` |
| **User OAuth2 Header** | `Authorization: Bearer <USER_ACCESS_TOKEN>` |
| **Core Bot SDK** | `npm install @jubbio/core` |
| **Voice SDK** | `npm install @jubbio/voice` |
| **OAuth2 React SDK** | `npm install @jubbio/auth @jubbio/auth-react` |

---

## 🚀 Quick Start Example

```javascript
import 'dotenv/config';
import { Client, GatewayIntentBits, SlashCommandBuilder } from '@jubbio/core';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent, // Privileged intent
  ],
});

client.on('ready', async () => {
  console.log(`✅ Logged in as ${client.user.username}!`);

  // Register Global Slash Command
  await client.rest.registerGlobalCommands([
    new SlashCommandBuilder()
      .setName('ping')
      .setDescription('Replies with Pong!')
      .toJSON(),
  ]);
});

client.on('interactionCreate', async (interaction) => {
  if (!interaction.isChatInputCommand()) return;

  if (interaction.commandName === 'ping') {
    await interaction.reply({ content: '🏓 Pong!', ephemeral: true });
  }
});

client.login(process.env.BOT_TOKEN);
```

---

## 📖 Master Reference Guide Contents

[`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) comprehensively covers 10 primary modules:

1. **Authentication & Token Security:** Bot Token vs. Bearer Token paradigms, security best practices, and privileged Gateway intents (`GUILD_MEMBERS`, `GUILD_PRESENCES`, `MESSAGE_CONTENT`).
2. **Messages API:** Standard messages, ephemeral messages (`flags: 64`), embeds, reactions, pinning, bulk deletion, multipart file uploads, and Direct Messages (DMs).
3. **Channels & Moderation:** Text/Voice/Category channel lifecycle, permission overrides, timeouts, kicks, bans, and bulk role mutations.
4. **Guilds, Roles & Invites:** Server configurations, custom emojis, bitfield permission creation, and invite link management.
5. **Application Commands (Slash Commands):** Global and guild commands, 11 argument option types, REST error codes (`50035`, `30032`), and mention tokens (`<@ID>`, `<@&ID>`, `<#ID>`).
6. **Interaction Callbacks:** 6 callback response types, deferred responses (`DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE`), and followup messages.
7. **Webhooks:** Webhook CRUD operations, execution, link buttons, Slack format compatibility, and GitHub webhook event mapping.
8. **OAuth2 & React SDK:** PKCE authentication flow, scope tables, `@jubbio/auth-react` context provider, and the `useJubbio()` hook.
9. **Voice Architecture (`@jubbio/voice`):** UDP voice socket connection, `AudioPlayer`, voice stream sources, and lifecycle management.
10. **Permissions, Collectors & Sharding:** `PermissionsBitField` 64-bit flags, `ShardingManager`, message collectors (`awaitMessages`), and formatting utilities.

---

## 📄 License

Distributed under the [MIT License](LICENSE).
Official Jubbio documentation contents and assets belong to [Jubbio](https://jubbio.com).

<!-- sync-activity: 2026-10-29T17:04:07Z (6c12bf9e) -->
