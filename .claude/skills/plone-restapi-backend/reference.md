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

---

## Example 8: Custom Link Integrity Check

```python
# services/linkintegrity.py
from plone import api
from plone.restapi.services import Service
from zc.relation.interfaces import ICatalog
from zope.component import getUtility

class LinkIntegrityService(Service):
    """Check for reference breaches before deletion."""

    def reply(self):
        uids = self.request.form.get("uids", [])
        if isinstance(uids, str):
            uids = [uids]

        breaches = []
        for uid in uids:
            obj = api.content.get(UID=uid)
            if obj is None:
                continue

            # Find objects referencing this one
            refs = self._find_references(obj)

            breaches.append({
                "@id": obj.absolute_url(),
                "@type": obj.portal_type,
                "title": obj.title,
                "breaches": [
                    {
                        "@id": ref.absolute_url(),
                        "title": ref.title,
                        "@type": ref.portal_type,
                    }
                    for ref in refs
                ]
            })

        return breaches

    def _find_references(self, target):
        """Find all objects that reference the target."""
        from zope.intid.interfaces import IIntIds
        intids = getUtility(IIntIds)
        catalog = getUtility(ICatalog)

        try:
            target_id = intids.getId(target)
        except KeyError:
            return []

        refs = []
        for rel in catalog.findRelations({"to_id": target_id}):
            if rel.from_object is not None:
                refs.append(rel.from_object)

        return refs
```

---

## Example 9: Navigation Root Service

```python
# services/navroot.py
from plone import api
from plone.restapi.services import Service
from plone.restapi.interfaces import ISerializeToJson
from zope.component import getMultiAdapter
from plone.base.interfaces import INavigationRoot

class NavRootService(Service):
    """Get navigation root information."""

    def reply(self):
        navroot = self._find_navroot()

        serializer = getMultiAdapter(
            (navroot, self.request), ISerializeToJson
        )

        return {
            "@id": f"{self.context.absolute_url()}/@navroot",
            "navroot": serializer()
        }

    def _find_navroot(self):
        """Walk up to find the navigation root."""
        obj = self.context
        while obj is not None:
            if INavigationRoot.providedBy(obj):
                return obj
            obj = obj.__parent__
        return api.portal.get()
```

---

## Example 10: Principals Search Service

```python
# services/principals.py
from plone import api
from plone.restapi.services import Service

class PrincipalsService(Service):
    """Search users and groups."""

    def reply(self):
        search = self.request.form.get("search", "")

        users = []
        groups = []

        if search:
            # Search users
            for user in api.user.get_users():
                if search.lower() in user.getId().lower():
                    users.append({
                        "@id": f"{api.portal.get().absolute_url()}/@users/{user.getId()}",
                        "id": user.getId(),
                        "fullname": user.getProperty("fullname", ""),
                        "email": user.getProperty("email", ""),
                    })

            # Search groups
            for group in api.group.get_groups():
                if search.lower() in group.getId().lower():
                    groups.append({
                        "@id": f"{api.portal.get().absolute_url()}/@groups/{group.getId()}",
                        "id": group.getId(),
                        "groupname": group.getId(),
                        "title": group.getProperty("title", ""),
                    })

        return {
            "users": users,
            "groups": groups,
        }
```

---

## Example 11: Transactions Service

```python
# services/transactions.py
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from ZODB.interfaces import IDatabase
from zope.component import getUtility

class TransactionsService(Service):
    """List and revert transactions."""

    def reply(self):
        db = getUtility(IDatabase)
        storage = db.storage

        # Get recent transactions
        transactions = []
        for txn in storage.undoLog(0, 50):
            transactions.append({
                "id": txn["id"],
                "time": txn["time"],
                "username": txn.get("user_name", ""),
                "description": txn.get("description", ""),
                "size": txn.get("size", 0),
            })

        return transactions


class TransactionsRevertService(Service):
    """Revert transactions."""

    def reply(self):
        data = json_body(self.request)
        transaction_ids = data.get("transaction_ids", [])

        if not transaction_ids:
            self.request.response.setStatus(400)
            return {"error": "transaction_ids required"}

        db = getUtility(IDatabase)

        for txn_id in transaction_ids:
            db.undo(txn_id)

        return {"message": "Transactions have been reverted successfully."}
```

---

## Example 12: User Schema Service

```python
# services/userschema.py
from plone.restapi.services import Service
from plone.restapi.types.utils import get_jsonschema_for_fti
from zope.interface import Interface
from zope import schema as zschema

class UserSchemaService(Service):
    """User profile and registration schemas."""

    def __init__(self, context, request):
        super().__init__(context, request)
        self.schema_type = None

    def publishTraverse(self, request, name):
        self.schema_type = name
        return self

    def reply(self):
        if self.schema_type == "registration":
            return self._get_registration_schema()
        return self._get_profile_schema()

    def _get_profile_schema(self):
        return {
            "type": "object",
            "fieldsets": [
                {
                    "id": "default",
                    "title": "Default",
                    "fields": ["fullname", "email", "description", "location"],
                }
            ],
            "properties": {
                "fullname": {"type": "string", "title": "Full Name"},
                "email": {"type": "string", "title": "Email", "widget": "email"},
                "description": {"type": "string", "title": "Biography", "widget": "textarea"},
                "location": {"type": "string", "title": "Location"},
            },
            "required": ["email"],
        }

    def _get_registration_schema(self):
        schema = self._get_profile_schema()
        schema["properties"]["username"] = {"type": "string", "title": "Username"}
        schema["properties"]["password"] = {"type": "string", "title": "Password", "widget": "password"}
        schema["properties"]["password_confirm"] = {"type": "string", "title": "Confirm Password", "widget": "password"}
        schema["required"] = ["username", "email", "password"]
        return schema
```

---

## Example 13: Upgrade Service

```python
# services/upgrade.py
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from Products.CMFPlone.factory import _DEFAULT_PROFILE
import logging

logger = logging.getLogger(__name__)

class UpgradeService(Service):
    """Site upgrade management."""

    def reply(self):
        from Products.CMFPlone.MigrationTool import MigrationTool
        mt = MigrationTool()

        return {
            "@id": f"{self.context.absolute_url()}/@upgrade",
            "versions": {
                "fs": mt.getFileSystemVersion(),
                "instance": mt.getInstanceVersion(),
            },
            "upgrade_steps": self._get_upgrade_steps(mt),
        }

    def _get_upgrade_steps(self, mt):
        steps = {}
        for step in mt.listUpgrades():
            key = f"{step['ssource']}-{step['sdest']}"
            if key not in steps:
                steps[key] = []
            steps[key].append({
                "id": step["id"],
                "title": step["title"],
            })
        return steps


class UpgradeRunService(Service):
    """Execute upgrade."""

    def reply(self):
        data = json_body(self.request)
        dry_run = data.get("dry_run", True)

        from Products.CMFPlone.MigrationTool import MigrationTool
        mt = MigrationTool()

        # Run upgrade
        report = mt.upgrade(REQUEST=self.request, dry_run=dry_run)

        return {
            "@id": f"{self.context.absolute_url()}/@upgrade",
            "dry_run": dry_run,
            "report": report,
            "upgraded": not dry_run,
            "versions": {
                "fs": mt.getFileSystemVersion(),
                "instance": mt.getInstanceVersion(),
            },
        }
```

---

## Example 14: Content Rules Assignment

```python
# services/contentrules.py
from plone import api
from plone.restapi.services import Service
from plone.restapi.deserializer import json_body
from plone.contentrules.engine.interfaces import IRuleStorage
from plone.contentrules.engine.interfaces import IRuleAssignmentManager
from zope.component import getUtility

class ContentRulesService(Service):
    """Content rules management for a context."""

    def reply(self):
        storage = getUtility(IRuleStorage)
        manager = IRuleAssignmentManager(self.context, None)

        if manager is None:
            return {"assigned_rules": [], "assignable_rules": [], "acquired_rules": []}

        # Get assigned rules
        assigned = []
        for rule_id in manager.keys():
            assignment = manager[rule_id]
            rule = storage.get(rule_id)
            if rule:
                assigned.append({
                    "id": rule_id,
                    "title": rule.title,
                    "enabled": assignment.enabled,
                    "bubbles": assignment.bubbles,
                })

        # Get assignable rules
        assignable = []
        for rule_id, rule in storage.items():
            if rule_id not in manager:
                assignable.append({
                    "id": rule_id,
                    "title": rule.title,
                    "trigger": rule.event.__identifier__,
                })

        return {
            "@id": f"{self.context.absolute_url()}/@content-rules/",
            "assigned_rules": assigned,
            "assignable_rules": assignable,
            "acquired_rules": [],
        }
```

---

## Example 15: Database Info Service

```python
# services/database.py
from plone.restapi.services import Service
from ZODB.interfaces import IDatabase
from zope.component import getUtility

class DatabaseService(Service):
    """ZODB database information."""

    def reply(self):
        db = getUtility(IDatabase)

        return {
            "@id": f"{self.context.absolute_url()}/@database",
            "db_name": db.getName(),
            "db_size": db.getSize(),
            "cache_size": db.getCacheSize(),
            "cache_length": db.cacheSize(),
            "cache_length_bytes": db.cacheDetailSize(),
            "database_size": db.objectCount(),
        }
```

---

## Example 16: Querystring Configuration

```python
# services/querystring.py
from plone.restapi.services import Service
from plone.app.querystring.interfaces import IQuerystringRegistryReader
from zope.component import getUtility

class QuerystringService(Service):
    """Querystring configuration for search."""

    def reply(self):
        reader = getUtility(IQuerystringRegistryReader)
        config = reader()

        return {
            "@id": f"{self.context.absolute_url()}/@querystring",
            "indexes": config.get("indexes", {}),
            "sortable_indexes": {
                k: v for k, v in config.get("indexes", {}).items()
                if v.get("sortable", False)
            },
        }
```

---

## Example 17: Sources and Query Sources

```python
# services/sources.py
from plone.restapi.services import Service
from zope.schema.interfaces import IContextSourceBinder, ISource
from zope.component import getUtility
from zope.schema import getFields

class SourcesService(Service):
    """Field-bound sources."""

    def __init__(self, context, request):
        super().__init__(context, request)
        self.field_name = None

    def publishTraverse(self, request, name):
        self.field_name = name
        return self

    def reply(self):
        if not self.field_name:
            self.request.response.setStatus(400)
            return {"error": "field_name required"}

        # Get the field from the content schema
        from plone.dexterity.interfaces import IDexterityContent
        if not IDexterityContent.providedBy(self.context):
            self.request.response.setStatus(400)
            return {"error": "Not a dexterity content"}

        from plone.dexterity.utils import getAdditionalSchemata
        from plone.behavior.interfaces import IBehavior

        schema = self.context.getTypeInfo().lookupSchema()
        field = schema.get(self.field_name)

        if field is None:
            # Check behaviors
            for behavior_schema in getAdditionalSchemata(self.context):
                field = behavior_schema.get(self.field_name)
                if field:
                    break

        if field is None:
            self.request.response.setStatus(404)
            return {"error": f"Field {self.field_name} not found"}

        source = field.source
        if IContextSourceBinder.providedBy(source):
            source = source(self.context)

        terms = []
        for term in source:
            terms.append({
                "token": term.token,
                "title": term.title or term.token,
            })

        return {
            "@id": f"{self.context.absolute_url()}/@sources/{self.field_name}",
            "items": terms,
            "items_total": len(terms),
        }
```

---

## Example 18: System Information Service

```python
# services/system.py
from plone.restapi.services import Service
import sys
import pkg_resources

class SystemService(Service):
    """Backend system information."""

    def reply(self):
        return {
            "@id": f"{self.context.absolute_url()}/@system",
            "python_version": sys.version,
            "plone_version": self._get_version("Products.CMFPlone"),
            "zope_version": self._get_version("Zope"),
            "cmf_version": self._get_version("Products.CMFCore"),
            "pil_version": self._get_version("Pillow"),
            "debug_mode": self._is_debug_mode(),
            "upgrade": self._needs_upgrade(),
        }

    def _get_version(self, package):
        try:
            return pkg_resources.get_distribution(package).version
        except pkg_resources.DistributionNotFound:
            return "N/A"

    def _is_debug_mode(self):
        from App.config import getConfiguration
        config = getConfiguration()
        return getattr(config, "debug_mode", False)

    def _needs_upgrade(self):
        from Products.CMFPlone.MigrationTool import MigrationTool
        mt = MigrationTool()
        return mt.needUpgrading()
```

---

## ZCML Registration Reference

Complete ZCML for all service types:

```xml
<configure
    xmlns="http://namespaces.zope.org/zope"
    xmlns:plone="http://namespaces.plone.org/plone">

  <!-- Content Operations -->
  <plone:service
      method="POST"
      for="plone.dexterity.interfaces.IDexterityContainer"
      factory=".services.CopyService"
      name="@copy"
      permission="zope2.View"
  />

  <plone:service
      method="POST"
      for="plone.dexterity.interfaces.IDexterityContainer"
      factory=".services.MoveService"
      name="@move"
      permission="zope2.View"
  />

  <!-- Comments -->
  <plone:service
      method="GET"
      for="plone.dexterity.interfaces.IDexterityContent"
      factory=".services.CommentsService"
      name="@comments"
      permission="zope2.View"
  />

  <!-- Locking -->
  <plone:service
      method="POST"
      for="plone.dexterity.interfaces.IDexterityContent"
      factory=".services.LockService"
      name="@lock"
      permission="cmf.ModifyPortalContent"
  />

  <!-- History -->
  <plone:service
      method="GET"
      for="plone.dexterity.interfaces.IDexterityContent"
      factory=".services.HistoryService"
      name="@history"
      permission="zope2.View"
  />

  <!-- Working Copy -->
  <plone:service
      method="POST"
      for="plone.dexterity.interfaces.IDexterityContent"
      factory=".services.WorkingCopyService"
      name="@workingcopy"
      permission="cmf.ModifyPortalContent"
  />

  <!-- Admin Services (portal root only) -->
  <plone:service
      method="GET"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.DatabaseService"
      name="@database"
      permission="cmf.ManagePortal"
  />

  <plone:service
      method="GET"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.SystemService"
      name="@system"
      permission="cmf.ManagePortal"
  />

  <plone:service
      method="GET"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.TransactionsService"
      name="@transactions"
      permission="cmf.ManagePortal"
  />

  <plone:service
      method="PATCH"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.TransactionsRevertService"
      name="@transactions"
      permission="cmf.ManagePortal"
  />

  <plone:service
      method="GET"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.UpgradeService"
      name="@upgrade"
      permission="cmf.ManagePortal"
  />

  <plone:service
      method="POST"
      for="Products.CMFPlone.interfaces.IPloneSiteRoot"
      factory=".services.UpgradeRunService"
      name="@upgrade"
      permission="cmf.ManagePortal"
  />

</configure>
```
