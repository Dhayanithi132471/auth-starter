[README.md](https://github.com/user-attachments/files/32301517/README.md)
# React + Python + PostgreSQL JWT authentication

A complete source project with React registration/login forms, a protected dashboard, a FastAPI backend, PostgreSQL storage, and JWT authentication. The interface is branded **Haven**; change the name and styling in `frontend/src/App.jsx` and `styles.css`.

## Start with Docker

Install Docker with Compose v2 and Python 3. Run these commands from this folder:

```bash
python scripts/setup_env.py
docker compose up --build
```

Open **http://localhost:8080**. Create an account to enter the dashboard. API documentation is at **http://localhost:8080/api/docs**. Migrations run automatically before the API starts.

The setup script generates random secrets, writes `.env`, and leaves existing settings intact. `.env` must not be committed or shared. Use `localhost` consistently; changing to `127.0.0.1` changes the browser origin and requires updating `ALLOWED_ORIGINS`.

Stop services with `docker compose down`. PostgreSQL data remains in the named volume. Avoid `docker compose down -v` unless you intend to permanently delete your development database.

## Included

- Registration with name, case-insensitive email, password, and frontend password confirmation.
- Login, reload-persistent authentication, protected dashboard, protected API example, and logout.
- Argon2id password hashing using pwdlib; passwords are never returned by the API.
- HS256 JWTs with expiry, issuer, audience, subject, and session ID checks.
- HTTP-only cookies, custom-header CSRF protection, and exact CORS origins.
- PostgreSQL session records so logout immediately revokes the current JWT.
- Database-enforced unique emails, SQLAlchemy models, and Alembic migrations.
- Shared PostgreSQL rate limits: 20 login attempts and 10 registration attempts per client IP per five-minute fixed window.
- Responsive forms, password visibility controls, validation feedback, request/loading states, and session expiry handling.
- Docker Compose and automated frontend/backend tests.

## How authentication works

1. React submits registration or login details as JSON.
2. FastAPI hashes a new password or verifies the stored Argon2id hash.
3. FastAPI creates a database session and signs a JWT containing its ID.
4. The JWT is placed in an HTTP-only `auth_access` cookie scoped to `/api`; React stores only the returned user profile and expiry in memory.
5. Protected endpoints validate both the JWT and its matching, unexpired database session.
6. Logout deletes the session row and clears the cookie. Replaying that JWT then fails.

Sessions expire after 30 minutes by default. This project intentionally requires a new login after expiry; it does not implement refresh tokens. Separate browsers have separate sessions. Logging in again in the same browser replaces that browser's previous session.

## API

All mutation requests require `X-CSRF-Protection: 1`. Browser origins must appear in `ALLOWED_ORIGINS`. Non-browser clients may omit `Origin` but must still send the custom header. Credentials and authentication use cookies, not localStorage or a bearer token returned in JSON.

| Method | Path | JSON body / result |
| --- | --- | --- |
| POST | `/api/auth/register` | `{ "name": "Alex", "email": "alex@example.com", "password": "a-long-passphrase" }` → user + expiry, sets cookie |
| POST | `/api/auth/login` | `{ "email": "alex@example.com", "password": "a-long-passphrase" }` → user + expiry, sets cookie |
| GET | `/api/auth/me` | Current user + expiry; requires valid session |
| POST | `/api/auth/logout` | No body; revokes current session, clears cookie, returns 204 |
| GET | `/api/private` | Example protected message |
| GET | `/api/health` | Checks database connectivity |

Example from a terminal:

```bash
curl -i -c cookies.txt http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -H 'X-CSRF-Protection: 1' \
  -d '{"name":"Alex","email":"alex@example.com","password":"a-long-passphrase"}'

curl -b cookies.txt http://localhost:8080/api/auth/me

curl -X POST -b cookies.txt -c cookies.txt \
  -H 'X-CSRF-Protection: 1' http://localhost:8080/api/auth/logout
```

Delete the local `cookies.txt` file after use; it contains an authentication credential. Interactive mutation calls in Swagger need the same custom header; use the React UI or curl examples for the full flow.

## Development without application containers

Requirements: Python 3.12+, Node 22.12+ (Node 24 recommended), and PostgreSQL 17. You can use the Compose database while running the application locally:

```bash
python scripts/setup_env.py
docker compose up -d db
```

Backend, in one terminal:

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
alembic upgrade head
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Frontend, in another terminal:

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:5173**. Vite proxies `/api` to `127.0.0.1:8000`. Direct API docs are at `http://127.0.0.1:8000/api/docs`. The backend reads the root `.env` automatically. For an existing PostgreSQL server, change `DATABASE_URL` in that file to `postgresql+psycopg://USER:PASSWORD@HOST:5432/DATABASE` (URL-encode special password characters).

## Tests

Frontend API client tests require only Node 22.12+:

```bash
cd frontend
npm test
npm run build
```

Backend integration tests use a separate, disposable PostgreSQL database:

```bash
# Run from the project root, after setup_env.py.
docker compose --profile test run --build --rm tests
docker compose --profile test stop test-db
```

The test service applies migrations and uses a temporary PostgreSQL volume. Tests cover registration, password storage, duplicate/concurrent registration, cookie flags, login, logout/replay, JWT tampering and claim validation, CSRF/CORS, input validation, and rate limiting.

To run pytest outside Docker, set `TEST_DATABASE_URL` to a disposable database whose name ends in `_test`, install `requirements-dev.txt`, and run `pytest -q` from `backend`. **Tests truncate all authentication tables in that test database.** Never point them at a database containing data you want to keep.

Validation performed when this project was generated: see `VERIFICATION.md`. The dependency manifests use bounded version ranges; registry access was unavailable, so no verified lockfiles are included. After a successful installation in your environment, retain the generated npm lockfile and pin your resolved Python dependencies before deployment.

## Project map

| Location | Purpose |
| --- | --- |
| `backend/app/routes.py` | Register, login, current session, logout |
| `backend/app/security.py` | Argon2id and JWT signing/validation |
| `backend/app/dependencies.py` | Server-side authorization checks |
| `backend/app/models.py` | Users, sessions, and rate-limit tables |
| `backend/migrations/` | Versioned database schema |
| `backend/tests/` | PostgreSQL integration tests |
| `frontend/src/AuthContext.jsx` | Client session lifecycle |
| `frontend/src/App.jsx` | Forms, routes, dashboard |
| `frontend/src/api.js` | Cookie-aware API client |
| `frontend/nginx.conf` | Static hosting and API proxy |

To protect a new FastAPI route, add the `CurrentUser` dependency, as shown in `/api/private`. Frontend route guards improve navigation; the backend dependency enforces access.

## Deployment configuration

The supplied Compose stack is configured for localhost development. For a public deployment, serve the frontend/API under one HTTPS origin, set `COOKIE_SECURE=true`, and replace `ALLOWED_ORIGINS` with that exact origin, for example `["https://accounts.example.com"]`. Keep the API and PostgreSQL private. JWT secrets and database credentials should come from your deployment's secret manager.

The Compose API trusts forwarding headers because it has no published port and receives them from Nginx, which overwrites incoming values. If you expose the API or change the proxy topology, restrict Uvicorn's trusted proxy IPs and configure trusted client address forwarding; otherwise IP rate limits may be spoofed or applied to the proxy. Local Vite development shares the proxy's loopback rate-limit bucket. The five-minute limits are a baseline and allow a burst at window boundaries; tune them for your traffic and add edge controls as needed.

Email ownership verification, password reset, MFA, and account recovery are not included. Registration reports duplicate emails, which reveals whether an address is registered. Add these flows and adapt that disclosure policy before using the project for a public account service. Back up PostgreSQL, apply dependency security updates, and configure database TLS when it crosses untrusted networks.

## Implementation references

- [FastAPI JWT and password hashing](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [OWASP custom-header CSRF protection](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Vite setup](https://vite.dev/guide/)

## Deploy to Render

This repository includes a Render Blueprint in `render.yaml` and a production Docker image in `Dockerfile.render`.

The Render deployment runs the React/Vite frontend, Nginx reverse proxy, and FastAPI backend in one web service, plus a PostgreSQL database. The browser uses the same HTTPS origin for the UI and `/api`, so the existing HTTP-only authentication cookie can remain same-origin.

1. Push the `auth-starter` directory to a GitHub repository.
2. In Render, choose **New -> Blueprint** and select that repository.
3. Render reads `render.yaml` and creates the web service and PostgreSQL database.
4. After the first deploy, open the service URL. If you changed the Render web-service name, update `ALLOWED_ORIGINS` in the web service environment variables to the exact `https://<your-service>.onrender.com` origin and redeploy.
5. The health endpoint is `/api/health`.

The database is created separately so its credentials are managed by Render. Do not commit a real `.env` file or database credentials.
