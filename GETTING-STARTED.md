# Getting Started

A step-by-step setup guide for `/build-product`. This guide assumes no prior experience with Claude Code, terminals, Git, or MCP integrations. Plan for 30 to 60 minutes the first time, mostly waiting for installs to complete.

If you already use Claude Code and have downloaded other skills before, skip to [Step 4 — Download the skill](#step-4--download-the-skill).

---

## Table of Contents

1. [What you need before you begin](#what-you-need-before-you-begin)
2. [Step 1 — Install Claude Code](#step-1--install-claude-code)
3. [Step 2 — A primer on using Claude Code](#step-2--a-primer-on-using-claude-code)
4. [Step 3 — Install Git](#step-3--install-git)
5. [Step 4 — Download the skill](#step-4--download-the-skill)
6. [Step 5 — Register the slash commands](#step-5--register-the-slash-commands)
7. [Step 6 — Connect your team's tools (MCPs)](#step-6--connect-your-teams-tools-mcps)
8. [Step 7 — Your first run](#step-7--your-first-run)
9. [Updating to the latest version](#updating-to-the-latest-version)
10. [If something goes wrong](#if-something-goes-wrong)
11. [Glossary](#glossary)

---

## What you need before you begin

Five things. Read through these first; the rest of this guide depends on them.

### A computer

Mac, Windows, or Linux. Any modern machine is fine.

### A Claude account on the Pro or Team plan

The free Claude plan does not include Claude Code, which this skill runs inside.

- Sign up at [https://claude.ai](https://claude.ai).
- Pro is $20 per month per user.
- Team plans are available for organizations.

### Claude Code installed

Claude Code is Anthropic's terminal-based version of Claude. It runs on your computer and can read files, write files, and run commands. That is how it produces a PRD on your disk and pushes tickets to Jira.

Claude Code is **not** the same as the claude.ai website. The website cannot run this skill. Install instructions are in [Step 1](#step-1--install-claude-code).

### Git installed

Git is a tool for downloading code from GitHub. You need it only to download this skill the first time, and to receive updates later.

You do not need a GitHub account to use Git for this purpose. Cloning a public repository works without an account. Install instructions are in [Step 3](#step-3--install-git).

### Access to your team's Jira

The skill creates Jira tickets under your account. You need:

- A login to your team's Jira (Cloud version, hosted at `*.atlassian.net`).
- Permission to create tickets in at least one project.
- For Confluence publishing (optional): permission to create or edit pages in at least one Confluence space.

If you do not know whether you have these permissions, the simplest test is to manually create a single Jira ticket in your team's project. If that works, you have the permissions you need for this skill.

> **A note on admin access.** Connecting Claude Code to your team's Atlassian workspace (Jira and Confluence) goes through an OAuth flow under your own account. Most teams allow this. Some companies require an IT administrator to pre-approve third-party integrations at the workspace level. If your first connection attempt fails with a permission error, contact your IT or Atlassian administrator and ask them to allow the Atlassian Remote MCP for your workspace.

---

## Step 1 — Install Claude Code

Follow the official installation guide at [https://docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code).

After installation, open your terminal (see [Step 2](#step-2--a-primer-on-using-claude-code) below for how to find it) and type:

```
claude --version
```

Then press Enter. If you see a version number (for example, `0.5.x`), Claude Code is installed correctly.

If you see `command not found` or a similar error, the install did not complete. Try restarting your terminal first. If that does not help, re-run the installer.

---

## Step 2 — A primer on using Claude Code

If you have not used a terminal before, this section explains the essentials. If you already use Claude Code, you can skip to [Step 3](#step-3--install-git).

### What a terminal is

A terminal is a text-based interface for giving your computer instructions. Instead of clicking icons, you type commands. When this guide says "type X and press Enter," you are typing the command into a terminal window.

### How to open your terminal

**On a Mac:** Press `Cmd + Space` to open Spotlight, type `Terminal`, then press Enter. A small window opens with a prompt that looks like `yourname@yourmachine ~ %`.

**On Windows:** Press `Win + R`, type `powershell`, then press Enter. A blue or black window opens.

**On Linux:** Open Terminal from your application menu, or press `Ctrl + Alt + T`.

The text ending in `%` (Mac) or `>` (Windows) is the prompt. You type your commands after the prompt. Do not type the prompt itself.

### Starting Claude Code

In the terminal, you can be in any folder when you start Claude Code. The home folder is fine for now. Type:

```
claude
```

Then press Enter. Claude Code starts. You will see a welcome message and a new prompt. That new prompt is where you type messages to Claude.

### Using slash commands

Inside Claude Code, typing `/` shows a list of available commands. This skill adds commands like `/build-product`, `/research-idea`, `/feature-kickoff`, and others.

To run a command:

1. Type `/`.
2. Keep typing the command name (for example, `/build-product`) or use the arrow keys to select it from the list.
3. Press Enter.

The skill takes over from there and walks you through questions.

### Exiting Claude Code

Type `/exit` and press Enter. Alternatively, press `Ctrl + C` twice.

---

## Step 3 — Install Git

Git is a one-time install per computer. You do not need a GitHub account.

### On a Mac

1. Open your terminal.
2. Type `git --version` and press Enter.
3. If a dialog appears offering to install developer tools, click Install. The install takes about 10 minutes.
4. When the install completes, type `git --version` again. You should see output like `git version 2.x.x`.

### On Windows

1. Download the installer from [https://git-scm.com/download/win](https://git-scm.com/download/win).
2. Run the installer. Accept the default options on every screen.
3. After installation, close any open PowerShell windows and open a fresh one.
4. Type `git --version`. You should see a version number.

### On Linux

- Debian or Ubuntu: `sudo apt install git`
- Fedora: `sudo dnf install git`
- Arch: `sudo pacman -S git`

---

## Step 4 — Download the skill

In your terminal (outside of Claude Code), paste the following command and press Enter:

```bash
git clone https://github.com/judy-eapen/build-product-skill.git ~/.claude/skills/build-product
```

You will see output similar to:

```
Cloning into '/Users/yourname/.claude/skills/build-product'...
remote: Enumerating objects: 200, done.
...
Resolving deltas: 100% (123/123), done.
```

The skill is now installed at `~/.claude/skills/build-product`. The `~` is shorthand for your home folder.

---

## Step 5 — Register the slash commands

Claude Code discovers slash commands in a specific folder (`~/.claude/commands/`). This one-time setup copies a small wrapper file for each command in this skill into that folder.

Paste this command (it is one long line) and press Enter:

```bash
mkdir -p ~/.claude/commands && for f in ~/.claude/skills/build-product/subprompts/*.md; do n=$(basename "$f" .md); printf 'Read and follow `~/.claude/skills/build-product/subprompts/%s.md`.\n' "$n" > ~/.claude/commands/"$n.md"; done
```

If the prompt returns without any error message, the registration worked. To verify, type:

```bash
ls ~/.claude/commands/
```

You should see a list of `.md` files including `build-product.md`, `research-idea.md`, `change-mode.md`, and others.

Now start Claude Code by typing `claude` and pressing Enter. Inside Claude Code, type `/`. The slash commands list should include `/build-product`.

---

## Step 6 — Connect your team's tools (MCPs)

### What an MCP is

An **MCP** (Model Context Protocol) is a connector that lets Claude Code talk to another service such as Jira, Confluence, Figma, Google Drive, or Slack. Each MCP you install adds a capability to Claude Code. The skill automatically skips any tool it cannot reach, so you only need to connect the ones you intend to use.

Connecting an MCP is a one-time setup per tool. The general flow is:

1. Add an entry to Claude Code's MCP configuration. The official guide is at [https://docs.claude.com/en/docs/claude-code/mcp](https://docs.claude.com/en/docs/claude-code/mcp).
2. Restart Claude Code so the new MCP is recognized.
3. The first time the skill calls the tool, Claude Code opens an OAuth flow in your browser. You log in with your existing account for that service to authorize Claude Code to act on your behalf.

### What each MCP is used for in this skill

| MCP | Used for | Required? |
|-----|----------|-----------|
| **Atlassian** (Jira + Confluence) | Step 11a creates Jira tickets. Step 11c optionally publishes a PRD page to Confluence. The `/share-for-review` and `/read-feedback` commands also use this MCP. | Jira: required for ticket export. Confluence: optional. |
| **Figma** | Step 7 generates a visual diagram as a Figma FigJam board. | Optional. The skill falls back to Mermaid syntax if Figma is not connected. |
| **Google Drive** | Step 11b optionally mirrors every artifact to a Drive folder. | Optional. |
| **Slack** | The `/share-for-review` command posts a Confluence page link to a Slack channel with reviewers tagged. | Optional. |

### Recommended order

Connect the Atlassian MCP first. That unlocks the primary value of this skill: real Jira tickets created from your approved breakdown. Add the other MCPs later as you decide you want them.

### Common OAuth failures

If the OAuth flow returns "permission denied" or "unauthorized":

1. Confirm you can log in to the service manually with the same account.
2. Check with your IT or workspace administrator that the integration is allowed at the workspace level. Atlassian administrators can disable third-party app access by policy; if that is your situation, ask the administrator to allow the Atlassian Remote MCP for your workspace.

---

## Step 7 — Your first run

In your terminal, navigate to any folder. Your home folder is fine:

```bash
cd ~
```

Start Claude Code:

```bash
claude
```

Inside Claude Code, type:

```
/build-product
```

Either keep typing to filter the list, or use the arrow keys to select `/build-product`. Press Enter.

### The questions you will be asked

The skill walks through these in order:

1. **Mode.** Choose `Fast mode` for your first run. Fast mode chains all automatic steps and pauses only at the three approval gates. Gated mode pauses after every step, which is useful later when you want to review each output more carefully.
2. **Feature name.** Whatever this feature is called. The skill converts spaces to hyphens and lowercases the name to derive a folder name (for example, `Customer Onboarding Flow` becomes `customer-onboarding-flow`).
3. **Seven intake questions.** The skill asks:
   1. Feature name (confirmed from Mode step)
   2. Jira project name (the project where tickets should be created)
   3. **Jira ticket conventions** (open-ended; describe labels, title formats, BE/FE split rules, custom field defaults, fields to leave blank, link conventions)
   4. Tech stack
   5. Product type (web app, mobile app, backend service, internal tool, AI feature, other)
   6. Whether the feature has a permission model (Yes / No / Not yet decided)
   7. Whether the feature has a backend or API surface (Yes / No / Not yet decided)
4. **The pipeline runs.** Research, codebase review, PRD generation, dual AI review, then Gate 1.

### What an approval gate looks like

When the pipeline pauses at a gate, you see a block like this:

```
━━━ APPROVAL NEEDED: Gate 1 — PRD ━━━

What was produced:
- PRD: ~/Desktop/Resources/PDLC Workflow Docs/[feature]/prd/[feature]-prd.md

━━━ QUALITY CHECK ━━━
Quality check passed. No issues found.

Progress:
[x] Step 1 — Research
[x] Step 2 — Codebase Review
[x] Step 3 — Create PRD
[x] Step 4 — Dual Review
[x] Step 5 — Apply Fixes
[ ] Step 6 — System Design ← next after approval

Say "approved" to continue, or give feedback to revise.
━━━
```

Read the produced artifact, then either:

- Type `approved` and press Enter to continue.
- Type your feedback in plain English (for example, "The data model is missing a soft-delete column. Add it and re-run from the data model section onward."). The skill will revise.

There are three gates: PRD (Gate 1), Designs (Gate 2), and User Stories (Gate 3). Real Jira tickets are not created until you approve all three.

### How long the full pipeline takes

For a small feature, expect 30 to 60 minutes. Larger features can take longer because of more rounds of revision at each gate.

You can stop and resume at any time. The skill writes a state file (`_pipeline-state.json`) after every step. On your next Claude Code session, it offers to resume from where you left off.

---

## Updating to the latest version

The skill is maintained as a GitHub repository. To pick up the latest changes (new commands, bug fixes, improved prompts), paste the following one-liner into your terminal and press Enter:

```bash
cd ~/.claude/skills/build-product && git pull && mkdir -p ~/.claude/commands && for f in subprompts/*.md; do n=$(basename "$f" .md); printf 'Read and follow `~/.claude/skills/build-product/subprompts/%s.md`.\n' "$n" > ~/.claude/commands/"$n.md"; done
```

This does two things in sequence:

1. **Pulls the latest commits from GitHub** into your local skill folder.
2. **Re-registers all slash commands** so any new commands added in this version become available the next time you start Claude Code.

After running the one-liner, restart Claude Code so it picks up the changes.

### What if `git pull` says my files are modified?

If you have edited any skill files locally (for example, customized `style-preferences.md` or any of the subprompts), Git may refuse to pull because it does not want to overwrite your changes. Two options:

- **Discard your local changes** and take the upstream version: `git stash && git pull`. Your local edits are saved in a stash you can restore later if needed.
- **Keep your local changes and merge upstream on top**: `git pull --rebase`. Git replays your local edits on top of the latest upstream commits. If there are conflicts, Git will pause and ask you to resolve them.

If you are not sure which to use, ask before running either.

### Seeing what changed in this update

Every update is documented in [CHANGELOG.md](./CHANGELOG.md) in the repository. Skim the top entry to see what is new before running through `/build-product` again — especially if a new intake question or new pipeline behavior was added.

### Recommended cadence

Check for updates roughly once a week, or any time you start a new feature. Updates are usually small and non-breaking. Major version bumps (e.g., v2.0.0 → v3.0.0) will be called out in CHANGELOG.md with an explicit "Breaking changes" section.

---

## If something goes wrong

| Symptom | What it usually means | What to try |
|---------|----------------------|-------------|
| `/build-product` does not appear in the slash command list when you type `/` in Claude Code | The slash command registration in Step 5 did not run, or it ran before Claude Code was installed | Re-run the Step 5 command. Restart Claude Code. |
| `command not found` when you type `git` or `claude` in the terminal | The tool is not installed, or your terminal session was opened before the install finished | Close the terminal window and open a fresh one. Re-run `git --version` or `claude --version`. |
| The Atlassian OAuth flow returns "permission denied" | Your IT or Atlassian administrator has not approved the Atlassian Remote MCP for your workspace | Contact your administrator and ask them to allow the integration. |
| The skill says an artifact is missing when you resume a session | A prior session crashed mid-step, leaving the state file out of sync with the artifacts on disk | The skill warns you about which step's output is missing. Re-run that specific step. |
| Mermaid diagrams render as "Error loading the extension" on a Confluence page | The Confluence space does not have the Mermaid plugin installed | Connect the Figma MCP. The skill will produce FigJam boards instead, which Confluence renders natively. |
| Jira tickets are created with the wrong labels or in the wrong format | Intake question #3 (Jira ticket conventions) was skipped or under-answered | Re-run the export with the conventions corrected. For an existing run, use `/change-mode` to propagate the corrected conventions. |

If you cannot resolve an issue from this table, file a question or bug report on the GitHub issues page: [https://github.com/judy-eapen/build-product-skill/issues](https://github.com/judy-eapen/build-product-skill/issues).

---

## Glossary

| Term | Meaning |
|------|---------|
| **Claude Code** | Anthropic's terminal-based version of Claude. Runs on your computer, reads and writes files, and runs commands. It is not the same as the claude.ai website. |
| **Terminal** | A text-based interface for giving your computer instructions. On a Mac, open via Spotlight by typing "Terminal." On Windows, use PowerShell. |
| **Git** | A tool for downloading code from GitHub. Used in this guide only to download the skill. No GitHub account is required for that. |
| **Skill** | A reusable set of instructions for Claude Code. This repository is one skill. Skills install into `~/.claude/skills/[skill-name]/`. |
| **Slash command** | A command you type in Claude Code that begins with `/`. Each subprompt in this skill is exposed as a slash command (for example, `/build-product`, `/research-idea`). |
| **MCP (Model Context Protocol)** | A connector that lets Claude Code talk to an external service such as Jira or Figma. One MCP per service. |
| **Pipeline** | The sequence of steps `/build-product` runs: research, codebase review, PRD, review, design, user stories, Jira export. |
| **Gate** | A pause in the pipeline where you approve before the work continues. There are three: PRD, Designs, User Stories. |
| **PRD** | Product Requirements Document. The primary artifact this skill produces. |
| **Gherkin** | A structured format for acceptance criteria using `Given / When / Then`. Used in user story acceptance criteria. |
| **FE / BE** | Front-End and Back-End. The skill produces separate tickets for each layer by default. The behavior is configurable at intake. |
| **Epic** | A Jira ticket type that groups related Story tickets under one umbrella. The skill creates one Epic per feature plus one Story per user story in the breakdown. |
| **FigJam** | Figma's whiteboard product. The skill uses it for visual diagrams (architecture, user journeys, per-flow diagrams). |
| **OAuth** | An authorization standard. The first time you connect an MCP to a service, your browser opens to log you in to that service and authorize Claude Code to act on your behalf. |

---

## Next steps

After your first successful run:

- Read [README.md](./README.md) for the command reference and architecture notes.
- Read [CHANGELOG.md](./CHANGELOG.md) to see what has changed in recent versions.
- Run `/team-status` periodically to see all features in your portfolio at once.
- If your team's Jira conventions change, update them at intake question #3 on your next `/build-product` run.
