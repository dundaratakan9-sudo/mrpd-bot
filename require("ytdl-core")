require("dotenv").config();

const ffmpeg = require("ffmpeg-static");
process.env.FFMPEG_PATH = ffmpeg;

const play = require("play-dl");

const {
  Client,
  GatewayIntentBits,
  Partials,
  PermissionsBitField,s
  ChannelType,
} = require("discord.js");

const {
  joinVoiceChannel,
  createAudioPlayer,
  createAudioResource,
  AudioPlayerStatus,
  VoiceConnectionStatus,
} = require("@discordjs/voice");


// -------- Client --------
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildVoiceStates,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ],
  partials: [Partials.Channel],
});

// -------- Crash guard --------
process.on("unhandledRejection", (e) => console.error("UNHANDLED:", e));
process.on("uncaughtException", (e) => console.error("UNCAUGHT:", e));

// -------- Music Queue --------
const queues = new Map(); // guildId -> { connection, player, songs: [{url, requestedBy}] }

async function playNext(guildId, textChannel) {
  const q = queues.get(guildId);
  if (!q || q.songs.length === 0) {
    try { q?.connection?.destroy(); } catch {}
    queues.delete(guildId);
    return;
  }

  const song = q.songs[0];
  const stream = ytdl(song.url, {
    filter: "audioonly",
    quality: "highestaudio",
    highWaterMark: 1 << 25,
  });

  const resource = createAudioResource(stream);
  q.player.play(resource);

  if (textChannel) {
    textChannel.send(`🎶 Çalıyor: ${song.url} (İsteyen: <@${song.requestedBy}>)`).catch(() => {});
  }
}

// -------- Ready & Slash Register (guild) --------
client.once("ready", async () => {
  console.log(`✅ Bot hazır: ${client.user.tag}`);

  const guild = client.guilds.cache.first();
  if (!guild) {
    console.log("❌ Bot hiçbir sunucuda değil.");
    return;
  }

  await guild.commands.set([
    { name: "ping", description: "Bot gecikmesini gösterir" },

    // Music
    {
      name: "play",
      description: "Müzik çalar (YouTube URL)",
      options: [{ name: "url", type: 3, description: "YouTube link", required: true }],
    },
    { name: "queue", description: "Müzik kuyruğunu gösterir" },
    { name: "skip", description: "Şarkıyı geçer" },
    { name: "stop", description: "Müziği durdurur ve kuyruk temizler" },

    // Role add/remove
    {
      name: "rolver",
      description: "Kullanıcıya rol verir",
      options: [
        { name: "kullanici", type: 6, description: "Kime?", required: true },
        { name: "rol", type: 8, description: "Hangi rol?", required: true },
      ],
    },
    {
      name: "rolal",
      description: "Kullanıcıdan rol alır",
      options: [
        { name: "kullanici", type: 6, description: "Kimden?", required: true },
        { name: "rol", type: 8, description: "Hangi rol?", required: true },
      ],
    },

    // Role info
    {
      name: "rolbilgi",
      description: "Sunucuda/rolde kaç kişi var gösterir",
      options: [{ name: "rol", type: 8, description: "İsteğe bağlı rol seç", required: false }],
    },

    // Channel lock/unlock
    {
      name: "kilitle",
      description: "Bu kanalı yazmaya kapatır",
      options: [{ name: "kanal", type: 7, description: "Hangi kanal? (opsiyonel)", required: false }],
    },
    {
      name: "ac",
      description: "Bu kanalı yazmaya açar",
      options: [{ name: "kanal", type: 7, description: "Hangi kanal? (opsiyonel)", required: false }],
    },

    // Small surprises
    { name: "coinflip", description: "Yazı tura atar" },
    {
      name: "8ball",
      description: "Sihirli 8 top",
      options: [{ name: "soru", type: 3, description: "Sorunu yaz", required: true }],
    },
    {
      name: "askolcer",
      description: "İki kişi arasındaki uyumu ölçer",
      options: [
        { name: "kisi1", type: 6, description: "1. kişi", required: true },
        { name: "kisi2", type: 6, description: "2. kişi", required: true },
      ],
    },
    {
      name: "say",
      description: "İstediğin mesajı bot yazsın (sürpriz)",
      options: [{ name: "mesaj", type: 3, description: "Ne yazsın?", required: true }],
    },
  ]);

  console.log("✅ Slash komutları yüklendi (guild).");
});

// -------- Interaction Handler --------
client.on("interactionCreate", async (interaction) => {
  try {
    if (!interaction.isChatInputCommand()) return;

    // ---- Ping
    if (interaction.commandName === "ping") {
      return interaction.reply(`🏓 Pong! ${client.ws.ping}ms`);
    }

    // =====================
    // MUSIC
    // =====================
    if (interaction.commandName === "play") {
      await interaction.deferReply();

      const url = interaction.options.getString("url");
      const member = interaction.member;

      if (!member.voice.channel) {
        return interaction.editReply("❌ Önce bir ses kanalına gir.");
      }
      if (!ytdl.validateURL(url)) {
        return interaction.editReply("❌ Geçersiz YouTube URL.");
      }

      const guildId = interaction.guild.id;
      let q = queues.get(guildId);

      if (!q) {
        const player = createAudioPlayer();

        const connection = joinVoiceChannel({
          channelId: member.voice.channel.id,
          guildId: interaction.guild.id,
          adapterCreator: interaction.guild.voiceAdapterCreator,
          selfDeaf: true,
        });

        q = { connection, player, songs: [] };
        queues.set(guildId, q);

        connection.subscribe(player);

        player.on(AudioPlayerStatus.Idle, () => {
          q.songs.shift();
          playNext(guildId, interaction.channel);
        });

        connection.on(VoiceConnectionStatus.Disconnected, () => {
          queues.delete(guildId);
        });
      }

      q.songs.push({ url, requestedBy: interaction.user.id });

      if (q.player.state.status !== AudioPlayerStatus.Playing) {
        await interaction.editReply("▶️ Müzik başlatılıyor...");
        await playNext(guildId, interaction.channel);
        return;
      }

      return interaction.editReply(`✅ Kuyruğa eklendi: ${url}`);
    }

    if (interaction.commandName === "queue") {
      const q = queues.get(interaction.guild.id);
      if (!q || q.songs.length === 0) return interaction.reply("📭 Kuyruk boş.");

      const list = q.songs
        .slice(0, 10)
        .map((s, i) => `${i === 0 ? "🎶" : "📝"} ${i + 1}. ${s.url} (<@${s.requestedBy}>)`)
        .join("\n");

      return interaction.reply(`📃 **Kuyruk (ilk 10):**\n${list}`);
    }

    if (interaction.commandName === "skip") {
      const q = queues.get(interaction.guild.id);
      if (!q) return interaction.reply("❌ Şu an çalan bir şey yok.");
      q.player.stop(true);
      return interaction.reply("⏭️ Geçildi!");
    }

    if (interaction.commandName === "stop") {
      const q = queues.get(interaction.guild.id);
      if (!q) return interaction.reply("❌ Şu an çalan bir şey yok.");

      q.songs = [];
      q.player.stop(true);
      try { q.connection.destroy(); } catch {}
      queues.delete(interaction.guild.id);

      return interaction.reply("⏹️ Durduruldu, kuyruk temizlendi.");
    }

    // =====================
    // ROLE ADD / REMOVE
    // =====================
    if (interaction.commandName === "rolver" || interaction.commandName === "rolal") {
      if (!interaction.member.permissions.has(PermissionsBitField.Flags.ManageRoles)) {
        return interaction.reply({ content: "❌ Yetkin yok! (Manage Roles)", ephemeral: true });
      }

      const member = interaction.options.getMember("kullanici");
      const role = interaction.options.getRole("rol");
      if (!member || !role) return interaction.reply({ content: "❌ Üye/rol bulunamadı.", ephemeral: true });

      const botMember = interaction.guild.members.me;
      if (!botMember) return interaction.reply({ content: "❌ Bot üyesi bulunamadı.", ephemeral: true });

      // bot rolü hedeften yüksek olmalı
      if (botMember.roles.highest.position <= role.position) {
        return interaction.reply({
          content: "❌ Bu rolü yönetemem. Botun en yüksek rolü hedef rolden daha üstte olmalı.",
          ephemeral: true,
        });
      }

      // kullanıcı rolü hedef rolden düşük olmalı (isteğe bağlı güvenlik)
      if (interaction.member.roles.highest.position <= role.position && interaction.guild.ownerId !== interaction.user.id) {
        return interaction.reply({
          content: "❌ Bu rol üzerinde işlem yapamazsın (rol hiyerarşisi).",
          ephemeral: true,
        });
      }

      if (interaction.commandName === "rolver") {
        await member.roles.add(role);
        return interaction.reply(`✅ ${member.user.tag} kişisine **${role.name}** rolü verildi.`);
      } else {
        await member.roles.remove(role);
        return interaction.reply(`✅ ${member.user.tag} kişisinden **${role.name}** rolü alındı.`);
      }
    }

    // =====================
    // ROLE INFO
    // =====================
    if (interaction.commandName === "rolbilgi") {
      await interaction.deferReply({ ephemeral: true });

      const role = interaction.options.getRole("rol");
      const guild = interaction.guild;

      await guild.members.fetch();

      const totalMembers = guild.memberCount;
      const humans = guild.members.cache.filter((m) => !m.user.bot).size;
      const bots = guild.members.cache.filter((m) => m.user.bot).size;

      if (!role) {
        return interaction.editReply(
          `📊 **Sunucu İstatistikleri**\n` +
            `👥 Toplam üye: **${totalMembers}**\n` +
            `🧑 İnsan: **${humans}**\n` +
            `🤖 Bot: **${bots}**\n` +
            `🎭 Rol sayısı: **${guild.roles.cache.size}**\n` +
            `💬 Kanal sayısı: **${guild.channels.cache.size}**`
        );
      }

      const roleCount = guild.members.cache.filter((m) => m.roles.cache.has(role.id)).size;

      return interaction.editReply(
        `🎭 **Rol Bilgisi**\n` +
          `Rol: **${role.name}**\n` +
          `Bu rolde olan kişi sayısı: **${roleCount}**\n` +
          `Rol ID: \`${role.id}\``
      );
    }

    // =====================
    // CHANNEL LOCK / UNLOCK
    // =====================
    if (interaction.commandName === "kilitle" || interaction.commandName === "ac") {
      if (!interaction.member.permissions.has(PermissionsBitField.Flags.ManageChannels)) {
        return interaction.reply({ content: "❌ Yetkin yok! (Manage Channels)", ephemeral: true });
      }

      const channel = interaction.options.getChannel("kanal") ?? interaction.channel;

      if (
        !channel ||
        (channel.type !== ChannelType.GuildText &&
          channel.type !== ChannelType.GuildAnnouncement &&
          channel.type !== ChannelType.PublicThread &&
          channel.type !== ChannelType.PrivateThread)
      ) {
        return interaction.reply({ content: "❌ Bu komut sadece yazı kanallarında çalışır.", ephemeral: true });
      }

      const everyoneId = interaction.guild.roles.everyone.id;

      if (interaction.commandName === "kilitle") {
        await channel.permissionOverwrites.edit(everyoneId, { SendMessages: false });
        return interaction.reply(`🔒 ${channel} kanalı kilitlendi.`);
      } else {
        await channel.permissionOverwrites.edit(everyoneId, { SendMessages: null });
        return interaction.reply(`🔓 ${channel} kanalı açıldı.`);
      }
    }

    // =====================
    // SURPRISES
    // =====================
    if (interaction.commandName === "coinflip") {
      const r = Math.random() < 0.5 ? "🪙 Yazı" : "🪙 Tura";
      return interaction.reply(r);
    }

    if (interaction.commandName === "8ball") {
      const soru = interaction.options.getString("soru");
      const cevaplar = [
        "Kesinlikle!",
        "Olabilir 👀",
        "Bence hayır.",
        "Tekrar sor, şimdi nazlıyım.",
        "Şansın yüksek 😄",
        "Hiç sanmıyorum.",
        "Evet ama dikkatli ol.",
        "Muhtemelen.",
        "Bu soruyu sormadın sayıyorum 😌",
        "Kaderin güzel yazılmış.",
      ];
      const sec = cevaplar[Math.floor(Math.random() * cevaplar.length)];
      return interaction.reply(`🎱 **Soru:** ${soru}\n**Cevap:** ${sec}`);
    }

    if (interaction.commandName === "askolcer") {
      const a = interaction.options.getUser("kisi1");
      const b = interaction.options.getUser("kisi2");
      const percent = Math.floor(Math.random() * 101);

      let yorum = "🤝 Normal";
      if (percent >= 85) yorum = "💍 Evlenin gitsin";
      else if (percent >= 70) yorum = "🔥 Çok iyi";
      else if (percent >= 50) yorum = "😄 İdare eder";
      else if (percent >= 30) yorum = "😬 Zor";
      else yorum = "🧊 Buz gibi";

      return interaction.reply(`❤️ ${a} × ${b}\n**Uyum:** **%${percent}**\n${yorum}`);
    }

    if (interaction.commandName === "say") {
      if (!interaction.member.permissions.has(PermissionsBitField.Flags.ManageMessages)) {
        return interaction.reply({ content: "❌ Yetkin yok! (Manage Messages)", ephemeral: true });
      }
      const msg = interaction.options.getString("mesaj");
      await interaction.reply({ content: "✅", ephemeral: true });
      return interaction.channel.send(msg);
    }
  } catch (err) {
    console.error(err);
    if (interaction?.deferred || interaction?.replied) {
      interaction.followUp({ content: "❌ Hata oluştu.", ephemeral: true }).catch(() => {});
    } else {
      interaction.reply({ content: "❌ Hata oluştu.", ephemeral: true }).catch(() => {});
    }
  }
});

// Optional: message trigger
client.on("messageCreate", (message) => {
  if (message.author.bot) return;
  if (message.content.toLowerCase() === "sa") message.reply("Aleyküm selam 👋");
});


client.login(process.env.TOKEN.trim());




