# Claude-Obsidian Implementation Options

Created: 2025-04-20

## Overview

This document outlines options for implementing the [claude-obsidian](https://github.com/AgriciDaniel/claude-obsidian) knowledge wiki pattern with Jimmy (Hermes agent).

---

## What is Claude-Obsidian?

A persistent, compounding knowledge wiki system based on Andrej Karpathy's LLM Wiki pattern. Key features:

- **Automatic ingestion**: Drop sources, AI extracts entities/concepts, creates linked pages
- **Compounding knowledge**: Each source builds on previous ones via cross-references
- **Query with citations**: Ask questions, get answers citing specific wiki pages
- **Session memory**: Hot cache persists between conversations
- **Vault maintenance**: Auto-lint for orphans, dead links, stale claims

---

## Option A: Manual Pattern (Start Now)

**Approach**: Use existing tools with natural language commands.

**How it works**:
- You say: *"Ingest this PDF into my wiki"*
- Jimmy reads the file, extracts key concepts
- Creates structured notes in appropriate folders
- Updates index and cross-references

**Pros**:
- No setup required - works immediately
- Flexible - adapt pattern as needed
- Uses existing Jimmy capabilities (file tools, web search, memory)

**Cons**:
- Less structured than automated system
- Requires explicit instructions each time
- No standardized templates

**Example workflow**:
```
User: "Ingest this paper on transformer architectures"
Jimmy: Reads PDF → Extracts concepts → Creates:
  - wiki/sources/transformer-paper.md
  - wiki/concepts/attention-mechanism.md
  - wiki/concepts/self-attention.md
  - Updates wiki/index.md
```

---

## Option B: Hermes Skill (Recommended)

**Approach**: Create a dedicated Hermes skill implementing the wiki pattern.

**How it works**:
- Skill directory: `~/.hermes/skills/obsidian-wiki/`
- Contains orchestrator logic, templates, and reference docs
- Jimmy loads skill automatically when relevant

**Pros**:
- Standardized workflow across sessions
- Pre-defined templates for entities, concepts, sources
- Can implement all claude-obsidian commands (`/wiki`, `/ingest`, `/lint`)
- Reusable - works in all future sessions

**Cons**:
- Requires initial development time
- Needs testing and refinement
- Must maintain the skill

**Skill structure**:
```
obsidian-wiki/
├── SKILL.md                    # Main orchestrator
├── references/
│   ├── wiki-pattern.md         # Karpathy's LLM Wiki pattern
│   ├── folder-structure.md     # Vault organization
│   ├── templates/
│   │   ├── entity-template.md
│   │   ├── concept-template.md
│   │   └── source-template.md
│   └── lint-rules.md           # Wiki health checks
└── scripts/
    └── ingest.py               # (optional) Helper scripts
```

**Commands to implement**:
| Command | Description |
|---------|-------------|
| `/wiki` | Check vault setup, show status |
| `/ingest [file]` | Read source, extract entities, create pages |
| `/query [topic]` | Search wiki, synthesize answer with citations |
| `/save [name]` | Save current conversation to wiki |
| `/lint` | Health check: orphans, dead links, gaps |
| `/autoresearch [topic]` | Multi-round web research, file findings |

---

## Option C: Clone & Test First

**Approach**: Clone claude-obsidian repo, test with Claude Code, then adapt.

**How it works**:
1. Clone `AgriciDaniel/claude-obsidian` to separate folder
2. Open in Obsidian as second vault
3. Use with Claude Code to experience full workflow
4. Identify which features to port to Hermes

**Pros**:
- See the original system in action
- Understand what works well vs. overhead
- Can borrow ideas, templates, structure
- No commitment - just testing

**Cons**:
- Requires Claude Code (separate from Hermes)
- Two vaults to manage
- Time investment to learn system

**Clone command**:
```bash
git clone https://github.com/AgriciDaniel/claude-obsidian ~/claude-obsidian-test
cd ~/claude-obsidian-test
bash bin/setup-vault.sh
```

---

## Comparison Matrix

| Factor | Option A (Manual) | Option B (Skill) | Option C (Clone First) |
|--------|-------------------|------------------|------------------------|
| Setup time | None | 1-2 hours | 30 minutes |
| Learning curve | Low | Medium | Medium |
| Flexibility | High | Medium (structured) | N/A (testing) |
| Long-term value | Low | High | Medium |
| Works with Jimmy | Yes | Yes | No (uses Claude Code) |
| Maintenance | None | Low | None (temporary) |

---

## My Recommendation

**Short term**: Start with Option A. Tell Jimmy to "ingest this into the wiki" and see how it feels.

**Medium term**: If the pattern proves useful, implement Option B (Hermes skill) for standardized workflow.

**If curious**: Try Option C to see the full claude-obsidian experience, then decide what to adopt.

---

## Technical Notes

### Local Vault Location
- **Path**: `/root/Documents/Obsidian Vault/`
- **GitHub**: github.com/jmolina500/obsidian-vault
- **Sync**: Manual push/pull (or set up Obsidian Git plugin for auto-sync)

### Required Tools
All options use Jimmy's existing capabilities:
- `skill_view` / `obsidian` skill - read vault notes
- `read_file` / `write_file` / `patch` - edit notes
- `web` / `search` - research for autoresearch
- `browser` - fetch web pages
- `terminal` - git operations
- `memory` - persistent preferences

### MCP Server (Optional)
For direct Obsidian integration (closer to claude-obsidian experience):
```bash
# Filesystem-based MCP (no plugin needed)
hermes mcp add-json obsidian-vault '{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@bitbonsai/mcpvault@latest", "/root/Documents/Obsidian Vault"]
}' --scope user
```

This would let Jimmy read/write vault notes via MCP instead of file tools.

---

## Next Steps

1. **Read this document** and consider which approach fits your workflow
2. **Let Jimmy know** your preference
3. **If Option A**: Start with "Jimmy, ingest [file] to my wiki"
4. **If Option B**: Jimmy will create the skill structure
5. **If Option C**: Clone and test, then report back

---

## References

- [claude-obsidian on GitHub](https://github.com/AgriciDaniel/claude-obsidian)
- [Karpathy's LLM Wiki Pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Deep Dive Blog Post](https://agricidaniel.com/blog/claude-obsidian-ai-second-brain)
