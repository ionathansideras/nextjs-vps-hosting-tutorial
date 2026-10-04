# Deployment & Setup Guide

1. Buy your domain and point it to your VPS’s IP address.

2. Connect to your VPS:

    ```bash
    ssh root@<your-vps-ip>
    ```

3. Install system dependencies (`node.js`, `nginx`, `MySQL`, `pm2`):

    ```bash
    sudo apt update
    sudo apt install -y nodejs npm
    sudo apt install -y mysql-server mysql-client
    npm install -g pm2
    sudo apt install -y nginx
    ```

4. Prepare application directory:

    ```bash
    sudo mkdir -p /var/www/<app-name>

    cd /var/www/<app-name>
    ```

5. Generate an SSH key & clone your repo:

    ```bash
    cd /var/www/
    ssh-keygen -t ed25519 -C "<your_email@example.com>"
    cat ~/.ssh/id_ed25519.pub   # copy this key into your Git repo’s deploy keys
    git clone git@github.com:your_username/your_repo.git
    ```

6. Install project dependencies:

    ```bash
    cd your_repo
    npm install
    ```

7. Configure environment variables:

    ```bash
    # Create .env
    cat > .env

    # Paste your variables, then:
    Ctrl+D
    ```

8. Create your PM2 ecosystem config (`ecosystem.config.js`):

    ```js
    module.exports = {
        apps: [
            {
                name: "<app-name>",
                script: "npm",
                args: "run start",
                instances: 1,
                autorestart: true,
                watch: false,
            },
        ],
    };
    ```

9. Set up MySQL:

    ```bash
    # Check if MySQL is running
    sudo ss -tap | grep mysql

    # Secure the root user with a password
    sudo mysql -u root

    ALTER USER 'root'@'localhost'
    IDENTIFIED WITH mysql_native_password
    BY '<your-password>';

    FLUSH PRIVILEGES;
    EXIT;

    # Push Prisma schema
    npx prisma db push
    ```

10. Create Nginx site file.

    ```bash
    cat > <site-name>
    ```

12. Configure Nginx:

    ```nginx
    # paste this

    server {
        listen 80;
        server_name <site-url> www.<site-url>;

        # OPTIONAL Serve static media if you have a static folder outside of public
        location /<folder name>/ {
            alias /var/www/<site-url>/;
            autoindex on;
        }

        # Proxy to Next.js
        location / {
            proxy_pass         http://127.0.0.1:3000;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection 'upgrade';
            proxy_set_header   Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        access_log  /var/log/nginx/<site-name>.access.log;
        error_log   /var/log/nginx/<site-name>.error.log warn;
    }
    ```

13. Activate your config:

    ```bash
    # Disable the default Nginx site
    rm /etc/nginx/sites-enabled/default

    # Enable your site
    ln -s /etc/nginx/sites-available/<site-config-name> /etc/nginx/sites-enabled/<site-config-name>

    # Test Nginx configuration
    nginx -t

    # Reload Nginx
    systemctl reload nginx
    ```

14. Create `deployment-actions.sh`:

    ```bash
    nano deployment_actions.sh
    ```

15. Add the code:

    ```bash
    #!/bin/bash

    set -e

    APP_DIR="/var/www/<app-name>"
    BRANCH="main"
    PM2_APP="<app-name>"

    echo "================================"
    echo "Starting deployment"
    echo "================================"

    cd "$APP_DIR"

    echo ""
    echo "==> Updating repository..."
    git fetch origin "$BRANCH"
    git reset --hard "origin/$BRANCH"

    echo ""
    echo "==> Installing dependencies..."
    npm ci

    echo ""
    echo "==> Building application..."
    npm run build

    echo ""
    echo "==> Starting/restarting PM2..."

    if pm2 describe "$PM2_APP" > /dev/null 2>&1; then
        pm2 restart "$PM2_APP"
    else
        pm2 start ecosystem.config.js
    fi

    echo ""
    echo "================================"
    echo "Deployment successful!"
    echo "================================"
    ```

16. Then make the file executable:

    ```bash
    chmod +x deployment_actions.sh
    ```

17. Run the `deployment-actions.sh`:

    ```bash
    ./deployment_actions.sh
    ```

18. Make sure the domain A record is pointing to your VPS IP.

19. Enable SSL with Certbot:

    ```bash
    sudo apt update
    sudo apt install -y certbot python3-certbot-nginx

    sudo certbot --nginx \
        -d <site-url> \
        -d www.<site-url> \
        --agree-tos \
        --redirect \
        --no-eff-email \
        -m <your email>

    # Restart nginx server
    sudo nginx -t
    sudo systemctl reload nginx
    ```
