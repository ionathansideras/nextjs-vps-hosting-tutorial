# Deployment & Setup Guide

1. Buy your domain and point it to your VPS’s IP address.

2. Connect to your VPS
    ```bash
    ssh root@<your-vps-ip>
    ```
3. Install system dependencies (`node.js, nginx, MySQL, pm2`)
    ```bash
    sudo apt update
    sudo apt install -y nodejs npm
    sudo apt install -y mysql-server mysql-client
    npm install -g pm2
    sudo apt install -y nginx
    ```

4. Prepare application directory
    ```bash
    sudo mkdir -p /var/www/<app-name>

    cd /var/www/<app-name>
    ```

5. Create your PM2 ecosystem config (`ecosystem.config.js`)
    ```js
    module.exports = {
        apps: [
            {
                name: "/var/www/<app-name>",
                script: "npm",
                args: "run start",
                instances: 1,
                autorestart: true,
                watch: false,
            },
        ],
    };
    ```
6. Generate an SSH key & clone your repo
    ```bash
    cd /var/www/
    ssh-keygen -t ed25519 -C "<your_email@example.com>"
    cat ~/.ssh/id_ed25519.pub   # copy this key into your Git repo’s deploy keys
    git clone git@github.com:your_username/your_repo.git
    ```
7. Install project dependencies
    ```bash
    cd your_repo
    npm install
    ```
8. Configure environment variables

    ```bash
    nano .env
    # Paste your variables, then:
    Ctrl+V            # Paste
    Ctrl+S            # Save
    Ctrl+X            # Exit
    cat .env          # Verify

    ```

9. Set up MySQL

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

10. Create Nginx site file
    ```bash
    sudo nano /etc/nginx/sites-available/<site-url>
    ```
11. Configure Nginx

    ```bash
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

12. Enable SSL with Certbot

    ```bash
    sudo apt update
    sudo apt install -y certbot python3-certbot-nginx
    sudo certbot --nginx \
    -d <site-url> \
    -d www.<site-url> \
    --agree-tos --redirect --no-eff-email \
    -m <your email>

    # restart nginx server
    sudo nginx -t
    sudo systemctl reload nginx
    ```

13. Build & start the app with PM2
    ```bash
    npm run build
    pm2 start ecosystem.config.js
    # To stop:
    pm2 stop ecosystem.config.js
    ```
