---
name: plone-restapi-backend
description: Develop Plone REST API backend services, endpoints, serializers, and deserializers. Use when creating custom @-endpoints, implementing REST services in Python, writing ZCML service registrations, customizing content serialization, or extending plone.restapi. Triggers on Plone backend API development, service classes, ZCML configuration, or REST endpoint implementation.
---

# Plone REST API Backend Development

Build custom REST API endpoints and services for Plone CMS using plone.restapi.

## Quick Start

```python
from plone.restapi.services import Service

class MyEndpoint(Service):
    def reply(self):
        return {"message": "Hello from custom endpoint"}
```

```xml
<plone:service
    method="GET"
    for="*"
    factory=".services.MyEndpoint"
    name="@my-endpoint"
    permission="zope2.View"
/>
```

## Service Architecture

### Base Service Class

All REST services inherit from `plone.restapi.services.Service`:

```python
from plone.restapi.services import Service

class MyService(Service):
    def __init__(self, context, request):
        super().__init__(context, request)
        # self.context - the content object
        # self.request - the HTTP request

    def reply(self):
        """Return JSON-serializable data."""
        return {"key": "value"}

    def publishTraverse(self, request, name):
        """Handle URL path segments after the service name."""
        # For endpoints like /@my-service/subpath
        pass
```

### HTTP Method Mapping

| Method | Use Case | Example |
|--------|----------|---------|
| GET | Retrieve data | `@my-items` |
| POST | Create content or trigger actions | `@my-action` |
| PATCH | Partial update | Update fields |
| PUT | Full replacement | Replace content |
| DELETE | Remove content | Delete item |

## ZCML Service Registration

### Basic Registration

```xml
<configure
    xmlns="http://namespaces.zope.org/zope"
    xmlns:plone="http://namespaces.plone.org/plone">

  <plone:service
      method="GET"
      for="*"
      factory=".services.MyService"
      name="@my-service"
      permission="zope2.View"
      layer="my.package.interfaces.IMyPackageLayer"
  />

</configure>
```

### For Specific Content Type

```xml
<plone:service
    method="GET"
    for="my.package.interfaces.IMyContentType"
    factory=".services.MyContentService"
    name="@content-info"
    permission="zope2.View"
/>
```

### Multiple Methods Same Endpoint

```xml
<plone:service
    method="GET"
    for="*"
    factory=".services.GetItems"
    name="@items"
    permission="zope2.View"
/>

<plone:service
    method="POST"
    for="*"
    factory=".services.CreateItem"
    name="@items"
    permission="cmf.AddPortalContent"
/>
```

### Common Permissions

| Permission | Use Case |
|------------|----------|
| `zope2.View` | Public read access |
| `cmf.AddPortalContent` | Create content |
| `cmf.ModifyPortalContent` | Edit content |
| `cmf.ReviewPortalContent` | Workflow transitions |
| `cmf.ManagePortal` | Admin only |

## Content Serialization

### Custom Serializer

```python
from plone.restapi.interfaces import ISerializeToJson
from plone.restapi.serializer.dxcontent import SerializeToJson
from zope.component import adapter
from zope.interface import implementer, Interface

@implementer(ISerializeToJson)
@adapter(IMyContentType, Interface)
class MyContentSerializer(SerializeToJson):
    def __call__(self, version=None, include_items=True):
        result = super().__call__(version, include_items)

        # Add custom fields
        result["custom_field"] = self.context.custom_field

        # Add computed values
        result["computed"] = self.compute_value()

        # Add related content
        result["related"] = self.serialize_related()

        return result

    def compute_value(self):
        return self.context.field1 + self.context.field2

    def serialize_related(self):
        return [
            {"@id": item.to_object.absolute_url(), "title": item.to_object.title}
            for item in (self.context.related_items or [])
        ]
```

Register in ZCML:

```xml
<adapter factory=".serializers.MyContentSerializer" />
```

### Summary Serializer (for listings)

```python
from plone.restapi.interfaces import ISerializeToJsonSummary
from zope.component import adapter
from zope.interface import implementer, Interface

@implementer(ISerializeToJsonSummary)
@adapter(IMyContent, Interface)
class MyContentSummarySerializer:
    def __init__(self, context, request):
        self.context = context
        self.request = request

    def __call__(self):
        return {
            "@id": self.context.absolute_url(),
            "@type": self.context.portal_type,
            "title": self.context.title,
            "description": self.context.description,
        }
```

## Content Deserialization

### Custom Deserializer

```python
from plone.restapi.interfaces import IDeserializeFromJson
from plone.restapi.deserializer.dxcontent import DeserializeFromJson
from plone.restapi.deserializer import json_body
from zope.component import adapter
from zope.interface import implementer, Interface

@implementer(IDeserializeFromJson)
@adapter(IMyContent, Interface)
class MyContentDeserializer(DeserializeFromJson):
    def __call__(self, data=None, validate_all=False, mask=None):
        if data is None:
            data = json_body(self.request)

        # Pre-process data before saving
        if "custom_field" in data:
            data["custom_field"] = data["custom_field"].strip().lower()

        return super().__call__(data, validate_all, mask)
```

## Expandable Elements

Add on-demand expandable data to responses:

```python
from plone.restapi.interfaces import IExpandableElement
from zope.component import adapter
from zope.interface import implementer, Interface

@implementer(IExpandableElement)
@adapter(Interface, Interface)
class StatsExpandable:
    def __init__(self, context, request):
        self.context = context
        self.request = request

    def __call__(self, expand=False):
        result = {
            "stats": {"@id": f"{self.context.absolute_url()}/@stats"}
        }
        if expand:
            result["stats"]["views"] = self.get_view_count()
            result["stats"]["children"] = len(self.context.objectIds())
        return result

    def get_view_count(self):
        # Your logic here
        return 42
```

Register:

```xml
<adapter factory=".expand.StatsExpandable" name="stats" />
```

Usage: `GET /plone/content?expand=stats`

## Batching

### Automatic Batching for Lists

```python
from plone.restapi.batching import HypermediaBatch
from plone.restapi.services import Service
from plone import api

class ListService(Service):
    def reply(self):
        catalog = api.portal.get_tool("portal_catalog")
        brains = catalog(portal_type="Document")

        batch = HypermediaBatch(self.request, brains)

        return {
            "@id": f"{self.context.absolute_url()}/@documents",
            "items_total": batch.items_total,
            "batching": batch.batching,
            "items": [self.serialize_brain(b) for b in batch]
        }

    def serialize_brain(self, brain):
        return {
            "@id": brain.getURL(),
            "title": brain.Title,
            "description": brain.Description,
        }
```

Query parameters: `?b_size=25&b_start=50`

## Error Handling

### Proper HTTP Status Codes

```python
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone.restapi.exceptions import DeserializationError

class MyService(Service):
    def reply(self):
        # 400 Bad Request
        data = json_body(self.request)
        if "required_field" not in data:
            self.request.response.setStatus(400)
            return {
                "error": {
                    "type": "BadRequest",
                    "message": "required_field is required"
                }
            }

        # 401 Unauthorized
        if not self.can_access():
            self.request.response.setStatus(401)
            return {"error": {"type": "Unauthorized", "message": "Login required"}}

        # 403 Forbidden
        if not self.has_permission():
            self.request.response.setStatus(403)
            return {"error": {"type": "Forbidden", "message": "Permission denied"}}

        # 404 Not Found
        item = self.find_item(data["id"])
        if item is None:
            self.request.response.setStatus(404)
            return {"error": {"type": "NotFound", "message": "Item not found"}}

        # 201 Created
        new_item = self.create_item(data)
        self.request.response.setStatus(201)
        return {"@id": new_item.absolute_url(), "title": new_item.title}
```

## Search and Catalog Queries

```python
from plone.restapi.services import Service
from plone import api

class SearchService(Service):
    def reply(self):
        query = self.request.form.get("q", "")
        portal_type = self.request.form.get("type")

        catalog = api.portal.get_tool("portal_catalog")

        search_query = {
            "SearchableText": f"*{query}*",
            "sort_on": "modified",
            "sort_order": "descending",
        }

        if portal_type:
            search_query["portal_type"] = portal_type

        brains = catalog(**search_query)

        return {
            "@id": f"{self.context.absolute_url()}/@search",
            "items_total": len(brains),
            "items": [
                {
                    "@id": brain.getURL(),
                    "@type": brain.portal_type,
                    "title": brain.Title,
                    "description": brain.Description,
                }
                for brain in brains[:100]
            ]
        }
```

## Workflow Integration

```python
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone import api

class WorkflowService(Service):
    def reply(self):
        data = json_body(self.request)
        transition = data.get("transition")
        comment = data.get("comment", "")

        try:
            api.content.transition(
                obj=self.context,
                transition=transition,
                comment=comment
            )
            return {
                "status": "success",
                "new_state": api.content.get_state(self.context)
            }
        except api.exc.InvalidParameterError as e:
            self.request.response.setStatus(400)
            return {"error": {"type": "BadRequest", "message": str(e)}}
```

## File and Image Handling

### File Upload

```python
from plone.restapi.services import Service
from plone.namedfile.file import NamedBlobFile

class FileUploadService(Service):
    def reply(self):
        if "file" not in self.request.form:
            self.request.response.setStatus(400)
            return {"error": "No file provided"}

        file_field = self.request.form["file"]

        blob = NamedBlobFile(
            data=file_field.read(),
            filename=file_field.filename,
            contentType=file_field.headers.get("content-type")
        )

        self.context.file = blob

        return {
            "filename": blob.filename,
            "size": blob.size,
            "content_type": blob.contentType
        }
```

### Image Scales

```python
from plone.restapi.services import Service
from plone.scale.interfaces import IImageScaleStorage

class ImageScaleService(Service):
    def reply(self):
        scale_name = self.request.form.get("scale", "preview")

        scales = IImageScaleStorage(self.context)
        scale = scales.scale(scale_name)

        if scale is None:
            self.request.response.setStatus(404)
            return {"error": "Scale not found"}

        return {
            "url": scale.url,
            "width": scale.width,
            "height": scale.height
        }
```

## Vocabularies

```python
from plone.restapi.services import Service
from zope.component import getUtility
from zope.schema.interfaces import IVocabularyFactory

class VocabularyService(Service):
    def reply(self):
        vocab_name = self.request.form.get("name")
        factory = getUtility(IVocabularyFactory, vocab_name)
        vocabulary = factory(self.context)

        return {
            "@id": f"{self.context.absolute_url()}/@vocabularies/{vocab_name}",
            "items": [
                {"token": term.token, "title": term.title}
                for term in vocabulary
            ]
        }
```

## Testing Services

```python
from plone.app.testing import SITE_OWNER_NAME, SITE_OWNER_PASSWORD
from plone.restapi.testing import RelativeSession
import unittest

class TestMyService(unittest.TestCase):
    layer = MY_FUNCTIONAL_TESTING

    def setUp(self):
        self.portal = self.layer["portal"]
        self.api_session = RelativeSession(self.portal.absolute_url())
        self.api_session.headers.update({"Accept": "application/json"})
        self.api_session.auth = (SITE_OWNER_NAME, SITE_OWNER_PASSWORD)

    def test_service_returns_data(self):
        response = self.api_session.get("/@my-service")
        self.assertEqual(response.status_code, 200)
        self.assertIn("expected_key", response.json())

    def test_service_requires_auth(self):
        self.api_session.auth = None
        response = self.api_session.get("/@my-service")
        self.assertEqual(response.status_code, 401)
```

## Dependencies

```python
# setup.py or pyproject.toml
install_requires=[
    "plone.restapi",
    "plone.api",
]
```

## Best Practices

1. **Use proper HTTP methods** - GET for reads, POST for creates, PATCH for updates
2. **Return meaningful status codes** - 200, 201, 400, 401, 403, 404
3. **Validate all input** - Use `json_body()` and validate required fields
4. **Apply least-privilege permissions** - Use most restrictive permission needed
5. **Use batching for lists** - Always batch large result sets
6. **Test all endpoints** - Write integration tests for services
7. **Document with OpenAPI** - Add Swagger documentation for complex APIs

---

## Built-in Endpoint Patterns

When extending or overriding built-in endpoints, follow these patterns:

### Content Copy/Move Service

```python
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone import api
from zope.component import getMultiAdapter

class CopyService(Service):
    """Custom copy service with validation."""

    def reply(self):
        data = json_body(self.request)
        source = data.get("source")

        if not source:
            self.request.response.setStatus(400)
            return {"error": "source is required"}

        sources = source if isinstance(source, list) else [source]
        results = []

        for src in sources:
            obj = self._resolve_object(src)
            if obj is None:
                continue

            clipboard = api.content.copy(source=obj, target=self.context)
            results.append({
                "source": obj.absolute_url(),
                "target": clipboard.absolute_url()
            })

        return results

    def _resolve_object(self, ref):
        # Handle UID, path, or URL
        if ref.startswith('http'):
            path = ref.split('/Plone')[-1]
            return api.content.get(path=path)
        elif len(ref) == 32:
            return api.content.get(UID=ref)
        else:
            return api.content.get(path=ref)
```

### Comments Service

```python
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone.app.discussion.interfaces import IConversation

class CommentsService(Service):
    """Custom comments handling."""

    def reply(self):
        conversation = IConversation(self.context)

        return {
            "@id": f"{self.context.absolute_url()}/@comments",
            "items_total": len(conversation),
            "items": [
                self._serialize_comment(comment)
                for comment in conversation.getComments()
            ]
        }

    def _serialize_comment(self, comment):
        return {
            "@id": f"{self.context.absolute_url()}/@comments/{comment.comment_id}",
            "comment_id": comment.comment_id,
            "text": {"data": comment.text, "mime-type": "text/plain"},
            "author_name": comment.author_name,
            "creation_date": comment.creation_date.isoformat(),
        }
```

### Locking Service

```python
from plone.restapi.services import Service
from plone.locking.interfaces import ILockable
from plone.restapi.deserializer import json_body

class LockService(Service):
    """Content locking service."""

    def reply(self):
        lockable = ILockable(self.context, None)
        if lockable is None:
            self.request.response.setStatus(400)
            return {"error": "Content is not lockable"}

        data = json_body(self.request)
        timeout = data.get("timeout", 600)
        stealable = data.get("stealable", True)

        lockable.lock(timeout=timeout)

        return {
            "locked": True,
            "stealable": stealable,
            "timeout": timeout,
            "token": lockable.lock_info()["token"],
        }
```

### History/Versioning Service

```python
from plone.restapi.services import Service
from Products.CMFEditions.interfaces import IArchivist

class HistoryService(Service):
    """Content versioning history."""

    def reply(self):
        archivist = IArchivist(self.context)
        history = []

        for version in archivist.queryHistory():
            history.append({
                "@id": f"{self.context.absolute_url()}/@history/{version.version_id}",
                "version": version.version_id,
                "actor": {"id": version.principal},
                "time": version.sys_metadata["timestamp"],
                "comments": version.sys_metadata.get("comment", ""),
            })

        return history
```

### Working Copy Service

```python
from plone.restapi.services import Service
from plone.app.iterate.interfaces import ICheckinCheckoutPolicy

class WorkingCopyService(Service):
    """Check-out/check-in workflow."""

    def reply(self):
        policy = ICheckinCheckoutPolicy(self.context, None)
        if policy is None:
            self.request.response.setStatus(400)
            return {"error": "Working copy not supported"}

        working_copy = policy.getWorkingCopy()
        baseline = policy.getBaseline()

        return {
            "working_copy": {
                "@id": working_copy.absolute_url(),
                "title": working_copy.title,
            } if working_copy else None,
            "working_copy_of": {
                "@id": baseline.absolute_url(),
            } if baseline else None,
        }
```

### Relations Service

```python
from plone.restapi.services import Service
from zc.relation.interfaces import ICatalog
from zope.component import getUtility
from zope.intid.interfaces import IIntIds
from plone.restapi.deserializer import json_body

class RelationsService(Service):
    """Manage content relations."""

    def reply(self):
        catalog = getUtility(ICatalog)
        intids = getUtility(IIntIds)

        # Query for relations
        relations = {}
        for relation in catalog.findRelations():
            rel_name = relation.from_attribute
            if rel_name not in relations:
                relations[rel_name] = {"items": [], "items_total": 0}

            relations[rel_name]["items"].append({
                "source": relation.from_object.absolute_url(),
                "target": relation.to_object.absolute_url(),
            })
            relations[rel_name]["items_total"] += 1

        return {
            "@id": f"{self.context.absolute_url()}/@relations",
            "relations": relations,
        }
```

### Sharing Service

```python
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone.api import portal

class SharingService(Service):
    """Local role assignments."""

    def reply(self):
        # Available roles from portal
        available_roles = [
            {"id": "Contributor", "title": "Can add"},
            {"id": "Editor", "title": "Can edit"},
            {"id": "Reader", "title": "Can view"},
            {"id": "Reviewer", "title": "Can review"},
        ]

        # Get local role assignments
        entries = []
        for principal, roles in self.context.get_local_roles():
            entries.append({
                "id": principal,
                "roles": {role: True for role in roles},
                "type": "user",  # or "group"
            })

        return {
            "@id": f"{self.context.absolute_url()}/@sharing",
            "available_roles": available_roles,
            "entries": entries,
            "inherit": getattr(self.context, "__ac_local_roles_block__", False) is False,
        }
```

### Aliases Service

```python
from plone.restapi.services import Service
from plone.app.redirector.interfaces import IRedirectionStorage
from zope.component import getUtility
from plone.restapi.deserializer import json_body

class AliasesService(Service):
    """URL alias management."""

    def reply(self):
        storage = getUtility(IRedirectionStorage)
        path = "/".join(self.context.getPhysicalPath())

        redirects = storage.redirects(path)

        return {
            "@id": f"{self.context.absolute_url()}/@aliases",
            "items": [
                {"path": redirect}
                for redirect in redirects
            ]
        }
```

### Translations Service

```python
from plone.restapi.services import Service
from plone.app.multilingual.interfaces import ITranslationManager

class TranslationsService(Service):
    """Multilingual content links."""

    def reply(self):
        manager = ITranslationManager(self.context)
        translations = manager.get_translations()

        return {
            "@id": f"{self.context.absolute_url()}/@translations",
            "items": [
                {"@id": obj.absolute_url(), "language": lang}
                for lang, obj in translations.items()
                if obj != self.context
            ]
        }
```

### Groups Service

```python
from plone.restapi.services import Service
from plone import api
from plone.restapi.deserializer import json_body

class GroupsService(Service):
    """User groups management."""

    def reply(self):
        query = self.request.form.get("query", "")
        limit = int(self.request.form.get("limit", 25))

        groups = api.group.get_groups()
        if query:
            groups = [g for g in groups if query.lower() in g.getId().lower()]

        return [
            {
                "@id": f"{api.portal.get().absolute_url()}/@groups/{g.getId()}",
                "id": g.getId(),
                "groupname": g.getId(),
                "title": g.getProperty("title", ""),
                "description": g.getProperty("description", ""),
                "email": g.getProperty("email", ""),
                "roles": list(g.getRoles()),
            }
            for g in groups[:limit]
        ]
```

### Control Panels Service

```python
from plone.restapi.services import Service
from plone.registry.interfaces import IRegistry
from zope.component import getUtility
from plone.restapi.deserializer import json_body

class ControlPanelService(Service):
    """Registry-based control panel."""

    def __init__(self, context, request):
        super().__init__(context, request)
        self.panel_id = None

    def publishTraverse(self, request, name):
        self.panel_id = name
        return self

    def reply(self):
        registry = getUtility(IRegistry)

        if not self.panel_id:
            # List all panels
            return self._list_panels()

        # Get specific panel
        return self._get_panel(registry)

    def _list_panels(self):
        return [
            {"@id": f"{self.context.absolute_url()}/@controlpanels/mail",
             "title": "Mail Settings", "group": "general"},
            # ... other panels
        ]

    def _get_panel(self, registry):
        # Return panel schema and data
        return {
            "@id": f"{self.context.absolute_url()}/@controlpanels/{self.panel_id}",
            "title": "Panel Title",
            "schema": {},  # JSON Schema
            "data": {},    # Current values
        }
```

### Add-ons Service

```python
from plone.restapi.services import Service
from Products.CMFPlone.interfaces import INonInstallable
from Products.GenericSetup import EXTENSION
from zope.component import getAllUtilitiesRegisteredFor

class AddonsService(Service):
    """Add-on management."""

    def reply(self):
        from Products.CMFPlone.factory import _get_packages

        packages = _get_packages()
        addons = []

        for pkg in packages:
            addons.append({
                "@id": f"{self.context.absolute_url()}/@addons/{pkg.project_name}",
                "id": pkg.project_name,
                "title": pkg.project_name,
                "version": pkg.version,
                "is_installed": self._is_installed(pkg.project_name),
            })

        return addons

    def _is_installed(self, addon_id):
        # Check if add-on is installed
        from Products.CMFPlone.utils import get_installer
        installer = get_installer(self.context)
        return installer.is_product_installed(addon_id)
```

### Site Info Service

```python
from plone.restapi.services import Service
from plone.registry.interfaces import IRegistry
from zope.component import getUtility

class SiteService(Service):
    """Site-wide information."""

    def reply(self):
        registry = getUtility(IRegistry)

        return {
            "@id": f"{self.context.absolute_url()}/@site",
            "plone.site_title": registry.get("plone.site_title", ""),
            "plone.default_language": registry.get("plone.default_language", "en"),
            "plone.available_languages": registry.get("plone.available_languages", ["en"]),
            "plone.portal_timezone": registry.get("plone.portal_timezone", "UTC"),
        }
```

### Types Schema Service

```python
from plone.restapi.services import Service
from plone.dexterity.interfaces import IDexterityFTI
from zope.component import getUtility

class TypesService(Service):
    """Content type schema management."""

    def __init__(self, context, request):
        super().__init__(context, request)
        self.type_id = None

    def publishTraverse(self, request, name):
        self.type_id = name
        return self

    def reply(self):
        if not self.type_id:
            # List all types
            from plone.dexterity.interfaces import IDexterityFTI
            from zope.component import getAllUtilitiesRegisteredFor

            return [
                {
                    "@id": f"{self.context.absolute_url()}/@types/{fti.getId()}",
                    "title": fti.title,
                    "addable": fti.global_allow,
                }
                for fti in getAllUtilitiesRegisteredFor(IDexterityFTI)
            ]

        # Get specific type schema
        fti = getUtility(IDexterityFTI, name=self.type_id)
        schema = fti.lookupSchema()

        return {
            "@id": f"{self.context.absolute_url()}/@types/{self.type_id}",
            "title": fti.title,
            "description": fti.description,
            "fieldsets": self._get_fieldsets(schema),
            "properties": self._get_properties(schema),
        }

    def _get_fieldsets(self, schema):
        # Extract fieldsets from schema
        return []

    def _get_properties(self, schema):
        # Extract field properties
        return {}
```

For complete examples, see [reference.md](reference.md).
