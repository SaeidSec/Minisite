# Step-by-Step Guide: Configuring SSL Certificates on Wazuh Dashboard using NGINX

This guide provides detailed instructions on how to configure SSL certificates for your Wazuh dashboard using NGINX as a reverse proxy. This setup enhances security by providing trusted SSL certificates (via Let's Encrypt) and offloading SSL decryption from the Wazuh dashboard, improving performance and ensuring a secure connection.

## Prerequisites

*   A perfectly working single-node manual deployment of Wazuh.
*   A registered domain name pointing to your Wazuh dashboard server's public IP address.
*   Basic understanding of Linux command-line operations.

## Phase 1: Setting up NGINX as a Reverse Proxy

### 1.1. Installing NGINX Software

Install NGINX on the same endpoint hosting your Wazuh dashboard. This guide assumes a Yum-based system (e.g., CentOS, RHEL, Fedora).

```bash
sudo yum install epel-release
sudo yum install nginx
```

### 1.2. Starting NGINX and Verifying Status

Start the NGINX service and confirm it is running correctly.

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

### 1.3. Opening Firewall Ports

Ensure that ports 80 (HTTP) and 443 (HTTPS) are open in your firewall to allow external access to NGINX.

```bash
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

## Phase 2: Configuring Proxy and Obtaining Certificates

### 2.1. Installing Snap and Certbot

Certbot, used for obtaining and renewing Let's Encrypt certificates, is often installed via Snap. First, install Snap, then Certbot.

```bash
sudo yum install epel-release
sudo yum upgrade
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
```

After installing Snap, install Certbot:

```bash
sudo yum remove certbot # Remove any existing OS package version of Certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### 2.2. Adjusting Wazuh Dashboard Port

To avoid port conflicts with NGINX (which will listen on 443), change the default port of the Wazuh dashboard. Edit the `/etc/wazuh-dashboard/opensearch_dashboards.yml` file.

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Locate the `server.port` line and change it to an available port number (e.g., `5601` or `8443`). Ensure `server.ssl.enabled` is `true` and the certificate paths are correct for the internal Wazuh dashboard SSL.

```yaml
server.host: 0.0.0.0
opensearch.hosts: https://127.0.0.1:9200
server.port: <YOUR_NEW_PORT_NUMBER> # e.g., 8443
opensearch.ssl.verificationMode: certificate
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
server.ssl.enabled: true
server.ssl.key: "/etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/wazuh-dashboard.pem"
# ... other configurations
```

### 2.3. Creating NGINX Configuration for Wazuh

Navigate to the NGINX configuration directory and create a new configuration file for Wazuh. This file will define how NGINX proxies requests to the Wazuh dashboard.

```bash
sudo unlink /etc/nginx/sites-enabled/default # Remove default NGINX site
sudo nano /etc/nginx/conf.d/wazuh.conf
```

Add the following initial configuration to `wazuh.conf`. This sets up NGINX to listen on port 80 and proxy requests to your Wazuh dashboard.

```nginx
server {
    listen 80 default_server;
    server_name <YOUR_DOMAIN_NAME>;

    location / {
        proxy_pass https://<WAZUH_DASHBOARD_IP_ADDRESS>:<YOUR_NEW_PORT_NUMBER>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Replace the placeholders:**

*   `<YOUR_DOMAIN_NAME>`: Your registered domain name (e.g., `wazuh.example.com`).
*   `<WAZUH_DASHBOARD_IP_ADDRESS>`: The IP address of your Wazuh dashboard server (e.g., `127.0.0.1` if NGINX is on the same server).
*   `<YOUR_NEW_PORT_NUMBER>`: The port you configured in `opensearch_dashboards.yml` (e.g., `8443`).

### 2.4. Restarting Wazuh Services

Restart the Wazuh dashboard and manager services to apply the port change.

```bash
sudo systemctl restart wazuh-dashboard
sudo systemctl restart wazuh-manager
```

### 2.5. Generating SSL Certificate with Certbot

Now, use Certbot to obtain a Let's Encrypt SSL certificate for your domain. Certbot will automatically detect your NGINX configuration and modify it to include the SSL settings.

```bash
sudo certbot --nginx -d <YOUR_DOMAIN_NAME>
```

Follow the prompts from Certbot. It will ask for an email address for urgent renewal notices and terms of service agreement. It will also ask if you want to redirect HTTP traffic to HTTPS, which is recommended.

### 2.6. Verifying NGINX Configuration

After Certbot runs, it will have updated your `/etc/nginx/conf.d/wazuh.conf` file. Verify that the configuration now includes the SSL directives. It should look similar to this:

```nginx
server {
    server_name <YOUR_DOMAIN_NAME>;

    location / {
        proxy_pass https://<WAZUH_DASHBOARD_IP_ADDRESS>:<YOUR_NEW_PORT_NUMBER>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<YOUR_DOMAIN_NAME>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<YOUR_DOMAIN_NAME>/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = <YOUR_DOMAIN_NAME>) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80 default_server;
    server_name <YOUR_DOMAIN_NAME>;
    return 404; # managed by Certbot
}
```

### 2.7. Restarting NGINX Service

Restart NGINX to load the new SSL configuration.

```bash
sudo systemctl restart nginx
```

## Phase 3: Accessing Wazuh Dashboard

Now you can access your Wazuh dashboard securely via your configured domain name using HTTPS.

Open your web browser and navigate to `https://<YOUR_DOMAIN_NAME>`.

You should see the Wazuh dashboard login page with a valid SSL certificate, indicated by a padlock icon in your browser's address bar.

## References

*   [1] Wazuh Documentation: Configuring SSL certificates on the Wazuh dashboard using NGINX. Available at: https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/configuring-third-party-certs/ssl-nginx.html
*   [2] OSINT Team Blog: How to Enable Free SSL (HTTPS) on Your Wazuh Dashboard with Let’s Encrypt and Nginx. Available at: https://osintteam.blog/how-to-enable-free-ssl-https-on-your-wazuh-dashboard-with-lets-encrypt-and-nginx-df8b3aefafa6
