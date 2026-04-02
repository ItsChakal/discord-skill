# Discord expert context — Cursor Skills & Claude app (discord.js v14)

This repository is an **[Agent Skill](https://agentskills.io/specification)** layout: `SKILL.md` + `references/`. **Cursor** and **Claude** load it in different ways:

| Where | What you install | How it works |
|--------|------------------|--------------|
| **Cursor** | A folder under `~/.cursor/skills/` | Cursor discovers `SKILL.md` and uses it in **Agent** mode. |
| **Claude (app)** — **recommended** | The **GitHub Release** asset **`discord.skill`** | A **pre-packaged skill** (ZIP) built for Claude’s native **Skills** uploader—no manual copy/paste. |
| **Claude (app)** — alternative | A **Project** on [claude.ai/projects](https://claude.ai/projects) | **Project instructions** + **project knowledge** files (same content as the repo, but configured by hand). |

The Release file is **not** a separate program: it is this skill **compiled/packaged** into the ZIP layout Anthropic expects (one top-level folder containing `SKILL.md` and `references/`). Upload it once under **Customize → Skills**; Claude loads metadata first, then the body and reference files when the skill is relevant.

---

## Install in Cursor (Agent Skills)

Cursor loads skills from directories that contain **`SKILL.md`** at the root.

### 1. Get the files

```bash
git clone https://github.com/ItsChakal/discord-skill.git
cd discord-skill
```

Repository: **[github.com/ItsChakal/discord-skill](https://github.com/ItsChakal/discord-skill)** (use your fork’s URL if you contribute via a fork).

### 2. Put the skill where Cursor expects it

**Symlink** (good if you keep editing the git clone):

```bash
ln -s "$(pwd)" ~/.cursor/skills/discord-expert
```

**Copy** (standalone):

```bash
mkdir -p ~/.cursor/skills
cp -R . ~/.cursor/skills/discord-expert
```

The folder name (`discord-expert`) is your choice. Required layout:

```text
~/.cursor/skills/discord-expert/
├── SKILL.md
└── references/
    ├── architecture.md
    ├── interactions.md
    ├── advanced.md
    └── gotchas.md
```

### 3. Use it in Cursor

1. Open any workspace in Cursor.
2. Open **Chat** or **Composer** and switch to **Agent** (not Ask-only).
3. Work on Discord / `discord.js` topics; Cursor may pick up the skill from the `description` in the `SKILL.md` YAML frontmatter. You can also add user **Rules** (Cursor **Settings → Rules**) such as: *Always use the Discord skill in `~/.cursor/skills/discord-expert` when editing the bot.*

Restart Cursor if the skill does not appear after a fresh install.

### Optional: Claude as the model inside Cursor

If your Cursor plan includes **Anthropic** models, choose a **Claude** model in the chat/composer model picker. **You do not install anything extra for Claude**—the same `~/.cursor/skills/...` folder applies to every model. Use **Agent** mode so the skill can drive tools and file edits.

---

## Install in the Claude app (Anthropic)

Claude does **not** read `~/.cursor/skills/`. You can either install the **packaged release** (native Skills) or recreate the content in a **Project**.

Skills on Claude require **[code execution to be enabled](https://support.anthropic.com/en/articles/12111783-create-and-edit-files-with-claude)** for your account. See also: [What are Skills?](https://support.anthropic.com/en/articles/12512176-what-are-skills), [Use Skills in Claude](https://support.anthropic.com/en/articles/12512180-use-skills-in-claude).

---

### A. Native Skill — GitHub Release `discord.skill` (recommended)

Each **[GitHub Release](https://github.com/ItsChakal/discord-skill/releases)** ships a **`discord.skill`** asset: a **ZIP file** in Anthropic’s packaged-skill format (folder + `SKILL.md` + `references/`). Think of it as the same repo **bundled for Claude**—not a different product, just a **ready-to-upload** build.

1. Open the repo’s **Releases** page and download **`discord.skill`** (or `discord.skill.zip`, depending on how the release is named).
2. In Claude (web: **[Customize → Skills](https://claude.ai/customize/skills)**), **upload** that file and **enable** the skill.
3. Chat as usual; Claude decides when to invoke the skill from the `description` in `SKILL.md` (progressive disclosure: metadata first, then full instructions and reference files when relevant).

Packaging rules follow Anthropic’s guide: [How to create custom Skills](https://support.anthropic.com/en/articles/12512198-how-to-create-custom-skills) (ZIP must contain **one top-level folder**; inside it, `SKILL.md` and optional `references/`, etc.). The open format is described at **[agentskills.io](https://agentskills.io/specification)**.

**Note:** On Claude.ai, the skill **`description` in YAML is limited to 200 characters** (stricter than Cursor). The Release build may ship a **shortened description** in `SKILL.md` so upload validation succeeds; the clone from `main` stays optimized for Cursor’s longer description until you align them.

---

### B. Manual setup — Claude Project (no Release file)

If you do not use the Release asset, mirror the repo with a **Project**.

Official overview: [How to create and manage projects](https://support.anthropic.com/en/articles/9519177-how-can-i-create-and-manage-projects).

1. Open **[claude.ai/projects](https://claude.ai/projects)** (or **Projects** in the Claude sidebar on web / desktop / mobile).
2. **+ New Project** → name it (e.g. *Discord bot — discord.js v14*).
3. **Set project instructions** → paste the body of `SKILL.md` (everything below the YAML `---` block), or the whole file.
4. **Project knowledge** → use **+** and upload:

   - `references/architecture.md`
   - `references/interactions.md`
   - `references/advanced.md`
   - `references/gotchas.md`  
   Optionally add **`SKILL.md`** as well.

5. Start **new chats inside that project** when working on your bot.

### C. Optional: global custom instructions

If your account offers global custom instructions, you can paste a **short** subset of the skill there. For this repo’s size, **A (Release)** or **B (Project)** is usually better.

---

### Maintainer: building the `discord.skill` Release asset

From a clean checkout, the ZIP must look like this (Anthropic expects **the folder as the root of the archive**, not loose files):

```text
discord.skill   ← ZIP file
└── discord/      ← single top-level folder; name must match `name:` in SKILL.md frontmatter
    ├── SKILL.md
    └── references/
        ├── architecture.md
        ├── interactions.md
        ├── advanced.md
        └── gotchas.md
```

Example (adjust paths if your clone lives elsewhere):

```bash
# From repo root: create inner folder matching frontmatter name: discord
rm -rf /tmp/discord-skill-build && mkdir -p /tmp/discord-skill-build/discord
cp SKILL.md /tmp/discord-skill-build/discord/
cp -R references /tmp/discord-skill-build/discord/
cd /tmp/discord-skill-build && zip -r discord.skill discord
# Upload discord.skill to GitHub Releases
```

If you shorten `description` for Claude’s 200-character limit, edit `SKILL.md` in the build folder **before** `zip`.

---

## Cursor: project-level skill (team)

To ship the skill **inside an app repository** for teammates using Cursor:

```bash
mkdir -p .cursor/skills
cp -R /path/to/this/repo .cursor/skills/discord-expert
```

Commit `.cursor/skills/` in your application repo.

---

## Contents of this repository

| Item | Purpose |
|------|---------|
| `SKILL.md` | Main instructions + YAML frontmatter (Cursor) |
| `references/architecture.md` | Project layout, handlers, deploy, sharding, env |
| `references/interactions.md` | Slash commands, buttons, modals, collectors, permissions |
| `references/advanced.md` | Threads, forums, voice, automod, webhooks, etc. |
| `references/gotchas.md` | Rate limits, intents, “Unknown Interaction”, etc. |

## License

Add your chosen license here once the repository is public.
