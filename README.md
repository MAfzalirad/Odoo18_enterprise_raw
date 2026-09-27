# Odoo 18 (Docker Compose) Development Setup

This document describes the development environment setup for running **Odoo 18** with **PostgreSQL** using **Docker Compose**, including support for **custom addons** and how to install/check the **CRM** module.

---

## 1. Prerequisites

- Ubuntu Linux
- Docker Engine + Docker Compose plugin installed

Verify:
```bash
docker --version
docker compose version
```

---

## 2. Project Structure

Recommended directory layout:

```text
odoo18-docker/
├── compose.yaml
├── config/
│   └── odoo.conf
└── custom-addons/
    └── (your custom modules go here)
```

- `compose.yaml` defines two services: `odoo` and `db`.
- `config/odoo.conf` is the Odoo configuration file.
- `custom-addons/` contains company custom modules.

---

## 3. Docker Compose Configuration

Create/confirm `compose.yaml`:

```yaml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    volumes:
      - db-data:/var/lib/postgresql/data

  odoo:
    image: odoo:18.0
    depends_on:
      - db
    ports:
      - "8069:8069"
    volumes:
      - odoo-data:/var/lib/odoo
      - ./config/odoo.conf:/etc/odoo/odoo.conf:ro
      - ./custom-addons:/mnt/extra-addons
    command: ["odoo", "-c", "/etc/odoo/odoo.conf", "--dev=all"]

volumes:
  db-data:
  odoo-data:
```

### What each volume/mount does

- `db-data:/var/lib/postgresql/data`
  Persists the PostgreSQL database (all Odoo data: CRM records, users, settings, installed modules).
- `odoo-data:/var/lib/odoo`
  Persists the Odoo filestore (attachments/uploads).
- `./config/odoo.conf:/etc/odoo/odoo.conf:ro`
  Mounts the Odoo config file into the container (read-only). This controls DB connection, master password, addons paths.
- `./custom-addons:/mnt/extra-addons`
  Mounts custom modules into the container so Odoo can load them.

---

## 4. Odoo Configuration

Create/confirm `config/odoo.conf`:

```ini
[options]
admin_passwd = Admin@123

db_host = db
db_port = 5432
db_user = odoo
db_password = odoo

addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons

; dev convenience (optional)
; log_level = debug
```

### Master Password (important)

When creating a new database in the web UI, the Master Password must match:

```ini
admin_passwd = Admin@123
```

If it does not match, Odoo shows:

> Database creation error: Access Denied

---

## 5. Start / Stop Odoo

From the project directory (where `compose.yaml` is located):

**Start:**
```bash
docker compose up -d
```

**View logs:**
```bash
docker compose logs -f odoo
```

**Stop (keeps volumes/data):**
```bash
docker compose down
```

---

## 6. Access Odoo

Open in browser:

- http://localhost:8069

Create a database using the database manager screen.

---

## 7. Persistence Rules (Do's and Don'ts)

**Data persists if volumes remain**

Recreating containers does not remove data if volumes remain:

- DB data stays in `db-data`
- Filestore stays in `odoo-data`

**⚠️ WARNING: This command deletes databases and attachments**

```bash
docker compose down -v
```

`-v` removes the named volumes, which wipes Postgres data and filestore.

**List current volumes:**
```bash
docker volume ls
```

---

## 8. Install CRM Module

### Install via UI

1. Go to **Apps**
2. Search **CRM**
3. Click **Install**

### Install via CLI (repeatable)

Replace `YOUR_DB` with your database name:

```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d YOUR_DB -i crm --stop-after-init
docker compose restart odoo
```

---

## 9. Check if CRM is Installed

### Option A (recommended): check in Postgres

List databases (if needed):

```bash
docker compose exec db psql -U odoo -d postgres -c "\l"
```

Check module state:

```bash
docker compose exec db psql -U odoo -d YOUR_DB -c "select name, state from ir_module_module where name='crm';"
```

Expected:

- `installed` → CRM installed
- `uninstalled` or no row → not installed

### Option B: check via Odoo UI

- Apps → search CRM
- Shows Installed / Uninstall button if installed

---

## 10. Custom Addons Development

Place custom modules here:

```text
custom-addons/<module_name>/
```

**Install a custom module** (replace `your_module` and `YOUR_DB`):

```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d YOUR_DB -i your_module --stop-after-init
docker compose restart odoo
```

**Update an existing module:**

```bash
docker compose exec odoo odoo -c /etc/odoo/odoo.conf -d YOUR_DB -u your_module --stop-after-init
docker compose restart odoo
```