# High-Level Design (HLD)

## Project: QuoteBoard
- **Repository:** [quoteboard-react-docker-starter](https://github.com/pravindiranms140-lgtm/quoteboard-react-docker-starter)
- **Architecture Pattern:** Decoupled Single Page Application (SPA) with Containerized Static Reverse Proxy

---

## 1. System Overview & Architecture Goals
QuoteBoard is structured as a client-side Single Page Application (SPA) built with React 18, React Router v6, and Vite, packaged and served via an ultra-lightweight Docker container running Nginx on Alpine Linux.

### Key Architectural Objectives:
1. **Instantaneous Client Navigation:** Allow instantaneous transitions between views without requesting whole HTML documents from the server.
2. **Deterministic SPA Route Resolution:** Solve the deep route / browser refresh 404 issue by using Nginx URL rewriting.
3. **Hardened Production Footprint:** Deliver an isolated, zero-Node runtime container with minimal dependencies and attack surface.

---

## 2. High-Level Architecture Diagram

```
+-----------------------------------------------------------------------------+
|                                Client Tier                                  |
|   [ End-User Web Browser (Desktop / Mobile) ]                               |
|        │                                                                    |
|        │  HTTP / HTTPS Request (e.g. GET /quote/1 or GET /)                 |
|        ▼                                                                    |
+-----------------------------------------------------------------------------+
|                         Docker Container (Port 80)                          |
|   +----------------------------------------------------------------------+  |
|   |                           Nginx Web Server                           |  |
|   |                                                                      |  |
|   |    URI Resolution Pipeline:                                          |  |
|   |      1. Does physical file exist at /usr/share/nginx/html$uri?       |  |
|   |         --> Yes: Serve static file (e.g., .js, .css, .ico)          |  |
|   |      2. Does physical directory exist at $uri/?                      |  |
|   |         --> Yes: Serve directory index                               |  |
|   |      3. Fallback: Rewrite to /index.html with HTTP 200               |  |
|   +----------------------------------------------------------------------+  |
|                                       │                                     |
|                                       ▼                                     |
|   +----------------------------------------------------------------------+  |
|   |                    Static Assets (/usr/share/nginx/html)             |  |
|   |    ├── index.html                                                    |  |
|   |    └── assets/ (app.js, styles.css, bundled chunks)                  |  |
|   +----------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------+
                                        │
                                        ▼ (Returned to Browser)
+-----------------------------------------------------------------------------+
|                    Browser Runtime (React 18 SPA Engine)                    |
|   +----------------------------------------------------------------------+  |
|   |                       React Router v6 (<Routes>)                     |  |
|   |                                                                      |  |
|   |    Route Path "/"           ──►  Renders <Home />                    |  |
|   |    Route Path "/quote/:id"  ──►  Renders <QuoteDetail />             |  |
|   +----------------------------------------------------------------------+  |
|   |                          In-Memory Data Store                        |  |
|   |    `src/data/quotes.js` (Curated quotes dataset)                     |  |
|   +----------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------+
```

---

## 3. Technology Stack & Component Justification

| Layer | Component | Version | Role & Justification |
|---|---|---|---|
| **UI Framework** | React | ^18.3.1 | Declarative component hierarchy, virtual DOM reconciliation, component state |
| **Routing** | React Router DOM | ^6.28.0 | Declarative client-side routing, URL parameter extraction (`useParams`), browser history management |
| **Build Tooling** | Vite | ^5.4.11 | Fast Rollup-based static compilation, ES module bundling, asset minification |
| **Web Server** | Nginx | Alpine Linux | Ultra-fast static asset delivery, event-driven non-blocking I/O, configurable URL rewriting |
| **Container Engine** | Docker | Engine 20+ | Reproducible multi-stage build pipeline, container isolation, minimal runtime size |

---

## 4. Multi-Stage Build & Packaging Strategy

To ensure security and minimize production resource usage, the containerization follows a **multi-stage build strategy**:

```
+-------------------------------------------------------------------+
| Stage 1: Build Phase (Base: node:20-alpine AS build)              |
|   1. Initialize working directory: /app                           |
|   2. Copy package*.json dependency manifests                     |
|   3. Run `npm ci` (deterministic dependency lockfile install)     |
|   4. Copy application source tree                                 |
|   5. Run `npm run build` (outputs compiled SPA to /app/dist)      |
+-------------------------------------------------------------------+
                               │
               Copies ONLY /app/dist static build
                               │
                               ▼
+-------------------------------------------------------------------+
| Stage 2: Runtime Phase (Base: nginx:alpine)                       |
|   1. Copy custom `nginx.conf` -> /etc/nginx/conf.d/default.conf   |
|   2. Copy static build -> /usr/share/nginx/html                   |
|   3. Expose port 80                                               |
|   4. Start Nginx daemon in foreground (`daemon off;`)             |
+-------------------------------------------------------------------+
```

### Why Node.js is Excluded from the Final Container:
1. **Security / Minimal Attack Surface:** Strips Node.js, npm, devDependencies, and shell compilers from production.
2. **Minimal Image Size:** Final production image content size is reduced to approximately ~26.4 MB (compared to >400 MB for a Node runtime container).
3. **High Performance & Concurrency:** Nginx serves static files using native kernel sendfile calls with negligible CPU and RAM consumption.

---

## 5. Routing Architecture & The SPA Refresh Problem

### 5.1 The Fundamental Challenge
In a Single Page Application, client-side route transitions (like navigating from `/` to `/quote/1`) occur entirely inside the browser's JavaScript engine via the HTML5 History API (`pushState`). The web server is not contacted during these transitions.

However, when a user:
- Directly navigates to `http://localhost:8080/quote/1`
- Refreshes the page while on `http://localhost:8080/quote/1`
- Shares or bookmarks the URL

The browser issues a direct HTTP `GET /quote/1` to the server. Without special configuration, Nginx looks on disk for `/usr/share/nginx/html/quote/1` or `/usr/share/nginx/html/quote/1/index.html`. Since that path does not exist, standard servers return **HTTP 404 Not Found**.

### 5.2 The Solution: Nginx `try_files` Directive
In `nginx.conf`, the routing fallback is handled as follows:
```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```
1. **`$uri`**: Nginx checks if a static file matching the URI exists (e.g., `/assets/index-D7x.js`). If so, it serves it directly.
2. **`$uri/`**: Nginx checks if a directory matching the URI exists.
3. **`/index.html`**: If neither matches, Nginx internally rewrites the request to `/index.html` and responds with HTTP 200.
4. **Client-Side Resolution:** The browser loads `index.html` and executes React Router, which inspects `window.location.pathname` (`/quote/1`) and renders the `<QuoteDetail />` component.

---

## 6. Non-Functional Architecture & Security Controls
- **Build Context Exclusion (`.dockerignore`):** Omits `node_modules`, `.git`, `dist`, and environment files, ensuring build context upload remains under 100 kB.
- **Reproducible Installs:** `npm ci` guarantees that builds use exact dependency versions pinned in `package-lock.json`.
- **Portability:** The container can run anywhere Docker runs (local development, AWS ECS, GCP Cloud Run, Kubernetes).
- **Graceful Error Handling:** Custom 50x error page handling configured in Nginx.
