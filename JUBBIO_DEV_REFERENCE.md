
# JUBBIO DEVELOPER PLATFORM & SDK REFERENCE MANUAL

> **Notice for AI Agents & LLMs:** This is an exhaustive, production-grade technical knowledge base and API specification for the **Jubbio** developer platform. It documents the **Bot REST API**, **Gateway WebSocket Protocol**, **jubbio.js SDK** (`@jubbio/core` and `@jubbio/voice`), and **OAuth2 PKCE Architecture**.
>
> Use this reference to generate accurate, idiomatically correct bot code, API calls, event handlers, UI component builders, and authentication flows without hallucinating Discord or Slack discrepancies.


## 1. Platform Architecture & Core Conventions

- **REST API Base URL:** `https://gateway.jubbio.com/api/v1`
- **WebSocket Gateway URL:** `wss://gateway.jubbio.com/`
- **OAuth2 Authorization URL:** `https://jubbio.com/oauth2/authorize`
- **OAuth2 Token Endpoint:** `https://gateway.jubbio.com/api/v1/oauth2/token`
- **Developer Portal:** `https://jubbio.com/dev`
- **Serialization:** JSON with strict `omitempty` conventions (fields that evaluate to `null`, `false`, `0`, or empty arrays `[]` are omitted from API responses).
- **Identifier Format:** 64-bit integer Snowflake IDs passed as strings (e.g., `"554200123456789012"`).
- **Rate Limits:** Return HTTP `429 Too Many Requests` with JSON body `{"error": "Rate limit aşıldı", "retry_after": 10}`. Default route limits: 30-50 requests/sec; Webhook execution: 30 requests / 60 seconds.


### Official NPM Packages

| Package | Version / Purpose | Key Exports |
| :--- | :--- | :--- |
| **`@jubbio/core`** | Primary Bot SDK & Gateway Client | `Client`, `GatewayIntentBits`, `EmbedBuilder`, `ButtonBuilder`, `ActionRowBuilder`, `StringSelectMenuBuilder`, `ModalBuilder`, `SlashCommandBuilder`, `PermissionsBitField`, `PermissionFlagsBits` |
| **`@jubbio/voice`** | Real-time UDP Audio & Voice Subsystem | `joinVoiceChannel`, `createAudioPlayer`, `createAudioResource`, `AudioPlayerStatus`, `VoiceConnectionStatus` |
| **`@jubbio/auth`** | OAuth2 PKCE SDK for Web & Node.js | `JubbioAuth`, `generatePKCE`, `exchangeCode` |
| **`@jubbio/auth-react`** | Official React Components & Hooks | `JubbioAuthProvider`, `JubbioLoginButton`, `useJubbio` |

```bash
npm install @jubbio/core
npm install @jubbio/voice    # Optional: For voice channels & audio streaming
npm install @jubbio/auth @jubbio/auth-react # Optional: For web app OAuth2 login
```


## 2. Authentication, Tokens & Security

All HTTP requests to the Jubbio REST API require an `Authorization` header:

```http
// 1. For Bot applications:
Authorization: Bot YOUR_BOT_TOKEN

// 2. For User impersonation / OAuth2 authorized applications:
Authorization: Bearer USER_ACCESS_TOKEN
```

### Token Comparison Matrix:
| Feature | Bot Token (`Bot`) | Bearer Token (`Bearer`) |
| :--- | :--- | :--- |
| **Header Prefix** | `Bot` | `Bearer` |
| **Target Architecture** | Automated bot backend services & daemons | External apps acting on behalf of users |
| **Origin** | Developer Portal → Applications → [App] → Bot Tab | OAuth2 Authorization Code Exchange |
| **Validity** | Permanent until manually regenerated | Access: 7 days, Refresh: 30 days |
| **Gateway Connection** | Full Gateway WebSocket access (`wss://`) | REST API only (No Gateway socket connection) |
| **Scope / Privileges** | Guild Role & Channel Permission Bitfields | Strictly constrained to authorized OAuth2 scopes |


### Privileged Gateway Intents

Bots requiring sensitive real-time data must have Privileged Intents explicitly enabled in the Developer Portal under the **Bot** tab:
1. **Guild Members (`GUILD_MEMBERS` / `1 << 1`):** Required to track member join/leave/update events and query full guild member lists.
2. **Presence Info (`GUILD_PRESENCES` / `1 << 8`):** Required for online/offline/dnd status updates.
3. **Message Content (`MESSAGE_CONTENT` / `1 << 15`):** Required to read the raw text `content` of messages sent by other users.


## 3. Quickstart Implementation Templates


### 3.1 Message-Based Bot (Node.js ES Modules)

```javascript
import 'dotenv/config';
import { Client, GatewayIntentBits } from '@jubbio/core';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent, // Requires privileged intent in Dev Portal
  ],
});

client.on('ready', () => {
  console.log(`✅ Connected as ${client.user.username} (ID: ${client.user.id})!`);
});

client.on('messageCreate', async (message) => {
  if (message.author.bot) return;

  if (message.content === '!ping') {
    await message.reply('🏓 Pong!');
  }

  if (message.content.startsWith('!echo ')) {
    const text = message.content.slice(6);
    await message.reply(text);
  }
});

client.login(process.env.BOT_TOKEN);
```


### 3.2 Modern Slash Command Bot

```javascript
import 'dotenv/config';
import { Client, GatewayIntentBits, SlashCommandBuilder } from '@jubbio/core';

const client = new Client({
  intents: [GatewayIntentBits.Guilds],
});

client.on('ready', async () => {
  console.log(`✅ ${client.user.username} is online!`);

  // Register Global Slash Commands (Propagates in ~1 hour across all guilds)
  await client.rest.registerGlobalCommands([
    new SlashCommandBuilder()
      .setName('ping')
      .setDescription('Replies with pong!')
      .toJSON(),
    new SlashCommandBuilder()
      .setName('echo')
      .setDescription('Repeats your input')
      .addStringOption(option =>
        option.setName('text').setDescription('Message to repeat').setRequired(true)
      )
      .toJSON(),
  ]);

  console.log('📝 Slash commands registered!');
});

client.on('interactionCreate', async (interaction) => {
  if (!interaction.isChatInputCommand()) return;

  if (interaction.commandName === 'ping') {
    await interaction.reply('🏓 Pong!');
  }

  if (interaction.commandName === 'echo') {
    const text = interaction.options.getString('text');
    await interaction.reply(`🗣️ ${text}`);
  }
});

client.login(process.env.BOT_TOKEN);
```


## 4. Complete Bot REST API Reference

All bot endpoints (except Application Commands) are prefixed with `/api/v1/bot/`.
Required Request Header: `Authorization: Bot <token>`
Content-Type: `application/json` (unless multipart file upload).


### 4.1 Messages API (`/bot/guilds/{guildId}/channels/{channelId}/messages`)

#### POST `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages`
Sends a message to the specified text channel. Automatically parses mentions (`<@userId>`, `<@&roleId>`, `<#channelId>`).

**JSON Request Body:**
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `content` | string | Optional* | Message text (max 2000 chars). At least `content` or `embeds` or file is required. |
| `embeds` | APIEmbed[] | Optional* | Array of up to 10 rich embed objects. |
| `components` | ActionRow[] | Optional | ActionRow components containing buttons or select menus. |
| `flags` | number | Optional | Set to `64` (`MessageFlags.Ephemeral`) for ephemeral message. |
| `target_user_id` | string | Optional | Required if `flags: 64` to indicate recipient. |
| `mentions` | MentionsData | Optional | Explicit mention configuration object. |
| `message_reference` | object | Optional | `{"message_id": "snowflake"}` for inline reply. |

**Success Response (201 Created):**

```json
{
  "id": "554200123456789012",
  "channel_id": "545980009056055296",
  "guild_id": "540528238699487232",
  "content": "Merhaba!",
  "type": "bot",
  "application_id": "551234567890123456",
  "created_at": "2026-03-10T12:00:00Z",
  "author": {
    "id": "2",
    "username": "MyBot",
    "display_name": "MyBot",
    "avatar_url": "https://cdn.example.com/avatar.png"
  },
  "embeds": [
    {
      "title": "Başlık",
      "description": "Açıklama",
      "color": 5793266
    }
  ]
}
```

#### Ephemeral Message (User-Only Temporary Message)
- **POST** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages`
- **Body:** `{"content": "Sadece sana görünür", "flags": 64, "target_user_id": "123456789"}`
- Not saved to database, transmitted over WebSocket only. Response: `201 Created`.

#### PATCH `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/{messageId}`
Edits a message previously sent by the bot. Response: `200 OK`.

#### DELETE `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/{messageId}`
Deletes a message. Response: `204 No Content`.

#### POST `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/bulk-delete`
Deletes 2-100 messages not older than 14 days. Requires `MANAGE_MESSAGES`.
- **Body:** `{"message_ids": ["554200123456789012", "554200123456789013"]}`
- **Response:** `200 OK` `{"deleted_count": 2}`

#### GET `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages`
Retrieves message history. Query params: `limit` (1-100, default 50), `before`, `after`, `around`. Response: `200 OK` (`Message[]`).

#### POST `/api/v1/bot/guilds/{guildId}/channels/{channelId}/attachments`
Uploads a file attachment. Uses `multipart/form-data` with fields `payload_json` and `files[0]`.


### 4.2 Reactions & Pinned Messages

- **Add Reaction:** **PUT** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/{messageId}/reactions/{emoji}/@me` → `204 No Content`
- **Remove Own Reaction:** **DELETE** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/{messageId}/reactions/{emoji}/@me` → `204 No Content`
- **Remove User Reaction:** **DELETE** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/messages/{messageId}/reactions/{emoji}/{userId}` → `204 No Content`
- **Pin Message:** **PUT** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/pins/{messageId}` (Limit: 50 pins/channel) → `204 No Content`
- **Unpin Message:** **DELETE** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/pins/{messageId}` → `204 No Content`
- **List Pins:** **GET** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/pins` → `200 OK` (`Message[]`)


### 4.3 Direct Messages (DM) API

- **Create DM Channel & Conversation:** **POST** `/api/v1/bot/dm` (Body: `{"recipient_id": "snowflake"}`) → `200 OK` `{"channel_id": "...", "recipient": {...}}`
- **Send Message to DM Channel:** **POST** `/api/v1/bot/dm/{channelId}/messages` (Body: `{"content": "..."}`) → `201 Created`
- **Get DM Channel Info:** **GET** `/api/v1/bot/dm/{channelId}` → `200 OK`
- **Get DM Message History:** **GET** `/api/v1/bot/dm/{channelId}/messages?limit=50` → `200 OK`


### 4.4 Channels & Categories API

#### POST `/api/v1/bot/guilds/{guildId}/channels`
Creates a channel or category. Requires `MANAGE_CHANNELS`.

**JSON Body:**
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | string | **Required** | Channel name (1-100 characters). |
| `type` | number | **Required** | `0` = GUILD_TEXT, `2` = GUILD_VOICE, `4` = GUILD_CATEGORY, `5` = GUILD_ANNOUNCEMENT, `13` = GUILD_STAGE_VOICE. |
| `topic` | string | Optional | Channel topic (0-1024 characters). |
| `parent_id` | string | Optional | Category snowflake ID. |
| `position` | number | Optional | Sorting position index. |
| `nsfw` | boolean | Optional | Age-restricted channel flag. |
| `rate_limit_per_user` | number | Optional | Slowmode duration in seconds (0-21600). |
| `permission_overwrites` | Overwrite[] | Optional | Array of initial permission overrides. |

- **DELETE Channel:** **DELETE** `/api/v1/bot/guilds/{guildId}/channels/{channelId}` → `200 OK`
- **DELETE Category:** **DELETE** `/api/v1/bot/guilds/{guildId}/categories/{categoryId}` → `204 No Content` (Channels under category are unparented, not deleted)
- **Set Channel Overwrites:** **PUT** `/api/v1/bot/channels/{channelId}/permissions/{overwriteId}` (Body: `{"type": 0, "allow": "1048576", "deny": "2097152"}`) → `200 OK`
- **Delete Overwrite:** **DELETE** `/api/v1/bot/channels/{channelId}/permissions/{overwriteId}` → `204 No Content`


### 4.5 Members & Moderation API

- **Get Member:** **GET** `/api/v1/bot/guilds/{guildId}/members/{userId}` → `200 OK`
- **List Members:** **GET** `/api/v1/bot/guilds/{guildId}/members?limit=100&after={userId}` → `200 OK` (`GuildMember[]`)
- **Timeout Member:** **POST** `/api/v1/bot/guilds/{guildId}/members/{userId}/timeout` (Body: `{"duration_seconds": 3600, "reason": "Spam"}`, max 28 days: `2419200`) → `200 OK`
- **Clear Timeout:** **POST** `/api/v1/bot/guilds/{guildId}/members/{userId}/timeout/clear` → `200 OK`
- **Kick Member:** **DELETE** `/api/v1/bot/guilds/{guildId}/members/{userId}` → `204 No Content`
- **Ban Member:** **PUT** `/api/v1/bot/guilds/{guildId}/bans/{userId}` (Body: `{"delete_message_days": 1, "reason": "Rules violation"}`) → `200 OK`
- **Unban Member:** **DELETE** `/api/v1/bot/guilds/{guildId}/bans/{userId}` → `204 No Content`
- **Edit Member Profile:** **PATCH** `/api/v1/bot/guilds/{guildId}/members/{userId}` (Body: `{"nick": "New Nick", "roles": [...], "mute": false, "deaf": false, "channel_id": "..."}`) → `200 OK`
- **Add Role to Member:** **PUT** `/api/v1/bot/guilds/{guildId}/members/{userId}/roles/{roleId}` → `204 No Content`
- **Remove Role from Member:** **DELETE** `/api/v1/bot/guilds/{guildId}/members/{userId}/roles/{roleId}` → `204 No Content`
- **Bulk Assign Roles:** **POST** `/api/v1/bot/guilds/{guildId}/members/roles/assign` (Body: `{"user_ids": [...], "role_ids": [...]}`) → `200 OK`
- **Bulk Remove Roles:** **POST** `/api/v1/bot/guilds/{guildId}/members/roles/remove` (Body: `{"user_ids": [...], "role_ids": [...]}`) → `200 OK`


### 4.6 Guilds, Roles, Invites & Emojis

- **Get Guild:** **GET** `/api/v1/bot/guilds/{guildId}?with_counts=true` → `200 OK`
- **List Guild Channels:** **GET** `/api/v1/bot/guilds/{guildId}/channels` → `200 OK`
- **List Guild Emojis:** **GET** `/api/v1/bot/guilds/{guildId}/emojis` → `200 OK`
- **List Bans:** **GET** `/api/v1/bot/guilds/{guildId}/bans` → `200 OK`
- **Get Ban:** **GET** `/api/v1/bot/guilds/{guildId}/bans/{userId}` → `200 OK`
- **List Roles:** **GET** `/api/v1/bot/guilds/{guildId}/roles` → `200 OK`
- **Create Role:** **POST** `/api/v1/bot/guilds/{guildId}/roles` (Body: `{"name": "Mod", "color": 3447003, "hoist": true, "permissions": "8", "mentionable": true}`) → `201 Created`
- **Edit Role:** **PATCH** `/api/v1/bot/guilds/{guildId}/roles/{roleId}` → `200 OK`
- **Delete Role:** **DELETE** `/api/v1/bot/guilds/{guildId}/roles/{roleId}` → `204 No Content`
- **Create Channel Invite:** **POST** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/invites` (Body: `{"max_age": 86400, "max_uses": 10, "temporary": false, "unique": true}`) → `201 Created`
- **Get Invite Info:** **GET** `/api/v1/bot/invites/{inviteCode}?with_counts=true` → `200 OK`
- **List Guild Invites:** **GET** `/api/v1/bot/guilds/{guildId}/invites` → `200 OK`
- **Delete Invite:** **DELETE** `/api/v1/bot/invites/{inviteCode}` → `204 No Content`
- **Get Public User Profile:** **GET** `/api/v1/bot/users/{userId}` → `200 OK`
- **Get Bot Self Profile:** **GET** `/api/v1/bot/users/@me` → `200 OK`


### 4.7 Application Commands (Slash Commands) & Mention Syntax

> [!IMPORTANT]
> Route Prefix Note: Application Command endpoints are under `/applications/{appId}/...` (NOT `/bot/...`), and accept both `Authorization: Bot <token>` and `Authorization: Bearer <jwt>`.
> Discord API compatible error format: `{"error": "...", "code": 50035, "field": "name"}`.

#### Global Commands (Propagate in ~1 hour across all guilds, limit 100):
- **POST** `/api/v1/applications/{appId}/commands` (Create)
- **GET** `/api/v1/applications/{appId}/commands` (List)
- **GET** `/api/v1/applications/{appId}/commands/{commandId}` (Get)
- **PATCH** `/api/v1/applications/{appId}/commands/{commandId}` (Update)
- **DELETE** `/api/v1/applications/{appId}/commands/{commandId}` (Delete)

#### Guild Commands (Instant update in specified guild):
- **POST** `/api/v1/applications/{appId}/guilds/{guildId}/commands` (Create)
- **GET** `/api/v1/applications/{appId}/guilds/{guildId}/commands` (List)
- **GET** `/api/v1/applications/{appId}/guilds/{guildId}/commands/{commandId}` (Get)
- **PATCH** `/api/v1/applications/{appId}/guilds/{guildId}/commands/{commandId}` (Update)
- **DELETE** `/api/v1/applications/{appId}/guilds/{guildId}/commands/{commandId}` (Delete)

#### Command Option Types (`type`):
| Type ID | Enum Name | Description |
| :--- | :--- | :--- |
| **1** | `SUB_COMMAND` | Subcommand option |
| **2** | `SUB_COMMAND_GROUP` | Subcommand group option |
| **3** | `STRING` | Text string parameter |
| **4** | `INTEGER` | 32-bit integer |
| **5** | `BOOLEAN` | True / false boolean |
| **6** | `USER` | User mention / Snowflake |
| **7** | `CHANNEL` | Channel selector |
| **8** | `ROLE` | Role selector |
| **9** | `MENTIONABLE` | User or role selector |
| **10** | `NUMBER` | Floating point number |
| **11** | `ATTACHMENT` | Uploaded file parameter |

#### Common Command Error Codes:
| Error Code | Meaning | Typical Trigger |
| :--- | :--- | :--- |
| `50035` | Invalid Form Body | Command name has uppercase letters or spaces; description > 100 chars; > 25 options |
| `30032` | Limit Exceeded | Application reached 100 global commands limit |
| `50001` | Missing Access | Attempted to manage commands belonging to a different Application ID |
| `10003` | Unknown Command | Command ID not found |

#### Mention Token Syntax:
| Format | Meaning | Example |
| :--- | :--- | :--- |
| `<@USER_ID>` | Mention User | `<@554200123456789012>` |
| `<@&ROLE_ID>` | Mention Role | `<@&540528238699487233>` |
| `<#CHANNEL_ID>` | Mention Channel | `<#545980009056055296>` |
| `@everyone` | Ping all guild members | `@everyone` |
| `@here` | Ping online guild members | `@here` |


### 4.8 Interactions Callback API

When an interaction occurs, the bot must acknowledge within 3 seconds:

#### POST `/api/v1/interactions/{interactionId}/{token}/callback`
**Interaction Callback Types (`InteractionResponseType`):**
| Type ID | Enum Constant | Description |
| :--- | :--- | :--- |
| **4** | `CHANNEL_MESSAGE_WITH_SOURCE` | Immediately send message reply |
| **5** | `DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE` | "Bot is thinking..." ACK (grants 15 mins to edit original response) |
| **6** | `DEFERRED_UPDATE_MESSAGE` | Acknowledge button/menu click without immediate visual update |
| **7** | `UPDATE_MESSAGE` | Immediately edit the message housing the clicked button/select menu |
| **9** | `APPLICATION_COMMAND_AUTOCOMPLETE_RESULT` | Return autocomplete search choices |
| **10** | `MODAL` | Pop up a modal dialog form |

- **Edit Original Response:** **PATCH** `/api/v1/interactions/webhooks/{appId}/{token}/messages/@original`
- **Delete Original Response:** **DELETE** `/api/v1/interactions/webhooks/{appId}/{token}/messages/@original`
- **Send Follow-up Message:** **POST** `/api/v1/interactions/webhooks/{appId}/{token}`


### 4.9 Webhooks API (Management & Execution)

- **Channel Webhooks:** **GET / POST** `/api/v1/bot/guilds/{guildId}/channels/{channelId}/webhooks` (Limit: 10 webhooks/channel)
- **Guild Webhooks:** **GET** `/api/v1/bot/guilds/{guildId}/webhooks`
- **Modify / Delete Webhook:** **PATCH / DELETE** `/api/v1/bot/webhooks/{webhookId}`
- **Execute Webhook:** **POST** `https://gateway.jubbio.com/api/v1/webhooks/{webhookId}/{webhookToken}?wait=true`
- **Execute with Link Buttons:** Webhooks cannot receive interactive callbacks, but can attach clickable URL buttons (`style: 5`).
- **Slack-Compatible Execution:** **POST** `https://gateway.jubbio.com/api/v1/webhooks/{webhookId}/{webhookToken}/slack`
- **GitHub-Compatible Execution:** **POST** `https://gateway.jubbio.com/api/v1/webhooks/{webhookId}/{webhookToken}/github` (Validates `X-Hub-Signature-256` HMAC SHA-256)

#### Supported GitHub Events:
| GitHub Event | Action | Jubbio Embed Appearance |
| :--- | :--- | :--- |
| `push` | Code pushed | Commit list (max 10), author, branch, green embed |
| `pull_request` | Opened / Merged / Closed | PR title, description, state color (Green/Purple/Red) |
| `issues` | Opened / Closed | Issue title, body snippet, state color |
| `issue_comment` | Comment added | Comment text snippet and direct link |
| `pull_request_review` | Review submitted | Review state badge and author |
| `release` | Published | Release title, tag, prerelease badge, gold embed |
| `fork` / `watch` | Forked / Starred | Milestone notice with user profile |
| `ping` | Webhook setup | Initial verification handshake embed |


## 5. OAuth2 Specification & SDK (@jubbio/auth)


### 5.1 OAuth2 Scopes

| Scope | Description |
| :--- | :--- |
| `identify` | User ID, username, discriminator, avatar, display name |
| `email` | Verified user email address |
| `guilds` | List of user's guilds (ID, name, icon, permission bitfield) |
| `guilds.members.read` | User's member profile and assigned roles in specific guilds |
| `connections` | Connected third-party accounts (GitHub, YouTube, etc.) |
| `relationships` | User friend list |
| `bot` | Adds bot to a guild (requires guild authorization) |
| `applications.commands` | Enables slash commands in the authorized guild |


### 5.2 Authorization Code Flow with PKCE

```text
// 1. Redirect to Jubbio Authorization
https://gateway.jubbio.com/api/v1/oauth2/authorize?client_id=CLIENT_ID&redirect_uri=https://app.com/callback&response_type=code&scope=identify%20email&state=RANDOM_STATE&code_challenge=HASH&code_challenge_method=S256

// 2. Token Exchange (POST https://gateway.jubbio.com/api/v1/oauth2/token)
// Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&code=CODE&redirect_uri=https://app.com/callback&client_id=CLIENT_ID&client_secret=CLIENT_SECRET&code_verifier=VERIFIER

// Response (200 OK):
{
  "access_token": "jbo_at_...",
  "token_type": "Bearer",
  "expires_in": 604800, // 7 days
  "refresh_token": "jbo_rt_...",
  "scope": "identify email"
}

// 3. Current User Endpoint (GET https://gateway.jubbio.com/api/v1/users/@me)
// Header: Authorization: Bearer jbo_at_...
```


### 5.3 React SDK Integration (@jubbio/auth-react)

```tsx
import React from 'react';
import { JubbioAuthProvider, JubbioLoginButton, useJubbio } from '@jubbio/auth-react';

const authConfig = {
  clientId: 'your_client_id',
  redirectUri: 'http://localhost:3000/callback',
  scopes: ['identify', 'email', 'guilds'],
  popup: true,
  autoStore: true,
};

export function App() {
  return (
    <JubbioAuthProvider config={authConfig}>
      <UserProfile />
    </JubbioAuthProvider>
  );
}

function UserProfile() {
  const { user, isLoggedIn, isLoading, logout } = useJubbio();

  if (isLoading) return <div>Yükleniyor...</div>;

  if (isLoggedIn && user) {
    return (
      <div>
        <img src={user.avatar} alt={user.username} />
        <h2>Hoş geldin, {user.display_name || user.username}!</h2>
        <p>Email: {user.email}</p>
        <button onClick={logout}>Çıkış Yap</button>
      </div>
    );
  }

  return (
    <JubbioLoginButton
      theme="dark" // "dark" | "light" | "purple"
      size="lg"    // "sm" | "md" | "lg"
      label="Jubbio ile Giriş Yap"
      showLogo={true}
    />
  );
}
```


## 6. jubbio.js SDK Deep Dive (@jubbio/core)


### 6.1 Client Class

```javascript
import { Client, GatewayIntentBits } from '@jubbio/core';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildVoiceStates,
  ],
  shards: 'auto',
  gatewayUrl: 'wss://gateway.jubbio.com',
  apiUrl: 'https://gateway.jubbio.com/api/v1',
});

// Properties:
// client.user       -> ClientUser (Bot user)
// client.guilds     -> GuildManager cache
// client.channels   -> ChannelManager cache
// client.users      -> UserManager cache
// client.rest       -> REST API client instance
// client.voice      -> Voice connection manager
// client.ws         -> Gateway WebSocket manager & ping latency

await client.login(process.env.BOT_TOKEN);
```


### 6.2 Gateway Events Reference

| Event Name | Handler Signature | Trigger Condition |
| :--- | :--- | :--- |
| `ready` | `(client: Client) => void` | Bot connection established & cache populated |
| `messageCreate` | `(message: Message) => Promise<void>` | Message sent in guild channel or DM |
| `messageUpdate` | `(oldMsg: Message, newMsg: Message) => void` | Message content or embeds edited |
| `messageDelete` | `(message: Message) => void` | Single message deleted |
| `messageDeleteBulk` | `(messages: Collection<string, Message>) => void` | Bulk message deletion executed |
| `interactionCreate` | `(interaction: BaseInteraction) => Promise<void>` | Slash command, button, select menu, or modal submitted |
| `guildCreate` | `(guild: Guild) => void` | Bot joins new guild or guild becomes available |
| `guildDelete` | `(guild: Guild) => void` | Bot kicked/banned from guild or guild deleted |
| `guildMemberAdd` | `(member: GuildMember) => void` | New member joined (requires `GUILD_MEMBERS`) |
| `guildMemberRemove` | `(member: GuildMember) => void` | Member left or was removed |
| `guildMemberUpdate` | `(oldMem: GuildMember, newMem: GuildMember) => void` | Nickname, roles, or timeout updated |
| `channelCreate` | `(channel: Channel) => void` | Channel created |
| `channelUpdate` | `(oldCh: Channel, newCh: Channel) => void` | Channel topic, name or perms updated |
| `channelDelete` | `(channel: Channel) => void` | Channel deleted |
| `voiceStateUpdate` | `(oldState: VoiceState, newState: VoiceState) => void` | User joins, leaves, moves, mutes or deafens |


### 6.3 Component Builders Reference


#### EmbedBuilder

```javascript
import { EmbedBuilder } from '@jubbio/core';

const embed = new EmbedBuilder()
  .setTitle('Sunucu Metrikleri')
  .setDescription('Anlık performans ve kullanıcı istatistikleri:')
  .setColor(0x00FF88) // Decimal hex color integer
  .setAuthor({ name: 'Sistem Yöneticisi', iconURL: 'https://cdn.example.com/admin.png' })
  .setThumbnail('https://cdn.example.com/logo.png')
  .addFields(
    { name: 'Aktif Üye', value: '1,420', inline: true },
    { name: 'Ping', value: '24ms', inline: true }
  )
  .setImage('https://cdn.example.com/graph.png')
  .setFooter({ text: 'Jubbio Monitoring Engine' })
  .setTimestamp();
```


#### ButtonBuilder & ActionRowBuilder

```javascript
import { ActionRowBuilder, ButtonBuilder, ButtonStyle } from '@jubbio/core';

const row = new ActionRowBuilder().addComponents(
  new ButtonBuilder()
    .setCustomId('btn_verify')
    .setLabel('Doğrula')
    .setStyle(ButtonStyle.Success)
    .setEmoji('🛡️'),
  new ButtonBuilder()
    .setCustomId('btn_cancel')
    .setLabel('Vazgeç')
    .setStyle(ButtonStyle.Danger),
  new ButtonBuilder()
    .setLabel('Dokümantasyon')
    .setStyle(ButtonStyle.Link)
    .setURL('https://jubbio.com/dev/docs')
);

// Limits: Max 5 ActionRows per message, max 5 buttons per ActionRow.
```


#### StringSelectMenuBuilder

```javascript
import { ActionRowBuilder, StringSelectMenuBuilder, StringSelectMenuOptionBuilder } from '@jubbio/core';

const menu = new StringSelectMenuBuilder()
  .setCustomId('select_category')
  .setPlaceholder('Bir konu seçin...')
  .setMinValues(1)
  .setMaxValues(1)
  .addOptions([
    new StringSelectMenuOptionBuilder()
      .setLabel('JavaScript')
      .setDescription('Node.js & TypeScript soruları')
      .setValue('cat_js')
      .setEmoji('🟨'),
    new StringSelectMenuOptionBuilder()
      .setLabel('Python')
      .setDescription('Django & FastAPI soruları')
      .setValue('cat_py')
      .setEmoji('🐍')
  ]);

const row = new ActionRowBuilder().addComponents(menu);
```


#### ModalBuilder & TextInputBuilder

```javascript
import { ModalBuilder, TextInputBuilder, TextInputStyle, ActionRowBuilder } from '@jubbio/core';

const modal = new ModalBuilder()
  .setCustomId('modal_ticket')
  .setTitle('Destek Bileti');

const subjectInput = new TextInputBuilder()
  .setCustomId('ticket_subject')
  .setLabel('Konu')
  .setStyle(TextInputStyle.Short)
  .setRequired(true)
  .setMaxLength(100);

const descInput = new TextInputBuilder()
  .setCustomId('ticket_desc')
  .setLabel('Açıklama')
  .setStyle(TextInputStyle.Paragraph)
  .setRequired(true)
  .setMinLength(10);

modal.addComponents(
  new ActionRowBuilder().addComponents(subjectInput),
  new ActionRowBuilder().addComponents(descInput)
);

// To render in interaction:
await interaction.showModal(modal);
```


## 7. Voice Subsystem (@jubbio/voice)

`@jubbio/voice` provides UDP audio streaming, Opus encoding, and audio player pipeline management.

```javascript
import {
  joinVoiceChannel,
  createAudioPlayer,
  createAudioResource,
  AudioPlayerStatus,
  VoiceConnectionStatus
} from '@jubbio/voice';

// 1. Connect to Voice Channel
const connection = joinVoiceChannel({
  channelId: voiceChannel.id,
  guildId: voiceChannel.guild.id,
  adapterCreator: voiceChannel.guild.voiceAdapterCreator,
  selfDeaf: true,
  selfMute: false,
});

// 2. Create Player
const player = createAudioPlayer();

// 3. Create Audio Resource (file path, stream, or URL)
const resource = createAudioResource('/assets/notification.mp3', {
  inlineVolume: true,
});
resource.volume?.setVolume(0.75); // 75% volume

// 4. Play and Subscribe
player.play(resource);
connection.subscribe(player);

player.on(AudioPlayerStatus.Playing, () => {
  console.log('🎵 Ses çalınıyor...');
});

player.on(AudioPlayerStatus.Idle, () => {
  console.log('⏹️ Ses tamamlandı.');
});

connection.on(VoiceConnectionStatus.Disconnected, () => {
  connection.destroy();
});
```


## 8. Permissions & PermissionFlagsBits Reference

Permissions in Jubbio use 64-bit BigInt bitfields handled by `PermissionsBitField`:

```javascript
import { PermissionsBitField, PermissionFlagsBits } from '@jubbio/core';

// Check permissions
if (member.permissions.has(PermissionFlagsBits.Administrator)) {
  // Administrator bypasses all channel restrictions
}

// Check multiple
const canModerate = member.permissions.has([
  PermissionFlagsBits.KickMembers,
  PermissionFlagsBits.BanMembers,
]);
```

### Complete PermissionFlagsBits Table:
| Flag Name | BigInt Bitwise Value | Description |
| :--- | :--- | :--- |
| `CreateInstantInvite` | `1n << 0n` | Create channel invite links |
| `KickMembers` | `1n << 1n` | Kick guild members |
| `BanMembers` | `1n << 2n` | Ban guild members |
| `Administrator` | `1n << 3n` | Full administrative bypass |
| `ManageChannels` | `1n << 4n` | Create, edit, and delete channels |
| `ManageGuild` | `1n << 5n` | Modify server settings and vanity URL |
| `AddReactions` | `1n << 6n` | Add emoji reactions to messages |
| `ViewAuditLog` | `1n << 7n` | View audit log records |
| `PrioritySpeaker` | `1n << 8n` | Lower others' volume during voice speech |
| `Stream` | `1n << 9n` | Video and screen sharing |
| `ViewChannel` | `1n << 10n` | Read channel history and see channel in sidebar |
| `SendMessages` | `1n << 11n` | Send text messages in channels |
| `SendTTSMessages` | `1n << 12n` | Send text-to-speech messages |
| `ManageMessages` | `1n << 13n` | Delete or pin other members' messages |
| `EmbedLinks` | `1n << 14n` | Render rich embeds |
| `AttachFiles` | `1n << 15n` | Upload files and media attachments |
| `ReadMessageHistory` | `1n << 16n` | Read historical channel messages |
| `MentionEveryone` | `1n << 17n` | Mention @everyone and @here |
| `UseExternalEmojis` | `1n << 18n` | Use custom emojis from other servers |
| `Connect` | `1n << 20n` | Connect to voice channels |
| `Speak` | `1n << 21n` | Transmit audio in voice channels |
| `MuteMembers` | `1n << 22n` | Server-mute members in voice |
| `DeafenMembers` | `1n << 23n` | Server-deafen members in voice |
| `MoveMembers` | `1n << 24n` | Move members between voice channels |
| `ChangeNickname` | `1n << 26n` | Change own nickname |
| `ManageNicknames` | `1n << 27n` | Change other members' nicknames |
| `ManageRoles` | `1n << 28n` | Create, edit, and assign roles lower than own |
| `ManageWebhooks` | `1n << 29n` | Create and manage channel webhooks |
| `ManageEmojisAndStickers` | `1n << 30n` | Manage custom emojis |
| `UseApplicationCommands` | `1n << 31n` | Execute slash commands |
| `ModerateMembers` | `1n << 40n` | Apply communication timeouts to members |


## 9. Collectors & Sharding Reference


### 9.1 Message & Interaction Collectors

```javascript
// 1. Await Single Message Response
const filter = (m) => m.author.id === targetUserId && m.content === 'onayla';
try {
  const collected = await channel.awaitMessages({ filter, max: 1, time: 30000, errors: ['time'] });
  console.log('Alınan yanıt:', collected.first().content);
} catch {
  console.log('Süre doldu!');
}

// 2. Component Interaction Collector
const collector = channel.createMessageComponentCollector({
  filter: (i) => i.customId === 'btn_vote' && !i.user.bot,
  time: 60000,
});

collector.on('collect', async (i) => {
  await i.reply({ content: 'Oyunuz kaydedildi!', ephemeral: true });
});

collector.on('end', (collected) => {
  console.log(`Toplanan oy sayısı: ${collected.size}`);
});
```


### 9.2 ShardingManager

```javascript
import { ShardingManager } from '@jubbio/core';

const manager = new ShardingManager('./bot.js', {
  token: process.env.BOT_TOKEN,
  totalShards: 'auto', // Auto calculates from Gateway recommendation
  respawn: true,
});

manager.on('shardCreate', (shard) => {
  console.log(`🚀 Shard #${shard.id} başlatıldı.`);
});

await manager.spawn();

// Broadcast evaluation across all worker processes:
const results = await manager.broadcastEval(c => c.guilds.cache.size);
const totalGuilds = results.reduce((acc, count) => acc + count, 0);
```


## 10. Enums & Markdown Formatting Syntax


### 10.1 Key Enums

- **ChannelType:** `GuildText` (0), `DM` (1), `GuildVoice` (2), `GroupDM` (3), `GuildCategory` (4), `GuildAnnouncement` (5), `GuildStageVoice` (13).
- **ButtonStyle:** `Primary` (1, Blue), `Secondary` (2, Gray), `Success` (3, Green), `Danger` (4, Red), `Link` (5, URL link).
- **TextInputStyle:** `Short` (1, single-line), `Paragraph` (2, multi-line).
- **ApplicationCommandType:** `ChatInput` (1), `User` (2), `Message` (3).


### 10.2 Formatting Helper Functions

```javascript
import {
  bold,
  italic,
  strikethrough,
  inlineCode,
  codeBlock,
  time,
  userMention,
  channelMention,
  roleMention,
  hyperlink,
  spoiler
} from '@jubbio/core';

const text = [
  bold('Kalın Metin'),
  italic('İtalik Metin'),
  strikethrough('Çizili Metin'),
  spoiler('Gizli Spoiler'),
  inlineCode('const a = 1;'),
  codeBlock('json', JSON.stringify({ hello: 'world' })),
  userMention('554200123456789012'), // <@554200123456789012>
  channelMention('545980009056055296'), // <#545980009056055296>
  roleMention('540528238699487233'), // <@&540528238699487233>
  time(new Date(), 'R'), // <t:1710000000:R> ("5 minutes ago")
  time(new Date(), 'F'), // <t:1710000000:F> ("Tuesday, March 10, 2026 12:00 PM")
  hyperlink('Jubbio Docs', 'https://jubbio.com/dev/docs')
].join('\n');
```

