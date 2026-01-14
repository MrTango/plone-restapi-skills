# Plone REST API Client Reference

Advanced patterns and complete examples for JavaScript and Python clients.

## TypeScript Client

Full-featured TypeScript client with proper typing:

```typescript
interface PloneContent {
  '@id': string;
  '@type': string;
  id: string;
  title: string;
  description?: string;
  text?: { 'content-type': string; data: string };
  created: string;
  modified: string;
  review_state?: string;
  [key: string]: unknown;
}

interface SearchResult {
  '@id': string;
  items_total: number;
  items: PloneContent[];
  batching?: {
    '@id': string;
    first: string;
    last: string;
    next?: string;
    prev?: string;
  };
}

interface LoginResponse {
  token: string;
}

interface CreateContentData {
  '@type': string;
  id?: string;
  title: string;
  description?: string;
  text?: { 'content-type': string; data: string };
  [key: string]: unknown;
}

class PloneClient {
  private baseUrl: string;
  private token: string | null = null;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
  }

  async login(username: string, password: string): Promise<LoginResponse> {
    const response = await fetch(`${this.baseUrl}/@login`, {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ login: username, password }),
    });

    if (!response.ok) {
      throw new Error(`Login failed: ${response.status}`);
    }

    const data: LoginResponse = await response.json();
    this.token = data.token;
    return data;
  }

  private async request<T>(
    path: string,
    options: RequestInit = {}
  ): Promise<T> {
    const headers: Record<string, string> = {
      'Accept': 'application/json',
      ...(options.headers as Record<string, string>),
    };

    if (this.token) {
      headers['Authorization'] = `Bearer ${this.token}`;
    }

    if (options.body && typeof options.body !== 'string') {
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
      return null as T;
    }

    return response.json();
  }

  async get<T = PloneContent>(path: string): Promise<T> {
    return this.request<T>(path);
  }

  async create(
    parentPath: string,
    data: CreateContentData
  ): Promise<PloneContent> {
    return this.request<PloneContent>(parentPath, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  async update(
    path: string,
    data: Partial<PloneContent>
  ): Promise<PloneContent> {
    return this.request<PloneContent>(path, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
  }

  async delete(path: string): Promise<void> {
    await this.request<void>(path, { method: 'DELETE' });
  }

  async search(params: {
    SearchableText?: string;
    portal_type?: string;
    path?: string;
    review_state?: string;
    sort_on?: string;
    sort_order?: 'ascending' | 'descending';
    b_size?: number;
    b_start?: number;
  }): Promise<SearchResult> {
    const searchParams = new URLSearchParams();
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined) {
        searchParams.set(key, String(value));
      }
    });
    return this.request<SearchResult>(`/@search?${searchParams}`);
  }
}
```

---

## Vue.js Composable

```javascript
import { ref, reactive, computed } from 'vue';

export function usePloneClient(baseUrl) {
  const token = ref(localStorage.getItem('plone_token'));
  const user = ref(null);
  const isAuthenticated = computed(() => !!token.value);

  async function request(path, options = {}) {
    const headers = {
      'Accept': 'application/json',
      ...options.headers,
    };

    if (token.value) {
      headers['Authorization'] = `Bearer ${token.value}`;
    }

    if (options.body && typeof options.body === 'object') {
      headers['Content-Type'] = 'application/json';
      options.body = JSON.stringify(options.body);
    }

    const response = await fetch(`${baseUrl}${path}`, { ...options, headers });

    if (response.status === 401) {
      token.value = null;
      localStorage.removeItem('plone_token');
      throw new Error('Unauthorized');
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return response.status === 204 ? null : response.json();
  }

  async function login(username, password) {
    const data = await request('/@login', {
      method: 'POST',
      body: { login: username, password },
    });
    token.value = data.token;
    localStorage.setItem('plone_token', data.token);
    return data;
  }

  function logout() {
    token.value = null;
    localStorage.removeItem('plone_token');
  }

  return {
    token,
    user,
    isAuthenticated,
    login,
    logout,
    get: (path) => request(path),
    create: (parent, data) => request(parent, { method: 'POST', body: data }),
    update: (path, data) => request(path, { method: 'PATCH', body: data }),
    delete: (path) => request(path, { method: 'DELETE' }),
    search: (query, opts = {}) => {
      const params = new URLSearchParams({ SearchableText: query, ...opts });
      return request(`/@search?${params}`);
    },
  };
}

// Usage in component
export default {
  setup() {
    const plone = usePloneClient('http://localhost:8080/Plone');

    async function loadPage() {
      const content = await plone.get('/my-page');
      console.log(content.title);
    }

    return { plone, loadPage };
  },
};
```

---

## Svelte Store

```javascript
import { writable, derived } from 'svelte/store';

function createPloneClient(baseUrl) {
  const token = writable(localStorage.getItem('plone_token'));
  const isAuthenticated = derived(token, ($token) => !!$token);

  let currentToken = null;
  token.subscribe((value) => {
    currentToken = value;
    if (value) {
      localStorage.setItem('plone_token', value);
    } else {
      localStorage.removeItem('plone_token');
    }
  });

  async function request(path, options = {}) {
    const headers = {
      'Accept': 'application/json',
      ...options.headers,
    };

    if (currentToken) {
      headers['Authorization'] = `Bearer ${currentToken}`;
    }

    if (options.body && typeof options.body === 'object') {
      headers['Content-Type'] = 'application/json';
      options.body = JSON.stringify(options.body);
    }

    const response = await fetch(`${baseUrl}${path}`, { ...options, headers });

    if (response.status === 401) {
      token.set(null);
      throw new Error('Unauthorized');
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return response.status === 204 ? null : response.json();
  }

  return {
    token,
    isAuthenticated,

    async login(username, password) {
      const data = await request('/@login', {
        method: 'POST',
        body: { login: username, password },
      });
      token.set(data.token);
      return data;
    },

    logout() {
      token.set(null);
    },

    get: (path) => request(path),
    create: (parent, data) => request(parent, { method: 'POST', body: data }),
    update: (path, data) => request(path, { method: 'PATCH', body: data }),
    delete: (path) => request(path, { method: 'DELETE' }),
  };
}

export const plone = createPloneClient('http://localhost:8080/Plone');
```

---

## Python CLI Tool

Command-line interface for Plone:

```python
#!/usr/bin/env python3
"""Plone REST API CLI tool."""

import argparse
import json
import sys
from typing import Optional

import requests


class PloneCLI:
    def __init__(self, base_url: str, token: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        self.session.headers.update({'Accept': 'application/json'})
        if token:
            self.session.headers['Authorization'] = f'Bearer {token}'

    def login(self, username: str, password: str) -> str:
        response = self.session.post(
            f'{self.base_url}/@login',
            json={'login': username, 'password': password}
        )
        response.raise_for_status()
        token = response.json()['token']
        self.session.headers['Authorization'] = f'Bearer {token}'
        return token

    def get(self, path: str) -> dict:
        response = self.session.get(f'{self.base_url}/{path.lstrip("/")}')
        response.raise_for_status()
        return response.json()

    def create(self, parent: str, content_type: str, title: str, **kwargs) -> dict:
        data = {'@type': content_type, 'title': title, **kwargs}
        response = self.session.post(
            f'{self.base_url}/{parent.lstrip("/")}',
            json=data
        )
        response.raise_for_status()
        return response.json()

    def update(self, path: str, **kwargs) -> dict:
        response = self.session.patch(
            f'{self.base_url}/{path.lstrip("/")}',
            json=kwargs
        )
        response.raise_for_status()
        return response.json()

    def delete(self, path: str) -> bool:
        response = self.session.delete(f'{self.base_url}/{path.lstrip("/")}')
        response.raise_for_status()
        return True

    def search(self, query: str, **kwargs) -> list:
        params = {'SearchableText': query, **kwargs}
        response = self.session.get(f'{self.base_url}/@search', params=params)
        response.raise_for_status()
        return response.json()['items']


def main():
    parser = argparse.ArgumentParser(description='Plone REST API CLI')
    parser.add_argument('--url', required=True, help='Plone site URL')
    parser.add_argument('--token', help='JWT token')
    parser.add_argument('--username', help='Username for login')
    parser.add_argument('--password', help='Password for login')

    subparsers = parser.add_subparsers(dest='command', required=True)

    # Get command
    get_parser = subparsers.add_parser('get', help='Get content')
    get_parser.add_argument('path', help='Content path')

    # Create command
    create_parser = subparsers.add_parser('create', help='Create content')
    create_parser.add_argument('parent', help='Parent path')
    create_parser.add_argument('--type', required=True, dest='content_type')
    create_parser.add_argument('--title', required=True)
    create_parser.add_argument('--id')
    create_parser.add_argument('--description')

    # Update command
    update_parser = subparsers.add_parser('update', help='Update content')
    update_parser.add_argument('path', help='Content path')
    update_parser.add_argument('--title')
    update_parser.add_argument('--description')

    # Delete command
    delete_parser = subparsers.add_parser('delete', help='Delete content')
    delete_parser.add_argument('path', help='Content path')

    # Search command
    search_parser = subparsers.add_parser('search', help='Search content')
    search_parser.add_argument('query', help='Search query')
    search_parser.add_argument('--type', dest='portal_type')
    search_parser.add_argument('--limit', type=int, default=10)

    args = parser.parse_args()

    cli = PloneCLI(args.url, args.token)

    if args.username and args.password:
        cli.login(args.username, args.password)

    try:
        if args.command == 'get':
            result = cli.get(args.path)
        elif args.command == 'create':
            kwargs = {}
            if args.id:
                kwargs['id'] = args.id
            if args.description:
                kwargs['description'] = args.description
            result = cli.create(args.parent, args.content_type, args.title, **kwargs)
        elif args.command == 'update':
            kwargs = {}
            if args.title:
                kwargs['title'] = args.title
            if args.description:
                kwargs['description'] = args.description
            result = cli.update(args.path, **kwargs)
        elif args.command == 'delete':
            cli.delete(args.path)
            result = {'status': 'deleted', 'path': args.path}
        elif args.command == 'search':
            kwargs = {'b_size': args.limit}
            if args.portal_type:
                kwargs['portal_type'] = args.portal_type
            result = cli.search(args.query, **kwargs)

        print(json.dumps(result, indent=2))

    except requests.HTTPError as e:
        print(f'Error: {e.response.status_code} - {e.response.text}', file=sys.stderr)
        sys.exit(1)


if __name__ == '__main__':
    main()
```

Usage:

```bash
# Login and get content
python plone_cli.py --url http://localhost:8080/Plone \
  --username admin --password secret \
  get /my-page

# Create document
python plone_cli.py --url http://localhost:8080/Plone \
  --token <jwt-token> \
  create /folder --type Document --title "New Doc" --description "A document"

# Search
python plone_cli.py --url http://localhost:8080/Plone \
  --token <jwt-token> \
  search "plone" --type Document --limit 20
```

---

## Python Data Classes

Type-safe client with dataclasses:

```python
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Any
from datetime import datetime
import requests


@dataclass
class PloneContent:
    id: str
    url: str
    type: str
    title: str
    description: str = ''
    text: Optional[str] = None
    created: Optional[datetime] = None
    modified: Optional[datetime] = None
    review_state: Optional[str] = None
    extra: Dict[str, Any] = field(default_factory=dict)

    @classmethod
    def from_json(cls, data: dict) -> 'PloneContent':
        return cls(
            id=data.get('id', ''),
            url=data.get('@id', ''),
            type=data.get('@type', ''),
            title=data.get('title', ''),
            description=data.get('description', ''),
            text=data.get('text', {}).get('data') if data.get('text') else None,
            created=datetime.fromisoformat(data['created'].rstrip('Z'))
                if data.get('created') else None,
            modified=datetime.fromisoformat(data['modified'].rstrip('Z'))
                if data.get('modified') else None,
            review_state=data.get('review_state'),
            extra={k: v for k, v in data.items()
                   if k not in ('id', '@id', '@type', 'title', 'description',
                               'text', 'created', 'modified', 'review_state')},
        )


@dataclass
class SearchResult:
    items: List[PloneContent]
    total: int
    has_more: bool

    @classmethod
    def from_json(cls, data: dict) -> 'SearchResult':
        items = [PloneContent.from_json(item) for item in data.get('items', [])]
        total = data.get('items_total', len(items))
        batching = data.get('batching', {})
        has_more = 'next' in batching
        return cls(items=items, total=total, has_more=has_more)


class TypedPloneClient:
    def __init__(self, base_url: str):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        self.session.headers.update({
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        })

    def login(self, username: str, password: str) -> str:
        response = self.session.post(
            f'{self.base_url}/@login',
            json={'login': username, 'password': password}
        )
        response.raise_for_status()
        token = response.json()['token']
        self.session.headers['Authorization'] = f'Bearer {token}'
        return token

    def get(self, path: str) -> PloneContent:
        response = self.session.get(f'{self.base_url}/{path.lstrip("/")}')
        response.raise_for_status()
        return PloneContent.from_json(response.json())

    def search(
        self,
        query: str = '',
        portal_type: Optional[str] = None,
        limit: int = 25,
        offset: int = 0
    ) -> SearchResult:
        params = {'b_size': limit, 'b_start': offset}
        if query:
            params['SearchableText'] = query
        if portal_type:
            params['portal_type'] = portal_type

        response = self.session.get(f'{self.base_url}/@search', params=params)
        response.raise_for_status()
        return SearchResult.from_json(response.json())


# Usage
client = TypedPloneClient('http://localhost:8080/Plone')
client.login('admin', 'secret')

# Get typed content
page: PloneContent = client.get('/my-page')
print(f"Title: {page.title}")
print(f"Modified: {page.modified}")

# Search with types
results: SearchResult = client.search('plone', portal_type='Document')
print(f"Found {results.total} items")
for item in results.items:
    print(f"  - {item.title} ({item.type})")
```

---

## Content Sync Script

Sync content between Plone sites:

```python
"""Sync content between Plone instances."""

import logging
from typing import Set

from plone_client import PloneClient  # Your client class

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def sync_content(
    source: PloneClient,
    target: PloneClient,
    source_path: str,
    target_path: str,
    portal_types: Set[str] = None,
):
    """Sync content from source to target."""

    # Get source items
    results = source.search(
        path=source_path,
        b_size=1000,
    )

    for item in results['items']:
        if portal_types and item['@type'] not in portal_types:
            continue

        # Calculate target path
        relative_path = item['@id'].replace(
            f"{source.base_url}{source_path}", ''
        ).lstrip('/')
        item_target_path = f"{target_path}/{relative_path}"

        # Check if exists
        try:
            existing = target.get(item_target_path)
            # Update if source is newer
            if item['modified'] > existing['modified']:
                logger.info(f"Updating: {item_target_path}")
                target.update(item_target_path, {
                    'title': item['title'],
                    'description': item.get('description', ''),
                })
        except Exception:
            # Create new
            logger.info(f"Creating: {item_target_path}")
            parent_path = '/'.join(item_target_path.split('/')[:-1])
            target.create(parent_path, {
                '@type': item['@type'],
                'id': item['id'],
                'title': item['title'],
                'description': item.get('description', ''),
            })


# Usage
source = PloneClient('http://source-plone:8080/Plone')
source.login('admin', 'secret')

target = PloneClient('http://target-plone:8080/Plone')
target.login('admin', 'secret')

sync_content(
    source, target,
    source_path='/news',
    target_path='/news',
    portal_types={'News Item', 'Document'},
)
```

---

## Copy and Move Operations

```javascript
// JavaScript - Copy content
async function copyContent(client, sourcePaths, destinationPath) {
  return client.fetch(`${destinationPath}/@copy`, {
    method: 'POST',
    body: { source: sourcePaths },
  });
}

// Move content
async function moveContent(client, sourcePaths, destinationPath) {
  return client.fetch(`${destinationPath}/@move`, {
    method: 'POST',
    body: { source: sourcePaths },
  });
}

// Usage
const result = await copyContent(client, ['/folder/doc1', '/folder/doc2'], '/archive');
// Returns: [{ source: '/folder/doc1', target: '/archive/copy_of_doc1' }, ...]
```

```python
# Python - Copy/Move
def copy_content(client, sources: list, destination: str) -> list:
    """Copy content to destination."""
    response = client.session.post(
        f'{client.base_url}/{destination.lstrip("/")}/@copy',
        json={'source': sources}
    )
    response.raise_for_status()
    return response.json()

def move_content(client, sources: list, destination: str) -> list:
    """Move content to destination."""
    response = client.session.post(
        f'{client.base_url}/{destination.lstrip("/")}/@move',
        json={'source': sources}
    )
    response.raise_for_status()
    return response.json()
```

---

## Comments Management

```javascript
// JavaScript - Comments
class CommentsManager {
  constructor(client) {
    this.client = client;
  }

  async list(contentPath) {
    return this.client.get(`${contentPath}/@comments`);
  }

  async add(contentPath, text) {
    return this.client.fetch(`${contentPath}/@comments/`, {
      method: 'POST',
      body: { text },
    });
  }

  async reply(contentPath, commentId, text) {
    return this.client.fetch(`${contentPath}/@comments/${commentId}`, {
      method: 'POST',
      body: { text },
    });
  }

  async update(contentPath, commentId, text) {
    return this.client.fetch(`${contentPath}/@comments/${commentId}`, {
      method: 'PATCH',
      body: { text },
    });
  }

  async delete(contentPath, commentId) {
    return this.client.fetch(`${contentPath}/@comments/${commentId}`, {
      method: 'DELETE',
    });
  }
}
```

```python
# Python - Comments
class CommentsManager:
    def __init__(self, client):
        self.client = client

    def list(self, content_path: str) -> dict:
        return self.client.get(f'{content_path}/@comments')

    def add(self, content_path: str, text: str) -> None:
        self.client.session.post(
            f'{self.client.base_url}/{content_path.lstrip("/")}/@comments/',
            json={'text': text}
        )

    def reply(self, content_path: str, comment_id: str, text: str) -> None:
        self.client.session.post(
            f'{self.client.base_url}/{content_path.lstrip("/")}/@comments/{comment_id}',
            json={'text': text}
        )

    def update(self, content_path: str, comment_id: str, text: str) -> None:
        self.client.session.patch(
            f'{self.client.base_url}/{content_path.lstrip("/")}/@comments/{comment_id}',
            json={'text': text}
        )

    def delete(self, content_path: str, comment_id: str) -> None:
        self.client.session.delete(
            f'{self.client.base_url}/{content_path.lstrip("/")}/@comments/{comment_id}'
        )
```

---

## Content Locking

```javascript
// JavaScript - Locking
class LockManager {
  constructor(client) {
    this.client = client;
  }

  async lock(path, options = {}) {
    const { stealable = true, timeout = 600 } = options;
    return this.client.fetch(`${path}/@lock`, {
      method: 'POST',
      body: { stealable, timeout },
    });
  }

  async getStatus(path) {
    return this.client.get(`${path}/@lock`);
  }

  async refresh(path) {
    return this.client.fetch(`${path}/@lock`, { method: 'PATCH' });
  }

  async unlock(path, force = false) {
    const params = force ? '?force=true' : '';
    return this.client.fetch(`${path}/@lock${params}`, { method: 'DELETE' });
  }

  // Update locked content with lock token
  async updateLocked(path, data, lockToken) {
    return this.client.fetch(path, {
      method: 'PATCH',
      headers: { 'Lock-Token': lockToken },
      body: data,
    });
  }
}

// Usage
const lockManager = new LockManager(client);
const lock = await lockManager.lock('/my-doc');
await lockManager.updateLocked('/my-doc', { title: 'New Title' }, lock.token);
await lockManager.unlock('/my-doc');
```

```python
# Python - Locking
class LockManager:
    def __init__(self, client):
        self.client = client

    def lock(self, path: str, stealable: bool = True, timeout: int = 600) -> dict:
        response = self.client.session.post(
            f'{self.client.base_url}/{path.lstrip("/")}/@lock',
            json={'stealable': stealable, 'timeout': timeout}
        )
        response.raise_for_status()
        return response.json()

    def get_status(self, path: str) -> dict:
        return self.client.get(f'{path}/@lock')

    def refresh(self, path: str) -> dict:
        response = self.client.session.patch(
            f'{self.client.base_url}/{path.lstrip("/")}/@lock'
        )
        response.raise_for_status()
        return response.json()

    def unlock(self, path: str, force: bool = False) -> None:
        params = {'force': 'true'} if force else {}
        response = self.client.session.delete(
            f'{self.client.base_url}/{path.lstrip("/")}/@lock',
            params=params
        )
        response.raise_for_status()

    def update_locked(self, path: str, data: dict, lock_token: str) -> dict:
        response = self.client.session.patch(
            f'{self.client.base_url}/{path.lstrip("/")}',
            json=data,
            headers={'Lock-Token': lock_token}
        )
        response.raise_for_status()
        return response.json()
```

---

## Version History

```javascript
// JavaScript - History
async function getHistory(client, path) {
  return client.get(`${path}/@history`);
}

async function getVersion(client, path, version) {
  return client.get(`${path}/@history/${version}`);
}

async function revertToVersion(client, path, version) {
  return client.fetch(`${path}/@history`, {
    method: 'PATCH',
    body: { version },
  });
}

// Usage
const history = await getHistory(client, '/my-doc');
console.log(`Total versions: ${history.length}`);
for (const entry of history) {
  console.log(`${entry.version}: ${entry.action} by ${entry.actor.id} at ${entry.time}`);
}
```

```python
# Python - History
def get_history(client, path: str) -> list:
    return client.get(f'{path}/@history')

def get_version(client, path: str, version: int) -> dict:
    return client.get(f'{path}/@history/{version}')

def revert_to_version(client, path: str, version: int) -> dict:
    response = client.session.patch(
        f'{client.base_url}/{path.lstrip("/")}/@history',
        json={'version': version}
    )
    response.raise_for_status()
    return response.json()
```

---

## Working Copy (Check-out/Check-in)

```javascript
// JavaScript - Working Copy
class WorkingCopyManager {
  constructor(client) {
    this.client = client;
  }

  async checkout(path) {
    return this.client.fetch(`${path}/@workingcopy`, { method: 'POST' });
  }

  async getStatus(path) {
    return this.client.get(`${path}/@workingcopy`);
  }

  async checkin(path) {
    return this.client.fetch(`${path}/@workingcopy`, { method: 'PATCH' });
  }

  async cancelCheckout(path) {
    return this.client.fetch(`${path}/@workingcopy`, { method: 'DELETE' });
  }
}

// Usage
const wc = new WorkingCopyManager(client);
const { '@id': workingCopyUrl } = await wc.checkout('/my-doc');
// Edit the working copy at workingCopyUrl
await client.update(workingCopyUrl, { title: 'Updated Title' });
// Check in changes
await wc.checkin('/my-doc');
```

---

## Relations Management

```javascript
// JavaScript - Relations
async function getRelations(client, options = {}) {
  const params = new URLSearchParams(options);
  return client.get(`/@relations?${params}`);
}

async function createRelations(client, items) {
  return client.fetch('/@relations', {
    method: 'POST',
    body: { items },
  });
}

async function deleteRelations(client, filter) {
  return client.fetch('/@relations', {
    method: 'DELETE',
    body: filter,
  });
}

// Usage
// Get all relations of a specific type
const relations = await getRelations(client, { relation: 'relatedItems' });

// Create a relation
await createRelations(client, [{
  relation: 'relatedItems',
  source: '/doc1',
  target: '/doc2',
}]);

// Delete by filter
await deleteRelations(client, { relation: 'relatedItems', source: '/doc1' });
```

---

## Sharing and Permissions

```javascript
// JavaScript - Sharing
async function getSharing(client, path, search = '') {
  const params = search ? `?search=${encodeURIComponent(search)}` : '';
  return client.get(`${path}/@sharing${params}`);
}

async function updateSharing(client, path, entries, inherit = true) {
  return client.fetch(`${path}/@sharing`, {
    method: 'POST',
    body: { entries, inherit },
  });
}

// Usage
const sharing = await getSharing(client, '/private-folder');
console.log('Available roles:', sharing.available_roles);

// Grant Editor role to a user
await updateSharing(client, '/private-folder', [{
  id: 'john',
  type: 'user',
  roles: { Editor: true, Reader: true },
}]);
```

```python
# Python - Sharing
def get_sharing(client, path: str, search: str = '') -> dict:
    params = {'search': search} if search else {}
    response = client.session.get(
        f'{client.base_url}/{path.lstrip("/")}/@sharing',
        params=params
    )
    response.raise_for_status()
    return response.json()

def update_sharing(client, path: str, entries: list, inherit: bool = True) -> None:
    response = client.session.post(
        f'{client.base_url}/{path.lstrip("/")}/@sharing',
        json={'entries': entries, 'inherit': inherit}
    )
    response.raise_for_status()
```

---

## URL Aliases

```javascript
// JavaScript - Aliases
async function getAliases(client, path = '') {
  const endpoint = path ? `${path}/@aliases` : '/@aliases';
  return client.get(endpoint);
}

async function createAliases(client, items, path = '') {
  const endpoint = path ? `${path}/@aliases` : '/@aliases';
  return client.fetch(endpoint, {
    method: 'POST',
    body: { items },
  });
}

async function deleteAliases(client, items, path = '') {
  const endpoint = path ? `${path}/@aliases` : '/@aliases';
  return client.fetch(endpoint, {
    method: 'DELETE',
    body: { items },
  });
}

// Usage
// Create an alias
await createAliases(client, [{ path: '/old-url' }], '/new-content');

// Get all aliases
const aliases = await getAliases(client);
```

---

## Translations (Multilingual)

```javascript
// JavaScript - Translations
async function getTranslations(client, path) {
  return client.get(`${path}/@translations`);
}

async function linkTranslation(client, path, targetId) {
  return client.fetch(`${path}/@translations`, {
    method: 'POST',
    body: { id: targetId },
  });
}

async function unlinkTranslation(client, path, language) {
  return client.fetch(`${path}/@translations`, {
    method: 'DELETE',
    body: { language },
  });
}

async function getTranslationLocator(client, path, targetLanguage) {
  return client.get(`${path}/@translation-locator?target_language=${targetLanguage}`);
}

// Usage with expand
const content = await client.get('/en/my-doc?expand=translations');
console.log(content['@components'].translations);
```

---

## Groups Management

```javascript
// JavaScript - Groups
class GroupsManager {
  constructor(client) {
    this.client = client;
  }

  async list(query = '', limit = 25) {
    const params = new URLSearchParams({ query, limit: String(limit) });
    return this.client.get(`/@groups?${params}`);
  }

  async get(groupname) {
    return this.client.get(`/@groups/${groupname}`);
  }

  async create(data) {
    return this.client.fetch('/@groups', {
      method: 'POST',
      body: data,
    });
  }

  async update(groupname, data) {
    return this.client.fetch(`/@groups/${groupname}`, {
      method: 'PATCH',
      body: data,
    });
  }

  async delete(groupname) {
    return this.client.fetch(`/@groups/${groupname}`, { method: 'DELETE' });
  }

  async addUsers(groupname, userIds) {
    const users = Object.fromEntries(userIds.map(id => [id, true]));
    return this.update(groupname, { users });
  }

  async removeUsers(groupname, userIds) {
    const users = Object.fromEntries(userIds.map(id => [id, false]));
    return this.update(groupname, { users });
  }
}

// Usage
const groups = new GroupsManager(client);
await groups.create({
  groupname: 'editors',
  title: 'Content Editors',
  roles: ['Editor'],
  users: ['john', 'jane'],
});
```

---

## Users Management

```javascript
// JavaScript - Users
class UsersManager {
  constructor(client) {
    this.client = client;
  }

  async list(options = {}) {
    const params = new URLSearchParams(options);
    return this.client.get(`/@users?${params}`);
  }

  async get(userid) {
    return this.client.get(`/@users/${userid}`);
  }

  async create(data) {
    return this.client.fetch('/@users', {
      method: 'POST',
      body: data,
    });
  }

  async update(userid, data) {
    return this.client.fetch(`/@users/${userid}`, {
      method: 'PATCH',
      body: data,
    });
  }

  async delete(userid) {
    return this.client.fetch(`/@users/${userid}`, { method: 'DELETE' });
  }

  async requestPasswordReset(userid) {
    return this.client.fetch(`/@users/${userid}/reset-password`, {
      method: 'POST',
    });
  }

  async setPassword(userid, newPassword, resetToken = null, oldPassword = null) {
    const body = { new_password: newPassword };
    if (resetToken) body.reset_token = resetToken;
    if (oldPassword) body.old_password = oldPassword;
    return this.client.fetch(`/@users/${userid}/reset-password`, {
      method: 'POST',
      body,
    });
  }
}
```

---

## Admin Operations

```javascript
// JavaScript - Add-ons Management
class AddonsManager {
  constructor(client) {
    this.client = client;
  }

  async list() {
    return this.client.get('/@addons');
  }

  async get(addonId) {
    return this.client.get(`/@addons/${addonId}`);
  }

  async install(addonId) {
    return this.client.fetch(`/@addons/${addonId}/install`, { method: 'POST' });
  }

  async uninstall(addonId) {
    return this.client.fetch(`/@addons/${addonId}/uninstall`, { method: 'POST' });
  }

  async upgrade(addonId) {
    return this.client.fetch(`/@addons/${addonId}/upgrade`, { method: 'POST' });
  }

  async getUpgradeable() {
    return this.client.get('/@addons?upgradeable=true');
  }
}

// Control Panels
async function getControlPanels(client) {
  return client.get('/@controlpanels');
}

async function getControlPanel(client, panelId) {
  return client.get(`/@controlpanels/${panelId}`);
}

async function updateControlPanel(client, panelId, data) {
  return client.fetch(`/@controlpanels/${panelId}`, {
    method: 'PATCH',
    body: data,
  });
}

// Registry
async function getRegistry(client, name = '') {
  if (name) {
    return client.get(`/@registry/${name}`);
  }
  return client.get('/@registry');
}

async function updateRegistry(client, values) {
  return client.fetch('/@registry/', {
    method: 'PATCH',
    body: values,
  });
}

// Site/System Info
async function getSiteInfo(client) {
  return client.get('/@site');
}

async function getSystemInfo(client) {
  return client.get('/@system');
}

async function getDatabaseInfo(client) {
  return client.get('/@database');
}
```

---

## Content Types Schema

```javascript
// JavaScript - Types Management
async function getTypes(client) {
  return client.get('/@types');
}

async function getTypeSchema(client, typeName) {
  return client.get(`/@types/${typeName}`);
}

async function addField(client, typeName, fieldData) {
  return client.fetch(`/@types/${typeName}`, {
    method: 'POST',
    body: fieldData,
  });
}

async function updateField(client, typeName, fieldName, properties) {
  return client.fetch(`/@types/${typeName}`, {
    method: 'PATCH',
    body: { properties: { [fieldName]: properties } },
  });
}

async function deleteField(client, typeName, fieldName) {
  return client.fetch(`/@types/${typeName}/${fieldName}`, { method: 'DELETE' });
}

// Usage
// Add a new email field
await addField(client, 'Document', {
  factory: 'Email',
  title: 'Author Email',
  required: true,
});
```

---

## Link Integrity Check

```javascript
// JavaScript - Check before delete
async function checkLinkIntegrity(client, uids) {
  const params = new URLSearchParams();
  uids.forEach(uid => params.append('uids', uid));
  return client.get(`/@linkintegrity?${params}`);
}

// Usage - Check before deleting content
const breaches = await checkLinkIntegrity(client, ['uid1', 'uid2']);
if (breaches.some(b => b.breaches.length > 0)) {
  console.warn('Deleting this content will break links!');
}
```

---

## Transactions Management

```javascript
// JavaScript - Transactions (Admin)
async function getTransactions(client) {
  return client.get('/@transactions');
}

async function revertTransactions(client, transactionIds) {
  return client.fetch('/@transactions', {
    method: 'PATCH',
    body: { transaction_ids: transactionIds },
  });
}

// Usage
const transactions = await getTransactions(client);
// Find and revert a specific transaction
const toRevert = transactions.find(t => t.description.includes('deleted'));
if (toRevert) {
  await revertTransactions(client, [toRevert.id]);
}
```

---

## Content Rules

```javascript
// JavaScript - Content Rules
async function getContentRules(client, path) {
  return client.get(`${path}/@content-rules/`);
}

async function assignRule(client, path, ruleId) {
  return client.fetch(`${path}/@content-rules/${ruleId}`, { method: 'POST' });
}

async function unassignRules(client, path, ruleIds) {
  return client.fetch(`${path}/@content-rules/`, {
    method: 'DELETE',
    body: { rule_ids: ruleIds },
  });
}

async function enableRule(client, path, ruleId) {
  return client.fetch(`${path}/@content-rules/`, {
    method: 'PATCH',
    body: { 'form.button.Enable': true, rule_id: ruleId },
  });
}
```

---

## Upgrade Management

```javascript
// JavaScript - Upgrade (Admin)
async function getUpgradeStatus(client) {
  return client.get('/@upgrade');
}

async function runUpgrade(client, dryRun = true) {
  return client.fetch('/@upgrade', {
    method: 'POST',
    body: { dry_run: dryRun },
  });
}

// Usage
const status = await getUpgradeStatus(client);
if (status.versions.fs !== status.versions.instance) {
  // Test upgrade first
  const dryRunResult = await runUpgrade(client, true);
  console.log(dryRunResult.report);

  // If looks good, run actual upgrade
  // await runUpgrade(client, false);
}
```
