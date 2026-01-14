# Plone REST API Backend Reference

Complete examples for advanced backend development patterns.

## Example 1: Content Listing with Filtering

```python
# services/listing.py
from plone import api
from plone.restapi.services import Service
from plone.restapi.batching import HypermediaBatch
from plone.restapi.interfaces import ISerializeToJsonSummary
from zope.component import getMultiAdapter


class ContentListing(Service):
    """List content with filtering and custom fields."""

    def reply(self):
        portal_type = self.request.get("type")
        state = self.request.get("state")
        sort_on = self.request.get("sort", "modified")
        sort_order = self.request.get("order", "descending")

        query = {
            "path": "/".join(self.context.getPhysicalPath()),
            "sort_on": sort_on,
            "sort_order": sort_order,
        }

        if portal_type:
            query["portal_type"] = portal_type
        if state:
            query["review_state"] = state

        catalog = api.portal.get_tool("portal_catalog")
        brains = catalog(**query)

        batch = HypermediaBatch(self.request, brains)

        items = []
        for brain in batch:
            obj = brain.getObject()
            serializer = getMultiAdapter(
                (obj, self.request), ISerializeToJsonSummary
            )
            item = serializer()
            item["review_state"] = brain.review_state
            item["modified"] = brain.modified.ISO8601()
            items.append(item)

        return {
            "@id": f"{self.context.absolute_url()}/@content-listing",
            "items_total": batch.items_total,
            "batching": batch.batching,
            "items": items,
        }
```

ZCML:

```xml
<plone:service
    method="GET"
    for="plone.dexterity.interfaces.IDexterityContainer"
    factory=".services.listing.ContentListing"
    name="@content-listing"
    permission="zope2.View"
/>
```

---

## Example 2: Bulk Operations

```python
# services/bulk.py
from plone import api
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone.restapi.interfaces import IDeserializeFromJson
from zope.component import getMultiAdapter


class BulkUpdate(Service):
    """Update multiple content items at once."""

    def reply(self):
        data = json_body(self.request)

        if "items" not in data:
            self.request.response.setStatus(400)
            return {"error": {"type": "BadRequest", "message": "items required"}}

        results = {"success": [], "errors": []}

        for item_data in data["items"]:
            uid = item_data.get("UID")
            if not uid:
                results["errors"].append({"UID": None, "error": "UID required"})
                continue

            obj = api.content.get(UID=uid)
            if obj is None:
                results["errors"].append({"UID": uid, "error": "Not found"})
                continue

            try:
                deserializer = getMultiAdapter(
                    (obj, self.request), IDeserializeFromJson
                )
                deserializer(data=item_data, validate_all=False)
                results["success"].append({
                    "UID": uid,
                    "@id": obj.absolute_url(),
                    "title": obj.title
                })
            except Exception as e:
                results["errors"].append({"UID": uid, "error": str(e)})

        status_code = 200 if not results["errors"] else 207
        self.request.response.setStatus(status_code)

        return {
            "updated": len(results["success"]),
            "failed": len(results["errors"]),
            "results": results
        }
```

---

## Example 3: Custom Content Type with Full Serialization

### Interface

```python
# interfaces.py
from zope.interface import Interface
from zope import schema


class IProject(Interface):
    """Project content type."""

    title = schema.TextLine(title="Title", required=True)
    description = schema.Text(title="Description", required=False)
    status = schema.Choice(
        title="Status",
        values=["planning", "active", "completed", "archived"],
        default="planning"
    )
    start_date = schema.Date(title="Start Date", required=False)
    end_date = schema.Date(title="End Date", required=False)
    budget = schema.Float(title="Budget", required=False)
    team_members = schema.List(
        title="Team Members",
        value_type=schema.TextLine(),
        required=False
    )
```

### Serializer

```python
# serializers.py
from plone.restapi.interfaces import ISerializeToJson
from plone.restapi.serializer.dxcontent import SerializeToJson
from zope.component import adapter
from zope.interface import implementer, Interface
from .interfaces import IProject
from datetime import date
from plone import api


@implementer(ISerializeToJson)
@adapter(IProject, Interface)
class ProjectSerializer(SerializeToJson):
    def __call__(self, version=None, include_items=True):
        result = super().__call__(version, include_items)

        # Computed fields
        result["duration_days"] = self._calculate_duration()
        result["is_overdue"] = self._check_overdue()
        result["team_size"] = len(self.context.team_members or [])
        result["tasks_count"] = self._count_tasks()

        return result

    def _calculate_duration(self):
        if self.context.start_date and self.context.end_date:
            return (self.context.end_date - self.context.start_date).days
        return None

    def _check_overdue(self):
        if self.context.end_date and self.context.status != "completed":
            return date.today() > self.context.end_date
        return False

    def _count_tasks(self):
        catalog = api.portal.get_tool("portal_catalog")
        path = "/".join(self.context.getPhysicalPath())
        return len(catalog(portal_type="Task", path=path))
```

### Stats Service

```python
# services/project.py
from plone import api
from plone.restapi.services import Service
from .interfaces import IProject


class ProjectStats(Service):
    """Get project statistics."""

    def reply(self):
        if not IProject.providedBy(self.context):
            self.request.response.setStatus(400)
            return {"error": "Not a project"}

        catalog = api.portal.get_tool("portal_catalog")
        path = "/".join(self.context.getPhysicalPath())

        all_tasks = catalog(portal_type="Task", path=path)
        completed_tasks = catalog(
            portal_type="Task", path=path, review_state="completed"
        )

        return {
            "@id": f"{self.context.absolute_url()}/@stats",
            "project": self.context.title,
            "status": self.context.status,
            "budget": self.context.budget,
            "team_size": len(self.context.team_members or []),
            "tasks": {
                "total": len(all_tasks),
                "completed": len(completed_tasks),
                "completion_rate": (
                    len(completed_tasks) / len(all_tasks) * 100
                    if all_tasks else 0
                )
            }
        }
```

---

## Example 4: CSV Export

```python
# services/export.py
from plone import api
from plone.restapi.services import Service
import csv
import io


class ExportCSV(Service):
    """Export folder contents as CSV."""

    def reply(self):
        catalog = api.portal.get_tool("portal_catalog")
        path = "/".join(self.context.getPhysicalPath())

        brains = catalog(
            path={"query": path, "depth": 1},
            sort_on="sortable_title"
        )

        output = io.StringIO()
        writer = csv.writer(output)

        writer.writerow(["Title", "Type", "URL", "Modified", "State"])

        for brain in brains:
            writer.writerow([
                brain.Title,
                brain.portal_type,
                brain.getURL(),
                brain.modified.strftime("%Y-%m-%d %H:%M"),
                brain.review_state
            ])

        self.request.response.setHeader("Content-Type", "text/csv")
        self.request.response.setHeader(
            "Content-Disposition",
            f'attachment; filename="{self.context.id}-export.csv"'
        )

        return output.getvalue()
```

---

## Example 5: Webhook Integration

```python
# services/webhook.py
from plone import api
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
import requests


class WebhookTrigger(Service):
    """Trigger webhook for content events."""

    def reply(self):
        data = json_body(self.request)
        webhook_url = data.get("webhook_url")
        event_type = data.get("event", "content.updated")

        if not webhook_url:
            self.request.response.setStatus(400)
            return {"error": "webhook_url required"}

        payload = {
            "event": event_type,
            "content": {
                "@id": self.context.absolute_url(),
                "@type": self.context.portal_type,
                "title": self.context.title,
                "modified": self.context.modified().ISO8601(),
                "modifier": api.user.get_current().getId()
            }
        }

        try:
            response = requests.post(
                webhook_url,
                json=payload,
                timeout=10,
                headers={"Content-Type": "application/json"}
            )
            return {"status": "sent", "response_code": response.status_code}
        except requests.RequestException as e:
            self.request.response.setStatus(502)
            return {"error": f"Webhook failed: {str(e)}"}
```

---

## Example 6: Custom Authentication Extension

```python
# services/auth.py
from plone.restapi.services import Service
from plone.restapi.services.auth import Login
from AccessControl import getSecurityManager


class CustomLogin(Login):
    """Extended login with additional claims."""

    def reply(self):
        result = super().reply()

        # Add custom claims
        user = getSecurityManager().getUser()
        result["user_info"] = {
            "id": user.getId(),
            "fullname": user.getProperty("fullname", ""),
            "email": user.getProperty("email", ""),
        }

        return result


class MultiAuthService(Service):
    """Service supporting multiple auth methods."""

    def reply(self):
        user = getSecurityManager().getUser()

        if user.getId() == "Anonymous User":
            self.request.response.setStatus(401)
            return {"error": "Authentication required"}

        return {
            "user": user.getId(),
            "roles": list(user.getRoles()),
        }
```

---

## Example 7: Path Traversal Endpoint

Handle sub-paths like `/@my-service/action/id`:

```python
# services/traversal.py
from plone.restapi.services import Service


class TraversableService(Service):
    """Service with path traversal."""

    def __init__(self, context, request):
        super().__init__(context, request)
        self.params = []

    def publishTraverse(self, request, name):
        self.params.append(name)
        return self

    def reply(self):
        if not self.params:
            return {"actions": ["list", "get", "stats"]}

        action = self.params[0]

        if action == "list":
            return self._list()
        elif action == "get" and len(self.params) > 1:
            return self._get(self.params[1])
        elif action == "stats":
            return self._stats()
        else:
            self.request.response.setStatus(404)
            return {"error": "Unknown action"}

    def _list(self):
        return {"items": ["item1", "item2"]}

    def _get(self, item_id):
        return {"id": item_id, "data": "..."}

    def _stats(self):
        return {"total": 42}
```

Usage:
- `GET /@my-service` → list actions
- `GET /@my-service/list` → list items
- `GET /@my-service/get/123` → get item 123
- `GET /@my-service/stats` → get stats
