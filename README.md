# Magento + Nginx + MariaDB + OpenSearch (Docker) — Obsidian Notes

A practical, reproducible setup for **Magento Open Source 2.4.8-p2** on Docker.  
This README doubles as your Obsidian note: it includes **what broke**, **why**, and **how we fixed it**.

---

## Stack

- **Magento Open Source** 2.4.8-p2
- **PHP-FPM** 8.2 with required extensions: `bcmath intl pdo_mysql soap xsl zip gd opcache ftp sockets`
- **Nginx** (serving Magento from `/pub`)
- **MariaDB** 10.6
- **OpenSearch** 2.12 (single-node, security disabled for dev)
- Docker Compose v2 (no `version:` key)

> Tip (WSL2/Linux): OpenSearch needs `vm.max_map_count = 262144`:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl --system
```

---

## Quick start

1) **Copy env file** and set a strong OpenSearch password:
```bash
cp .env.example .env
# edit .env and set OPENSEARCH_PASSWORD=Mag3nto!2025DevSecure (or your own strong value)
```

2) **Up the stack**:
```bash
docker compose up -d --build
```

3) **Create Magento project** (into the bind mount `/var/www/html` → `./src` on host):
```bash
docker compose exec php bash -lc '
export COMPOSER_MEMORY_LIMIT=-1
composer --version
composer config --global http-basic.repo.magento.com <public_key> <private_key>
composer create-project --repository-url=https://repo.magento.com/ --no-interaction --prefer-dist \
  magento/project-community-edition /var/www/html
'
```

4) **Permissions (dev-safe)**:
```bash
docker compose exec php bash -lc '
cd /var/www/html && \
mkdir -p var/cache var/page_cache var/session generated pub/static pub/media app/etc && \
chown -R www-data:www-data var generated pub/static pub/media app/etc && \
find var generated pub/static pub/media app/etc -type d -exec chmod 775 {} + && \
find var generated pub/static pub/media app/etc -type f -exec chmod 664 {} + && \
chmod g+s var generated pub/static pub/media app/etc && \
chmod u+x bin/magento
'
```

5) **Install Magento**:
```bash
docker compose exec php php -d memory_limit=-1 bin/magento setup:install \
  --base-url="http://localhost:8080/" \
  --db-host=db --db-name=magento --db-user=magento --db-password=magento \
  --admin-firstname=Admin --admin-lastname=User \
  --admin-email=admin@example.com --admin-user=admin --admin-password='Admin123!' \
  --language=en_US --currency=USD --timezone=UTC --use-rewrites=1 \
  --search-engine=opensearch --opensearch-host=opensearch --opensearch-port=9200 \
  --opensearch-timeout=30
```

6) **Base URLs & dev tuning**:
```bash
docker compose exec php bin/magento setup:store-config:set \
  --base-url="http://localhost:8080/" \
  --base-url-secure="http://localhost:8080/"
docker compose exec php bin/magento config:set dev/static/sign 0
docker compose exec php bin/magento deploy:mode:set developer
```

7) **Deploy static assets** (copy mode, low memory pressure):
```bash
docker compose exec php bash -lc '
cd /var/www/html && \
find pub/static -mindepth 1 -maxdepth 1 ! -name ".htaccess" -exec rm -rf {} + ; \
rm -rf var/view_preprocessed/* generated/* || true
'
docker compose exec php php -d memory_limit=2G bin/magento setup:static-content:deploy -f \
  --area frontend --theme Magento/luma  en_US --jobs 1 -s standard
docker compose exec php php -d memory_limit=2G bin/magento setup:static-content:deploy -f \
  --area adminhtml --theme Magento/backend en_US --jobs 1 -s standard
docker compose exec php bin/magento cache:flush
```

8) **Open it**:
- Storefront → <http://localhost:8080>
- Admin → `docker compose exec php bin/magento info:adminuri` (open that path under `http://localhost:8080/…`)

---

## Services

- **Nginx** on port **8080** → serves `/var/www/html/pub`
- **PHP-FPM** listens on `php:9000`
- **MariaDB** (user/db/pass): `magento / magento / magento`
- **OpenSearch** with `plugins.security.disabled=true` (dev). Provide `OPENSEARCH_INITIAL_ADMIN_PASSWORD` anyway (2.12+ requirement).

---

## Files

- `docker-compose.yml` — stack definition (no `version:` key)
- `php/Dockerfile` — PHP 8.2 FPM + extensions + Composer + sane `php.ini`
- `nginx/default.conf` — vhost serving from `/pub`
- `src/` — bind-mounted app code

---

## Troubleshooting log (what we hit + fixes)

### 1) `bin/magento: no such file or directory`
**Cause:** Project path empty.  
**Fix:** Run `composer create-project … /var/www/html` inside the PHP container.

---

### 2) Composer requires **ext-ftp**, then **ext-sockets**
**Fix:** Add to Dockerfile:
```Dockerfile
docker-php-ext-install bcmath intl pdo_mysql soap xsl zip gd opcache ftp sockets
```
Rebuild `php` image.

---

### 3) OpenSearch: `No alive nodes found in your cluster`
**Causes:** Weak/missing admin password (2.12+), or low `vm.max_map_count`.  
**Fix:** Set strong `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, add `plugins.security.disabled=true`, set `vm.max_map_count`, restart `opensearch`.

---

### 4) `Forbidden` storefront / Admin 404
**Cause:** Nginx served `/var/www/html` instead of `/var/www/html/pub`.  
**Fix:** Change `root` and reload Nginx.

---

### 5) `cache_dir … is not writable`
**Cause:** Bind mount perms.  
**Fix:** dev-safe ownership & perms on `var`, `generated`, `pub/static`, `pub/media`, `app/etc`.

---

### 6) `Allowed memory size of 134217728 bytes exhausted`
**Fix:** Create `/usr/local/etc/php/conf.d/99-magento.ini` with `memory_limit=2G`, restart PHP, pass `php -d memory_limit=2G` for heavy CLI tasks.

---

### 7) `502 Bad Gateway`
**Fixes that worked:**
- Ensure Nginx `root /var/www/html/pub;`
- Increase PHP memory for FPM (via `99-magento.ini`)
- Optional Nginx FastCGI buffers for large headers:
```nginx
fastcgi_buffer_size 64k;
fastcgi_buffers 16 32k;
fastcgi_busy_buffers_size 64k;
fastcgi_temp_file_write_size 64k;
```

---

### 8) Unstyled pages (CSS/JS 404)
**Fix sequence:** base URLs include `:8080`, disable static signing in dev, clean & redeploy static assets (copy mode), ensure `pub/static/...` files exist.

---

### 9) Admin blocked by 2FA + email not configured
**Option A (keep 2FA, no email):** Force Google/TOTP and reset user:
```bash
docker compose exec php bin/magento config:set twofactorauth/general/force_providers google
docker compose exec php bin/magento twofactorauth:reset admin || true
docker compose exec php bin/magento cache:flush
```
**Option B (dev-only):** Disable dependent first, then core 2FA:
```bash
docker compose exec php bin/magento module:disable Magento_AdminAdobeImsTwoFactorAuth
docker compose exec php bin/magento setup:upgrade
docker compose exec php bin/magento module:disable Magento_TwoFactorAuth
docker compose exec php bin/magento cache:flush
```

---

## Handy commands

```bash
docker compose ps
docker compose logs -f nginx
docker compose logs -f php
docker compose logs -f opensearch

docker compose exec php bin/magento info:adminuri
docker compose exec php bin/magento cache:flush
docker compose exec php php -d memory_limit=2G bin/magento indexer:reindex
docker compose exec php php -d memory_limit=2G bin/magento setup:static-content:deploy -f en_US --jobs 1

docker compose exec php bin/magento admin:user:unlock admin
docker compose exec php bin/magento admin:user:change-password --username=admin --password='NewStrongPass123!'
```

---

## Optional: MailHog (capture emails locally)

Add service (and open <http://localhost:8025>):

```yaml
mailhog:
  image: mailhog/mailhog
  ports:
    - "8025:8025"
```

Configure Magento SMTP:
```bash
docker compose exec php bin/magento config:set system/smtp/host mailhog
docker compose exec php bin/magento config:set system/smtp/port 1025
docker compose exec php bin/magento cache:flush
```

---

## Notes

- Compose v2 ignores `version:` — we omit it to silence warnings.
- Keep `/pub` as document root for security & routing.
- For production: add TLS, Redis, and proper file ownership without 777.
