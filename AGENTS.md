# AGENTS.md

## Cursor Cloud specific instructions

Homelable is a self-hosted homelab visualization app made of three components in this repo:

| Service | Path | Dev command | Port | Required? |
| --- | --- | --- | --- | --- |
| Backend (FastAPI + SQLite + nmap scanner) | `backend/` | `backend/.venv/bin/uvicorn app.main:app --reload --port 8000` (run from `backend/`) | 8000 | Yes |
| Frontend (React + Vite SPA) | `frontend/` | `npm run dev` (run from `frontend/`) | 5173 (dev) | Yes |
| MCP server (AI integration) | `mcp/` | `mcp/.venv/bin/uvicorn app.main:app --port 8001` | 8001 | Optional |

The startup update script already installs all dependencies (backend `.venv`, `mcp/.venv`, frontend `node_modules`), installs the `nmap` system package into the base image, and creates `backend/.env` if it is missing. Standard lint/test/build commands are documented in `CONTRIBUTING.md` — don't duplicate them; the notes below only cover non-obvious caveats.

### Non-obvious caveats

- **`backend/.env` must NOT contain an active `MCP_API_KEY=` line.** The backend `Settings` model forbids unknown keys, and when settings are loaded from a `.env` *file* (dev mode) every key in the file is validated — so the `MCP_API_KEY` entry from the root `.env.example` makes the backend (and `pytest`) fail at import with `extra_forbidden`. `MCP_API_KEY` belongs only to the `mcp` service. In Docker this never surfaces because env vars are read individually. The update script builds `backend/.env` by copying `.env.example` with that line stripped; keep it that way if you recreate the file. `MCP_SERVICE_KEY`, `LIVEVIEW_KEY`, `HOMEPAGE_API_KEY`, and the Proxmox/Zigbee/Z-Wave keys *are* valid backend settings and can stay.
- **Default dev login is `admin` / `admin`** (from the bcrypt hash in `.env.example`). Backend auth is verified working via `POST /api/v1/health` and `POST /api/v1/auth/login`.
- **The Vite dev server proxies `/api` and websockets to `http://localhost:8000`**, so the backend must be running for the frontend to work (unless building the standalone `VITE_STANDALONE=true` variant).
- **Python:** the base image has Python 3.12, which runs everything fine even though `backend/requirements.txt` says "3.13+". CI uses 3.11 for the backend and 3.13 for the mcp service; both are compatible.
- **nmap** is required for the network scanner feature and for ping status checks; some scan types (SYN/OS detection) need root or `cap_net_raw`, which is not granted in this VM — connect-scan fallback still works.
- **Known pre-existing test failure (not environment-related):** `backend/tests/migrations/test_legacy_node_columns.py::test_the_newest_backup_that_still_has_the_columns_is_the_one_read` fails because the test hardcodes backup version strings (`3.3.0`/`3.3.1`/`3.3.2`) while `VERSION` is now `3.3.5`, so `init_db()` writes a newer `back-3.3.5` backup the test doesn't expect. All other backend tests (902), all frontend tests (2066), and all MCP tests (132) pass, along with `ruff`, `mypy`, ESLint, and `tsc`.
- The SQLite DB lives at `backend/data/homelab.db` in dev (created on first backend start). Saving a canvas from the UI persists nodes there via the API.
