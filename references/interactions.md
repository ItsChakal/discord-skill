# Interactions Reference — discord.js v14

## Slash commands

### Basic command with options

```ts
import {
  SlashCommandBuilder, ChatInputCommandInteraction,
  PermissionFlagsBits
} from 'discord.js';

export const data = new SlashCommandBuilder()
  .setName('ban')
  .setDescription('Ban a user from the server')
  .setDefaultMemberPermissions(PermissionFlagsBits.BanMembers)
  .setDMPermission(false)
  .addUserOption(opt =>
    opt.setName('user').setDescription('User to ban').setRequired(true)
  )
  .addStringOption(opt =>
    opt.setName('reason').setDescription('Ban reason').setMaxLength(512)
  )
  .addIntegerOption(opt =>
    opt.setName('delete_days').setDescription('Days of messages to delete')
       .setMinValue(0).setMaxValue(7)
  );

export async function execute(interaction: ChatInputCommandInteraction) {
  const target = interaction.options.getMember('user');
  const reason = interaction.options.getString('reason') ?? 'No reason provided';
  const deleteDays = interaction.options.getInteger('delete_days') ?? 0;

  if (!target) {
    return interaction.reply({ content: 'User not found in this server.', ephemeral: true });
  }

  await interaction.deferReply({ ephemeral: true });

  await (target as GuildMember).ban({ reason, deleteMessageDays: deleteDays });
  await interaction.editReply(`Banned **${target.user?.tag}** — ${reason}`);
}
```

### Subcommands and subcommand groups

```ts
new SlashCommandBuilder()
  .setName('config')
  .setDescription('Bot configuration')
  .addSubcommandGroup(group =>
    group.setName('logging')
         .setDescription('Logging settings')
         .addSubcommand(sub =>
           sub.setName('set-channel')
              .setDescription('Set log channel')
              .addChannelOption(opt => opt.setName('channel').setRequired(true))
         )
         .addSubcommand(sub =>
           sub.setName('disable').setDescription('Disable logging')
         )
  )
  .addSubcommand(sub =>
    sub.setName('reset').setDescription('Reset all settings')
  );

// In execute():
const group = interaction.options.getSubcommandGroup(false); // false = not required
const sub = interaction.options.getSubcommand();

if (group === 'logging') {
  if (sub === 'set-channel') { /* ... */ }
  if (sub === 'disable') { /* ... */ }
} else if (sub === 'reset') { /* ... */ }
```

---

## Buttons

```ts
import {
  ActionRowBuilder, ButtonBuilder, ButtonStyle,
  ComponentType, ButtonInteraction
} from 'discord.js';

// Create the message with buttons
const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
  new ButtonBuilder()
    .setCustomId('confirm')
    .setLabel('Confirm')
    .setStyle(ButtonStyle.Danger)
    .setEmoji('⚠️'),
  new ButtonBuilder()
    .setCustomId('cancel')
    .setLabel('Cancel')
    .setStyle(ButtonStyle.Secondary),
  new ButtonBuilder()
    .setLabel('Discord Docs')
    .setURL('https://discord.js.org')
    .setStyle(ButtonStyle.Link) // Link buttons have no customId
);

const reply = await interaction.reply({
  content: 'Are you sure?',
  components: [row],
  fetchReply: true,
});

// Collect a single button click from the same user
const collector = reply.createMessageComponentCollector({
  componentType: ComponentType.Button,
  filter: i => i.user.id === interaction.user.id,
  time: 30_000,
  max: 1,
});

collector.on('collect', async (btnInteraction: ButtonInteraction) => {
  if (btnInteraction.customId === 'confirm') {
    await btnInteraction.update({ content: 'Confirmed!', components: [] });
    // do the thing
  } else {
    await btnInteraction.update({ content: 'Cancelled.', components: [] });
  }
});

collector.on('end', (_, reason) => {
  if (reason === 'time') {
    // Disable buttons after timeout — don't leave dangling interactive UI
    const disabled = new ActionRowBuilder<ButtonBuilder>().addComponents(
      row.components.map(b =>
        ButtonBuilder.from(b as ButtonComponent).setDisabled(true)
      )
    );
    interaction.editReply({ components: [disabled] }).catch(() => {});
  }
});
```

**Key pattern:** Always disable or remove buttons when the collector ends — leaving active buttons
that will throw "Unknown Interaction" on click is a bad user experience.

---

## Select menus

### String select

```ts
import { StringSelectMenuBuilder, StringSelectMenuOptionBuilder } from 'discord.js';

const menu = new StringSelectMenuBuilder()
  .setCustomId('role_select')
  .setPlaceholder('Pick a role…')
  .setMinValues(1)
  .setMaxValues(3)
  .addOptions(
    new StringSelectMenuOptionBuilder()
      .setLabel('Gaming')
      .setValue('gaming')
      .setEmoji('🎮')
      .setDefault(false),
    new StringSelectMenuOptionBuilder()
      .setLabel('Music')
      .setValue('music')
      .setEmoji('🎵'),
  );

// Handling:
// interaction.isStringSelectMenu() → true
// interaction.values → string[] of selected values
```

### User / Role / Channel / Mentionable selects (v14)

```ts
import { UserSelectMenuBuilder, RoleSelectMenuBuilder } from 'discord.js';

// User select — interaction.users is a Collection<string, User>
new UserSelectMenuBuilder().setCustomId('user_pick').setMaxValues(5);

// Role select — interaction.roles is a Collection<string, Role | APIRole>
new RoleSelectMenuBuilder().setCustomId('role_pick');
```

---

## Modals

```ts
import {
  ModalBuilder, TextInputBuilder, TextInputStyle, ActionRowBuilder,
  ModalSubmitInteraction
} from 'discord.js';

// Show modal (must be the initial response — cannot defer first)
const modal = new ModalBuilder()
  .setCustomId('feedback_modal')
  .setTitle('Share Feedback')
  .addComponents(
    new ActionRowBuilder<TextInputBuilder>().addComponents(
      new TextInputBuilder()
        .setCustomId('subject')
        .setLabel('Subject')
        .setStyle(TextInputStyle.Short)
        .setMaxLength(100)
        .setRequired(true)
    ),
    new ActionRowBuilder<TextInputBuilder>().addComponents(
      new TextInputBuilder()
        .setCustomId('body')
        .setLabel('Details')
        .setStyle(TextInputStyle.Paragraph)
        .setMinLength(10)
        .setMaxLength(2000)
        .setRequired(true)
    )
  );

await interaction.showModal(modal);

// Collect the submission (in the same command handler or a global interactionCreate listener)
const submitted = await interaction
  .awaitModalSubmit({ filter: i => i.customId === 'feedback_modal', time: 300_000 })
  .catch(() => null);

if (!submitted) return; // timed out

const subject = submitted.fields.getTextInputValue('subject');
const body = submitted.fields.getTextInputValue('body');

await submitted.reply({ content: `Feedback received: **${subject}**`, ephemeral: true });
```

**Important:** `showModal()` must be the *initial* response to an interaction. You cannot defer and then show a modal. If you need to fetch data before showing the modal, you must do it synchronously before the 3-second deadline, or restructure your flow.

---

## Autocomplete

```ts
// In the command file — autocomplete is a separate export
export async function autocomplete(interaction: AutocompleteInteraction) {
  const focusedOption = interaction.options.getFocused(true);
  // focusedOption.name = which option is focused, .value = current typed value

  let choices: ApplicationCommandOptionChoiceData[] = [];

  if (focusedOption.name === 'color') {
    const allColors = ['Red', 'Green', 'Blue', 'Purple', 'Orange', 'Yellow'];
    choices = allColors
      .filter(c => c.toLowerCase().startsWith(focusedOption.value.toLowerCase()))
      .slice(0, 25) // Discord max: 25 choices
      .map(c => ({ name: c, value: c.toLowerCase() }));
  }

  await interaction.respond(choices);
}
```

Autocomplete interactions have their own 3-second response window and **cannot** be deferred — respond immediately. Hit a DB or cache, not a slow external API.

---

## Context menus

```ts
import {
  ContextMenuCommandBuilder, ApplicationCommandType,
  UserContextMenuCommandInteraction,
  MessageContextMenuCommandInteraction,
} from 'discord.js';

// User context menu (right-click → Apps → <name>)
export const data = new ContextMenuCommandBuilder()
  .setName('View Profile')
  .setType(ApplicationCommandType.User);

export async function execute(interaction: UserContextMenuCommandInteraction) {
  const target = interaction.targetUser;
  await interaction.reply({ content: `User: ${target.tag}`, ephemeral: true });
}
```

```ts
// Message context menu
export const data = new ContextMenuCommandBuilder()
  .setName('Quote')
  .setType(ApplicationCommandType.Message);

export async function execute(interaction: MessageContextMenuCommandInteraction) {
  const msg = interaction.targetMessage;
  await interaction.reply(`> ${msg.content}\n— ${msg.author.tag}`);
}
```

Context menus are registered the same way as slash commands and show up in the deploy script automatically if they export `data.toJSON()`.

---

## Embeds

```ts
import { EmbedBuilder } from 'discord.js';

const embed = new EmbedBuilder()
  .setColor(0x5865F2) // Discord blurple
  .setAuthor({ name: interaction.user.tag, iconURL: interaction.user.displayAvatarURL() })
  .setTitle('Title (max 256 chars)')
  .setURL('https://example.com')
  .setDescription('Description (max 4096 chars)')
  .addFields(
    { name: 'Inline A', value: 'value', inline: true },
    { name: 'Inline B', value: 'value', inline: true },
    { name: '\u200B', value: '\u200B' }, // blank spacer field
  )
  .setThumbnail('https://example.com/thumb.png')
  .setImage('https://example.com/image.png')
  .setFooter({ text: 'Footer text', iconURL: 'https://example.com/icon.png' })
  .setTimestamp();

await interaction.reply({ embeds: [embed] });
```

**Limits:** 25 fields, 6000 total characters across all text fields, 10 embeds per message.

---

## Collectors — all types

```ts
// Message collector
const msgCollector = channel.createMessageCollector({
  filter: (m) => m.author.id === userId,
  time: 60_000,
  max: 1,
});

msgCollector.on('collect', msg => { /* ... */ });
msgCollector.on('end', (collected, reason) => {
  // reason: 'time' | 'max' | 'user' | 'messageDelete' | etc.
});

// Promise-based (simpler for single response)
const collected = await channel.awaitMessages({
  filter: m => m.author.id === userId,
  max: 1,
  time: 30_000,
  errors: ['time'], // throw on timeout instead of resolving with empty collection
});
```

---

## Permissions

```ts
import { PermissionFlagsBits } from 'discord.js';

// Check if member has permission
if (!interaction.memberPermissions?.has(PermissionFlagsBits.ManageRoles)) {
  return interaction.reply({ content: 'You need Manage Roles.', ephemeral: true });
}

// Check bot permissions in channel
const botPerms = interaction.channel?.permissionsFor(interaction.guild!.members.me!);
if (!botPerms?.has(PermissionFlagsBits.SendMessages)) {
  return interaction.reply({ content: 'I cannot send messages here.', ephemeral: true });
}

// Set on command definition — Discord enforces this server-side
new SlashCommandBuilder()
  .setDefaultMemberPermissions(PermissionFlagsBits.Administrator)
  .setDMPermission(false);
```
