<div align="center">

# Jubbio Developer Documentation for LLMs & AI Agents

[![Jubbio API](https://img.shields.io/badge/Jubbio_API-v1-blue.svg)](https://jubbio.com/dev/docs)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)](https://nodejs.org/)
[![llms.txt](https://img.shields.io/badge/llms.txt-compliant-orange.svg)](https://llmstxt.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**An exhaustive, production-grade technical reference manual for the Jubbio developer ecosystem, optimized for LLMs, AI agents, and bot developers.**

[Full Technical Reference](JUBBIO_DEV_REFERENCE.md) • [llms.txt Standard](llms.txt) • [Full Ingestion Text](llms-full.txt) • [Official Docs](https://jubbio.com/dev/docs)

</div>

---

## 📌 About This Repository

**Jubbio** is a modern real-time chat, community, and voice platform. Its developer ecosystem features a rich WebSocket Gateway protocol, REST APIs, and official TypeScript/JavaScript libraries (`@jubbio/core`, `@jubbio/voice`, `@jubbio/auth`).

Because Jubbio is a newer platform, standard foundation LLMs often lack up-to-date training data for its endpoints, payload schemas, and library methods, leading to hallucinations or false assumptions based on Discord or Slack APIs.

**This repository solves that problem** by providing a 100% verified, comprehensive, machine-readable reference file extracted directly from the live Jubbio Developer Portal.

---

## 🤖 How to Use with AI Agents & Coding Assistants

### 1. Cursor IDE / Windsurf
Add this repository or the raw `JUBBIO_DEV_REFERENCE.md` file to your `@Docs` or Project Rules:
- Point your Cursor Docs indexer to: `https://raw.githubusercontent.com/saintberat/jubbio-docs-for-llms/main/llms.txt`
- Or add `JUBBIO_DEV_REFERENCE.md` into your `.cursorrules` / project workspace context.

### 2. Claude Projects / ChatGPT Custom GPTs
Upload `JUBBIO_DEV_REFERENCE.md` (or `llms-full.txt`) into the **Knowledge Base** of your Claude Project or Custom GPT.

### 3. API Prompt Context
When asking an LLM to generate Jubbio bots, pass the content of `JUBBIO_DEV_REFERENCE.md` as context in your system prompt.

---

## ⚡ Quick Platform Summary

| Component | Endpoint / Value |
| :--- | :--- |
| **REST API Base URL** | `https://gateway.jubbio.com/api/v1` |
| **WebSocket Gateway** | `wss://gateway.jubbio.com/` |
| **OAuth2 Authorize URL** | `https://jubbio.com/oauth2/authorize` |
| **OAuth2 Token URL** | `https://gateway.jubbio.com/api/v1/oauth2/token` |
| **Bot Auth Header** | `Authorization: Bot <BOT_TOKEN>` |
| **User OAuth2 Header** | `Authorization: Bearer <USER_ACCESS_TOKEN>` |
| **Primary Package** | `npm install @jubbio/core` |
| **Voice Package** | `npm install @jubbio/voice` |
| **OAuth2 React SDK** | `npm install @jubbio/auth @jubbio/auth-react` |

---

## 🚀 Quickstart Example

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

  // Register Global Slash Commands
  await client.rest.registerGlobalCommands([
    new SlashCommandBuilder()
      .setName('ping')
      .setDescription('Replies with pong!')
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

## 📖 What's Covered in the Full Reference

The main reference document [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) contains:

1. **Authentication & Tokens:** Bot token vs Bearer token matrix, security rules, and privileged Gateway intents (`GUILD_MEMBERS`, `GUILD_PRESENCES`, `MESSAGE_CONTENT`).
2. **Messages API:** Normal messages, ephemeral messages (`flags: 64`), embeds, reactions, pins, bulk delete, attachments (multipart), and DMs.
3. **Channels & Moderation:** Channel creation/deletion, categories, permission overrides, timeouts, kicks, bans, and bulk role management.
4. **Guilds, Roles & Invites:** Server settings, custom emojis, role creation/editing with bitfields, and invite management.
5. **Application Commands:** Global vs Guild commands, all 11 option types, error codes (`50035`, `30032`, etc.), and mention tokens (`<@ID>`, `<@&ID>`, `<#ID>`).
6. **Interaction Callbacks:** 6 callback response types, deferring replies (`DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE`), and follow-up messages.
7. **Webhooks:** Bot webhook CRUD, URL execution, link buttons, Slack format, and GitHub event mappings.
8. **OAuth2 & React SDK:** PKCE flow, all scopes, `@jubbio/auth-react` provider and hooks.
9. **`@jubbio/voice`:** UDP voice connection, AudioPlayer, streaming resources, and event lifecycle.
10. **Permissions & Sharding:** `PermissionsBitField` BigInt bitfields, `ShardingManager`, collectors (`awaitMessages`), and formatting helpers.

---

## 📄 License

This repository is licensed under the [MIT License](LICENSE).
Official documentation content belongs to [Jubbio](https://jubbio.com).
