# Running Supabase Postgres with Automatic SSL Certificate Reloads

In environments where you don't (or can't) modify postgresql.conf directly, you can keep SSL certificates up to date by introducing a small wrapper script that copies refreshed Let's Encrypt certificates into place every 24 hours and reloads Postgres. This guide shows how to wire it up inside docker-compose.yml, using nginx-proxy-manager for certificate issuance and proxying.

## Table of Contents
1. Architecture Overview
2. docker-compose.yml
3. Wrapper Script: pg_ssl_wrapper.sh
4. How It Works
5. Notes and Best Practices

## Architecture Overview
- nginx-proxy-manager handles TLS certificates via Let's Encrypt, exposing ports 80/443/81.
- A custom wrapper starts Postgres with SSL enabled, then watches the certificate directory and reloads Postgres when certificates change.
- Certificates are shared into the Postgres container via a bind mount from the nginx data directory.

## docker-compose.yml

```yaml
version: "3.8"

networks:
  supabase_internal:
    name: supabase_internal
    driver: bridge
  proxy_network:
    name: proxy_network
    driver: bridge

services:
  nginx:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    networks:
      - proxy_network
      - supabase_internal
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./nginx-data:/data
      - ./nginx-letsencrypt:/etc/letsencrypt
      - ./nginx-snippets:/snippets:ro
    environment:
      TZ: Europe/Berlin
      PROXY_REAL_IP_RECURSIVE: "on"
      PROXY_REAL_IP_FROM: "10.0.0.0/24"

  supabase-db:
    container_name: supabase-db
    image: supabase/postgres:15.6.1.145
    restart: unless-stopped
    ports:
      - "5432:5432"
    networks:
      - supabase_internal
    depends_on:
      vector:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10
    command: ["bash", "/usr/local/bin/pg_ssl_wrapper.sh"]
    environment:
      POSTGRES_HOST: /var/run/postgresql
      PGPORT: ${POSTGRES_PORT}
      POSTGRES_PORT: ${POSTGRES_PORT}
      PGPASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATABASE: ${POSTGRES_DB}
      POSTGRES_DB: ${POSTGRES_DB}
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXP: ${JWT_EXPIRY}
      POSTGRES_MAX_CONNECTIONS: ${DB_MAX_CONNECTIONS}
      POSTGRES_SHARED_BUFFERS: ${DB_SHARED_BUFFERS}
      POSTGRES_WORK_MEM: ${DB_WORK_MEM}
      POSTGRES_MAINTENANCE_WORK_MEM: ${DB_MAINTENANCE_WORK_MEM}
      POSTGRES_EFFECTIVE_CACHE_SIZE: ${DB_EFFECTIVE_CACHE_SIZE}
      POSTGRES_RANDOM_PAGE_COST: ${DB_RANDOM_PAGE_COST}
      POSTGRES_SYNCHRONOUS_COMMIT: ${DB_SYNCHRONOUS_COMMIT}
      POSTGRES_TEMP_FILE_LIMIT: ${DB_TEMP_FILE_LIMIT}
      POSTGRES_WAL_BUFFERS: ${DB_WAL_BUFFERS}
      POSTGRES_MAX_WAL_SIZE: ${DB_MAX_WAL_SIZE}
    volumes:
      # Initialization and migrations
      - ./volumes/db/realtime.sql:/docker-entrypoint-initdb.d/migrations/99-realtime.sql:Z
      - ./volumes/db/webhooks.sql:/docker-entrypoint-initdb.d/init-scripts/98-webhooks.sql:Z
      - ./volumes/db/roles.sql:/docker-entrypoint-initdb.d/init-scripts/99-roles.sql:Z
      - ./volumes/db/jwt.sql:/docker-entrypoint-initdb.d/init-scripts/99-jwt.sql:Z
      - ./volumes/db/_supabase.sql:/docker-entrypoint-initdb.d/migrations/97-_supabase.sql:Z
      - ./volumes/db/logs.sql:/docker-entrypoint-initdb.d/migrations/99-logs.sql:Z
      - ./volumes/db/pooler.sql:/docker-entrypoint-initdb.d/migrations/99-pooler.sql:Z

      # Data directory
      - ./volumes/db/data:/var/lib/postgresql/data:Z

      # Custom config & certificates
      - db-config:/etc/postgresql-custom
      - ./nginx-letsencrypt:/etc/letsencrypt_host:ro
      - ./volumes/db/pg_certs:/etc/postgresql/custom_certs:Z
      - ./pg_ssl_wrapper.sh:/usr/local/bin/pg_ssl_wrapper.sh:ro

volumes:
  db-config:
```

Key command:

```
command: ["bash", "/usr/local/bin/pg_ssl_wrapper.sh"]
```

This runs our wrapper script instead of the default entrypoint so we can manage SSL certs.

## Wrapper Script: pg_ssl_wrapper.sh

Place this file next to your docker-compose.yml and make it executable (chmod +x pg_ssl_wrapper.sh):

```bash
#!/bin/bash
# pg_ssl_wrapper.sh
set -e

CERT_SRC_DIR="/etc/letsencrypt_host/live/npm-32"
CERT_DEST_DIR="/etc/postgresql/custom_certs"
PG_DATA_DIR="/var/lib/postgresql/data"

FULLCHAIN_SRC="${CERT_SRC_DIR}/fullchain.pem"
KEY_SRC="${CERT_SRC_DIR}/privkey.pem"
FULLCHAIN_DEST="${CERT_DEST_DIR}/server.crt"
KEY_DEST="${CERT_DEST_DIR}/server.key"
CURRENT_CERT_SUM_FILE="${CERT_DEST_DIR}/current.cert.md5"
CURRENT_KEY_SUM_FILE="${CERT_DEST_DIR}/current.key.md5"

log() {
  echo "$(date +'%Y-%m-%d %H:%M:%S') - INFO: $1"
}
error_log() {
  echo "$(date +'%Y-%m-%d %H:%M:%S') - ERROR: $1" >&2
}

copy_and_set_perms_and_sums() {
  log "Copying certificates and updating checksums..."
  mkdir -p "$CERT_DEST_DIR"

  if [[ ! -f "$FULLCHAIN_SRC" || ! -f "$KEY_SRC" ]]; then
    error_log "Certificate source files not found."
    return 1
  fi

  cp "$FULLCHAIN_SRC" "$FULLCHAIN_DEST"
  cp "$KEY_SRC"     "$KEY_DEST"
  chown postgres:postgres "$FULLCHAIN_DEST" "$KEY_DEST"
  chmod 0644 "$FULLCHAIN_DEST"
  chmod 0600 "$KEY_DEST"

  md5sum "$FULLCHAIN_DEST" | awk '{print $1}' > "$CURRENT_CERT_SUM_FILE"
  md5sum "$KEY_DEST"      | awk '{print $1}' > "$CURRENT_KEY_SUM_FILE"
  chown postgres:postgres "$CURRENT_CERT_SUM_FILE" "$CURRENT_KEY_SUM_FILE"

  log "Certificates copied; checksums stored."
  return 0
}

# 1. Initial copy (so Postgres can start with SSL immediately)
if ! copy_and_set_perms_and_sums; then
  error_log "Initial SSL setup failed (Postgres may start without SSL)."
fi

# 2. Start Postgres in the background with SSL enabled
log "Launching PostgreSQL..."
gosu postgres postgres \
  -D "$PG_DATA_DIR" \
  -c config_file=/etc/postgresql/postgresql.conf \
  -c ssl=on \
  -c ssl_cert_file="$FULLCHAIN_DEST" \
  -c ssl_key_file="$KEY_DEST" \
  -c max_connections=800 \
  -c log_min_messages=fatal &

PG_PID=$!
trap 'log "Shutting down PostgreSQL..."; kill -SIGTERM "$PG_PID"; wait "$PG_PID"; exit 0;' SIGINT SIGTERM

# 3. Wait for full initialization, then enter update loop
log "Waiting 30s before monitoring certs..."
sleep 30

while true; do
  log "Checking for updated certificates..."
  src_cert_sum=$(md5sum "$FULLCHAIN_SRC" | awk '{print $1}')
  src_key_sum=$(md5sum "$KEY_SRC"     | awk '{print $1}')
  dest_cert_sum=$(cat "$CURRENT_CERT_SUM_FILE" 2>/dev/null || echo "")
  dest_key_sum=$(cat "$CURRENT_KEY_SUM_FILE" 2>/dev/null || echo "")

  if [[ "$src_cert_sum" != "$dest_cert_sum" || "$src_key_sum" != "$dest_key_sum" ]]; then
    log "Change detected—updating certs and reloading Postgres."
    if copy_and_set_perms_and_sums; then
      gosu postgres pg_ctl reload -D "$PG_DATA_DIR" -s \
        && log "Postgres reloaded successfully."
    else
      error_log "Failed to update certs; Postgres not reloaded."
    fi
  else
    log "No changes detected."
  fi

  log "Sleeping 24 hours before next check."
  sleep 86400
done
```

## How It Works
1. **Initial Copy**: On container start, the script copies Let's Encrypt fullchain.pem and privkey.pem into custom_certs inside the Postgres data volume, then stores MD5 checksums.
2. **Background Postgres**: Postgres launches with SSL enabled, pointing at the copied certs.
3. **Monitoring Loop**: Every 24 h (after a 30 s startup delay), the script recomputes checksums; if they changed, it copies new certs and reloads Postgres (pg_ctl reload).

## Notes and Best Practices
- **File Permissions**:
  - server.key must be 0600, owner postgres.
  - server.crt can be 0644.
- **Graceful Shutdown**: Catches SIGINT/SIGTERM to stop Postgres cleanly.
- **Certificate Path**: Adjust CERT_SRC_DIR to where your proxy keeps live certs.
- **Compose Version**: Uses format 3.8 but works in most modern Docker setups.
- **Renewal Frequency**: nginx-proxy-manager renews ~every 60 days; shorten the loop if you need faster detection (e.g. every 6 h).
