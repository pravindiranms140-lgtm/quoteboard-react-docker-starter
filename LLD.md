# Low-Level Design (LLD)

## Project: QuoteBoard
- **Repository:** [quoteboard-react-docker-starter](https://github.com/pravindiranms140-lgtm/quoteboard-react-docker-starter)
- **Module Breakdown:** Component Hierarchy, Data Contracts, Routing Controllers, and Nginx / Docker Specifications

---

## 1. Directory Structure & File Hierarchy

```text
quoteboard-react-docker-starter/
├── .dockerignore              # Excludes node_modules, dist, git metadata from Docker context
├── .gitignore                # Git repository ignore rules
├── Dockerfile                # Multi-stage production Docker build manifest
├── HLD.md                    # High-Level Design document
├── index.html                # Vite HTML5 root entry point
├── LLD.md                    # Low-Level Design document
├── nginx.conf                # Nginx SPA try_files configuration
├── package.json              # NPM package metadata and build scripts
├── package-lock.json         # Deterministic dependency version lockfile
├── PRD.md                    # Product Requirements Document
├── README.md                 # Project summary and developer quickstart guide
├── vite.config.js            # Vite build configuration with React plugin
└── src/
    ├── App.css               # Styling rules for layout, cards, badges, and detail views
    ├── App.jsx               # Top-level routing switch (<Routes>, <Route>)
    ├── index.css             # Base reset, CSS variables, and global fonts
    ├── main.jsx              # React DOM root render and BrowserRouter provider
    ├── data/
    │   └── quotes.js         # Curated in-memory quotes data fixture
    └── pages/
        ├── Home.jsx          # Catalog landing view rendering quote cards
        └── QuoteDetail.jsx   # Dynamic single-quote view with route inspector
```

---

## 2. Component Specifications & Logic

### 2.1 `main.jsx` (Application Bootstrapper)
- **File:** `src/main.jsx`
- **Responsibilities:**
  - Initializes React root using `ReactDOM.createRoot`.
  - Wraps root component tree inside `<React.StrictMode>` and `<BrowserRouter>`.
- **Code Snippet:**
  ```jsx
  import React from 'react';
  import ReactDOM from 'react-dom/client';
  import { BrowserRouter } from 'react-router-dom';
  import App from './App';
  import './index.css';

  ReactDOM.createRoot(document.getElementById('root')).render(
    <React.StrictMode>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </React.StrictMode>
  );
  ```

---

### 2.2 `App.jsx` (Route Switch Controller)
- **File:** `src/App.jsx`
- **Responsibilities:**
  - Configures route table using `react-router-dom`.
  - Maps `/` to `<Home />`.
  - Maps `/quote/:id` to `<QuoteDetail />`.
  - Wraps views in `.app-shell` layout container.
- **Code Snippet:**
  ```jsx
  import React from 'react';
  import { Routes, Route } from 'react-router-dom';
  import Home from './pages/Home';
  import QuoteDetail from './pages/QuoteDetail';
  import './App.css';

  export default function App() {
    return (
      <div className="app-shell">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/quote/:id" element={<QuoteDetail />} />
        </Routes>
      </div>
    );
  }
  ```

---

### 2.3 `pages/Home.jsx` (Quote Catalog Page)
- **File:** `src/pages/Home.jsx`
- **Responsibilities:**
  - Imports curated `quotes` array from `../data/quotes`.
  - Renders application header (`<h1>QuoteBoard</h1>`, subtitle).
  - Iterates over quotes mapping them to `.quote-card` elements.
  - Generates client-side `<Link>` components to `/quote/${quote.id}`.
- **Render Hierarchy:**
  ```text
  <Home>
    ├── <header className="app-header">
    │     ├── <h1>QuoteBoard</h1>
    │     └── <p className="subtitle">Curated quotes for developers and thinkers</p>
    └── <main className="quotes-grid">
          └── {quotes.map} -> <div className="quote-card">
                ├── <span className="quote-category">{category}</span>
                ├── <p className="quote-text">"{text}"</p>
                ├── <p className="quote-author">— {author}</p>
                └── <Link to="/quote/:id">View Quote #{id} →</Link>
  ```

---

### 2.4 `pages/QuoteDetail.jsx` (Single Quote Detail View)
- **File:** `src/pages/QuoteDetail.jsx`
- **Responsibilities:**
  - Extracts route parameter `:id` using `useParams()`.
  - Converts `id` string to integer via `parseInt(id, 10)`.
  - Looks up matching quote in `quotes` collection.
  - **Found State:** Renders `.featured-card` with quote badge, category, text, author, and route status indicator.
  - **Not Found State (`!quote`):** Renders `.detail-container.not-found` with "Quote Not Found" message and a button returning to `/`.
- **Lookup Logic:**
  ```javascript
  const { id } = useParams();
  const quote = quotes.find((q) => q.id === parseInt(id, 10));
  ```

---

## 3. Data Schema & Model Specification

The quote data model is defined in `src/data/quotes.js`:

```typescript
interface Quote {
  id: number;          // Unique numerical identifier (1-based index)
  text: string;        // Full quotation text
  author: string;      // Person or thinker credited
  category: string;    // Classification tag (e.g. Inspiration, Technology, Design)
}
```

### Initial Curated Dataset:
| ID | Author | Category | Content |
|---|---|---|---|
| **1** | Franklin D. Roosevelt | Inspiration | *"The only limit to our realization of tomorrow is our doubts of today."* |
| **2** | Edsger W. Dijkstra | Technology | *"Simplicity is prerequisite for reliability."* |
| **3** | Martin Fowler | Software Engineering | *"Any fool can write code that a computer can understand. Good programmers write code that humans can understand."* |
| **4** | John Johnson | Problem Solving | *"First, solve the problem. Then, write the code."* |
| **5** | Kent Beck | Design | *"Make it work, make it right, make it fast."* |

---

## 4. Infrastructure & Server Configurations

### 4.1 Multi-Stage Dockerfile Analysis (`Dockerfile`)
```dockerfile
# Stage 1: Build the React application
FROM node:20-alpine AS build

WORKDIR /app

# Copy package manifests before source code to leverage Docker layer caching
COPY package*.json ./

# Install dependencies cleanly and deterministically
RUN npm ci

# Copy remaining source code and generate production build
COPY . .
RUN npm run build

# Stage 2: Serve the static build with Nginx
FROM nginx:alpine

# Copy custom Nginx configuration with SPA try_files fallback
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy only the compiled static assets from Stage 1
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

#### Layer Optimization Strategy:
- **Manifest caching:** `COPY package*.json ./` followed by `RUN npm ci` ensures dependencies are re-installed only when dependencies change, not when application code changes.
- **Stage separation:** The `node:20-alpine` build image containing compilers, npm cache, and source files is discarded; only the compiled `/app/dist` folder is transferred to the lightweight `nginx:alpine` runtime image.

---

### 4.2 Nginx SPA Configuration Analysis (`nginx.conf`)
```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

#### Directive Breakdown:
- `listen 80;`: Listens on HTTP standard container port 80.
- `root /usr/share/nginx/html;`: Points root web directory to compiled static assets from Vite.
- `try_files $uri $uri/ /index.html;`:
  1. Tests if the exact file exists on disk (e.g., `/assets/index-D7x.js`). If found, streams file with appropriate MIME type.
  2. Tests if a directory exists at `$uri/`.
  3. If both tests fail (as in deep routes like `/quote/1`), falls back to `/index.html` with status HTTP 200, allowing React Router to take over routing on the client.

---

### 4.3 Build Context Exclusion (`.dockerignore`)
```text
node_modules
dist
.git
.env*
```
Prevents leaking local modules, build directories, and git history into the Docker build context.

---

## 5. Detailed Sequence Flows

### 5.1 Hard Browser Refresh on `/quote/1`
```
User Browser                  Nginx (Port 80)                 React App Engine
     │                               │                              │
     ├─── 1. GET /quote/1 ──────────►│                              │
     │                               │ 2. Check /quote/1 on disk    │
     │                               │    (Not Found)               │
     │                               │ 3. Fallback to /index.html   │
     │◄── 4. HTTP 200 (index.html) ──┤                              │
     │                               │                              │
     ├─── 5. GET /assets/app.js ────►│                              │
     │◄── 6. HTTP 200 (JS bundle) ───┤                              │
     │                                                              │
     ├─── 7. Executes Bundle ──────────────────────────────────────►│
     │                                                              │ 8. React Router reads pathname
     │                                                              │ 9. Matches /quote/:id (id=1)
     │                                                              │ 10. Looks up quote ID 1
     │◄── 11. Displays Quote #1 Detail View ────────────────────────┤
```

### 5.2 Invalid Route Request (`/quote/999`)
```
User Browser                  Nginx (Port 80)                 React App Engine
     │                               │                              │
     ├─── 1. GET /quote/999 ────────►│                              │
     │                               │ 2. Fallback to /index.html   │
     │◄── 3. HTTP 200 (index.html) ──┤                              │
     │                                                              │
     ├─── 4. Executes Bundle ──────────────────────────────────────►│
     │                                                              │ 5. React Router matches id=999
     │                                                              │ 6. Lookup returns undefined
     │◄── 7. Renders "Quote Not Found" UI ──────────────────────────┤
```

---

## 6. Verification & Quality Assurance Checklist
- [x] **Vite Dev Server:** Starts with zero errors (`npm run dev`).
- [x] **Production Build:** Generates static assets in `/dist` without errors (`npm run build`).
- [x] **Docker Image Build:** Builds cleanly via multi-stage cache (`docker build -t quoteboard .`).
- [x] **Docker Run:** Listens on configured port (`docker run -d -p 8080:80 quoteboard`).
- [x] **Direct Path Refresh:** Hard refresh on `/quote/1` returns HTTP 200 (Nginx `try_files` working).
- [x] **Boundary Check:** Non-numeric or missing quote IDs render graceful "Quote Not Found" UI.
