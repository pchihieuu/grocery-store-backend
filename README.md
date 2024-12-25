# Set up Backend service

## Installing Warden

### Prerequisites

Docker Desktop for Mac 2.2.0.0 or later or Docker for Linux (Warden has been tested on Fedora 29 and Ubuntu 18.10) or Docker for Windows

docker-compose version 2 or later is required (this can be installed via brew, apt, dnf, or pip3 as needed)

Mutagen 0.11.4 or later is required for environments leveraging sync sessions on Mac OS. Warden will attempt to install this via brew if not present.

### Installing via Homebrew

Warden may be installed via Homebrew on both macOS and Linux hosts:

```bash
brew install wardenenv/warden/warden
warden svc up
```

### Automatic DNS Resolution

On Linux environments, you will need to configure your DNS to resolve `*.test` to `127.0.0.1` or use `/etc/hosts` entries. 

```plaintext
127.0.0.1 app.grocery-store.test
```

On Mac OS this configuration is automatic via the BSD per-TLD resolver configuration found at `/etc/resolver/test`. On Windows manual configuration of the network adapter DNS server is required.

## Setup Magento 2

### Step 1: Clone the repository

Clone your project repository to your local machine:

```bash
git clone https://github.com/pchihieuu/grocery-store-backend.git
cd grocery-store-backend
```

### Step 2: Sign SSL certificate

Sign an SSL certificate for use with the project (the input here should match the value of `TRAEFIK_DOMAIN` in the above `.env` example file):

```bash
warden sign-certificate grocery-store.test
```

### Step 3: Start project environment

Next you’ll want to start the project environment:

```bash
warden env up
```

### Step 4: Enter the PHP container shell

Drop into a shell within the project environment. Commands following this step in the setup procedure will be run from within the `php-fpm` docker container this launches you into:

```bash
warden shell
```

Configure global Magento Marketplace credentials

```bash
composer global config http-basic.repo.magento.com <username> <password>
```

### Step 5: Install libraries

```bash
composer install
```

### Step 6: Install Magento application

Install the application and you should be all set:

```bash
## Install Application
bin/magento setup:install \
    --backend-frontname=backend \
    --amqp-host=rabbitmq \
    --amqp-port=5672 \
    --amqp-user=guest \
    --amqp-password=guest \
    --db-host=db \
    --db-name=magento \
    --db-user=magento \
    --db-password=magento \
    --search-engine=opensearch \
    --opensearch-host=opensearch \
    --opensearch-port=9200 \
    --opensearch-index-prefix=magento2 \
    --opensearch-enable-auth=0 \
    --opensearch-timeout=15 \
    --http-cache-hosts=varnish:80 \
    --session-save=redis \
    --session-save-redis-host=redis \
    --session-save-redis-port=6379 \
    --session-save-redis-db=2 \
    --session-save-redis-max-concurrency=20 \
    --cache-backend=redis \
    --cache-backend-redis-server=redis \
    --cache-backend-redis-db=0 \
    --cache-backend-redis-port=6379 \
    --page-cache=redis \
    --page-cache-redis-server=redis \
    --page-cache-redis-db=1 \
    --page-cache-redis-port=6379

## Configure Application
bin/magento config:set --lock-env web/unsecure/base_url \
    "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"

bin/magento config:set --lock-env web/secure/base_url \
    "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"

bin/magento config:set --lock-env web/secure/offloader_header X-Forwarded-Proto

bin/magento config:set --lock-env web/secure/use_in_frontend 1
bin/magento config:set --lock-env web/secure/use_in_adminhtml 1
bin/magento config:set --lock-env web/seo/use_rewrites 1

bin/magento config:set --lock-env system/full_page_cache/caching_application 2
bin/magento config:set --lock-env system/full_page_cache/ttl 604800

bin/magento config:set --lock-env catalog/search/enable_eav_indexer 1

bin/magento config:set --lock-env dev/static/sign 0

bin/magento deploy:mode:set -s developer
bin/magento cache:disable block_html full_page

bin/magento indexer:reindex
bin/magento cache:flush
```

### Step 7 : Create admin user and configure two-factor authentication

Generate an admin user and configure 2FA for OTP:

```bash
## Generate localadmin user
ADMIN_PASS="$(pwgen -n1 16)"
ADMIN_USER=localadmin

bin/magento admin:user:create \
    --admin-password="${ADMIN_PASS}" \
    --admin-user="${ADMIN_USER}" \
    --admin-firstname="Local" \
    --admin-lastname="Admin" \
    --admin-email="${ADMIN_USER}@example.com"
printf "u: %s\np: %s\n" "${ADMIN_USER}" "${ADMIN_PASS}"

## Configure 2FA provider
OTPAUTH_QRI=
# Python 2: TFA_SECRET=$(python -c "import base64; print base64.b32encode('$(pwgen -A1 128)')" | sed 's/=*$//')
# Python 3:
TFA_SECRET=$(python3 -c "import base64; print(base64.b32encode(bytearray('$(pwgen -A1 128)', 'ascii')).decode('utf-8'))" | sed 's/=*$//')
OTPAUTH_URL=$(printf "otpauth://totp/%s%%3Alocaladmin%%40example.com?issuer=%s&secret=%s" \
    "${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}" "${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}" "${TFA_SECRET}"
)

bin/magento config:set --lock-env twofactorauth/general/force_providers google
bin/magento security:tfa:google:set-secret "${ADMIN_USER}" "${TFA_SECRET}"

printf "%s\n\n" "${OTPAUTH_URL}"
printf "2FA Authenticator Codes:\n%s\n" "$(oathtool -s 30 -w 10 --totp --base32 "${TFA_SECRET}")"

segno "${OTPAUTH_URL}" -s 4 -o "pub/media/${ADMIN_USER}-totp-qr.png"
printf "%s\n\n" "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/media/${ADMIN_USER}-totp-qr.png?t=$(date +%s)"
```

### Step 8: Disable FA authentication

```bash
bin/magento module:disable -f Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth
```

### Step 9: Access the application

Open the following URLs in your browser:

- Frontend: https://app.grocery-store.test/
- Admin Panel: https://app.grocery-store.test/backend/
- RabbitMQ: https://rabbitmq.grocery-store.test/
- OpenSearch: https://elasticsearch.grocery-store.test/