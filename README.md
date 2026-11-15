# CHECKPOINT https://github.com/dhurghamCreation/checkpoint-health

CHECKPOINT is a **simulation-only, offline-first prototype** for a fictional Mars-transit crew. It is not a medical device, an operational mission system, a diagnosis, or a source of approved clinical guidance. Do not enter real personal or health information. All readings, thresholds, accounts, communications, checklists, and decisions require human review; the supplied source material in `reference/` is unverified, not an approved clinical procedure.

## Local setup and run

The supplied Dockerfile targets Python 3.12 and Node.js 22; npm 10 is specified in `package.json`. From the repository root, with those tools available:

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
npm ci
npm run build
python -m backend.app
```

Open **http://127.0.0.1:5000**. Flask serves the built frontend and API from one local service. The application defaults to a file at `data/checkpoint.db`; set `CHECKPOINT_DB_PATH` before starting it to choose another persistent file. `HOST` and `PORT` change the Flask listener; the default `HOST` is loopback. Simulated sensor and message-scheduling workers run by default: `CHECKPOINT_SIMULATION=0 python -m backend.app` disables both. Stop the service with Ctrl-C.

The downloadable prototype archive includes a prebuilt `dist/` frontend. If that directory is present and you are not changing the frontend, you can skip `npm ci` and `npm run build` after installing the Python requirements. The version-controlled source does not include `dist/` and needs the build steps above.

For frontend development, start the backend as above and, in a second terminal in the repository root, run:

```bash
npm run dev -- --host 127.0.0.1
```

Open **http://127.0.0.1:3000**; Vite proxies `/api` to the backend on port 5000. Without the explicit host option, the Vite configuration binds to all interfaces. Its development-preview content security policy permits inline scripts and WebSocket connections for hot reload, unlike the stricter built Flask app; do not treat the Vite preview as a hardened deployment. Avoid exposing either demo service on an untrusted network. Dependency installation requires network access; once built and installed, local operation does not depend on an Earth connection, but the browser still needs to reach its local application service.

## Demo accounts and security

A fresh database seeds these publicly documented accounts; each initially uses password **`checkpoint-demo`**:

| Username | Demo role |
|---|---|
| `commander` | Commander |
| `medical` | Medical Officer |
| `operator` | Mission Operator |
| `admin` | Administrator |

Roles are enforced by the server, passwords are hashed, and sessions expire. The **Security** section can flag the current account's unchanged demo password and lets a user change it; a successful change revokes that account's existing sessions. New and reset passwords must be 12–256 characters and must not be the demo password or one of a limited local denylist. Repeated failed sign-ins are temporarily limited per account; a separate installation-wide burst budget counts successful and failed sign-ins, and may temporarily block legitimate users during a burst. API responses carry security and no-store headers.

The initial demo credentials remain publicly known until each account is changed; they demonstrate roles, **not production security**. Existing databases retain any changed accounts and passwords rather than reseeding on restart. The throttle does not prevent distributed denial of service or credential spraying, browser session tokens remain vulnerable to same-origin script compromise, and local HTTP is not encrypted. Changing credentials alone does not make the prototype safe for real data or deployment.

The default built app permits embedding by any origin so the managed Preview can work; this leaves a clickjacking risk. Outside Preview, an operator can set `CHECKPOINT_FRAME_ANCESTORS` to a space-separated list of explicitly trusted HTTPS origins before starting Flask. An incorrect allowlist can prevent intended embedding, and the Vite development server uses its own preview policy. Configure transport security and independently review the deployment before exposure to untrusted users.

SQLite records, including medical-style records, are **not encrypted by the application**. Use only fictional information; protect the machine, database, and backups, and use an encrypted disk if appropriate. No production security assessment or independent clinical, regulatory, or mission certification is claimed.

## Database backup and retention

SQLite uses **WAL mode**: committed changes can be in the `-wal` file. **Do not copy only the main database file while the service is running.** Use SQLite's online backup API to make a consistent backup, including committed WAL contents. The following example works while the service is running and refuses to overwrite an existing backup. Set `CHECKPOINT_DB_PATH` in this shell to the same regular database file used by the service; otherwise the example uses `CHECKPOINT_DATABASE` or the default path. It does not handle SQLite `file:` URIs or in-memory databases.

```bash
# Run from the repository root. Choose a protected backup destination.
umask 077
DB_PATH="${CHECKPOINT_DB_PATH:-${CHECKPOINT_DATABASE:-$PWD/data/checkpoint.db}}"
BACKUP_PATH="$HOME/checkpoint-backup-$(date -u +%Y%m%dT%H%M%SZ).db"
python - "$DB_PATH" "$BACKUP_PATH" <<'PY'
import sqlite3
import sys
from contextlib import closing
from pathlib import Path

source_path = Path(sys.argv[1]).expanduser().resolve(strict=True)
backup_path = Path(sys.argv[2]).expanduser().resolve()
if source_path == backup_path:
    raise SystemExit('Backup must be a different file from the source.')

# Exclusive creation avoids accidentally replacing an existing backup.
with backup_path.open('xb'):
    pass
try:
    with closing(sqlite3.connect(source_path.as_uri() + '?mode=ro', uri=True, timeout=20)) as source:
        with closing(sqlite3.connect(str(backup_path), timeout=20)) as backup:
            source.backup(backup)
            if backup.execute('PRAGMA integrity_check').fetchall() != [('ok',)]:
                raise RuntimeError('Backup integrity check failed')
except Exception:
    backup_path.unlink(missing_ok=True)  # discard any incomplete backup
    raise
print('Backup created:', backup_path)
PY
```

Keep backups protected: they are also unencrypted by the application. Periodically test restoration to a **separate** path, without overwriting a running database, and verify that the restored application opens the expected records. A backup is not a substitute for a tested recovery process.

The `retention_days` setting is stored and configurable, but the application currently performs **no automatic deletion or archival** based on it. It is not an enforced retention schedule.

## Simulation boundaries and scope

- **Earth Link is local simulation only.** No real spacecraft, Earth endpoint, or external message transport is connected. The offline switch represents the simulated Earth link, not the browser-to-service connection. The default delay is 18.7 minutes one way (37.4 minutes round trip); message state and synchronization are locally simulated.
- **Hosted preview is ephemeral.** A development preview/container filesystem can be reset and is not durable storage or proof of a fully disconnected installation. For a durable local experiment, run on a machine with persistent storage and validate backup, restore, and disconnected operation there.
- The prototype includes crew and measurement records, alert rules, stored-data charts, radiation/environment views, simulated communications, medical/medication and incident workflows, administrative audit, settings, and CSV/JSON/PDF exports. The illustrative Three.js orbital scene is not real flight telemetry. Operations Desk tasks and handovers are local, visible to all signed-in roles, and intended only for non-medical fictional information; its entries are not included in general exports. The command palette opens with Ctrl/Command+K, and the dashboard can generate a non-medical simulated briefing. Capabilities and unresolved items are summarized in `TODO.md`; they are not a claim of full SRS acceptance.
- Audit events use a locally verified hash chain and database update/delete guards, but there is **no independent external anchor or signature**. A privileged attacker able to rewrite the database and chain, including truncation or recomputation, may evade detection. This is tamper evidence with important limits, not immutable or independently verifiable storage.

## Build and tests

From the repository root, after setup:

```bash
npm run build
python -m pytest tests -q
node --test tests/*.test.mjs
```

The first command runs TypeScript checking and a Vite production build. The Python suite uses Flask's test client and isolated temporary SQLite databases; it covers selected validation, authorization, telemetry, alerts, message timing/synchronization, audit integrity, and restart behavior. The Node suite exercises frontend polling, history, and chart utilities. See `tests/README.md` for fixtures and interpretation. The timing check runs five sequential in-process requests with the simulator disabled; it is **not** a browser, load, concurrency, memory, network, or long-duration benchmark. Passing it does not establish the SRS performance targets, operational reliability, comprehensive security, accessibility, clinical safety, or independent compliance.
