# Plone REST API Skills

A Claude Code plugin that provides comprehensive skills for working with Plone REST API - both backend development (Python/Plone) and client development (JavaScript/TypeScript/Python).

## Installation

### Install from GitHub

Add the marketplace and install the plugin:

```bash
claude plugin marketplace add https://github.com/MrTango/plone-restapi-skills
claude plugin install plone-restapi-skills
```

Restart Claude Code to load the plugin.

### Verify Installation

Check installed plugins:

```bash
claude plugin list
```

Or within Claude Code, use the `/plugins` command.

### Update

Update the plugin to the latest version:

```bash
claude plugin update plone-restapi-skills
```

### Uninstall

```bash
claude plugin uninstall plone-restapi-skills
```

## Available Skills

This plugin provides two skills that are **automatically activated** when Claude Code detects relevant context in your conversation.

### plone-restapi-backend

**Purpose:** Develop Plone REST API backend services in Python

**Auto-triggers on:**
- Creating custom `@`-endpoints
- Writing REST services in Python
- ZCML service registrations
- Content serialization/deserialization
- Extending plone.restapi

**Features:**
- Service class patterns (`plone.restapi.services.Service`)
- ZCML registration examples
- Custom serializers (`ISerializeToJson`, `ISerializeToJsonSummary`)
- Custom deserializers (`IDeserializeFromJson`)
- Expandable elements for on-demand data
- Batching with `HypermediaBatch`
- Error handling with proper HTTP status codes
- Workflow integration
- File and image handling
- Vocabulary services
- Testing patterns

### plone-restapi-client

**Purpose:** Consume Plone REST API from JavaScript/TypeScript and Python clients

**Auto-triggers on:**
- REST API client code
- `@navigation`, `@contextnavigation`, `@breadcrumbs` endpoints
- fetch/axios/requests with Plone
- Frontend integration with Plone

**Features:**
- Complete endpoint reference table
- JavaScript client (Fetch API, Axios)
- TypeScript client with full typing
- Python client (requests, aiohttp async)
- React hooks and components
- Vue.js composables
- Svelte stores
- Navigation and breadcrumb handling
- Common patterns (pagination, error handling, batch operations)

## How It Works

Once installed, the skills are **automatically loaded** when Claude Code detects relevant keywords or patterns in your conversation. You don't need to manually invoke them.

For example, when you ask:
- "How do I create a custom REST API endpoint in Plone?" - triggers `plone-restapi-backend`
- "How do I fetch navigation from Plone REST API?" - triggers `plone-restapi-client`
- "Create a Python client for Plone REST API" - triggers `plone-restapi-client`

## Endpoint Coverage

The client skill covers all major Plone REST API endpoints:

| Category | Endpoints |
|----------|-----------|
| Authentication | `@login`, `@logout`, `@users`, `@groups`, `@principals`, `@roles` |
| Content | CRUD operations, `@copy`, `@move`, `@workflow`, `@history`, `@lock` |
| Navigation | `@navigation`, `@contextnavigation`, `@breadcrumbs`, `@navroot` |
| Search | `@search`, `@querystring-search`, `@querystring` |
| Comments | `@comments` (list, add, reply, update, delete) |
| Relations | `@relations`, `@aliases` |
| Sharing | `@sharing`, `@linkintegrity` |
| Translations | `@translations`, `@translation-locator` |
| Vocabularies | `@vocabularies`, `@sources`, `@querysources`, `@types` |
| Administration | `@addons`, `@controlpanels`, `@registry`, `@system`, `@database` |

## Development

### Plugin Structure

```
.claude-plugin/
├── plugin.json                              # Plugin manifest
└── skills/
    ├── plone-restapi-backend/
    │   ├── SKILL.md                         # Backend skill definition
    │   └── reference.md                     # Advanced examples
    └── plone-restapi-client/
        ├── SKILL.md                         # Client skill definition
        └── reference.md                     # Advanced examples
```

### Adding New Skills

1. Create a new directory under `.claude-plugin/skills/`
2. Add a `SKILL.md` file with YAML frontmatter:

```yaml
---
name: my-skill-name
description: Description with trigger keywords...
---

# Skill Content

Your skill documentation here...
```

3. Optionally add a `reference.md` for extended examples

### Using btca for Documentation

This project includes a `btca.config.jsonc` for querying Plone documentation:

```bash
# Query plone.restapi docs
btca ask -r ploneRestapi -q "How do I create a custom endpoint?"

# Query multiple sources
btca ask -r ploneRestapi -r plone -q "Content type REST API support"
```

## Resources

- [Plone REST API Documentation](https://plonerestapi.readthedocs.io/)
- [plone.restapi on GitHub](https://github.com/plone/plone.restapi)
- [Plone Documentation](https://6.docs.plone.org/)
- [Claude Code Documentation](https://docs.anthropic.com/claude-code)

## License

MIT
