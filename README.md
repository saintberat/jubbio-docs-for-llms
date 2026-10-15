<div align="center">

# LLM'ler ve Yapay Zeka Ajanları İçin Jubbio Geliştirici Dokümantasyonu

[![Jubbio API](https://img.shields.io/badge/Jubbio_API-v1-blue.svg)](https://jubbio.com/dev/docs)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)](https://nodejs.org/)
[![llms.txt](https://img.shields.io/badge/llms.txt-uyumlu-orange.svg)](https://llmstxt.org)
[![License: MIT](https://img.shields.io/badge/Lisans-MIT-yellow.svg)](LICENSE)

**Jubbio geliştirici ekosistemi için hazırlanmış; LLM'ler, yapay zeka kodlama ajanları ve bot geliştiricileri için optimize edilmiş eksiksiz teknik referans deposu.**

[Teknik Referans Kılavuzu (İngilizce)](JUBBIO_DEV_REFERENCE.md) • [llms.txt Standardı](llms.txt) • [Tam Düz Metin (llms-full.txt)](llms-full.txt) • [Resmi Dokümantasyon](https://jubbio.com/dev/docs)

</div>

---

## 📌 Bu Repo Hakkında

**Jubbio**, modern bir gerçek zamanlı mesajlaşma, topluluk ve ses platformudur. Geliştirici ekosistemi; zengin bir WebSocket Gateway protokolü, REST API'leri ve resmi TypeScript/JavaScript kütüphanelerini (`@jubbio/core`, `@jubbio/voice`, `@jubbio/auth`) içerir.

Jubbio yeni bir platform olduğu için, genel yapay zeka modelleri (LLM'ler) güncel eğitim verilerinde Jubbio'nun endpoint'lerine, payload şemalarına ve kütüphane metotlarına sahip değildir. Bu durum, modellerin Discord veya Slack API'lerini taklit ederek hatalı (halüsinatif) kodlar üretmesine yol açar.

**Bu repo bu sorunu çözmektedir:** Canlı Jubbio Geliştirici Portalı'ndan doğrulanarak çıkarılmış, makine tarafından okunabilir (machine-readable) ve en ince ayrıntısına kadar eksiksiz bir referans sunar.

> [!NOTE]
> **Dil Yapılandırması:** Yapay zeka modellerinin teknik API parametrelerini, veri tiplerini ve metot imzalarını sıfır hatayla işleyebilmesi için LLM'lerin okuduğu teknik dosyalar ([`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md), [`llms.txt`](llms.txt) ve [`llms-full.txt`](llms-full.txt)) **İngilizce** teknik standartta tutulmuştur.

---

## 🤖 Yapay Zeka Ajanları ve Kodlama Araçlarıyla Kullanım

### 1. Cursor IDE / Windsurf
Bu depoyu veya `llms.txt` dosyasını `@Docs` ya da Proje Kurallarınıza ekleyin:
- Cursor Docs dizinine şu URL'yi ekleyin: `https://raw.githubusercontent.com/saintberat/jubbio-docs-for-llms/main/llms.txt`
- Veya [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) dosyasını `.cursorrules` ya da projenizin çalışma alanına context olarak dahil edin.

### 2. Claude Projects / ChatGPT Custom GPT'ler
[`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) (veya `llms-full.txt`) dosyasını Claude Project veya Özel GPT'nizin **Knowledge (Bilgi Tabanı)** bölümüne yükleyin.

### 3. API & Sistem İstemi (System Prompt)
Bir LLM'e Jubbio botu veya entegrasyonu yazdırırken, [`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) içeriğini sistem isteminize (system prompt) referans context olarak ekleyin.

---

## ⚡ Hızlı Platform Özeti

| Bileşen | Değer / Endpoint |
| :--- | :--- |
| **REST API Base URL** | `https://gateway.jubbio.com/api/v1` |
| **WebSocket Gateway** | `wss://gateway.jubbio.com/` |
| **OAuth2 Giriş URL** | `https://jubbio.com/oauth2/authorize` |
| **OAuth2 Token URL** | `https://gateway.jubbio.com/api/v1/oauth2/token` |
| **Bot Yetkilendirme Başlığı** | `Authorization: Bot <BOT_TOKEN>` |
| **Kullanıcı OAuth2 Başlığı** | `Authorization: Bearer <USER_ACCESS_TOKEN>` |
| **Ana Bot Kütüphanesi** | `npm install @jubbio/core` |
| **Ses (Voice) Kütüphanesi** | `npm install @jubbio/voice` |
| **OAuth2 React SDK** | `npm install @jubbio/auth @jubbio/auth-react` |

---

## 🚀 Hızlı Başlangıç Örneği

```javascript
import 'dotenv/config';
import { Client, GatewayIntentBits, SlashCommandBuilder } from '@jubbio/core';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent, // Ayrıcalıklı intent
  ],
});

client.on('ready', async () => {
  console.log(`✅ ${client.user.username} olarak giriş yapıldı!`);

  // Global Slash Komutu Kaydet
  await client.rest.registerGlobalCommands([
    new SlashCommandBuilder()
      .setName('ping')
      .setDescription('Pong ile yanıt verir!')
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

## 📖 Master Referans Kılavuzunun İçeriği

[`JUBBIO_DEV_REFERENCE.md`](JUBBIO_DEV_REFERENCE.md) dosyası şu 10 ana bölümü eksiksiz kapsar:

1. **Kimlik Doğrulama & Token Güvenliği:** Bot Token ve Bearer Token farkları, güvenlik kuralları ve ayrıcalıklı Gateway intent'leri (`GUILD_MEMBERS`, `GUILD_PRESENCES`, `MESSAGE_CONTENT`).
2. **Mesajlar API:** Normal mesajlar, geçici mesajlar (`flags: 64`), embed'ler, reaksiyonlar, pinleme, toplu mesaj silme, çok parçalı (multipart) dosya yükleme ve Direkt Mesajlar (DM).
3. **Kanallar & Moderasyon:** Metin/Ses/Kategori kanalları oluşturma/silme, izin override'ları, timeout (zaman aşımı), kick, ban ve toplu rol yönetimi.
4. **Guild'ler, Roller & Davetler:** Sunucu ayarları, özel emojiler, bitfield ile rol oluşturma/düzenleme ve davet bağlantısı yönetimi.
5. **Uygulama Komutları (Slash Commands):** Global ve Guild komutları, 11 seçenek tipi, hata kodları tablosu (`50035`, `30032` vb.) ve etiketleme token'ları (`<@ID>`, `<@&ID>`, `<#ID>`).
6. **Interaction Callback'leri:** 6 yanıt tipi, yanıt geciktirme (`DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE`) ve takip (followup) mesajları.
7. **Webhook'lar:** Webhook CRUD, çalıştırma, link butonları desteği, Slack uyumluluğu ve GitHub webhook event haritası.
8. **OAuth2 & React SDK:** PKCE akışı, scope tablosu, `@jubbio/auth-react` sağlayıcısı ve `useJubbio()` hook'u.
9. **`@jubbio/voice` Ses Mimarisi:** UDP ses bağlantısı, `AudioPlayer`, ses akışı kaynakları ve yaşam döngüsü.
10. **İzinler, Toplayıcılar & Sharding:** `PermissionsBitField` 64-bit flag'leri, `ShardingManager`, toplayıcılar (`awaitMessages`) ve formatlama yardımcıları.

---

## 📄 Lisans

Bu repo [MIT Lisansı](LICENSE) ile korunmaktadır.
Resmi dokümantasyon içerikleri [Jubbio](https://jubbio.com)'ya aittir.
