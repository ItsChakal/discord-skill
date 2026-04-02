# Architecture Reference — discord.js v14

## Recommended project structure

```
my-bot/
├── src/
│   ├── index.ts              # Entry point — create client, load handlers, login
│   ├── client.ts             # Extended Client class with commands Collection
│   ├── commands/
│   │   ├── utility/
│   │   │   ├── ping.ts
│   │   │   └── help.ts
│   │   └── moderation/
│   │       └── ban.ts
│   ├── events/
│   │   ├── ready.ts
│   │   └── interactionCreate.ts
│   └── deploy-commands.ts    # Standalone script, not imported by bot
├── .env
├── tsconfig.json
└── package.json
```

The key architectural principle: **deploy-commands.ts runs separately** from the bot process. It uses
`@discordjs/rest` directly — no gateway connection — to register slash commands. This keeps the bot
startup fast and avoids accidentally re-registering commands on every restart.

---

## Extended Client class (TypeScript)

```ts
// src/client.ts
import { Client, Collection, GatewayIntentBits, Partials } from 'discord.js';

export interface Command {
  data: { name: string; toJSON(): unknown };
  execute(interaction: ChatInputCommandInteraction): Promise<void>;
  autocomplete?(interaction: AutocompleteInteraction): Promise<void>;
}

export class BotClient extends Client {
  public commands = new Collection<string, Command>();

  constructor() {
    super({
      intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        // Add privileged intents only if needed and enabled in dev portal:
        // GatewayIntentBits.MessageContent,
        // GatewayIntentBits.GuildMembers,
      ],
      partials: [Partials.Channel], // Needed for DM message events
    });
  }
}
```

---

## Command handler

```ts
// src/index.ts (command loading portion)
import { readdirSync } from 'fs';
import { join } from 'path';
import { BotClient } from './client.js';

const client = new BotClient();

// Recursively load commands from all subdirectories
const commandsPath = join(import.meta.dirname, 'commands');

function loadCommands(dir: string) {
  for (const entry of readdirSync(dir, { withFileTypes: true })) {
    const fullPath = join(dir, entry.name);
    if (entry.isDirectory()) {
      loadCommands(fullPath);
    } else if (entry.name.endsWith('.js')) {
      const command = await import(fullPath);
      if ('data' in command && 'execute' in command) {
        client.commands.set(command.data.name, command);
      } else {
        console.warn(`[WARN] ${fullPath} is missing data or execute`);
      }
    }
  }
}

// Note: top-level await requires "type": "module" in package.json
await loadCommands(commandsPath);
```

---

## Event handler

```ts
// src/events/interactionCreate.ts
import type { Interaction } from 'discord.js';
import type { BotClient } from '../client.js';

export default {
  name: 'interactionCreate',
  once: false,
  async execute(interaction: Interaction, client: BotClient) {
    if (interaction.isChatInputCommand()) {
      const command = client.commands.get(interaction.commandName);
      if (!command) return;

      try {
        await command.execute(interaction);
      } catch (error) {
        console.error(`Error executing /${interaction.commandName}:`, error);
        const reply = { content: 'Something went wrong.', ephemeral: true };
        if (interaction.deferred || interaction.replied) {
          await interaction.editReply(reply).catch(() => {});
        } else {
          await interaction.reply(reply).catch(() => {});
        }
      }
    } else if (interaction.isAutocomplete()) {
      const command = client.commands.get(interaction.commandName);
      await command?.autocomplete?.(interaction).catch(console.error);
    }
  },
};
```

```ts
// src/index.ts (event loading portion)
const eventsPath = join(import.meta.dirname, 'events');
for (const file of readdirSync(eventsPath).filter(f => f.endsWith('.js'))) {
  const event = await import(join(eventsPath, file));
  const handler = (...args: unknown[]) => event.default.execute(...args, client);
  if (event.default.once) {
    client.once(event.default.name, handler);
  } else {
    client.on(event.default.name, handler);
  }
}
```

---

## Command file template

```ts
// src/commands/utility/ping.ts
import { SlashCommandBuilder, ChatInputCommandInteraction } from 'discord.js';

export const data = new SlashCommandBuilder()
  .setName('ping')
  .setDescription('Check bot latency');

export async function execute(interaction: ChatInputCommandInteraction) {
  const sent = await interaction.reply({ content: 'Pinging…', fetchReply: true });
  const latency = sent.createdTimestamp - interaction.createdTimestamp;
  await interaction.editReply(`Pong! Roundtrip: **${latency}ms** | API: **${interaction.client.ws.ping}ms**`);
}
```

---

## Deployment script

```ts
// src/deploy-commands.ts
import { REST, Routes } from 'discord.js';
import { readdirSync } from 'fs';
import { join } from 'path';

const { TOKEN, CLIENT_ID, GUILD_ID } = process.env;
if (!TOKEN || !CLIENT_ID) throw new Error('Missing TOKEN or CLIENT_ID');

const rest = new REST({ version: '10' }).setToken(TOKEN);

// Collect command JSON from all command files
const commands: unknown[] = [];
const commandsPath = join(import.meta.dirname, 'commands');

function collectCommands(dir: string) {
  for (const entry of readdirSync(dir, { withFileTypes: true })) {
    const fullPath = join(dir, entry.name);
    if (entry.isDirectory()) {
      collectCommands(fullPath);
    } else if (entry.name.endsWith('.js')) {
      const cmd = await import(fullPath);
      if (cmd.data?.toJSON) commands.push(cmd.data.toJSON());
    }
  }
}
await collectCommands(commandsPath);

// Guild commands → instant (dev). Global commands → up to 1 hour (prod).
const route = GUILD_ID
  ? Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID)
  : Routes.applicationCommands(CLIENT_ID);

await rest.put(route, { body: commands });
console.log(`Registered ${commands.length} commands (${GUILD_ID ? 'guild' : 'global'})`);
```

**Strategy:** Keep `GUILD_ID` set in `.env.development` (instant updates). Clear it for production
builds to register globally. Never use global registration for commands in active development — the
1-hour propagation delay will slow you down considerably.

---

## Sharding

Only shard when approaching **2,500 guilds** (Discord's limit per connection). Before that, sharding
adds complexity for no benefit.

```ts
// src/shard.ts — entry point when sharding
import { ShardingManager } from 'discord.js';

const manager = new ShardingManager('./dist/index.js', {
  token: process.env.TOKEN,
  totalShards: 'auto', // Discord recommends one shard per 1,000 guilds
});

manager.on('shardCreate', shard => {
  console.log(`[Shard ${shard.id}] Spawned`);
});

await manager.spawn();
```

**Accessing cross-shard data:**
```ts
// Sum guild counts across all shards
const counts = await client.shard!.broadcastEval(c => c.guilds.cache.size);
const total = counts.reduce((a, b) => a + b, 0);
```

---

## Environment config

```
# .env
TOKEN=your_bot_token
CLIENT_ID=your_application_id
GUILD_ID=your_test_guild_id   # unset in production

# .env.production (for prod)
TOKEN=prod_token
CLIENT_ID=prod_client_id
# GUILD_ID not set → global registration
```

```ts
// Validate at startup — fail fast is better than mysterious errors later
const required = ['TOKEN', 'CLIENT_ID'];
for (const key of required) {
  if (!process.env[key]) throw new Error(`Missing required env var: ${key}`);
}
```

---

## Graceful shutdown

```ts
async function shutdown() {
  console.log('Shutting down...');
  client.destroy();
  process.exit(0);
}

process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);

// Surface unhandled rejections — don't let the bot die silently
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason);
});
```
