# Project: Odoo 17 Migration to Docker on Windows Server 2022 (with External Database)

## 1. Introduction

Migrating Odoo to a Dockerized environment offers several advantages, including:
*   **Portability:** Run Odoo consistently across different environments.
*   **Scalability:** Easily scale Odoo instances up or down.
*   **Isolation:** The Odoo application runs in an isolated container.
*   **Simplified Management:** Easier updates and maintenance of the Odoo application using Docker Compose.
*   **Reproducibility:** Define your Odoo application environment as code.

This document outlines the steps to migrate an existing Odoo 17 instance from a CentOS 7 Hyper-V VM to Docker Desktop running on Windows Server 2022, with the **PostgreSQL database hosted externally by the client.**

## 2. Prerequisites

*   **Windows Server 2022:** With Hyper-V role enabled (if Docker Desktop requires it, though it usually uses WSL2).
*   **Docker Desktop for Windows:** Installed and configured on Windows Server 2022. Ensure it's using the WSL 2 backend.
*   **Administrative access:** To both the CentOS 7 VM and the Windows Server 2022 machine.
*   **Sufficient disk space:** On the Windows Server for Docker images, Odoo filestore, and backups of the filestore/addons.
*   **Network connectivity:**
    *   Between the CentOS VM and the Windows Server for data transfer (filestore, addons).
    *   From the Windows Server 2022 (where Docker runs) to the client's external PostgreSQL database server (ensure firewalls allow this connection on the correct port).
*   **Client's External PostgreSQL Database:**
    *   A running PostgreSQL server (version compatible with Odoo 17, typically 12+).
    *   Connection details: hostname/IP, port, database name, user, and password with appropriate permissions for Odoo.
    *   The client is responsible for the availability, maintenance, and backup of this external database.

## 3. Migration Phases

### Phase 1: Backup Odoo Data and Custom Addons from CentOS 7 VM

This is the most critical phase. Ensure you have complete and verified backups.

*   **3.1. Stop Odoo Service on CentOS 7:**
    *   Command: `sudo systemctl stop odoo17.service`

*   **3.2. Backup Odoo Database (to be restored on Client's External DB):**
    *   Command (example):
        ```bash
        pg_dump -h localhost -U <odoo_db_user> -Fc -f /path/to/backup/odoo_database.dump <odoo_database_name>
        ```
    *   This dump file will be provided to the client to restore on their external PostgreSQL server.

*   **3.3. Backup Odoo Filestore:**
    *   Command (example):
        ```bash
        sudo tar -czvf /path/to/backup/odoo_filestore.tar.gz /path/to/your/odoo/filestore/<odoo_database_name>
        ```

*   **3.4. Backup Custom Addons:**
    *   Command (example):
        ```bash
        sudo tar -czvf /path/to/backup/custom_addons.tar.gz /path/to/your/odoo/custom_addons_folder
        ```

*   **3.5. Backup Odoo Configuration File (Optional but Recommended for Reference):**
    *   Command (example): `sudo cp /etc/odoo/odoo.conf /path/to/backup/odoo.conf.backup.original`

*   **3.6. Transfer Backups to Windows Server 2022:**
    *   Transfer `odoo_filestore.tar.gz` and `custom_addons.tar.gz` to the Windows Server.
    *   Provide `odoo_database.dump` to the client for restoration on their database server.

### Phase 2: Prepare Docker Environment on Windows Server 2022

*   **4.1. Create Project Directory Structure:**
    *   Example: `C:\Docker\Odoo17` (or relatively `Odoo17/` within your workspace).
    *   Subdirectories:
        *   `config`: For the Odoo configuration file (`odoo.conf`).
        *   `addons`: For custom Odoo addons.
        *   `backups`: For placing the transferred `odoo_filestore.tar.gz` initially.
    *   **Note on Data Storage:**
        *   **Database:** Hosted externally by the client. No local Docker volume for the database is used.
        *   **Odoo Filestore:** Docker will manage persistent storage for the Odoo filestore via a named volume (`odoo_data` as defined in `docker-compose.yml`).

*   **4.2. Create/Update `odoo.conf` file (File: `Odoo17/config/odoo.conf`):**
    *   This file is crucial for connecting to the client's external database.
    *   **Key settings for external Dockerized Odoo:**
        ```ini
        [options]
        admin_passwd = your_superadmin_password ; Change this
        db_host = <CLIENT_DB_HOSTNAME_OR_IP> ; Provided by client
        db_port = <CLIENT_DB_PORT>           ; Provided by client (e.g., 5432)
        db_user = <CLIENT_DB_USER>           ; Provided by client
        db_password = <CLIENT_DB_PASSWORD>   ; Provided by client
        db_name = <ODOO_DATABASE_NAME_ON_EXTERNAL_SERVER> ; Provided by client
        db_sslmode = prefer                  ; Or 'require', 'verify-full' - confirm with client
        addons_path = /mnt/extra-addons,/usr/lib/python3/dist-packages/odoo/addons
        data_dir = /var/lib/odoo             ; Odoo filestore path inside container
        ; list_db = True ; Set to False in production after setup
        ; proxy_mode = True ; If running behind a reverse proxy
        ```
        *   **Obtain all `<CLIENT_DB_...>` details from the client.**

*   **4.3. Prepare Custom Addons:**
    *   Extract `custom_addons.tar.gz` into `Odoo17/addons`.

*   **4.4. `Dockerfile` for Odoo (File: `Odoo17/Dockerfile`):**
    *   (Content remains the same as previously defined - focuses on Odoo app, not DB)
        ```Dockerfile
        # Use the official Odoo 17 image as a base
        ARG ODOO_VERSION=17.0
        FROM odoo:${ODOO_VERSION}
        ENV USER=odoo
        ENV HOME=/home/odoo
        # ... (rest of Dockerfile for system/python dependencies if needed) ...
        EXPOSE 8069 8071 8072
        ```

*   **4.5. `docker-compose.yml` (File: `Odoo17/docker-compose.yml`):**
    *   This file now only defines the Odoo service and its filestore volume.
    *   Content:
        ```yaml
        version: '3.8'

        services:
          odoo:
            build:
              context: .
              dockerfile: Dockerfile
              args:
                ODOO_VERSION: 17.0
            container_name: odoo17
            ports:
              - "8069:8069" # Odoo HTTP
              - "8071:8071" # Odoo Cron/Workers
              # - "8072:8072" # Odoo Longpolling
            volumes:
              - ./config/odoo.conf:/etc/odoo/odoo.conf
              - ./addons:/mnt/extra-addons
              - odoo_data:/var/lib/odoo # Filestore and other Odoo data
            # DB Environment variables are primarily set in odoo.conf for external DB
            restart: unless-stopped
            networks:
              - odoo_network

        volumes:
          odoo_data: # Defines the named volume for Odoo filestore

        networks:
          odoo_network:
            driver: bridge
        ```

### Phase 3: Data Restoration

*   **5.1. Client Restores Database:**
    *   The client is responsible for restoring the `odoo_database.dump` (from Phase 1) onto their external PostgreSQL server.
    *   They should ensure the database name, user, and permissions match what will be configured in `odoo.conf`.
    *   Coordinate with the client to confirm when this step is completed.

*   **5.2. Restore Odoo Filestore into Docker Volume:**
    *   Extract `odoo_filestore.tar.gz` on the Windows host (e.g., to `C:\Temp\extracted_filestore\<odoo_database_name_on_external_server>\`).
    *   Copy the contents into the Odoo container's filestore location (which maps to the `odoo_data` volume).
        Ensure the Odoo container (even if it fails to fully connect) has been started once with `docker-compose up -d odoo` to initialize the volume structure if needed, or find the volume path on host (can be complex with WSL2).
        A reliable method is to copy into the running (or temporarily started) container:
        ```powershell
        # Ensure 'odoo17' container is at least created/started
        # Example: <odoo_database_name_on_external_server> is 'prod_db_external'
        # Your extracted filestore is at C:\Temp\extracted_filestore\prod_db_external
        docker cp C:\Temp\extracted_filestore\prod_db_external odoo17:/var/lib/odoo/filestore/
        ```
        This places the filestore into the `odoo_data` volume. The target folder name inside `/var/lib/odoo/filestore/` should match the `db_name` used by Odoo.

*   **5.3. Start Odoo Application:**
    *   In the `Odoo17` directory, run: `docker-compose up -d`
    *   This builds/starts the Odoo container, which will attempt to connect to the client's external database.

*   **5.4. Check Logs:**
    *   Monitor Odoo logs for successful database connection and startup:
        *   `docker logs odoo17 -f`
    *   Troubleshoot any connection errors (check `odoo.conf`, network connectivity to DB, client DB firewall).

### Phase 4: Configuration and Testing

*   **6.1. Access Odoo:**
    *   Browser: `http://localhost:8069` or `http://<windows_server_ip>:8069`.
*   **6.2. Test Thoroughly:**
    *   Log in, verify data (ensure it's from the restored DB), custom addons, attachments (filestore), user permissions, workflows, reports.
*   **6.3. Update Addons List (if needed):**
    *   In Odoo: `Apps` -> `Update Apps List` (developer mode).

### Phase 5: Go-Live and Post-Migration

*   **7.1. Final Data Sync (Coordinated with Client):**
    *   If downtime needs to be minimized:
        1.  Stop old Odoo on CentOS.
        2.  Perform a final `pg_dump`.
        3.  Client restores this final dump to their external DB.
        4.  Sync any new filestore changes to the Docker volume.
*   **7.2. DNS Changes:**
    *   Point Odoo domain to the Windows Server 2022 IP.
*   **7.3. SSL Configuration (Reverse Proxy - CRITICAL for Production):**
    *   Implement a reverse proxy (Nginx, Traefik, or IIS on Windows host) for SSL/TLS (HTTPS).
    *   Set `proxy_mode = True` in `odoo.conf`.
*   **7.4. Monitoring:**
    *   Monitor Docker container (`docker stats`) and Odoo application logs.
    *   Client is responsible for monitoring their external database.
*   **7.5. Backup Strategy for Dockerized Odoo:**
    *   **Database:** Client is responsible for backing up their external PostgreSQL database. Document their strategy.
    *   **Odoo Filestore (from `odoo_data` volume):**
        ```powershell
        docker run --rm -v odoo_data:/data -v C:\Docker\Odoo17\backups:/backup alpine tar -czvf /backup/odoo_filestore_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .
        ```
    *   **Custom Addons & Config:** Back up the `Odoo17/addons` and `Odoo17/config` directories from the host.
*   **7.6. Decommission Old CentOS 7 VM:**
    *   After confirming stability and client approval.

## 8. Important Considerations & Potential Challenges

*   **Database Connectivity:** This is the primary new point of failure. Ensure robust network connection and correct firewall rules between Windows Server (Docker) and the client's database.
*   **Database Performance:** Performance can be affected by network latency to the external database.
*   **Client Coordination:** Clear communication and coordination with the client are essential for database restoration, configuration, and backups.
*   **Security:** Ensure secure connection to the external database (SSL/TLS, strong passwords, limited user privileges).
*   **Permissions:** For addons and filestore within Docker volumes.
*   **wkhtmltopdf:** Ensure correct version is in Docker image for PDF reports.
*   **Resource Allocation for Docker Desktop:** Sufficient CPU/RAM/disk for Docker on Windows Server.
*   **Python Dependencies for Custom Addons:** Manage via `requirements.txt` and `Dockerfile`.
*   **Windows Server Firewall:** Allow inbound connections on exposed ports (e.g., 8069, and 80/443 for reverse proxy).

This plan provides a roadmap. Always adapt paths, names, and passwords to your specific setup and **test thoroughly in a staging environment before production migration.** 