# WORKWAY Plugins

> The tool recedes; the outcome remains.

Claude Code plugins for WORKWAY workflow automation.

## Installation

Add to your `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "workway": {
      "source": {
        "source": "github",
        "repo": "WORKWAYCO/workway-plugins"
      }
    }
  },
  "enabledPlugins": {
    "harness@workway": true,
    "design-canon@workway": true,
    "integrations@workway": true,
    "platform@workway": true
  }
}
```

## Available Plugins

### harness

Multi-session autonomous work orchestration via Beads.

**Skills:**
- `harness-spec` - Generate and execute harness specs for autonomous multi-session work

**Agents:**
- `harness` - Autonomous work orchestrator with complexity detection and model routing

**Use when:** Work exceeds a single session, multiple features involved, or you need checkpoints.

---

### design-canon

Heideggerian design philosophy and canon review.

**Skills:**
- `heidegger-design` - Zuhandenheit design principles and Dieter Rams' 10 Principles

**Agents:**
- `canon-reviewer` - Review code/designs against CREATE SOMETHING's Heideggerian canon
- `workflow-designer` - Design compound workflows that embody Zuhandenheit

**Use when:** Making architectural, UX, or naming decisions. Reviewing code for alignment.

---

### integrations

BaseAPIClient patterns for service integrations.

**Skills:**
- `workway-integrations` - Build integrations following the BaseAPIClient pattern

**Agents:**
- `integration-builder` - Build new service integrations (Zoom, Slack, Gmail, etc.)

**Use when:** Creating or modifying service integrations.

---

### platform

WORKWAY platform development patterns.

**Skills:**
- `platform-api-patterns` - Hono/D1/auth patterns for API endpoints
- `database-migrations` - Safe D1 migration patterns with SQLite constraints

**Use when:** Building API endpoints, modifying authentication, or creating database migrations.

## Philosophy

These plugins embody **Zuhandenheit** - the tool should recede into transparent use. You invoke a skill or agent to achieve an outcome; the mechanism disappears.

### Canon

Before every commit, ask:

1. **Zuhandenheit**: Does the tool recede into transparent use?
2. **Weniger, aber besser**: Can anything be removed?
3. **Outcome Test**: Can you describe the value without mentioning technology?

## License

MIT
