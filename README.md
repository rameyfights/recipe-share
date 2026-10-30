# Recipe Share

A recipe sharing app. Post a recipe with a photo, browse what everyone else has made, like
the ones you want to cook, and get a notification when someone adds something new.

Built with React and Express, backed by MongoDB, and running as three Docker services behind
Nginx.

## What it does

- Browse recipes on the home page, with filtering by author and sorting
- Sign up / log in (passwords hashed with bcrypt)
- Create a recipe with a title, ingredients, steps, and an image upload
- Edit and delete your own recipes, from a "My Recipes" page
- Like recipes
- Open a recipe detail page
- Live notifications when someone posts a new recipe (Socket.IO)

## Stack

**Frontend**

- React 18 + Vite
- React Router
- TanStack React Query for server state
- Socket.IO client
- Plain CSS (no UI library)

**Backend**

- Node + Express (ES modules, `"type": "module"`)
- MongoDB + Mongoose
- JWT + bcrypt for auth
- Multer for image uploads
- Socket.IO
- Jest + `mongodb-memory-server` for tests

**Infra**

- Docker Compose: `blog-frontend` (Nginx), `blog-backend` (Express), `blog-database` (Mongo)
- Nginx serves the built React app and reverse-proxies `/api/*` to the backend
- GitHub Actions for lint / test / build

The Compose services are still named `blog-*` — that's left over from the course starter I
branched this off, and renaming them means touching `nginx.conf`, so I haven't.

## Running it

The point of the Compose setup is that you don't have to install Mongo or fight with CORS.
From the repo root:

```bash
docker compose up --build -d
```

Then open <http://localhost:3000>.

If you get a **502 from Nginx**, it's almost always one of two things:

1. The backend container isn't up yet — check `docker compose logs blog-backend`.
2. `nginx.conf` is proxying to the wrong place. It should be exactly
   `proxy_pass http://blog-backend:3001;` with no `$request_uri` on the end.

## Dev mode (hot reload)

To avoid rebuilding the image on every change, run the two halves separately:

```bash
# terminal 1
cd backend
npm install
npm run dev

# terminal 2 (repo root)
npm install
npm run dev
```

Vite proxies `/api/v1` and `/socket.io` to `http://localhost:3001`, so the frontend doesn't
need to know where the backend lives. You'll need Mongo on `27017` — the Compose database
service is enough: `docker compose up -d blog-database`.

## Why every API call starts with `/api/v1`

This was the biggest source of pain in the project, so I'm writing it down.

The frontend always calls the API with a **relative** path (`/api/v1`, see `src/api/client.js`).
It never hardcodes `http://localhost:3001` or a Codespaces URL. That means:

- In Vite dev, `vite.config.js` proxies `/api/v1` to port 3001.
- In Docker, Nginx proxies `/api/*` to `http://blog-backend:3001`.
- In a Codespace, nothing needs editing when the forwarded URL changes.

`backend/src/app.js` has a CORS allowlist for localhost ports plus a regex for
`*.app.github.dev`. If you add an origin and forget to add it there, you get an error that
looks like a network failure in the console. I lost an afternoon to that at least once.

## Tests

Backend only, for now:

```bash
cd backend
npm test
```

Tests use `mongodb-memory-server`, so no real database is needed. The first run is slow
because it downloads a MongoDB binary.

## CI

Three workflows in `.github/workflows/`:

- `backend-ci.yaml` — `npm ci`, lint, Jest on Node 20
- `frontend-ci.yaml` — `npm ci`, lint, Vite build on Node 20
- `docker-build.yaml` — builds both images and pushes them to GHCR (no deploy step)

There used to be a `cd.yaml` that built the images and deployed both services to Cloud Run.
It never worked, so I removed it instead of leaving a red X on every push. Why it never
worked is in "rough edges" below.

## Known rough edges

Listing these is more useful than pretending the project is finished:

- **The deploy pipeline is gone.** The frontend image runs Nginx, and `nginx.conf` proxies to
  the Compose hostname `blog-backend`. On Cloud Run there is no such host, so the frontend
  would serve the page and then 502 on every API call. Fixing it properly means templating
  `nginx.conf` with the backend's Cloud Run URL at container start. The backend would also
  need a real `DATABASE_URL` (MongoDB Atlas) and a `JWT_SECRET`.
- **Login doesn't return a JWT yet.** There are two functions, `loginUser` (signs a token)
  and `loginUserSimple` (doesn't), and the route currently calls the simple one. The frontend
  stores the user object in `localStorage`. So auth "works" but isn't enforced server-side.
- **One PATCH route has a hardcoded default user id.** A shortcut to get the edit form
  working that I never replaced with the logged-in user.
- **Uploads live on the container's filesystem.** Fine locally, gone on restart or redeploy.
  The real fix is object storage.
- **The frontend has a test file and no test runner.** `src/components/__tests__/Recipe.test.jsx`
  is there; I never wired up Vitest. The backend tests are the real ones.
- **No pagination.** Every recipe is fetched at once. Fine for a class project, not past a
  few hundred rows.
- **It's all JavaScript.** If I rewrote it I'd use TypeScript — most of my bugs were shape
  mismatches between an API response and what a component expected.
- There's a stray file called `-a` in the root (output of a mistyped `git log` redirect) and
  a `.husky/pre-commut` typo, both still in the tree.

## What I'd do next

1. Finish the JWT flow: sign a token on login, verify it in middleware, stop trusting
   `localStorage` for identity.
2. Add Vitest + React Testing Library for the frontend.
3. Paginate `GET /api/v1/recipes` and add search.
4. Move uploads to object storage.
5. Get the Cloud Run deploy working (Atlas + templated Nginx config) and bring `cd.yaml` back.

## Background

This started as my final project for CS 618. The course built a blog app step by step; this
is a rebuild in a domain I picked, with a different set of problems — image uploads, real-time
notifications, and a real Docker/Nginx setup instead of a dev server.
