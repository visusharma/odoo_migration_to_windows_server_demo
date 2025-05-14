# Odoo 17 Dockerized Setup (with External Database)

This project provides a Dockerized setup for running Odoo 17, designed to connect to an externally hosted PostgreSQL database.

## Overview

-   **Odoo Version:** 17.0
-   **Application Server:** Odoo runs as a Docker container managed by Docker Compose.
-   **Database:** PostgreSQL is hosted and managed externally (e.g., by the client on a separate server or cloud service). The Odoo application container connects to this external database.
-   **Filestore:** Odoo's filestore (for attachments and binary data) is managed using a Docker named volume (`odoo_data`) for persistence.

## Project Structure

```
Odoo17/
├── addons/               # Place your custom Odoo addons here
├── config/               # Contains odoo.conf for Odoo configuration
│   └── odoo.conf         # Main Odoo server configuration file (CRITICAL: configure DB connection here)
├── Dockerfile            # Defines the Odoo application Docker image (builds from official odoo:17.0)
├── docker-compose.yml    # Defines the Odoo service and manages containers
├── Plan.md               # Detailed migration and setup plan (assumes external DB)
└── README.md             # This file
```

## Prerequisites

-   Docker Desktop installed and running.
-   Access to an external PostgreSQL database with connection details (host, port, user, password, database name).

## Setup and Configuration

1.  **Configure Odoo (`config/odoo.conf`):**
    *   Update `config/odoo.conf` with the correct connection parameters for your **external PostgreSQL database**. Key parameters include:
        *   `db_host`
        *   `db_port`
        *   `db_user`
        *   `db_password`
        *   `db_name`
        *   `db_sslmode` (if required by your external database)
    *   Also, set the `admin_passwd` (master administrator password).

2.  **Custom Addons:**
    *   Place any custom Odoo modules into the `addons/` directory.

3.  **Build and Run Odoo:**
    *   Open a terminal in the `Odoo17` directory.
    *   Run the command: `docker-compose up -d`
    *   This will build the Odoo Docker image (if it's the first time or `Dockerfile` changed) and start the Odoo service.

4.  **Access Odoo:**
    *   Once the container is running, Odoo should be accessible at `http://localhost:8069` in your web browser.

## Key Files

-   `docker-compose.yml`: Defines the `odoo` service. It mounts:
    *   `./config/odoo.conf` to `/etc/odoo/odoo.conf` in the container.
    *   `./addons` to `/mnt/extra-addons` in the container.
    *   A named volume `odoo_data` to `/var/lib/odoo` for the filestore.
-   `Dockerfile`: Builds the Odoo image from the official `odoo:17.0` base. It can be extended to include additional system dependencies or Python packages if required by custom addons.
-   `config/odoo.conf`: **Crucial for database connection.** This is where you tell Odoo how to find and authenticate with your external database.

## Data Management

-   **Database Data:** Managed externally by the client or on a separate database server. Backups and maintenance of the database are handled externally.
-   **Odoo Filestore:** Stored in the Docker named volume `odoo_data`. This volume persists even if the Odoo container is stopped or removed. You should implement a backup strategy for this volume (see `Plan.md` for examples).
-   **Custom Addons & Configuration:** Reside in the `./addons` and `./config` directories on the host and should be part of your regular project backups.

## Migration

For details on migrating an existing Odoo instance to this setup, refer to `Plan.md`. 