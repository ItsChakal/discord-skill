# Advanced Features Reference — discord.js v14

## Threads

```ts
import { ChannelType, ThreadAutoArchiveDuration } from 'discord.js';

// Create a public thread on a message
const thread = await message.startThread({
  name: 'Discussion',
  autoArchiveDuration: ThreadAutoArchiveDuration.OneDay, // 60 | 1440 | 4320 | 10080
  reason: 'Started discussion thread',
});

// Create thread directly on a text channel (no parent message)
const thread = await textChannel.threads.create({
  name: 'Support Thread',
  autoArchiveDuration: ThreadAutoArchiveDuration.OneWeek,
  type: ChannelType.PrivateThread, // requires COMMUNITY or Nitro Boost lv2
  invitable: false, // private threads only — whether non-mods can add members
});

// Manage members
await thread.members.add(userId);
await thread.members.remove(userId);

// Archive / unarchive
await thread.setArchived(true);
await thread.setArchived(false);

// Listen for thread events
client.on('threadCreate', async (thread) => {
  if (!thread.joinable) return;
  await thread.join(); // Bot must join to receive messages in the thread
});
```

**Intent note:** `GatewayIntentBits.Guilds` includes basic thread events. For `threadMembersUpdate`,
add `GatewayIntentBits.GuildMembers` (privileged).

---

## Forum channels

Forum channels (`ChannelType.GuildForum`) contain threads (called "posts") as their primary content.

```ts
import { ChannelType, ForumChannel } from 'discord.js';

const forum = guild.channels.cache.get(forumChannelId) as ForumChannel;

// Get available tags
const tagMap = Object.fromEntries(forum.availableTags.map(t => [t.name, t.id]));

// Create a post (always creates a thread)
const post = await forum.threads.create({
  name: 'Bug Report: Login fails on mobile',
  message: {
    content: 'Describe the bug…',
    embeds: [bugEmbed],
  },
  appliedTags: [tagMap['Bug'], tagMap['Needs Triage']].filter(Boolean),
});

// Apply/remove tags on an existing post
const currentTags = post.appliedTags;
await post.setAppliedTags([...currentTags, tagMap['In Progress']]);
```

---

## Voice connections (@discordjs/voice)

```bash
npm install @discordjs/voice @discordjs/opus # or: libsodium-wrappers ffmpeg-static
```

```ts
import {
  joinVoiceChannel,
  createAudioPlayer,
  createAudioResource,
  AudioPlayerStatus,
  VoiceConnectionStatus,
  entersState,
  getVoiceConnection,
} from '@discordjs/voice';
import { createReadStream } from 'fs';

// Join a voice channel
const connection = joinVoiceChannel({
  channelId: voiceChannel.id,
  guildId: guild.id,
  adapterCreator: guild.voiceAdapterCreator,
  selfDeaf: true, // Bot appears deafened (recommended)
});

// Wait for the connection to be ready before playing
await entersState(connection, VoiceConnectionStatus.Ready, 20_000);

// Create and configure a player
const player = createAudioPlayer();
connection.subscribe(player);

// Play audio from a file
const resource = createAudioResource(createReadStream('audio.mp3'), {
  inlineVolume: true, // enables resource.volume.setVolume()
});
resource.volume?.setVolume(0.5); // 0.0 – 1.0

player.play(resource);

// Await playback to finish
await entersState(player, AudioPlayerStatus.Idle, 5 * 60 * 1_000);

// Disconnect
connection.destroy();
```

**Playing from a URL (e.g., YouTube via ytdl-core):**
```ts
import ytdl from 'ytdl-core';
const stream = ytdl('https://youtu.be/...', { filter: 'audioonly', quality: 'highestaudio' });
const resource = createAudioResource(stream);
player.play(resource);
```

**Retrieve existing connection (e.g., in a stop command):**
```ts
const connection = getVoiceConnection(interaction.guildId!);
connection?.destroy();
```

---

## Auto moderation

```ts
import {
  AutoModerationRuleTriggerType,
  AutoModerationRuleEventType,
  AutoModerationActionType,
} from 'discord.js';

// Create a keyword rule
await guild.autoModerationRules.create({
  name: 'Banned Words',
  enabled: true,
  eventType: AutoModerationRuleEventType.MessageSend,
  triggerType: AutoModerationRuleTriggerType.Keyword,
  triggerMetadata: {
    keywordFilter: ['badword', 'otherword'],
    regexPatterns: ['^(buy|sell)\\s+cheap'],
  },
  actions: [
    {
      type: AutoModerationActionType.BlockMessage,
      metadata: { customMessage: 'Your message contained a banned word.' },
    },
    {
      type: AutoModerationActionType.SendAlertMessage,
      metadata: { channelId: modLogChannelId },
    },
    {
      type: AutoModerationActionType.Timeout,
      metadata: { durationSeconds: 60 },
    },
  ],
  exemptRoles: [modRoleId],
  exemptChannels: [botCommandsChannelId],
});

// Listen for rule executions
client.on('autoModerationActionExecution', (execution) => {
  console.log(
    `AutoMod rule "${execution.ruleTriggerType}" fired in #${execution.channel?.name} ` +
    `for ${execution.user?.tag}: "${execution.content}"`
  );
});
```

Trigger types: `Keyword`, `Spam`, `KeywordPreset`, `MentionSpam`, `MemberProfile`.

---

## Scheduled events (guild events)

```ts
import {
  GuildScheduledEventEntityType,
  GuildScheduledEventPrivacyLevel,
  GuildScheduledEventStatus,
} from 'discord.js';

// Create an event
const event = await guild.scheduledEvents.create({
  name: 'Weekly Game Night',
  scheduledStartTime: new Date(Date.now() + 3_600_000), // 1 hour from now
  scheduledEndTime: new Date(Date.now() + 7_200_000),   // 2 hours from now
  privacyLevel: GuildScheduledEventPrivacyLevel.GuildOnly,
  entityType: GuildScheduledEventEntityType.Voice,
  channel: voiceChannel, // Required for Voice type
  description: 'Come hang out and game!',
  image: 'https://example.com/banner.png',
});

// Fetch subscribers (users who RSVP'd)
const subscribers = await event.fetchSubscribers({ withMember: true });

// Update status
await event.setStatus(GuildScheduledEventStatus.Active);   // start early
await event.setStatus(GuildScheduledEventStatus.Completed); // end

// Events
client.on('guildScheduledEventCreate', event => { /* */ });
client.on('guildScheduledEventUpdate', (oldEvent, newEvent) => { /* */ });
client.on('guildScheduledEventUserAdd', (event, user) => { /* RSVP'd */ });
```

---

## Webhooks

```ts
// Create a webhook in a channel
const webhook = await channel.createWebhook({
  name: 'My Webhook',
  avatar: 'https://example.com/avatar.png',
  reason: 'For notifications',
});

// Send through webhook — no bot token consumed, faster rate limit
await webhook.send({
  content: 'Hello from webhook!',
  username: 'Custom Name',  // override the webhook's display name
  avatarURL: 'https://example.com/custom-avatar.png',
  embeds: [embed],
  threadId: thread.id, // send into a thread via webhook
});

// Use an existing webhook by URL (e.g., from env vars)
import { WebhookClient } from 'discord.js';
const webhookClient = new WebhookClient({ url: process.env.WEBHOOK_URL! });
await webhookClient.send('Message from anywhere, no bot client needed');
```

**Rate limits:** Webhooks share the channel's rate limit: ~5 requests per 2 seconds.
`WebhookClient` is useful for logging, alerts, and integrations that don't need a full bot.

---

## Entitlements and monetization

```ts
// Fetch entitlements for a user
const entitlements = await client.application!.entitlements.fetch({
  userId: interaction.user.id,
  skuIds: [premiumSkuId],
  excludeEnded: true,
});

if (!entitlements.size) {
  // User does not have the premium SKU — prompt upgrade
  return interaction.reply({
    content: '⭐ This is a premium feature.',
    components: [premiumUpgradeButton],
    ephemeral: true,
  });
}

// Listen for new purchases
client.on('entitlementCreate', async (entitlement) => {
  const user = await client.users.fetch(entitlement.userId!);
  await user.send(`Thanks for subscribing! Your perks are now active.`);
});

// SKUs (your premium offerings)
const skus = await client.application!.skus.fetch();
```

---

## Polls (Discord native polls, v14.15+)

```ts
import { PollLayoutType } from 'discord.js';

await channel.send({
  poll: {
    question: { text: 'Best programming language?' },
    answers: [
      { text: 'TypeScript', emoji: '💙' },
      { text: 'Python', emoji: '🐍' },
      { text: 'Rust', emoji: '🦀' },
    ],
    duration: 24, // hours
    allowMultiselect: false,
    layoutType: PollLayoutType.Default,
  },
});

client.on('messagePollVoteAdd', (answer, userId) => { /* */ });
client.on('messagePollVoteRemove', (answer, userId) => { /* */ });
```

---

## Role management patterns

```ts
// Add/remove a single role
await member.roles.add(roleId, 'Opted into gaming pings');
await member.roles.remove(roleId);

// Set multiple roles at once (replaces the entire role list)
await member.roles.set([roleId1, roleId2], 'Role sync');

// Create a role
const role = await guild.roles.create({
  name: 'Premium',
  color: 0xFFD700,
  hoist: true,     // show separately in member list
  mentionable: false,
  permissions: [],
  position: 3,
});
```
