---
name: frappe-vue-pwa
description: Build full-stack Frappe v15 + Vue 3 SPA applications with PWA support, Web Push notifications, and real-time features. Use this skill when creating Frappe custom apps with Vue frontends, PWA wrappers, or project management tools.
triggers:
  - frappe vue app
  - frappe spa
  - frappe pwa
  - vue 3 frappe
  - custom frappe app
  - frappe frontend
  - bench custom app
  - frappe push notifications
  - frappe service worker
  - frappe vue router
  - frappe pinia store
  - frappe real-time
---

# Frappe v15 + Vue 3 SPA + PWA Development

This skill documents proven patterns for building production-grade Frappe v15 applications with a decoupled Vue 3 SPA frontend, PWA capabilities, and Web Push notifications. Based on the Next PMS project (a full project management system).

## Architecture Overview

```
frappe-bench/apps/your_app/
├── frontend/                    # Vue 3 SPA (Vite build)
│   ├── src/
│   │   ├── main.js              # Vue app bootstrap
│   │   ├── App.vue              # Root with sidebar + router-view
│   │   ├── router/index.js      # Vue Router (lazy-loaded, guarded)
│   │   ├── store/               # Pinia stores (domain-driven)
│   │   ├── views/               # Page components
│   │   ├── components/          # Reusable UI components
│   │   ├── utils/frappe.js      # Frappe API wrapper with caching
│   │   ├── utils/realtime.js    # Socket.IO composables
│   │   └── composables/         # Vue 3 composables
│   ├── vite.config.js
│   └── package.json
├── your_app/
│   ├── hooks.py                 # App config, doc events, routes
│   ├── api/                     # Whitelisted Python endpoints
│   │   ├── crud.py              # CRUD operations
│   │   ├── dashboard.py         # Aggregated queries
│   │   ├── permissions.py       # Role-based access
│   │   ├── push.py              # Web Push (VAPID)
│   │   └── pwa.py               # Service worker serving
│   ├── www/your-app/index.py    # SPA entry (bootstrap + asset loading)
│   ├── public/
│   │   ├── manifest.json        # PWA manifest
│   │   ├── js/sw.js             # Service worker
│   │   ├── icons/               # PWA icons
│   │   └── frontend/            # Built Vue assets (Vite output)
│   └── your_app/doctype/        # Frappe DocTypes
└── pyproject.toml
```

## Step-by-Step: Creating a New Frappe + Vue App

### Step 1: Scaffold the Frappe App

```bash
cd ~/frappe-bench
bench new-app your_app
```

### Step 2: Set Up Vue 3 Frontend

Create `frontend/` directory with Vite + Vue 3:

**frontend/package.json:**
```json
{
  "name": "your-app-frontend",
  "type": "module",
  "dependencies": {
    "pinia": "^2.1.0",
    "vue": "^3.4.0",
    "vue-router": "^4.3.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "sass-embedded": "^1.98.0",
    "vite": "^5.4.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  }
}
```

**frontend/vite.config.js:**
```javascript
import path from "path";
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig(({ command }) => ({
  plugins: [vue()],
  base: command === "serve" ? "/" : "/assets/your_app/frontend/",
  build: {
    outDir: path.resolve(__dirname, "../your_app/public/frontend"),
    emptyOutDir: true,
    target: "es2020",
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
  server: {
    port: 8081,
    proxy: {
      "/api": { target: "http://localhost:8000" },
      "/assets": { target: "http://localhost:8000" },
    },
  },
}));
```

### Step 3: Create the Frappe API Wrapper

This is the most critical utility. It wraps Frappe's REST API with request deduplication and TTL caching.

**frontend/src/utils/frappe.js:**
```javascript
const pendingRequests = new Map();
const cache = new Map();

function getCsrfToken() {
  const cookie = document.cookie.match(/csrf_token=([^;]+)/);
  if (cookie) return decodeURIComponent(cookie[1]);
  if (window.csrf_token) return window.csrf_token;
  if (window.frappe?.csrf_token) return window.frappe.csrf_token;
  return null;
}

export async function call(method, args = {}, opts = {}) {
  const cacheKey = JSON.stringify({ method, args });

  // Return cached if valid
  if (opts.cache) {
    const cached = cache.get(cacheKey);
    if (cached && Date.now() - cached.time < (opts.cache || 5000)) {
      return cached.data;
    }
  }

  // Deduplicate concurrent identical requests
  if (pendingRequests.has(cacheKey)) {
    return pendingRequests.get(cacheKey);
  }

  const promise = fetch(`/api/method/${method}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Frappe-CSRF-Token": getCsrfToken(),
    },
    credentials: "include",
    body: JSON.stringify(args),
  })
    .then(async (r) => {
      const data = await r.json();
      if (data.exc) throw new Error(data.exc);
      return data.message;
    })
    .finally(() => pendingRequests.delete(cacheKey));

  pendingRequests.set(cacheKey, promise);

  const result = await promise;
  if (opts.cache) {
    cache.set(cacheKey, { data: result, time: Date.now() });
  }
  return result;
}

export async function getList(doctype, options = {}) {
  return call("frappe.client.get_list", {
    doctype,
    ...options,
  });
}

export async function getDoc(doctype, name) {
  return call("frappe.client.get", { doctype, name });
}

export async function setValue(doctype, name, field, value) {
  return call("frappe.client.set_value", { doctype, name, fieldname: field, value });
}
```

### Step 4: Set Up Vue Router with Lazy Loading

**frontend/src/router/index.js:**
```javascript
import { createRouter, createWebHistory } from "vue-router";

const routes = [
  { path: "/", redirect: "/dashboard" },
  {
    path: "/dashboard",
    component: () => import("@/views/DashboardView.vue"),
  },
  {
    path: "/project/:id",
    component: () => import("@/views/ProjectDetailView.vue"),
    props: true,
  },
  {
    path: "/task/:id",
    component: () => import("@/views/TaskDetailView.vue"),
    props: true,
  },
];

const router = createRouter({
  history: createWebHistory("/your-app/"),
  routes,
});

// Route guard for role-based access
router.beforeEach((to) => {
  const settings = useSettingsStore();
  if (to.meta.requiresAdmin && !settings.isAdmin) {
    return "/dashboard";
  }
});

export default router;
```

### Step 5: Create Pinia Stores (Domain-Driven)

**Pattern: One store per domain entity.**

```javascript
// frontend/src/store/projects.js
import { defineStore } from "pinia";
import { call } from "@/utils/frappe";

export const useProjectStore = defineStore("projects", {
  state: () => ({
    projects: [],
    currentProject: null,
    loading: false,
  }),
  actions: {
    async fetchProjects() {
      this.loading = true;
      try {
        this.projects = await call(
          "your_app.api.dashboard.get_all_projects_summary",
          {},
          { cache: 30000 }
        );
      } finally {
        this.loading = false;
      }
    },
  },
});
```

### Step 6: SPA Entry Point (Frappe WWW Page)

**your_app/www/your-app/index.py:**
```python
import os, json, re, frappe

no_cache = 1

def get_context(context):
    csrf_token = frappe.sessions.get_csrf_token()
    frappe.db.commit()

    # Find built Vue assets (hashed filenames)
    assets_dir = frappe.get_app_path("your_app", "public", "frontend", "assets")
    js_file = css_file = ""
    if os.path.exists(assets_dir):
        for f in os.listdir(assets_dir):
            if f.startswith("index-") and f.endswith(".js"):
                js_file = f
            elif f.startswith("index-") and f.endswith(".css"):
                css_file = f

    context.update({
        "csrf_token": csrf_token,
        "js_file": js_file,
        "css_file": css_file,
    })
```

**your_app/www/your-app/index.html:**
```html
{% extends "templates/web.html" %}
{% block page_content %}
<div id="app"></div>
<script>
  window.csrf_token = "{{ csrf_token }}";
</script>
{% if css_file %}<link rel="stylesheet" href="/assets/your_app/frontend/assets/{{ css_file }}">{% endif %}
{% if js_file %}<script type="module" src="/assets/your_app/frontend/assets/{{ js_file }}"></script>{% endif %}
{% endblock %}
```

### Step 7: Configure hooks.py

```python
app_name = "your_app"
app_title = "Your App"

# SPA route catch-all
website_route_rules = [
    {"from_route": "/your-app/<path:app_path>", "to_route": "your-app"}
]

# Service worker headers
after_request = ["your_app.utils.add_sw_headers"]

# Row-level security
permission_query_conditions = {
    "Your DocType": "your_app.api.permissions.query_conditions",
}
has_permission = {
    "Your DocType": "your_app.api.permissions.has_permission",
}

# Doc event hooks for side effects
doc_events = {
    "Your DocType": {
        "on_update": "your_app.api.notifications.on_update",
    },
}
```

### Step 8: PWA Setup

**your_app/public/manifest.json:**
```json
{
  "name": "Your App",
  "short_name": "YourApp",
  "start_url": "/your-app/",
  "scope": "/your-app/",
  "display": "standalone",
  "theme_color": "#ffffff",
  "icons": [
    {"src": "/assets/your_app/icons/icon-192x192.png", "sizes": "192x192", "type": "image/png"},
    {"src": "/assets/your_app/icons/icon-512x512.png", "sizes": "512x512", "type": "image/png"}
  ]
}
```

**Service Worker serving (your_app/api/pwa.py):**
```python
import frappe

@frappe.whitelist(allow_guest=True)
def sw():
    sw_path = frappe.get_app_path("your_app", "public", "js", "sw.js")
    with open(sw_path, "r") as f:
        content = f.read()
    frappe.response["type"] = "page"
    frappe.response["page_content"] = content
    frappe.local.response.headers["Content-Type"] = "application/javascript"
    frappe.local.response.headers["Service-Worker-Allowed"] = "/your-app/"
```

**Service Worker header utility (your_app/utils.py):**
```python
import frappe

def add_sw_headers(response):
    if hasattr(frappe.local, "request") and "/sw" in frappe.local.request.path:
        response.headers["Service-Worker-Allowed"] = "/your-app/"
    return response
```

### Step 9: Web Push Notifications

**your_app/api/push.py:**
```python
import json, frappe
from pywebpush import webpush, WebPushException

@frappe.whitelist()
def get_vapid_public_key():
    return frappe.conf.get("push_vapid_public_key")

@frappe.whitelist()
def save_push_subscription(subscription):
    sub = json.loads(subscription) if isinstance(subscription, str) else subscription
    if not frappe.db.exists("Your Push Subscription", {"endpoint": sub["endpoint"]}):
        frappe.get_doc({
            "doctype": "Your Push Subscription",
            "user": frappe.session.user,
            "endpoint": sub["endpoint"],
            "p256dh": sub["keys"]["p256dh"],
            "auth": sub["keys"]["auth"],
        }).insert(ignore_permissions=True)
    frappe.db.commit()

@frappe.whitelist()
def send_push_to_user(user, title, body, url=None):
    subs = frappe.get_all("Your Push Subscription",
        filters={"user": user},
        fields=["endpoint", "p256dh", "auth"])

    for sub in subs:
        try:
            webpush(
                subscription_info={
                    "endpoint": sub.endpoint,
                    "keys": {"p256dh": sub.p256dh, "auth": sub.auth},
                },
                data=json.dumps({"title": title, "body": body, "url": url}),
                vapid_private_key=frappe.conf.get("push_vapid_private_key"),
                vapid_claims={"sub": f"mailto:{frappe.conf.get('push_vapid_email')}"},
            )
        except WebPushException as e:
            if e.response and e.response.status_code in (404, 410):
                frappe.delete_doc("Your Push Subscription", sub.name)
```

### Step 10: Permission System Pattern

**your_app/api/permissions.py:**
```python
import frappe

def is_admin():
    return "System Manager" in frappe.get_roles()

def get_user_projects():
    """Returns None if admin (access all), or list of project names."""
    if is_admin():
        return None
    return frappe.get_all("Your Project Member",
        filters={"user": frappe.session.user},
        pluck="parent")

def query_conditions(user):
    """Row-level security for list queries."""
    if is_admin():
        return ""
    projects = get_user_projects()
    if not projects:
        return "1=0"
    return f"`tabYour Project`.name IN ({','.join(repr(p) for p in projects)})"

def check_access(project):
    projects = get_user_projects()
    if projects is not None and project not in projects:
        frappe.throw("No access", frappe.PermissionError)
```

## Key Patterns & Best Practices

### API Design
- **One endpoint per action**: `update_task_status()`, not a generic PATCH
- **Bulk fetch**: `get_bulk_task_assignees(task_names)` instead of N+1 queries
- **Cache dashboard queries**: 30-second TTL for summary data
- **Enqueue heavy work**: `frappe.enqueue()` for push notifications, emails

### Frontend Patterns
- **Lazy-load views**: `() => import("@/views/View.vue")` in router
- **Domain stores**: One Pinia store per entity (projects, tasks, timer)
- **Event bus for local actions**: Emit after own user's changes, listen for refresh
- **Real-time for remote changes**: Socket.IO for other users' actions

### Deployment Workflow
```bash
# On server:
cd ~/frappe-bench/apps/your_app && git pull
cd ~/frappe-bench
bench migrate                    # Update DB schema
bench build --app your_app       # Build Frappe bundle
cd apps/your_app/frontend && yarn && yarn build  # Build Vue SPA
sudo supervisorctl restart all   # Restart workers
```

### Common Pitfalls

1. **`bench build` vs `yarn build`**: `bench build` only compiles the Frappe bundle (hooks JS/CSS). The Vue SPA must be built separately with `cd frontend && yarn build`.

2. **`bench migrate` on wrong site**: Multi-site setups require `bench use <site>` or `bench --site <site> migrate`. Check with `ls sites/`.

3. **Null API parameters**: Frappe doesn't handle `null` well. Use conditional arg inclusion:
   ```javascript
   const args = { task: id, url: url };
   const title = titleRef.value.trim();
   if (title) args.title = title;
   await call("your_app.api.method", args);
   ```

4. **Service Worker scope**: SW must be served from the app's scope path. Use `Service-Worker-Allowed` header to allow broader scope.

5. **CSRF token in SPA**: Must be injected via `window.csrf_token` in the entry template, or read from cookies.

6. **Frappe caching on migrate**: After adding new DocTypes, always run `bench migrate` on the correct site. The error `ProgrammingError: ('DocType', 'Your DocType')` means the table wasn't created.

7. **Permission query conditions**: Must return empty string `""` for full access, or `"1=0"` for no access. Never return `None`.

8. **Vue Router base path**: Must match `website_route_rules` in hooks.py. Use `createWebHistory("/your-app/")`.

### TWA Android Wrapper (Optional)

To wrap the PWA as an Android APK using Trusted Web Activity:

1. Create Android project with `build.gradle` targeting SDK 34
2. Add dependencies: `androidx.browser:browser:1.8.0`, `com.google.androidbrowserhelper:androidbrowserhelper:2.5.0`
3. Configure `AndroidManifest.xml` with `LauncherActivity` pointing to your PWA URL
4. Build with `./gradlew assembleDebug`
5. For production: generate signed APK with keystore, set up Digital Asset Links

## Quick Reference

| Task | Command |
|------|---------|
| New Frappe app | `bench new-app your_app` |
| Install on site | `bench --site your_site install-app your_app` |
| Dev frontend | `cd frontend && yarn dev` |
| Build frontend | `cd frontend && yarn build` |
| Build Frappe bundle | `bench build --app your_app` |
| Migrate DB | `bench migrate` |
| Create DocType | Add JSON + Python in `your_app/your_app/doctype/` |
| Add API endpoint | Add `@frappe.whitelist()` function in `your_app/api/` |
| Generate VAPID keys | `bench execute your_app.api.push.generate_vapid_keys` |
| Restart server | `sudo supervisorctl restart all` |

## Workflow: Feature Confirmation Protocol

Every time a feature is confirmed working on the server:

1. **Update CLAUDE.md** in the project repo — add entry under "Confirmed Features" with date and brief description
2. **Update this skill file** if the feature introduces a new reusable pattern (e.g., new API pattern, new component type, new integration)
3. **Commit the docs** alongside the feature code

This ensures future sessions have full context of what's deployed and working.
