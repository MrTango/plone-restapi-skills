# Plone REST API Skills

Skills for working with Plone REST API in Claude Code.

## Installation

### Add as Custom Marketplace

Add this repository as a custom marketplace in Claude Code:

```bash
claude settings add marketplace https://github.com/MrTango/plone-restapi-skills
```

Or manually add to your `~/.claude/settings.json`:

```json
{
  "marketplaces": [
    "https://github.com/MrTango/plone-restapi-skills"
  ]
}
```

### Install Individual Skills

After adding the marketplace, install skills:

```bash
claude skill install plone-restapi-client
```

## Available Skills

### plone-restapi-client

Consume Plone REST API from JavaScript and Python clients.

**Triggers on:**
- REST API client code
- `@navigation`, `@contextnavigation`, `@breadcrumbs` endpoints
- fetch/axios/requests with Plone
- Frontend integration with Plone

**Features:**
- Complete endpoint reference
- JavaScript client (Fetch API, Axios)
- Python client (requests, aiohttp)
- React hooks and components
- Navigation and breadcrumb handling
- Common patterns (pagination, error handling, batch operations)

## Development

To add new skills, create a directory under `.claude/skills/` with a `SKILL.md` file and update `marketplace.json`.

## License

MIT
