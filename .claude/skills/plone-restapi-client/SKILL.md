---
name: plone-restapi-client
description: Consume Plone REST API from JavaScript and Python clients. Use when fetching content, navigation, breadcrumbs from Plone, authenticating with JWT, making CRUD operations via HTTP, or building frontend apps. Triggers on REST API client code, @navigation, @contextnavigation, @breadcrumbs, fetch/axios/requests with Plone, or frontend integration.
---

# Plone REST API Client Development

Build JavaScript and Python clients to consume Plone REST API.

## API Basics

### Base URL and Headers

All requests require:

```
Accept: application/json
Content-Type: application/json  (for POST/PATCH/PUT)
Authorization: Bearer <token>   (for authenticated requests)
```

### Complete Endpoint Reference

#### Authentication & Users
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@login` | POST | Get JWT token |
| `/@logout` | POST | Invalidate token |
| `/@users` | GET/POST | List or create users |
| `/@users/{userid}` | GET/PATCH/DELETE | Read, update, delete user |
| `/@users/{userid}/reset-password` | POST | Request password reset |
| `/@groups` | GET/POST | List or create groups |
| `/@groups/{groupname}` | GET/PATCH/DELETE | Read, update, delete group |
| `/@principals` | GET | Search users and groups |
| `/@roles` | GET | List available roles |
| `/@userschema` | GET | Get user profile schema |
| `/@userschema/registration` | GET | Get registration form schema |

#### Content Operations
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/path/to/content` | GET | Get content |
| `/parent-path` | POST | Create content |
| `/path/to/content` | PATCH | Update content |
| `/path/to/content` | DELETE | Delete content |
| `/@copy` | POST | Copy content to destination |
| `/@move` | POST | Move content to destination |
| `/@workflow/{transition}` | POST | Execute workflow transition |
| `/@history` | GET | Get version history |
| `/@history/{version}` | GET | Get specific version |
| `/@history` | PATCH | Revert to previous version |
| `/@lock` | POST/GET/PATCH/DELETE | Manage content locks |
| `/@workingcopy` | POST/GET/PATCH/DELETE | Check-out/check-in workflow |

#### Navigation & Structure
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@navigation` | GET | Site navigation tree |
| `/@contextnavigation` | GET | Context-aware navigation |
| `/@breadcrumbs` | GET | Breadcrumb trail |
| `/@navroot` | GET | Navigation root info |
| `/@actions` | GET | Available portal actions |

#### Search & Query
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@search` | GET | Search content |
| `/@querystring-search` | POST | Advanced query search |
| `/@querystring` | GET | Get querystring config |

#### Comments & Discussion
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@comments` | GET | List comments |
| `/@comments/` | POST | Add comment |
| `/@comments/{id}` | POST | Reply to comment |
| `/@comments/{id}` | PATCH | Update comment |
| `/@comments/{id}` | DELETE | Delete comment |

#### Aliases & Redirects
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@aliases` | GET | List URL aliases |
| `/@aliases` | POST | Create aliases |
| `/@aliases` | DELETE | Remove aliases |

#### Relations
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@relations` | GET | Query relations |
| `/@relations` | POST | Create relations |
| `/@relations` | DELETE | Remove relations |
| `/@relations/rebuild` | POST | Rebuild broken relations |

#### Sharing & Permissions
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@sharing` | GET | Get sharing info |
| `/@sharing` | POST | Update sharing |
| `/@linkintegrity` | GET | Check link breaches |

#### Translations (Multilingual)
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@translations` | GET | List translations |
| `/@translations` | POST | Link translations |
| `/@translations` | DELETE | Unlink translation |
| `/@translation-locator` | GET | Get translation folder |

#### Vocabularies & Types
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@vocabularies` | GET | List vocabularies |
| `/@vocabularies/{name}` | GET | Get vocabulary terms |
| `/@sources/{field}` | GET | Get source terms |
| `/@querysources/{field}` | GET | Query source terms |
| `/@types` | GET | List content types |
| `/@types/{type}` | GET/POST/PATCH/PUT/DELETE | Manage type schema |

#### Administration (requires Manager role)
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/@addons` | GET | List add-ons |
| `/@addons/{id}/install` | POST | Install add-on |
| `/@addons/{id}/uninstall` | POST | Uninstall add-on |
| `/@addons/{id}/upgrade` | POST | Upgrade add-on |
| `/@controlpanels` | GET | List control panels |
| `/@controlpanels/{id}` | GET/PATCH | Read/update panel |
| `/@registry` | GET | List registry records |
| `/@registry/{name}` | GET | Get registry value |
| `/@registry/` | PATCH | Update registry |
| `/@site` | GET | Get site info |
| `/@system` | GET | Get system info |
| `/@database` | GET | Get database info |
| `/@upgrade` | GET/POST | Check/run upgrades |
| `/@transactions` | GET/PATCH | List/revert transactions |
| `/@content-rules` | GET/POST/PATCH/DELETE | Manage content rules |

---

## JavaScript Client

### Using Fetch API

```javascript
class PloneClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
    this.token = null;
  }

  async login(username, password) {
    const response = await fetch(`${this.baseUrl}/@login`, {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ login: username, password }),
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const data = await response.json();
    this.token = data.token;
    return data;
  }

  async logout() {
    await this.fetch('/@logout', { method: 'POST' });
    this.token = null;
  }

  async fetch(path, options = {}) {
    const headers = {
      'Accept': 'application/json',
      ...options.headers,
    };

    if (this.token) {
      headers['Authorization'] = `Bearer ${this.token}`;
    }

    if (options.body && typeof options.body === 'object') {
      headers['Content-Type'] = 'application/json';
      options.body = JSON.stringify(options.body);
    }

    const response = await fetch(`${this.baseUrl}${path}`, {
      ...options,
      headers,
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new Error(error.message || `HTTP ${response.status}`);
    }

    if (response.status === 204) {
      return null;
    }

    return response.json();
  }

  // CRUD Operations
  async get(path) {
    return this.fetch(path);
  }

  async create(parentPath, data) {
    return this.fetch(parentPath, {
      method: 'POST',
      body: data,
    });
  }

  async update(path, data) {
    return this.fetch(path, {
      method: 'PATCH',
      body: data,
    });
  }

  async delete(path) {
    return this.fetch(path, { method: 'DELETE' });
  }

  // Search
  async search(query, options = {}) {
    const params = new URLSearchParams({
      SearchableText: query,
      ...options,
    });
    return this.fetch(`/@search?${params}`);
  }

  // Workflow
  async transition(path, transition, comment = '') {
    return this.fetch(`${path}/@workflow/${transition}`, {
      method: 'POST',
      body: { comment },
    });
  }
}
```

### Usage Examples

```javascript
const client = new PloneClient('http://localhost:8080/Plone');

// Login
await client.login('admin', 'secret');

// Get content
const page = await client.get('/my-page');
console.log(page.title);

// Create content
const newDoc = await client.create('/folder', {
  '@type': 'Document',
  id: 'my-new-doc',
  title: 'My New Document',
  description: 'A description',
  text: {
    'content-type': 'text/html',
    data: '<p>Hello World</p>',
  },
});

// Update content
await client.update('/folder/my-new-doc', {
  title: 'Updated Title',
});

// Search
const results = await client.search('plone', {
  portal_type: 'Document',
  b_size: 10,
});

// Workflow transition
await client.transition('/folder/my-new-doc', 'publish');

// Delete
await client.delete('/folder/my-new-doc');

// Logout
await client.logout();
```

### Using Axios

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: 'http://localhost:8080/Plone',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
  },
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('plone_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle auth errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('plone_token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// Login
async function login(username, password) {
  const { data } = await api.post('/@login', { login: username, password });
  localStorage.setItem('plone_token', data.token);
  return data;
}

// CRUD
const getContent = (path) => api.get(path).then(r => r.data);
const createContent = (parent, data) => api.post(parent, data).then(r => r.data);
const updateContent = (path, data) => api.patch(path, data).then(r => r.data);
const deleteContent = (path) => api.delete(path);

// Search
async function search(query, options = {}) {
  const { data } = await api.get('/@search', {
    params: { SearchableText: query, ...options },
  });
  return data.items;
}
```

### React Hook Example

```javascript
import { useState, useEffect, useCallback } from 'react';
import DOMPurify from 'dompurify';

function usePloneContent(path) {
  const [content, setContent] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchContent = useCallback(async () => {
    try {
      setLoading(true);
      const response = await fetch(`/api${path}`, {
        headers: {
          'Accept': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('token')}`,
        },
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      const data = await response.json();
      setContent(data);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [path]);

  useEffect(() => {
    fetchContent();
  }, [fetchContent]);

  return { content, loading, error, refetch: fetchContent };
}

// Usage - IMPORTANT: Always sanitize HTML content before rendering
function DocumentView({ path }) {
  const { content, loading, error } = usePloneContent(path);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  // Sanitize HTML content to prevent XSS attacks
  const sanitizedHtml = DOMPurify.sanitize(content.text?.data || '');

  return (
    <article>
      <h1>{content.title}</h1>
      <p>{content.description}</p>
      <div dangerouslySetInnerHTML={{ __html: sanitizedHtml }} />
    </article>
  );
}
```

---

## Python Client

### Using Requests

```python
import requests
from typing import Optional, Dict, Any, List


class PloneClient:
    """Plone REST API client."""

    def __init__(self, base_url: str):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        self.session.headers.update({
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        })

    def login(self, username: str, password: str) -> Dict[str, Any]:
        """Authenticate and store JWT token."""
        response = self.session.post(
            f'{self.base_url}/@login',
            json={'login': username, 'password': password}
        )
        response.raise_for_status()
        data = response.json()
        self.session.headers['Authorization'] = f"Bearer {data['token']}"
        return data

    def logout(self) -> None:
        """Invalidate token."""
        self.session.post(f'{self.base_url}/@logout')
        self.session.headers.pop('Authorization', None)

    def get(self, path: str) -> Dict[str, Any]:
        """Get content by path."""
        response = self.session.get(f'{self.base_url}/{path.lstrip("/")}')
        response.raise_for_status()
        return response.json()

    def create(self, parent_path: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Create content in parent."""
        response = self.session.post(
            f'{self.base_url}/{parent_path.lstrip("/")}',
            json=data
        )
        response.raise_for_status()
        return response.json()

    def update(self, path: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Partial update content."""
        response = self.session.patch(
            f'{self.base_url}/{path.lstrip("/")}',
            json=data
        )
        response.raise_for_status()
        return response.json()

    def delete(self, path: str) -> bool:
        """Delete content."""
        response = self.session.delete(
            f'{self.base_url}/{path.lstrip("/")}'
        )
        response.raise_for_status()
        return True

    def search(
        self,
        query: str = '',
        portal_type: Optional[str] = None,
        path: Optional[str] = None,
        review_state: Optional[str] = None,
        sort_on: str = 'modified',
        sort_order: str = 'descending',
        b_size: int = 25,
        b_start: int = 0,
    ) -> Dict[str, Any]:
        """Search content."""
        params = {
            'b_size': b_size,
            'b_start': b_start,
            'sort_on': sort_on,
            'sort_order': sort_order,
        }

        if query:
            params['SearchableText'] = query
        if portal_type:
            params['portal_type'] = portal_type
        if path:
            params['path.query'] = path
        if review_state:
            params['review_state'] = review_state

        response = self.session.get(
            f'{self.base_url}/@search',
            params=params
        )
        response.raise_for_status()
        return response.json()

    def transition(
        self,
        path: str,
        transition: str,
        comment: str = ''
    ) -> Dict[str, Any]:
        """Execute workflow transition."""
        response = self.session.post(
            f'{self.base_url}/{path.lstrip("/")}/@workflow/{transition}',
            json={'comment': comment}
        )
        response.raise_for_status()
        return response.json()

    def get_vocabulary(self, name: str) -> List[Dict[str, str]]:
        """Get vocabulary items."""
        response = self.session.get(
            f'{self.base_url}/@vocabularies/{name}'
        )
        response.raise_for_status()
        return response.json().get('items', [])

    def upload_file(
        self,
        parent_path: str,
        filename: str,
        file_data: bytes,
        content_type: str = 'application/octet-stream',
        title: Optional[str] = None,
    ) -> Dict[str, Any]:
        """Upload a file."""
        import base64

        data = {
            '@type': 'File',
            'id': filename,
            'title': title or filename,
            'file': {
                'filename': filename,
                'content-type': content_type,
                'data': base64.b64encode(file_data).decode('ascii'),
                'encoding': 'base64',
            }
        }

        return self.create(parent_path, data)

    def upload_image(
        self,
        parent_path: str,
        filename: str,
        image_data: bytes,
        content_type: str = 'image/jpeg',
        title: Optional[str] = None,
    ) -> Dict[str, Any]:
        """Upload an image."""
        import base64

        data = {
            '@type': 'Image',
            'id': filename,
            'title': title or filename,
            'image': {
                'filename': filename,
                'content-type': content_type,
                'data': base64.b64encode(image_data).decode('ascii'),
                'encoding': 'base64',
            }
        }

        return self.create(parent_path, data)
```

### Usage Examples

```python
# Initialize client
client = PloneClient('http://localhost:8080/Plone')

# Login
client.login('admin', 'secret')

# Get content
page = client.get('/my-page')
print(page['title'])

# Create document
doc = client.create('/folder', {
    '@type': 'Document',
    'id': 'my-doc',
    'title': 'My Document',
    'description': 'A description',
    'text': {
        'content-type': 'text/html',
        'data': '<p>Hello World</p>',
    },
})

# Update
client.update('/folder/my-doc', {'title': 'Updated Title'})

# Search
results = client.search(
    query='plone',
    portal_type='Document',
    b_size=10
)
for item in results['items']:
    print(f"- {item['title']}: {item['@id']}")

# Workflow
client.transition('/folder/my-doc', 'publish', comment='Ready for review')

# Upload file
with open('document.pdf', 'rb') as f:
    client.upload_file('/folder', 'document.pdf', f.read(), 'application/pdf')

# Delete
client.delete('/folder/my-doc')

# Logout
client.logout()
```

### Async Python Client (aiohttp)

```python
import aiohttp
from typing import Optional, Dict, Any


class AsyncPloneClient:
    """Async Plone REST API client."""

    def __init__(self, base_url: str):
        self.base_url = base_url.rstrip('/')
        self.token = None
        self._session = None

    async def __aenter__(self):
        self._session = aiohttp.ClientSession(
            headers={'Accept': 'application/json'}
        )
        return self

    async def __aexit__(self, *args):
        await self._session.close()

    def _headers(self) -> Dict[str, str]:
        headers = {'Accept': 'application/json'}
        if self.token:
            headers['Authorization'] = f'Bearer {self.token}'
        return headers

    async def login(self, username: str, password: str) -> Dict[str, Any]:
        async with self._session.post(
            f'{self.base_url}/@login',
            json={'login': username, 'password': password},
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            data = await response.json()
            self.token = data['token']
            return data

    async def get(self, path: str) -> Dict[str, Any]:
        async with self._session.get(
            f'{self.base_url}/{path.lstrip("/")}',
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            return await response.json()

    async def create(
        self, parent_path: str, data: Dict[str, Any]
    ) -> Dict[str, Any]:
        async with self._session.post(
            f'{self.base_url}/{parent_path.lstrip("/")}',
            json=data,
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            return await response.json()

    async def update(
        self, path: str, data: Dict[str, Any]
    ) -> Dict[str, Any]:
        async with self._session.patch(
            f'{self.base_url}/{path.lstrip("/")}',
            json=data,
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            return await response.json()

    async def delete(self, path: str) -> bool:
        async with self._session.delete(
            f'{self.base_url}/{path.lstrip("/")}',
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            return True

    async def search(self, query: str, **kwargs) -> Dict[str, Any]:
        params = {'SearchableText': query, **kwargs}
        async with self._session.get(
            f'{self.base_url}/@search',
            params=params,
            headers=self._headers()
        ) as response:
            response.raise_for_status()
            return await response.json()


# Usage
async def main():
    async with AsyncPloneClient('http://localhost:8080/Plone') as client:
        await client.login('admin', 'secret')

        # Parallel requests
        import asyncio
        pages = await asyncio.gather(
            client.get('/page1'),
            client.get('/page2'),
            client.get('/page3'),
        )

        for page in pages:
            print(page['title'])
```

---

## Navigation

### @navigation - Site Navigation Tree

Returns the global navigation tree from the site root.

```javascript
// JavaScript
const navigation = await client.get('/@navigation');

// Response structure
{
  "@id": "http://localhost:8080/Plone/@navigation",
  "items": [
    {
      "@id": "http://localhost:8080/Plone/news",
      "title": "News",
      "description": "Site news",
      "review_state": "published",
      "items": [
        {
          "@id": "http://localhost:8080/Plone/news/article1",
          "title": "Article 1"
        }
      ]
    },
    {
      "@id": "http://localhost:8080/Plone/events",
      "title": "Events",
      "description": "Upcoming events"
    }
  ]
}
```

```python
# Python
navigation = client.get('/@navigation')
for item in navigation['items']:
    print(f"- {item['title']}: {item['@id']}")
    # Handle nested items
    for child in item.get('items', []):
        print(f"  - {child['title']}")
```

**Query Parameters:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| `expand.navigation.depth` | Levels to include | 1 |

```javascript
// Get 3 levels deep
const nav = await client.get('/@navigation?expand.navigation.depth=3');
```

### @contextnavigation - Context-Aware Navigation

Returns navigation relative to the current context (folder/section).

```javascript
// JavaScript - Get navigation for a specific section
const contextNav = await client.get('/news/@contextnavigation');

// Response structure
{
  "@id": "http://localhost:8080/Plone/news/@contextnavigation",
  "available": true,
  "has_custom_name": false,
  "items": [
    {
      "@id": "http://localhost:8080/Plone/news/article1",
      "title": "Article 1",
      "description": "First article",
      "is_current": false,
      "is_in_path": false,
      "is_folderish": false,
      "normalized_id": "article1",
      "review_state": "published",
      "thumb": "",
      "type": "News Item"
    },
    {
      "@id": "http://localhost:8080/Plone/news/article2",
      "title": "Article 2",
      "is_current": true,
      "is_in_path": true
    }
  ],
  "title": "News"
}
```

```python
# Python
context_nav = client.get('/folder/@contextnavigation')

# Build a sidebar menu
for item in context_nav.get('items', []):
    css_class = 'active' if item.get('is_current') else ''
    print(f"<li class='{css_class}'>{item['title']}</li>")
```

**Query Parameters:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| `expand.contextnavigation.topLevel` | Start level | 0 |
| `expand.contextnavigation.bottomLevel` | End level | 0 |
| `expand.contextnavigation.includeTop` | Include root | false |
| `expand.contextnavigation.currentFolderOnly` | Only current folder | false |
| `expand.contextnavigation.name` | Custom portlet name | null |

```javascript
// Get only current folder contents, 2 levels deep
const nav = await client.get(
  '/folder/@contextnavigation?expand.contextnavigation.bottomLevel=2&expand.contextnavigation.currentFolderOnly=true'
);
```

### @breadcrumbs - Breadcrumb Trail

Returns the path from site root to current content.

```javascript
// JavaScript
const breadcrumbs = await client.get('/news/article/@breadcrumbs');

// Response
{
  "@id": "http://localhost:8080/Plone/news/article/@breadcrumbs",
  "items": [
    {
      "@id": "http://localhost:8080/Plone",
      "title": "Home"
    },
    {
      "@id": "http://localhost:8080/Plone/news",
      "title": "News"
    },
    {
      "@id": "http://localhost:8080/Plone/news/article",
      "title": "My Article"
    }
  ]
}
```

```python
# Python - Build breadcrumb HTML
breadcrumbs = client.get('/path/to/content/@breadcrumbs')

html_parts = []
for item in breadcrumbs['items']:
    html_parts.append(f'<a href="{item["@id"]}">{item["title"]}</a>')
breadcrumb_html = ' &gt; '.join(html_parts)
```

### Using Expand for Combined Requests

Fetch content with navigation in a single request:

```javascript
// Get content with navigation, breadcrumbs, and actions
const content = await client.get(
  '/my-page?expand=navigation,breadcrumbs,actions,contextnavigation'
);

// Access expanded data
console.log(content.title);                    // Page title
console.log(content['@components'].navigation);     // Site nav
console.log(content['@components'].breadcrumbs);    // Breadcrumbs
console.log(content['@components'].contextnavigation); // Context nav
console.log(content['@components'].actions);        // Available actions
```

```python
# Python - Single request with multiple components
content = client.get('/my-page?expand=navigation,breadcrumbs,contextnavigation')

# Access components
nav_items = content['@components']['navigation']['items']
breadcrumbs = content['@components']['breadcrumbs']['items']
context_nav = content['@components']['contextnavigation']['items']
```

### React Navigation Component

```javascript
import { useState, useEffect } from 'react';

function useNavigation(path = '') {
  const [navigation, setNavigation] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchNav() {
      try {
        const response = await fetch(
          `/api${path}/@navigation?expand.navigation.depth=2`,
          { headers: { 'Accept': 'application/json' } }
        );
        const data = await response.json();
        setNavigation(data);
      } finally {
        setLoading(false);
      }
    }
    fetchNav();
  }, [path]);

  return { navigation, loading };
}

function NavigationMenu() {
  const { navigation, loading } = useNavigation();

  if (loading) return <nav>Loading...</nav>;

  return (
    <nav>
      <ul>
        {navigation?.items?.map((item) => (
          <li key={item['@id']}>
            <a href={item['@id']}>{item.title}</a>
            {item.items?.length > 0 && (
              <ul>
                {item.items.map((child) => (
                  <li key={child['@id']}>
                    <a href={child['@id']}>{child.title}</a>
                  </li>
                ))}
              </ul>
            )}
          </li>
        ))}
      </ul>
    </nav>
  );
}

function Breadcrumbs({ path }) {
  const [crumbs, setCrumbs] = useState([]);

  useEffect(() => {
    fetch(`/api${path}/@breadcrumbs`, {
      headers: { 'Accept': 'application/json' }
    })
      .then(r => r.json())
      .then(data => setCrumbs(data.items || []));
  }, [path]);

  return (
    <nav aria-label="breadcrumb">
      <ol>
        {crumbs.map((crumb, i) => (
          <li key={crumb['@id']}>
            {i < crumbs.length - 1 ? (
              <a href={crumb['@id']}>{crumb.title}</a>
            ) : (
              <span>{crumb.title}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

### Python Navigation Helper

```python
def build_nav_tree(client, path='', depth=2):
    """Build navigation tree with specified depth."""
    nav = client.get(f'{path}/@navigation?expand.navigation.depth={depth}')
    return nav.get('items', [])

def get_section_nav(client, section_path):
    """Get navigation for a specific section."""
    return client.get(f'{section_path}/@contextnavigation')

def get_breadcrumb_path(client, content_path):
    """Get breadcrumb trail for content."""
    crumbs = client.get(f'{content_path}/@breadcrumbs')
    return [
        {'title': item['title'], 'url': item['@id']}
        for item in crumbs.get('items', [])
    ]

# Usage
nav_tree = build_nav_tree(client, depth=3)
section = get_section_nav(client, '/news')
crumbs = get_breadcrumb_path(client, '/news/my-article')
```

---

## Common Patterns

### Pagination

```javascript
// JavaScript
async function getAllItems(client, query) {
  const items = [];
  let start = 0;
  const size = 100;

  while (true) {
    const result = await client.search(query, { b_start: start, b_size: size });
    items.push(...result.items);

    if (items.length >= result.items_total) break;
    start += size;
  }

  return items;
}
```

```python
# Python
def get_all_items(client, query):
    items = []
    start = 0
    size = 100

    while True:
        result = client.search(query, b_start=start, b_size=size)
        items.extend(result['items'])

        if len(items) >= result['items_total']:
            break
        start += size

    return items
```

### Error Handling

```javascript
// JavaScript
async function safeRequest(client, path) {
  try {
    return await client.get(path);
  } catch (error) {
    if (error.message.includes('401')) {
      // Re-authenticate
      await client.login(username, password);
      return client.get(path);
    }
    if (error.message.includes('404')) {
      return null;
    }
    throw error;
  }
}
```

```python
# Python
from requests.exceptions import HTTPError

def safe_get(client, path):
    try:
        return client.get(path)
    except HTTPError as e:
        if e.response.status_code == 401:
            client.login(username, password)
            return client.get(path)
        if e.response.status_code == 404:
            return None
        raise
```

### Batch Operations

```javascript
// JavaScript - parallel creates
async function createMany(client, parentPath, items) {
  return Promise.all(
    items.map(item => client.create(parentPath, item))
  );
}
```

```python
# Python - parallel with asyncio
import asyncio

async def create_many(client, parent_path, items):
    return await asyncio.gather(*[
        client.create(parent_path, item)
        for item in items
    ])
```

For more advanced patterns, see [reference.md](reference.md).
