# Specification: Plone REST API Agent Skills

## Overview

This task creates comprehensive agent skills for Plone REST API development covering both backend implementation (Python/Plone) and client consumption (TypeScript/JavaScript and Python). The skills will enable developers to build custom REST API endpoints, services, serializers, and deserializers on the server side, while also supporting frontend developers building REST API clients. The skills must comprehensively cover all Plone REST API Usage and Endpoints chapters from the official documentation (https://plonerestapi.readthedocs.io/en/latest/).

## Workflow Type

**Type**: feature

**Rationale**: This is a new feature implementation that creates comprehensive documentation-based skills. It requires extracting information from external documentation, synthesizing it into actionable skill content, and organizing it for effective agent assistance. The task involves creating new content files and enhancing existing skill structures.

## Task Scope

### Services Involved
- **plone-restapi-backend** (primary) - Backend development skill for Python/Plone REST API implementation
- **plone-restapi-client** (primary) - Client development skill for JavaScript/TypeScript and Python API consumption

### This Task Will:
- [ ] Fetch and analyze all Plone REST API documentation chapters using btca
- [ ] Enhance `.claude/skills/plone-restapi-backend/SKILL.md` with complete endpoint coverage
- [ ] Enhance `.claude/skills/plone-restapi-backend/reference.md` with additional examples
- [ ] Enhance `.claude/skills/plone-restapi-client/SKILL.md` with complete endpoint coverage
- [ ] Enhance `.claude/skills/plone-restapi-client/reference.md` with additional examples
- [ ] Ensure all REST API Usage chapters are covered
- [ ] Ensure all Endpoints chapters are documented
- [ ] Verify skills do NOT reference btca tool as a dependency

### Out of Scope:
- Creating new skill directories beyond the existing structure
- Adding btca as a runtime dependency in skills
- Implementing actual Plone backend code
- Creating working client applications
- Automated testing of skills

## Service Context

### plone-restapi-backend Skill

**Tech Stack:**
- Language: Markdown (skill documentation)
- Framework: Claude Code Skills format
- Key directories: `.claude/skills/plone-restapi-backend/`

**Entry Point:** `.claude/skills/plone-restapi-backend/SKILL.md`

**How to Run:**
```bash
# Skills are automatically loaded by Claude Code agent
# No runtime execution required
```

**Supporting Files:**
- `reference.md` - Extended examples and patterns

### plone-restapi-client Skill

**Tech Stack:**
- Language: Markdown (skill documentation)
- Framework: Claude Code Skills format
- Key directories: `.claude/skills/plone-restapi-client/`

**Entry Point:** `.claude/skills/plone-restapi-client/SKILL.md`

**How to Run:**
```bash
# Skills are automatically loaded by Claude Code agent
# No runtime execution required
```

**Supporting Files:**
- `reference.md` - Extended examples and patterns

## Files to Modify

| File | Service | What to Change |
|------|---------|---------------|
| `.claude/skills/plone-restapi-backend/SKILL.md` | backend | Enhance with complete REST API backend patterns from all documentation chapters |
| `.claude/skills/plone-restapi-backend/reference.md` | backend | Add more advanced examples covering all endpoint types |
| `.claude/skills/plone-restapi-client/SKILL.md` | client | Enhance with complete endpoint reference from all documentation chapters |
| `.claude/skills/plone-restapi-client/reference.md` | client | Add more advanced client patterns and examples |

## Files to Reference

These files show patterns to follow:

| File | Pattern to Copy |
|------|----------------|
| `.claude/skills/plone-restapi-backend/SKILL.md` | Existing skill format with YAML frontmatter, code examples, and tables |
| `.claude/skills/plone-restapi-client/SKILL.md` | Existing skill format with complete endpoint tables and client examples |
| `AGENTS.md` | btca usage patterns for fetching documentation |

## Patterns to Follow

### SKILL.md Frontmatter Pattern

From `.claude/skills/plone-restapi-backend/SKILL.md`:

```yaml
---
name: plone-restapi-backend
description: Develop Plone REST API backend services, endpoints, serializers, and deserializers. Use when creating custom @-endpoints, implementing REST services in Python, writing ZCML service registrations, customizing content serialization, or extending plone.restapi. Triggers on Plone backend API development, service classes, ZCML configuration, or REST endpoint implementation.
---
```

**Key Points:**
- Concise but comprehensive description
- Include trigger keywords for skill activation
- List all relevant use cases

### Endpoint Table Pattern

From `.claude/skills/plone-restapi-client/SKILL.md`:

```markdown
#### Authentication & Users
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@login` | POST | Get JWT token |
| `/@logout` | POST | Invalidate token |
```

**Key Points:**
- Group endpoints by category
- Include method and brief description
- Use consistent formatting

### Code Example Pattern

From existing skills:

```python
from plone.restapi.services import Service

class MyEndpoint(Service):
    def reply(self):
        return {"message": "Hello from custom endpoint"}
```

**Key Points:**
- Complete, runnable examples
- Include necessary imports
- Show typical usage patterns

## Requirements

### Functional Requirements

1. **Complete Backend Coverage**
   - Description: All REST API backend development patterns documented
   - Acceptance: Skill covers service creation, ZCML registration, serializers, deserializers, expandables, batching, error handling

2. **Complete Client Coverage**
   - Description: All REST API endpoints documented for client consumption
   - Acceptance: Skill covers all endpoint categories: auth, content, navigation, search, comments, aliases, relations, sharing, translations, vocabularies, admin

3. **Multi-Language Client Support**
   - Description: Client examples in JavaScript/TypeScript and Python
   - Acceptance: Each endpoint category has examples in both JS/TS and Python

4. **btca Independence**
   - Description: Skills must not depend on btca at runtime
   - Acceptance: No references to btca in SKILL.md or reference.md files

5. **Documentation-Driven**
   - Description: Content sourced from official Plone REST API documentation
   - Acceptance: All endpoints and patterns verified against https://plonerestapi.readthedocs.io/en/latest/

### Edge Cases

1. **Missing Endpoint Documentation** - If official docs are incomplete, note limitations in skill comments
2. **Deprecated Endpoints** - Mark any deprecated endpoints clearly
3. **Version Differences** - Note any version-specific behavior where applicable

## Implementation Notes

### DO
- Use btca tool during implementation to fetch latest documentation
- Follow the existing skill format with YAML frontmatter
- Organize endpoints by logical category (auth, content, navigation, etc.)
- Include complete, copy-paste-ready code examples
- Cover both simple and advanced use cases
- Cross-reference between backend and client skills where relevant

### DON'T
- Include btca in the final skill content
- Create partial or incomplete endpoint tables
- Skip error handling patterns
- Omit type definitions for TypeScript examples
- Leave placeholder content

## Development Environment

### Data Gathering Commands

```bash
# Fetch Plone REST API documentation using btca
btca ask -r ploneRestapi -q "List all REST API endpoints"
btca ask -r ploneRestapi -q "How to create custom REST API services"
btca ask -r ploneRestapi -q "Authentication and JWT tokens"
btca ask -r ploneRestapi -q "Content serialization and deserialization"
```

### Skill Location
- Backend: `.claude/skills/plone-restapi-backend/`
- Client: `.claude/skills/plone-restapi-client/`

### Required Tools
- `btca` - For fetching documentation (implementation phase only)
- Text editor for Markdown files

## Success Criteria

The task is complete when:

1. [ ] Backend skill covers all service development patterns (services, ZCML, serializers, deserializers, expandables)
2. [ ] Client skill covers ALL endpoint categories from official documentation
3. [ ] All endpoint tables are complete with method and description
4. [ ] JavaScript/TypeScript examples provided for each major endpoint category
5. [ ] Python examples provided for each major endpoint category
6. [ ] No references to btca in any skill file
7. [ ] Skills follow existing format patterns (YAML frontmatter, code blocks, tables)
8. [ ] Reference files contain advanced patterns and complete examples
9. [ ] All code examples are syntactically correct
10. [ ] Skills are self-contained and actionable

## QA Acceptance Criteria

**CRITICAL**: These criteria must be verified by the QA Agent before sign-off.

### Documentation Completeness Tests
| Test | File | What to Verify |
|------|------|----------------|
| Backend Skill Structure | `.claude/skills/plone-restapi-backend/SKILL.md` | Has valid YAML frontmatter, all sections present |
| Client Skill Structure | `.claude/skills/plone-restapi-client/SKILL.md` | Has valid YAML frontmatter, all sections present |
| Backend Reference | `.claude/skills/plone-restapi-backend/reference.md` | Contains advanced examples |
| Client Reference | `.claude/skills/plone-restapi-client/reference.md` | Contains advanced examples |

### Content Coverage Tests
| Test | Skill | What to Verify |
|------|-------|----------------|
| Authentication Endpoints | client | @login, @logout, @users, @groups, @principals |
| Content Endpoints | client | CRUD operations, @copy, @move, @workflow, @history, @lock |
| Navigation Endpoints | client | @navigation, @contextnavigation, @breadcrumbs, @navroot |
| Search Endpoints | client | @search, @querystring-search |
| Comments Endpoints | client | @comments operations |
| Relations Endpoints | client | @relations, @aliases |
| Admin Endpoints | client | @addons, @controlpanels, @registry, @system |
| Service Patterns | backend | Service class, ZCML registration, HTTP methods |
| Serialization Patterns | backend | ISerializeToJson, ISerializeToJsonSummary |
| Deserialization Patterns | backend | IDeserializeFromJson, json_body |

### Code Quality Tests
| Test | What to Verify |
|------|----------------|
| Python Syntax | All Python code blocks are valid syntax |
| JavaScript Syntax | All JavaScript/TypeScript code blocks are valid syntax |
| YAML Frontmatter | Valid YAML in all SKILL.md files |
| Markdown Tables | All tables render correctly |

### Independence Tests
| Test | What to Verify |
|------|----------------|
| No btca References | grep -r "btca" .claude/skills/ returns no matches |
| Self-Contained | Skills reference only their own files and external Plone docs |

### Browser/Manual Verification
| Page/Component | Location | Checks |
|----------------|----------|--------|
| Backend SKILL.md | `.claude/skills/plone-restapi-backend/SKILL.md` | Renders correctly in Markdown viewer |
| Client SKILL.md | `.claude/skills/plone-restapi-client/SKILL.md` | Renders correctly in Markdown viewer |

### QA Sign-off Requirements
- [ ] All skill files have valid YAML frontmatter
- [ ] All endpoint categories from official docs are covered
- [ ] Code examples are syntactically correct (can be verified with linters)
- [ ] No btca references in skill files
- [ ] Tables render correctly in Markdown
- [ ] Cross-references between skills are valid
- [ ] Description fields trigger on relevant keywords
- [ ] Examples are copy-paste ready
- [ ] No placeholder or TODO content remains
