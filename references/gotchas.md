# Gotchas & Debugging Reference — discord.js v14

## "Unknown Interaction" (error code 10062)

This is the most common bot error. It means Discord's interaction token is no longer valid.

**Root causes:**
1. **3-second deadline missed** — you didn't call `reply()`, `deferReply()`, or `showModal()` within 3 seconds of receiving the interaction.
2. **Double-reply** — you called `reply()` twice (e.g., both in a command handler and a generic interactionCreate listener).
3. **Async code without await** — a `reply()` call fired without `await`, and then an `editReply()` ran before it completed.
4. **Production restarts** — bot restarted while Discord still has an old interaction open. Users clicking old buttons send interactions to your bot; if the bot doesn't recognize the `customId`, it won't reply and Discord marks it unknown.

**Solutions:**

```ts
// Pattern 1: Always defer if work takes > 2 seconds
export async function execute(interaction: ChatInputCommandInteraction) {
  await interaction.deferReply(); // <-- do this first, always
  const result = await someSlowApi();
  await interaction.editReply(result);
}

// Pattern 2: Centralized error handling
try {
  await command.execute(interaction);
} catch (err) {
  const payload = { content: 'Something went wrong.', ephemeral: true };
  if (interaction.deferred || interaction.replied) {
    await interaction.editReply(payload).catch(() => {});
  } else {
    await interaction.reply(payload).catch(() => {});
  }
}

// Pattern 3: Guard against stale button interactions
client.on('interactionCreate', async interaction => {
  if (interaction.isButton()) {
    const handled = knownCustomIds.has(interaction.customId);
    if (!handled) {
      await interaction.reply({ content: 'This button is no longer active.', ephemeral: true });
      return;
    }
  }
});
```

---

## Ephemeral messages — what you can and can't do

- ✅ Reply ephemerally: `interaction.reply({ content: '...', ephemeral: true })`
- ✅ Defer ephemerally: `interaction.deferReply({ ephemeral: true })`
- ❌ You **cannot change** ephemeral after the first response — pick it once and stick with it
- ❌ `followUp()` after an ephemeral reply is **public** unless you also set `ephemeral: true` on the followUp
- ❌ `fetchReply: true` on ephemeral replies sometimes fails — the reply isn't stored the same way
- ❌ You cannot edit or delete ephemeral messages after 15 minutes

---

## Missing intents / empty caches

**MessageContent (privileged):**
```ts
// Symptom: message.content is "" for messages received via gateway
// Fix: add intent AND enable in Discord Developer Portal
intents: [GatewayIntentBits.MessageContent]
// If bot is verified (> 100 guilds), must be approved in dev portal
```

**GuildMembers (privileged):**
```ts
// Symptom: guild.members.cache is mostly empty; guildMemberAdd never fires
intents: [GatewayIntentBits.GuildMembers]
// Also required for: fetchMembers(), member.presence, member.roles in DMs
```

**Partials for DM reactions/messages:**
```ts
// Symptom: DM message events not firing, or messageReactionAdd missing reactions on old messages
partials: [Partials.Channel, Partials.Message, Partials.Reaction]
// Then always check if partial before accessing properties:
if (message.partial) await message.fetch();
```

---

## Rate limits

discord.js handles rate limits automatically (queues requests, waits for reset). However:

```ts
// Monitor global rate limits if you're doing bulk operations
client.rest.on('rateLimited', info => {
  console.warn(`Rate limited: ${info.route} — retry after ${info.timeToReset}ms`);
});

// Bulk REST operations — stagger if needed
const delay = (ms: number) => new Promise(r => setTimeout(r, ms));
for (const user of users) {
  await dmChannel.send('...');
  await delay(500); // avoid hitting DM rate limits (~10-20/min per bot)
}
```

**Key limits to know:**
- Global: 50 req/s
- Message create: ~5/5s per channel
- Webhook: 5 req/2s per webhook
- DM: ~10-20 DMs per minute (unofficial, varies)
- Bulk delete: messages must be < 14 days old; max 100 per call
- Invalid requests (4xx responses): 10,000 per 10 minutes → 24-hour ban if exceeded

---

## Caching pitfalls

**Members not in cache:**
```ts
// Don't rely on guild.members.cache.get(userId) — it won't have everyone
// Always fetch when you need a specific member:
const member = await guild.members.fetch(userId);

// Or enable fetchAllMembers (expensive, not recommended for large guilds):
// client = new Client({ fetchAllMembers: true })
```

**Old messages:**
```ts
// guild.channels.cache only holds the last ~200 messages per channel
// For older messages, fetch:
const msg = await channel.messages.fetch(messageId);
```

**Stale guild data:**
```ts
// If guild data looks wrong, refresh it:
const freshGuild = await guild.fetch();
```

---

## Token security

- **Never commit your token.** Use `.env` + `.gitignore`.
- If a token is exposed (committed, logged, etc.), **immediately regenerate it** in the Discord Developer Portal. Old token is invalid immediately.
- Use separate applications for dev and prod — don't share tokens.
- For CI/CD, store tokens as encrypted secrets, not environment files.

```ts
// Catch token errors on startup
client.login(process.env.TOKEN).catch(err => {
  if (err.code === 'TokenInvalid') {
    console.error('Invalid token. Regenerate at discord.com/developers/applications');
  }
  process.exit(1);
});
```

---

## Avoiding the event loop block

Discord.js runs on Node's single-threaded event loop. Any synchronous blocking freezes the entire bot.

```ts
// ❌ BAD — blocks event loop during computation
client.on('interactionCreate', interaction => {
  const result = computeHeavyThing(largeArray); // CPU-intensive, blocks
  interaction.reply(result);
});

// ✅ GOOD — offload to worker or split across ticks
import { Worker } from 'worker_threads';

client.on('interactionCreate', async interaction => {
  await interaction.deferReply();
  const result = await runInWorker(largeArray); // non-blocking
  await interaction.editReply(result);
});

// For moderately heavy work, setTimeout(0) or setImmediate() can help
// yield control between iterations:
for (const chunk of chunks(bigArray, 100)) {
  processChunk(chunk);
  await new Promise(r => setImmediate(r)); // yield to event loop
}
```

---

## Permissions v2 — server-side vs code-side

Since Discord's **Permissions v2** update (2022), slash command permissions are controlled server-side
by admins in Server Settings → Integrations. `setDefaultMemberPermissions()` only sets the *default*
— server admins can override it.

```ts
// What this does: hides the command from users who lack ManageGuild by default
.setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild)

// For absolute enforcement, ALSO check in execute():
if (!interaction.memberPermissions?.has(PermissionFlagsBits.ManageGuild)) {
  return interaction.reply({ content: 'Insufficient permissions.', ephemeral: true });
}
```

Don't rely solely on `setDefaultMemberPermissions` for security-critical commands.

---

## Common TypeScript issues

```ts
// "interaction.member is User | APIInteractionGuildMember | null"
// Cast properly:
import { GuildMember } from 'discord.js';
const member = interaction.member as GuildMember;

// "guild may be null" — always guard in non-DM commands
if (!interaction.inCachedGuild()) return; // narrows types to guild context

// Collection vs Map — discord.js uses Collection, which extends Map
// Collection has extra methods: .find(), .filter(), .map(), .some(), .every()
const result = client.commands.find(cmd => cmd.data.name === 'ping');

// SlashCommandBuilder loses type info in toJSON() — use satisfies:
const cmd = {
  data: new SlashCommandBuilder().setName('...'),
  execute: async (i: ChatInputCommandInteraction) => { /* ... */ },
} satisfies Command;
```

---

## ESM vs CommonJS

discord.js v14 is ESM-first. Using `require()` still works but you'll have conflicts with
ESM-only packages (`@discordjs/voice`, `chalk`, etc.).

**Recommended setup:**
```json
// package.json
{
  "type": "module"
}
```
```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022"
  }
}
```

With `"type": "module"`, use `import.meta.dirname` instead of `__dirname`, and all imports need
explicit `.js` extensions (even for `.ts` source files):
```ts
import { BotClient } from './client.js'; // correct in ESM TS
```

---

## Debugging checklist

When something isn't working, run through:

1. ☐ **Are intents enabled** in Developer Portal AND in code?
2. ☐ **Did you re-register commands** after adding/renaming options?
3. ☐ **Are you on a guild command** during dev (not global)?
4. ☐ **Is `await` missing** before an interaction method?
5. ☐ **Is there a try/catch** hiding the real error?
6. ☐ **Does the bot have permission** in that specific channel?
7. ☐ **Is the correct bot token** being used (not a copy from a different app)?
8. ☐ **Is the error logged?** Set `client.on('warn', ...)` and `client.on('error', ...)`.
