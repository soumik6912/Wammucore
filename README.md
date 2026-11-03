# WammuCore

WammuCore is a Proxmox-based VPS management panel for hosting providers.

This repository contains:
- Panel backend (Laravel)
- Admin/User frontend (React + Vite)
- Paymenter extension files under `extensions/paymenter/`

## Documentation

Use the install docs for direct host deployment:
- [installdocs/README.md](installdocs/README.md)
- [installdocs/Panel/Getting-Started.md](installdocs/Panel/Getting-Started.md)
- [installdocs/Panel/Webserver-Configuration.md](installdocs/Panel/Webserver-Configuration.md)
- [installdocs/Panel/Updating-the-Panel.md](installdocs/Panel/Updating-the-Panel.md)
- [installdocs/Panel/MySQL-Backups.md](installdocs/Panel/MySQL-Backups.md)

## Local Docker Testing (PC)

Use this when you want to test locally before deploying to VPS.

1. Create local env file:

```bash
cp .env.docker.example .env
# PowerShell:
# Copy-Item .env.docker.example .env
```

2. Start Docker stack:

```bash
docker compose up -d --build database redis php caddy workers
```

3. Install dependencies and build frontend (tool container):

```bash
docker compose run --rm --profile tools workspace composer install
docker compose run --rm --profile tools workspace npm install
docker compose run --rm --profile tools workspace npm run build
```

4. Initialize app:

```bash
docker compose exec php php artisan key:generate --force
docker compose exec php php artisan migrate --force
docker compose exec php php artisan optimize:clear
```

5. Open panel:
- http://localhost:8080

Useful commands:

```bash
docker compose logs -f php workers
docker compose down
docker compose down -v
```

## Zero-Downtime Multi-Tenant Migration (Reseller Foundation)

The codebase now includes safe multi-tenant foundation migrations and host-based tenant resolution.

### What this keeps safe
- Existing users are preserved.
- Existing VPS records are preserved.
- Proxmox VM IDs, IP assignments, and server mappings are untouched.

### Rollout steps
1. Run migrations:
```bash
php artisan migrate --force
```

2. Initialize default reseller and backfill all legacy data:
```bash
php artisan tenancy:initialize-default-reseller --name="WammuHost Direct" --domain=your-current-panel-domain.com
```

3. (Optional after verification) enforce `reseller_id` as NOT NULL:
```bash
php artisan tenancy:initialize-default-reseller --domain=your-current-panel-domain.com --enforce-not-null
```

4. Enable tenancy in `.env`:
```env
TENANCY_ENABLED=true
TENANCY_STRICT_HOST_MATCHING=true
TENANCY_MASTER_ADMIN_HOSTS=admin.yourplatform.com
```

5. Clear cached config:
```bash
php artisan optimize:clear
```

## Packaging

- `panel.tar` is the panel-only production archive (no `node_modules`, no `codex`, no temp bundles)

Codex-only reference material lives under `codex/` and should not be shipped in production archives.

## Safe Panel Upgrade (Keep Existing DB/User Data)

Use this flow on Ubuntu 22 to update panel files without deleting existing users/orders/services data:

1. Upload `panel.tar` to the server.
2. Extract to a temporary directory.
3. Sync files into your live panel directory.
4. Run migrations and clear caches.

Example commands:

```bash
cd /var/www
mkdir -p panel_update_tmp
tar -xf panel.tar -C panel_update_tmp

# Sync files to live panel path (change /var/www/wammucore if needed)
rsync -a --delete --exclude='.env' panel_update_tmp/ /var/www/wammucore/

cd /var/www/wammucore
php artisan down || true
php artisan migrate --force
php artisan optimize:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan queue:restart
php artisan up || true
```

Notes:
- This does **not** drop existing database data.
- `php artisan migrate --force` only applies new table/column migrations.
- Keep your existing `.env` file and DB credentials unchanged.
