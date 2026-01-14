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
