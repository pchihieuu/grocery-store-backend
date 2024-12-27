# Grocery Store Backend Setup

This repository contains the backend setup for the Grocery Store project, which leverages Magento 2 for local development using Warden and Docker. The following instructions will guide you through the steps for setting up and configuring the environment, including DNS resolution, SSL certificates, and Magento 2 installation.

## Prerequisites

Before you begin, make sure you have the following tools installed on your system:

- **Docker Desktop for Mac** (v2.2.0.0 or later), **Docker for Linux**, or **Docker for Windows**.
- **Docker Compose** (v2 or later).
- **Mutagen** (v0.11.4 or later) for environments that require sync sessions on macOS.
- **Homebrew** (for macOS or Linux).

### Docker Configuration

By default, Docker Desktop allocates 2GB of RAM, which may cause performance issues when running Magento actions (like `sampledata:deploy`). It is recommended to allocate at least 6GB of RAM. You can change this in **Preferences -> Resources -> Advanced -> Memory**.

### DNS Configuration

- On **Linux**, configure your DNS to resolve `*.test` to `127.0.0.1` or add entries to `/etc/hosts`:
  ```
  127.0.0.1 app.grocery-store.test
  ```
- On **MacOS**, this configuration is automatic through the `/etc/resolver/test` file.
- On **Windows (via WSL2)**, configure the network adapter DNS server or manually edit the hosts file.

### Install Warden via Homebrew

You can install **Warden** via Homebrew on both macOS and Linux:
```bash
brew install wardenenv/warden/warden
warden svc up
```

### Install Warden on Windows (via WSL2)

1. Install **WSL2** and **Ubuntu 20.04** or another compatible Linux version.
2. Enable **WSL2 integration** in Docker for Windows.
3. In your WSL terminal, install Homebrew and Warden:
   ```bash
   wsl
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
   brew install wardenenv/warden/warden
   warden svc up
   ```
4. Configure DNS entries by adding them to your Windows hosts file or setting `127.0.0.1` as the first DNS server in your network adapter.

## Magento 2 Setup

Follow these steps to set up Magento 2 locally:

### Step 1: Clone the Repository

Clone the project repository to your local machine:
```bash
git clone https://github.com/pchihieuu/grocery-store-backend.git
cd grocery-store-backend
```

### Step 2: Sign SSL Certificate

Sign an SSL certificate for the project:
```bash
warden sign-certificate grocery-store.test
```

### Step 3: Start the Environment

Start the project environment using Warden:
```bash
warden env up
```

### Step 4: Enter PHP Container Shell

Enter the PHP container shell to run Magento commands:
```bash
warden shell
```

### Step 5: Configure Magento Marketplace Credentials

Set your Magento Marketplace credentials:
```bash
composer global config http-basic.repo.magento.com <username> <password>
```

### Step 6: Install Magento

Run the Magento installation:
```bash
bin/magento setup:install     --backend-frontname=backend     --amqp-host=rabbitmq     --amqp-port=5672     --amqp-user=guest     --amqp-password=guest     --db-host=db     --db-name=magento     --db-user=magento     --db-password=magento     --search-engine=opensearch     --opensearch-host=opensearch     --opensearch-port=9200     --opensearch-index-prefix=magento2     --opensearch-enable-auth=0     --opensearch-timeout=15     --http-cache-hosts=varnish:80     --session-save=redis     --session-save-redis-host=redis     --session-save-redis-port=6379     --session-save-redis-db=2     --session-save-redis-max-concurrency=20     --cache-backend=redis     --cache-backend-redis-server=redis     --cache-backend-redis-db=0     --cache-backend-redis-port=6379     --page-cache=redis     --page-cache-redis-server=redis     --page-cache-redis-db=1     --page-cache-redis-port=6379
```

### Step 7: Configure the Application

Set the appropriate URLs and caching settings for Magento:
```bash
bin/magento config:set --lock-env web/unsecure/base_url "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"
bin/magento config:set --lock-env web/secure/base_url "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"
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

### Step 8: Create Admin User and Configure 2FA

Generate the admin user and configure two-factor authentication:
```bash
ADMIN_PASS="$(pwgen -n1 16)"
ADMIN_USER=localadmin

bin/magento admin:user:create     --admin-password="${ADMIN_PASS}"     --admin-user="${ADMIN_USER}"     --admin-firstname="Local"     --admin-lastname="Admin"     --admin-email="${ADMIN_USER}@example.com"

printf "u: %s
p: %s
" "${ADMIN_USER}" "${ADMIN_PASS}"
```

### Step 9: Disable 2FA Authentication

If you want to disable the 2FA module:
```bash
bin/magento module:disable -f Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth
```

### Step 10: Access the Application

Once everything is set up, you can access the application through the following URLs:
- Frontend: [https://app.grocery-store.test/](https://app.grocery-store.test/)
- Admin Panel: [https://app.grocery-store.test/backend/](https://app.grocery-store.test/backend/)
- RabbitMQ: [https://rabbitmq.grocery-store.test/](https://rabbitmq.grocery-store.test/)
- OpenSearch: [https://elasticsearch.grocery-store.test/](https://elasticsearch.grocery-store.test/)

### Step 11: Tear Down Environment

To completely destroy the environment, use the following command:
```bash
warden env down -v
```

This will tear down the Docker containers, volumes, and all other resources associated with the project.

## Contributing

Feel free to fork the repository and contribute by opening pull requests for any improvements or fixes you come across.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.





