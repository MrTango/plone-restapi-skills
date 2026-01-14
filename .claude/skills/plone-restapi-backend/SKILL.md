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

For complete examples, see [reference.md](reference.md).
