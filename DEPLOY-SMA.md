# Cacti - Tester Environment Deployment Notes

## Fork Info

- **Fork URL:** https://github.com/alevern/cacti
- **Baseline branch:** `tester-env-baseline`
- **Baseline commit:** `d9de6dff576beb685c6c5c904e284aef3501922c`
- **Upstream:** https://github.com/Cacti/cacti
- **Fork purpose:** Tester-env submodule for RL agent training scenarios

## Deployment

CLI: `deployment/cacti/tester-env` (located in tester-env repo, not in the fork)

Commands:
```
deployment/cacti/tester-env deploy   # Build and start containers, run installer
deployment/cacti/tester-env seed     # Populate deterministic Northwind seed data
deployment/cacti/tester-env verify   # Health-check web/login/DB + seed counts
deployment/cacti/tester-env reset    # Stop containers, remove containers + volumes
deployment/cacti/tester-env stop     # Stop containers
deployment/cacti/tester-env logs     # Tail web container logs
deployment/cacti/tester-env status   # Show container status
```

Full reset-deploy-seed-verify cycle:
```
./deployment/cacti/tester-env reset && ./deployment/cacti/tester-env deploy && ./deployment/cacti/tester-env seed && ./deployment/cacti/tester-env verify
```

## Build Method

- **Method:** `docker compose up --build` using upstream `docker-compose.yml`
- **Source:** Mounted as bind volume into web container at `/var/www/html/cacti`
- **Web image:** `php:8.4-fpm-bookworm` (built locally with Apache + PHP-FPM)
- **DB image:** `mariadb:11.8`
- **Services:** web (port 8089), db (port 3306)
- **Volumes:** cacti_db, cacti_cache, cacti_rra, cacti_logs
- **Project name:** `tester-env-cacti`

## Runtime Fixups

1. **php.ini:** Suppress `E_DEPRECATED` and `E_STRICT`, disable `display_errors` (PHP 8.4 warnings pollute CLI installer stdout, breaking `json_decode`)
2. **fping:** Install `fping` package in web container (required for device availability checks)
3. **Cache dirs:** Create `cache/{boost,mibcache,realtime,spikekill}` subdirectories
4. **Admin password:** Set admin password to `MD5('admin123')`, disable `must_change_password` via direct DB update (CLI installer runs as root with random password)
5. **Log ownership:** `chown -R www-data:www-data log/ cache/ rra/` (installer runs as root, php-fpm as www-data)

## Caveats

- Upstream `docker-compose.yml` hardcodes `container_name: cacti_web / cacti_db`, preventing parallel `--run-id` isolation without editing compose file
- CLI installer runs as root via `docker exec`, creating log files owned by root; without post-install `chown`, php-fpm (www-data) cannot write `cacti.log`, causing "System log file is not available for writing" FATAL and session recursion
- Only single-run deploys with default container names are supported currently

## Credentials

- **Admin user:** `admin`
- **Admin password:** `admin123`
- **DB user:** `cacti`
- **DB password:** `cacti`
- **DB name:** `cacti`
- **DB root password:** `root`

## URLs

- **Local:** http://localhost:8089/cacti/
- **Browser MCP:** http://host.docker.internal:8089/cacti/

## Seed Data — Northwind Network Operations

**Theme:** Northwind Network Operations

**Sites (3):**
1. Portland Edge DC (PDX-EDGE) — regional edge and checkout ingress
2. Denver Core DC (DEN-CORE) — core switching and billing systems
3. Remote Retail Labs (BOI-LAB) — remote lab sensors and backup links

**Devices (6):**
1. Portland Edge Router (NW-PDX-RTR-01) — UP, primary customer ingress router
2. Portland Checkout API Probe (NW-PDX-API-22) — RECOVERING, synthetic API latency probe
3. Denver Core Switch Stack (NW-DEN-SW-10) — UP, core switch stack
4. Denver Billing Database (NW-DEN-DB-45) — DOWN, TCP timeout after storage maintenance
5. Retail Lab Sensor Gateway (NW-BOI-IOT-12) — DISABLED, firmware validation paused
6. Retail Lab Long Haul Backup Link (NW-BOI-WAN-77) — ERROR, SNMP authentication failure

**Status mix:** 2 up, 1 recovering, 1 down, 1 error, 1 disabled

**Graph Tree:** Northwind Network Operations (1 tree, 9 tree items: 3 site headers + 6 device leaves)

## Determinism

- Two full reset+deploy+seed+verify cycles passed with identical counts
- Immediate reseed+verify also passed with exact counts and names
- Seed uses deterministic MariaDB fixture with fixed external IDs (NW-* prefix)

## Verification

`deployment/cacti/tester-env verify` checks:
- Web server responds at /cacti/
- Login page contains "Cacti" text
- Admin user exists in database
- Seed counts: 3 sites, 6 devices, 1 tree, 9 tree items
- Status mix: 2 up, 1 recovering, 1 down, 1 error, 1 disabled
- All 6 device names match exactly

## Browser Smoke Evidence

- Login page loads with CSRF token
- Login succeeds with admin/admin123
- Devices CRUD visible (create/list/delete devices)
- Seeded devices, sites, and graph tree verified in browser UI

## Mutation Smoke Evidence

- Source code change (e.g., page title) → rebuild → visible change confirmed in browser
- Source restored and rebuilt → original behavior restored

## Reset Path

`docker compose down -v` removes containers and all named volumes (DB, cache, RRA, logs).
Full reset is deterministic — after reset + deploy, installer creates fresh database schema.
Seed script is idempotent (DELETE old NW-* rows, then INSERT fresh).
