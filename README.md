# Hathor Skills

Official Claude Code skills for [Hathor Network](https://hathor.network) development.

## Available Skills

<<<<<<< Updated upstream
- **hathor-blueprints** - Specialist skill for creating Hathor blockchain blueprints (nano contracts)
=======
| Skill | Description |
|-------|-------------|
| **hathor-blueprint** | Create Hathor blockchain blueprints (nano contracts) - Python 3.11 smart contracts |

## Marketplace

This repository includes a `marketplace.json` file that describes all available skills. This can be used for automated skill discovery and installation.

## Installation

### Personal Installation (Recommended)

Install the skill for use across all your Claude Code sessions:

```bash
mkdir -p ~/.claude/skills
cp -r hathor-blueprints ~/.claude/skills/hathor-blueprint
```

Note: The directory name should match the `name` field in SKILL.md (which is `hathor-blueprint`, not `hathor-blueprints`).

After installation, the skill will be automatically available in all new Claude Code sessions.

### Project-Level Installation

To make the skill available to your team in a specific repository:

```bash
mkdir -p .claude/skills
cp -r hathor-blueprints .claude/skills/hathor-blueprint
git add .claude/skills/hathor-blueprint/
git commit -m "Add hathor-blueprint skill"
```

Team members will have access to the skill when working in this repository.

### Folder-Level Installation (Multiple Projects)

To share the skill across multiple projects in a folder without installing globally, use symlinks:

```bash
# From this repository location, create a symlink in each project
cd /path/to/your/project1
ln -s ~/hathor-skills/hathor-blueprints .claude/skills/hathor-blueprint

cd /path/to/your/project2
ln -s ~/hathor-skills/hathor-blueprints .claude/skills/hathor-blueprint
```

## Using the Skills

### hathor-blueprint

The skill activates automatically when you:
- Ask to build/debug Hathor blueprints
- Mention nano contracts or Hathor smart contracts
- Reference Hathor blockchain development

Or invoke it explicitly:
```
/hathor-blueprint
```

## How It Works

- **Automatic discovery**: Claude Code loads skills from `~/.claude/skills/` (personal) or `.claude/skills/` (project) at startup
- **Persistent**: Personal skills are available in every new session automatically
- **Smart triggering**: Claude uses the skill when you mention relevant keywords (blueprints, nano contracts, Hathor)
>>>>>>> Stashed changes
