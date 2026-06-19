# Artemis Cosmos Mission Scripting Guide

Community resources for writing missions in Artemis Cosmos using MAST and PyMAST.

- **[SKILLS.md](SKILLS.md)** — Practical guide for mission authors: syntax, patterns, addons, GUI, PyMAST, and more.
- **[mission_patterns.md](mission_patterns.md)** — Patterns and techniques learned from studying real missions (HereThereBeMonsters, Infinite_Cosmos, WalkTheLine, Lucky_13, theta_quadrant, MiningDays, modding_tools, remote_mission_pick).

---

## Using this guide with an AI assistant (Claude, ChatGPT, etc.)

MAST is a niche language that AI assistants don't know out of the box. Giving the AI `SKILLS.md` as context at the start of a conversation lets it generate correct MAST syntax, use the right patterns, and avoid confusing Cosmos with the old Artemis XML scripting format.

### With Claude.ai or ChatGPT (web chat)

1. Open a new conversation.
2. Paste the full contents of `SKILLS.md` as your first message with a brief framing:
   > *"I'm writing an Artemis Cosmos mission. Use this reference guide for MAST syntax and patterns: [paste SKILLS.md contents]"*
3. Describe the mission you want and ask for help.

You can also paste the raw GitHub URL and ask the AI to fetch it directly — though pasting the content is more reliable across all interfaces.

### With Claude Code (CLI / IDE extension)

1. Drop `SKILLS.md` into your mission folder.
2. Create a `CLAUDE.md` file in the same folder that references it:
   ```
   @SKILLS.md
   ```
3. Claude Code will automatically include it as context for every conversation in that project — no pasting needed.

### What this prevents

Without the guide, AI assistants tend to invent plausible-looking but incorrect MAST syntax, mix up `shared`/`default`/`->END` semantics, or write XML-style scripting from the old Artemis 2.x game. The guide anchors the AI to how Cosmos actually works.
