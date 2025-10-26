# ENGARAGE - Codespace-ready Scaffold

This document contains a complete, runnable scaffold for a minimal ENGARAGE-like app that you can open in **GitHub Codespaces**. It includes a **React (Vite) frontend**, a **Node + Express backend**, Docker configuration, and a Codespaces devcontainer. Copy these files into a new GitHub repository, then open the repo in Codespaces.

---

## Quick steps (summary)
1. Create a new GitHub repository and copy the files below into it (preserve paths).
2. Push to GitHub.
3. In the GitHub repo page, click **Code → Codespaces → Create codespace on main** (or open Codespaces from VS Code).
4. Codespaces will build the devcontainer and start both services. See logs in the Terminal.
5. Frontend: http://localhost:5173 (forwarded by Codespaces). Backend: http://localhost:4000

---

## File: `README.md`
```md
# ENGARAGE - Codespace Scaffold

Minimal full-stack scaffold (Vite React frontend + Express backend) prepared for GitHub Codespaces.

## How to run in Codespaces
1. Open this repository in GitHub Codespaces.
2. Codespaces will automatically build the devcontainer and run the `start` task which runs `docker-compose up --build`.
3. The frontend will be available at the forwarded port 5173; backend at 4000.

## Local (non-Codespace) quick run
- Requires Docker & Docker Compose installed.

```bash
docker-compose up --build
```

---

## Project structure
- .devcontainer/
- docker-compose.yml
- frontend/
- backend/

---

## Notes
- This scaffold is intentionally minimal to jumpstart development. Add authentication, database (Postgres), storage, and third-party integrations (Twilio, SendGrid, Stripe) as you progress.
```
```

---

## File: `.devcontainer/devcontainer.json`
```json
{
  "name": "ENGARAGE Codespace",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "workspace",
  "workspaceFolder": "/workspace",
  "shutdownAction": "stopCompose",
  "settings": {
    "terminal.integrated.shell.linux": "/bin/bash"
  },
  "extensions": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "ms-azuretools.vscode-docker"
  ],
  "postCreateCommand": "./scripts/post_create.sh",
  "forwardPorts": [5173, 4000],
  "remoteUser": "vscode"
}
```

---

## File: `docker-compose.yml`
```yaml
version: '3.8'
services:
  frontend:
    build: ./frontend
    ports:
      - '5173:5173'
    volumes:
      - ./frontend:/workspace/frontend:cached
    environment:
      - NODE_ENV=development
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - '4000:4000'
    volumes:
      - ./backend:/workspace/backend:cached
    environment:
      - NODE_ENV=development

  workspace:
    image: mcr.microsoft.com/vscode/devcontainers/javascript-node:0-18
    volumes:
      - ./:/workspace:cached
    working_dir: /workspace
    command: sleep infinity
    depends_on:
      - frontend
      - backend
```

---

## File: `scripts/post_create.sh`
```bash
#!/usr/bin/env bash
set -e
# Run installs for frontend and backend inside the workspace container
echo "Installing frontend dependencies..."
cd /workspace/frontend
npm install

echo "Installing backend dependencies..."
cd /workspace/backend
npm install

echo "Post-create finished. To run the app: docker-compose up --build"
```

Make the script executable by `chmod +x scripts/post_create.sh`.

---

## Backend

### File: `backend/Dockerfile`
```Dockerfile
FROM node:18-bullseye
WORKDIR /workspace/backend
COPY package*.json ./
RUN npm install
COPY . ./
EXPOSE 4000
CMD ["npm", "run", "dev"]
```

### File: `backend/package.json`
```json
{
  "name": "engarage-backend",
  "version": "0.1.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon --watch . --exec node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
```

### File: `backend/index.js`
```js
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const app = express();
app.use(cors());
app.use(bodyParser.json());

let nextId = 1;
const workOrders = [
  { id: nextId++, title: 'Replace brake pads - Car A', status: 'Queued' },
  { id: nextId++, title: 'Oil change - Car B', status: 'In Progress' }
];

app.get('/api/health', (req, res) => res.json({ ok: true }));

app.get('/api/workorders', (req, res) => {
  res.json(workOrders);
});

app.post('/api/workorders', (req, res) => {
  const { title } = req.body;
  if (!title) return res.status(400).json({ error: 'title required' });
  const newOrder = { id: nextId++, title, status: 'Queued' };
  workOrders.unshift(newOrder);
  res.status(201).json(newOrder);
});

app.patch('/api/workorders/:id/status', (req, res) => {
  const id = Number(req.params.id);
  const { status } = req.body;
  const order = workOrders.find(o => o.id === id);
  if (!order) return res.status(404).json({ error: 'not found' });
  order.status = status || order.status;
  res.json(order);
});

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Backend listening on http://localhost:${PORT}`));
```

---

## Frontend

### File: `frontend/Dockerfile`
```Dockerfile
FROM node:18-bullseye
WORKDIR /workspace/frontend
COPY package*.json ./
RUN npm install
COPY . ./
EXPOSE 5173
CMD ["npm", "run", "dev"]
```

### File: `frontend/package.json`
```json
{
  "name": "engarage-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "vite": "^5.1.0",
    "@vitejs/plugin-react": "^4.0.0"
  }
}
```

### File: `frontend/vite.config.js`
```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
    port: 5173,
    proxy: {
      '/api': 'http://backend:4000'
    }
  }
})
```

### File: `frontend/index.html`
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ENGARAGE - Demo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

### File: `frontend/src/main.jsx`
```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(<App />)
```

### File: `frontend/src/App.jsx`
```jsx
import React from 'react'
import WorkOrdersList from './pages/WorkOrders/WorkOrdersList'

export default function App() {
  return (
    <div style={{ fontFamily: 'Inter, Arial, sans-serif', padding: 16 }}>
      <header style={{ marginBottom: 24 }}>
        <h1>ENGARAGE — Demo (Codespace)</h1>
      </header>
      <main>
        <WorkOrdersList />
      </main>
    </div>
  )
}
```

### File: `frontend/src/pages/WorkOrders/WorkOrdersList.jsx`
```jsx
import React, { useEffect, useState } from 'react'

export default function WorkOrdersList() {
  const [orders, setOrders] = useState([])
  const [loading, setLoading] = useState(true)
  const [newTitle, setNewTitle] = useState('')

  useEffect(() => {
    fetch('/api/workorders')
      .then((r) => r.json())
      .then((data) => {
        setOrders(data)
        setLoading(false)
      })
      .catch((err) => {
        console.error(err)
        setLoading(false)
      })
  }, [])

  async function createOrder(e) {
    e.preventDefault()
    if (!newTitle) return
    const res = await fetch('/api/workorders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title: newTitle })
    })
    const created = await res.json()
    setOrders((prev) => [created, ...prev])
    setNewTitle('')
  }

  if (loading) return <div>Loading work orders…</div>

  return (
    <div>
      <form onSubmit={createOrder} style={{ marginBottom: 16 }}>
        <input
          placeholder="New work order title"
          value={newTitle}
          onChange={(e) => setNewTitle(e.target.value)}
          style={{ padding: 8, marginRight: 8 }}
        />
        <button style={{ padding: '8px 12px' }}>Create</button>
      </form>

      <ul style={{ listStyle: 'none', padding: 0 }}>
        {orders.map((o) => (
          <li key={o.id} style={{ marginBottom: 8, padding: 12, background: '#fff', borderRadius: 6, boxShadow: '0 0 4px rgba(0,0,0,0.06)' }}>
            <div style={{ fontWeight: 600 }}>{o.title}</div>
            <div style={{ fontSize: 13, color: '#666' }}>Status: {o.status}</div>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### File: `frontend/src/styles.css`
```css
body { background: #f5f7fb; margin: 0; }
#root { max-width: 900px; margin: 24px auto; }
```

---

## Optional: GitHub Actions workflow (CI) — `./.github/workflows/ci.yml`
```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    services:
      docker:
        image: docker:20.10.16
        options: --privileged
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
      - name: Build frontend
        run: |
          cd frontend
          npm ci
          npm run build
      - name: Lint backend
        run: |
          cd backend
          npm ci
```

---

## Final notes
- This scaffold is intentionally straightforward so Codespaces builds quickly.
- Next steps you may want me to implement for you right away (pick any):
  - Add Postgres + TypeORM/Prisma + migrations.
  - Add Authentication (JWT + refresh tokens) and role-based guards.
  - Implement invoice PDF generation (Puppeteer or PDFKit).
  - Convert frontend to TypeScript and add component library.


---

*End of scaffold.*

