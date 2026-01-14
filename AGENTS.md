# AGENTS.md

## btca

When you need up-to-date information about technologies used in this project, use btca to query source repositories directly.

**Available resources**: ploneRestapi, plone, ploneVolto

### Usage

```bash
btca ask -r <resource> -q "<question>"
```

Use multiple `-r` flags to query multiple resources at once:

```bash
btca ask -r ploneRestapi -r plone -q "How do I create a custom content type with REST API support?"
```
