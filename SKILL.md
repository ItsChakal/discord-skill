---
name: discord
description: >
  Expert-level Discord bot development with discord.js v14, the Discord REST API, and bot architecture.
  Use this skill whenever the user is building, debugging, or architecting a Discord bot or integration.
  Triggers on: discord.js, slash commands, interactions, embeds, buttons, modals, select menus, autocomplete,
  context menus, collectors, voice channels, threads, forum channels, automod, sharding, webhooks,
  Discord OAuth2, gateway intents, bot deployment, command registration, rate limits, GatewayIntentBits,
  ActionRowBuilder, EmbedBuilder, SlashCommandBuilder, ModalBuilder, @discordjs/voice, @discordjs/rest —
  anything touching Discord development. Use this skill even if the user just mentions "my bot" or "Discord server".
---

# Discord Expert Skill

You are working with an expert Discord developer. Skip the basics — assume they know JavaScript/TypeScript
and the overall Discord model. Focus on precision: correct API shapes, current v14 patterns, architectural
guidance, and sharp debugging.

## What to do when this skill activates

1. **Read the user's request carefully.** Identify whether it's a new feature, a debugging session,
   an architecture question, or a deployment/ops problem.
2. **Pull in the right reference file(s)** from `references/` based on what you need (see index below).
   Don't load all of them — just the relevant one(s).
3. **Produce production-quality code** with proper error handling, TypeScript types where applicable,
   and clear inline comments explaining non-obvious design choices.
4. **Flag gotchas proactively.** If the user's approach will hit a rate limit, a missing intent,
   or a known interaction timing issue — say so before they hit it.

## Reference file index

| File | When to read it |
|------|-----------------|
| `references/architecture.md` | Project structure, command/event handlers, deploy scripts, sharding, env config, TypeScript client setup |
| `references/interactions.md` | Slash commands, buttons, selects, modals, autocomplete, context menus, collectors, permission flags |
| `references/advanced.md` | Threads, forum channels, voice (`@discordjs/voice`), automod, scheduled events, entitlements, webhooks |
| `references/gotchas.md` | "Unknown Interaction", rate limits, missing intents, caching issues, ephemeral quirks, graceful shutdown |

## Quick orientation — discord.js v14 essentials

**All enums are now from `discord-api-types` and are PascalCase singular:**
```ts
import {
  GatewayIntentBits, Partials, ButtonStyle, ChannelType,
  ApplicationCommandType, TextInputStyle
} from 'discord.js'; // re-exported from discord-api-types
```

**Builder names changed from v13:**
| v13 | v14 |
|-----|-----|
| `MessageEmbed` | `EmbedBuilder` |
| `MessageButton` | `ButtonBuilder` |
| `MessageSelectMenu` | `StringSelectMenuBuilder` |
| `MessageActionRow` | `ActionRowBuilder` |
| `Intents.FLAGS.X` | `GatewayIntentBits.X` |

**Critical interaction timing:**
- Initial response: **3 seconds** — must call `reply()`, `deferReply()`, or `showModal()`
- After deferral: **15 minutes** for `editReply()` / `followUp()`
- Exceeding these windows → "Unknown Interaction" (Discord error code 10062)

**Privileged intents** (require Developer Portal toggle + verification for large bots):
- `GatewayIntentBits.GuildMembers`
- `GatewayIntentBits.GuildPresences`
- `GatewayIntentBits.MessageContent`

## General coding standards for this skill

- Use `async/await` throughout; never `.then()` chains in bot code
- Always `await` interaction methods — missing `await` causes silent "already responded" errors
- Wrap command execute handlers in try/catch; on error, attempt `editReply` if already deferred
- Use `Collection<string, T>` (not plain `Map`) for command/event stores — it's discord.js's extended Map
- Never store sensitive data (tokens, secrets) anywhere except environment variables
- Prefer `interaction.options.get('name', true)` (required: true) over non-null assertions

## How to present code

- Provide complete, runnable files (not fragments) unless the user asks for a snippet
- When writing TypeScript, use `satisfies` for command objects to get type safety without widening
- Add a brief explanation after each significant file: what it does and any decisions the user should be aware of
- If multiple approaches are valid, briefly note the trade-offs and make a recommendation
