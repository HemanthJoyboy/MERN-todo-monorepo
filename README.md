# MERN Todo Monorepo (Node + Express + React + Neon Postgres)

A small but realistic microservice-style Todo app. "MongoDB" is swapped for
**Neon Postgres** since that's what you're using — everything else (Express,
React, Node) is standard MERN.

## 1. Architecture / repo structure

```
mern-todo-monorepo/
├── client/                  # React (Vite) frontend            → port 3000
├── services/
│   ├── api-gateway/         # single entry point, proxies requests → port 4000
│   ├── auth-service/        # register/login, issues JWTs        → port 4001
│   └── todo-service/        # CRUD todos, verifies JWTs          → port 4002
├── ecosystem.config.js      # PM2 config to run all 4 processes
└── package.json             # npm workspaces root
```

This mirrors a real setup: the **client never talks to auth-service or
todo-service directly** — it only calls the **api-gateway**, which proxies
requests to the right internal service. That's the standard "API Gateway"
pattern used in real microservice systems (keeps a single public port,
lets you add/remove/scale internal services without touching the frontend).

### Why 3 backend services + 1 frontend
| Service | Responsibility | Port |
|---|---|---|
| `client` | React UI | 3000 |
| `api-gateway` | Public entrypoint, routes `/api/auth/*` and `/api/todos/*` to the right service | 4000 |
| `auth-service` | User register/login, password hashing, issues JWT | 4001 |
| `todo-service` | Todo CRUD, protected by JWT | 4002 |

Only **port 3000 (frontend)** and **port 4000 (gateway)** need to be public.
4001/4002 should stay internal (only reachable by the gateway on the same
machine) — see EC2 security group notes below.

## 2. Data flow (step by step)

**Register/Login:**
1. React app (`client/src/components/Login.jsx`) calls `authApi.login()` →
   `fetch('http://<gateway>:4000/api/auth/login', ...)`
2. `api-gateway` receives it at `/api/auth/login`, strips the `/api/auth`
   prefix, and proxies to `auth-service` at `http://localhost:4001/login`
3. `auth-service` checks the `users` table in Neon Postgres, verifies the
   bcrypt password hash, and signs a JWT with `JWT_SECRET`
4. Response (user + token) flows back through the gateway to the browser
5. Client stores `token` in `localStorage`

**Fetching/creating todos:**
1. Client calls `todoApi.list()` → `GET /api/todos` with header
   `Authorization: Bearer <token>`
2. Gateway proxies to `todo-service` at `http://localhost:4002/`
3. `todo-service`'s `requireAuth` middleware verifies the JWT **using the
   same `JWT_SECRET`** (no network call to auth-service needed — this is
   why both services must share the same secret)
4. It queries the `todos` table filtered by `user_id = decoded token id`
5. Rows flow back through the gateway to the client

This is why `JWT_SECRET` in `auth-service/.env` and `todo-service/.env`
**must be identical** — auth-service issues tokens, todo-service verifies
them independently, without a shared session store.

## 3. Where to make Neon Postgres changes

You only need to touch **`.env` files** — no code changes required.

1. Create a Neon project → copy the connection string (looks like
   `postgres://user:pass@ep-xxxx.us-east-2.aws.neon.tech/neondb?sslmode=require`)
2. Paste it into:
   - `services/auth-service/.env` → `DATABASE_URL=...`
   - `services/todo-service/.env` → `DATABASE_URL=...`
   (You can use the same Neon database for both, or two Neon databases —
   either works since each service only touches its own table.)
3. Run the schema against Neon. Easiest way: open the Neon SQL editor and
   paste the contents of:
   - `services/auth-service/sql/schema.sql` (creates `users`)
   - `services/todo-service/sql/schema.sql` (creates `todos`)

   Or from your machine with `psql`:
   ```bash
   psql "$DATABASE_URL" -f services/auth-service/sql/schema.sql
   psql "$DATABASE_URL" -f services/todo-service/sql/schema.sql
   ```
4. Set a strong random value for `JWT_SECRET` in **both** `auth-service/.env`
   and `todo-service/.env` (must match exactly). Generate one with:
   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```

No other code needs editing — `pg.Pool` in each service's `src/db.js`
reads `DATABASE_URL` from `.env` automatically, with
`ssl: { rejectUnauthorized: false }` already set (required by Neon).

## 4. Running locally (before EC2)

```bash
# from repo root
npm install    # installs all workspaces (client + 3 services) at once

# copy env files and fill in your Neon URL + JWT secret
cp services/auth-service/.env.example services/auth-service/.env
cp services/todo-service/.env.example services/todo-service/.env
cp services/api-gateway/.env.example services/api-gateway/.env
cp client/.env.example client/.env

# run schema.sql files against Neon (see step 3 above)

# start each service in its own terminal
npm run start:auth       # port 4001
npm run start:todo       # port 4002
npm run start:gateway    # port 4000
npm --workspace=client run dev   # port 3000 (dev mode with hot reload)
```

Open `http://localhost:3000`.

## 5. Deploying to a single EC2 instance

### 5.1 Launch & connect
- Launch an Ubuntu 22.04 EC2 instance (t3.small is plenty for a demo)
- In the **Security Group**, open inbound:
  - port **22** (SSH) — your IP only
  - port **3000** (frontend) — 0.0.0.0/0 (or restrict as needed)
  - port **4000** (api-gateway) — 0.0.0.0/0
  - **Do NOT open 4001/4002 publicly** — they're only called by the
    gateway over `localhost`, so keep them closed to the internet
- SSH in: `ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>`

### 5.2 Install Node.js, git, PM2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs git
sudo npm install -g pm2 serve
node -v   # confirm v20.x
```

### 5.3 Get the code onto the box
Push this repo to GitHub, then on the EC2 instance:
```bash
git clone <your-repo-url> mern-todo-monorepo
cd mern-todo-monorepo
npm install
```
(Or `scp -r` the folder from your machine if you don't want to use git.)

### 5.4 Configure environment files
```bash
cp services/auth-service/.env.example services/auth-service/.env
cp services/todo-service/.env.example services/todo-service/.env
cp services/api-gateway/.env.example services/api-gateway/.env
cp client/.env.example client/.env

nano services/auth-service/.env   # paste Neon DATABASE_URL + JWT_SECRET
nano services/todo-service/.env   # same DATABASE_URL/JWT_SECRET as above
nano services/api-gateway/.env    # defaults are fine (localhost URLs)
nano client/.env                  # VITE_API_BASE_URL=http://<EC2_PUBLIC_IP>:4000
```

### 5.5 Build the frontend
The client is a static React app — build it once, then serve the static
files (don't run Vite's dev server in production):
```bash
npm --workspace=client run build   # outputs client/dist
```

### 5.6 Run everything with PM2
`ecosystem.config.js` at the repo root already defines all 4 processes:
```bash
pm2 start ecosystem.config.js
pm2 status                # see all 4 processes running
pm2 logs                  # tail logs from all services
pm2 logs auth-service     # logs from one service
```
Make PM2 survive reboots:
```bash
pm2 save
pm2 startup               # run the command it prints (sets up systemd)
```

Useful PM2 commands:
```bash
pm2 restart api-gateway   # restart one service after a code/env change
pm2 restart all
pm2 stop all
pm2 delete all
```

### 5.7 Verify
```bash
curl http://localhost:4000/health   # api-gateway
curl http://localhost:4001/health   # auth-service
curl http://localhost:4002/health   # todo-service
```
Then visit `http://<EC2_PUBLIC_IP>:3000` in a browser.

### 5.8 (Optional, recommended) Put Nginx in front
For a real deployment you'd usually front port 3000/4000 with Nginx on
port 80, and use a domain + TLS (Let's Encrypt/Certbot) instead of exposing
raw Node ports. Not required for a demo, but worth knowing:
```bash
sudo apt-get install -y nginx
# configure /etc/nginx/sites-available/default to reverse-proxy
# / -> localhost:3000
# /api -> localhost:4000
```

## 6. Making code changes later
| Change | Where |
|---|---|
| Add a new todo field | `todo-service/sql/schema.sql` (ALTER TABLE) + `todo-service/src/routes/todos.js` + `client/src/components/TodoList.jsx` |
| Add a new API route | New file under `services/<service>/src/routes/`, mount it in that service's `src/index.js` |
| Add a new backend service | New folder under `services/`, add its proxy rule in `services/api-gateway/src/index.js`, add it to `ecosystem.config.js` |
| Change ports | Edit the relevant `.env` (and `ecosystem.config.js`'s `args`/`env` if serving the client on a different port) |
| Rotate JWT secret | Update `JWT_SECRET` in **both** `auth-service/.env` and `todo-service/.env`, then `pm2 restart auth-service todo-service` |
