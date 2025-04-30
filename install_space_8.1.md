# DSpace 8.1 Installation Guide for Africarxiv

This guide provides detailed instructions for installing DSpace 8.1 on an Ubuntu server using the embedded Tomcat (runnable JAR) approach for the backend.

## System Requirements

- Ubuntu 20.04 LTS or newer
- At least 4GB RAM (8GB recommended)
- At least 20GB free disk space
- Internet connectivity for downloading packages

## 1. Install Prerequisites

First, update your system and install the necessary dependencies:

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install Java JDK 17
sudo apt install openjdk-17-jdk -y

# Install build tools
sudo apt install maven ant -y

# Install PostgreSQL and its extensions
sudo apt install postgresql postgresql-contrib -y

# Install utility packages
sudo apt install wget curl -y
```

## 2. Install and Configure Apache Solr

DSpace requires Apache Solr for search functionality:

```bash
# Create directory for Solr
sudo mkdir -p /opt/solr

# Download Solr 8.11.4 (security-patched version)
cd /tmp
wget https://archive.apache.org/dist/lucene/solr/8.11.4/solr-8.11.4.tgz

# Extract Solr
tar xzf solr-8.11.4.tgz
sudo cp -R solr-8.11.4/* /opt/solr

# Create solr user for security
sudo useradd -M -r solr
sudo chown -R solr:solr /opt/solr

# Create a systemd service file for Solr
sudo bash -c 'cat > /etc/systemd/system/solr.service << EOF
[Unit]
Description=Apache Solr
After=network.target

[Service]
User=solr
Group=solr
Environment=SOLR_HOME=/opt/solr
ExecStart=/opt/solr/bin/solr start -f
ExecStop=/opt/solr/bin/solr stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF'

# Enable and start Solr service
sudo systemctl daemon-reload
sudo systemctl enable solr
sudo systemctl start solr
```

## 3. Setup PostgreSQL Database for DSpace

Create a database and user for DSpace:

```bash
# Switch to postgres user
sudo -u postgres bash

# Create a dspace database user (you'll be prompted for a password)
createuser --no-superuser --pwprompt dspace

# Create a dspace database
createdb --owner=dspace --encoding=UNICODE dspace

# Enable the pgcrypto extension (required by DSpace)
psql dspace -c "CREATE EXTENSION pgcrypto;"

# Exit postgres user shell
exit
```

## 4. Create DSpace User

Create a dedicated system user for running DSpace:

```bash
sudo useradd -m dspace
```

## 5. Download and Extract DSpace

```bash
# Switch to dspace user
sudo -u dspace bash

# Download DSpace 8.1
cd /home/dspace
wget https://github.com/DSpace/DSpace/archive/refs/tags/dspace-8.1.tar.gz

# Extract DSpace
tar -xzvf dspace-8.1.tar.gz

# Exit dspace user shell
exit

# Create DSpace installation directory
sudo mkdir -p /dspace
sudo chown dspace:dspace /dspace
```

## 6. Configure DSpace Backend

Create and configure the local.cfg file:

```bash
# Switch to dspace user
sudo -u dspace bash

# Copy the example config file
cd /home/dspace/dspace-8.1/dspace/config
cp local.cfg.EXAMPLE local.cfg

# Edit local.cfg with your configuration
nano local.cfg
exit
```

Add the following configuration to your local.cfg (adjust values as needed):

```properties
# DSpace installation directory
dspace.dir = /dspace

# DSpace URLs
dspace.hostname = your-server-ip-or-domain
dspace.server.url = http://your-server-ip-or-domain:8080/server
dspace.ui.url = http://your-server-ip-or-domain

# Database configuration
db.url = jdbc:postgresql://localhost:5432/dspace
db.driver = org.postgresql.Driver
db.dialect = org.hibernate.dialect.PostgreSQL94Dialect
db.username = dspace
db.password = your-database-password
db.schema = public

# Email configuration
mail.server = smtp.example.com
mail.from.address = dspace@example.com
mail.feedback.recipient = dspace-admin@example.com
mail.admin = dspace-admin@example.com

# Solr server configuration
solr.server = http://localhost:8983/solr
```

## 7. Build and Install DSpace

Build and install the DSpace backend:

```bash
# Switch to dspace user
sudo -u dspace bash

# Build DSpace installation package
cd /home/dspace/dspace-8.1
mvn package

# Install DSpace
cd /home/dspace/dspace-8.1/dspace/target/dspace-installer
ant fresh_install

# Initialize the database
/dspace/bin/dspace database migrate

# Exit dspace user shell
exit
```

## 8. Configure Solr for DSpace

Copy DSpace's Solr cores to the Solr installation:

```bash
# Copy Solr configuration
sudo cp -R /dspace/solr/* /opt/solr/server/solr/configsets
sudo chown -R solr:solr /opt/solr/server/solr/configsets

# Restart Solr to apply changes
sudo systemctl restart solr
```

## 9. Create DSpace Administrator Account

Create the initial administrator account:

```bash
sudo -u dspace /dspace/bin/dspace create-administrator
# Follow the prompts to create your admin account
```

## 10. Deploy DSpace Backend Using Embedded Tomcat

Create a systemd service for the DSpace backend:

```bash
sudo bash -c 'cat > /etc/systemd/system/dspace-server.service << EOF
[Unit]
Description=DSpace Server
After=network.target

[Service]
User=dspace
Group=dspace
WorkingDirectory=/dspace
ExecStart=/usr/bin/java -jar /dspace/webapps/server-boot.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF'

# Enable and start the DSpace server
sudo systemctl daemon-reload
sudo systemctl enable dspace-server
sudo systemctl start dspace-server
```

## 11. Install Node.js and Yarn for the Frontend

```bash
# Install Node.js v20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Yarn package manager
sudo npm install -g yarn

# Install PM2 for process management
sudo npm install -g pm2
```

## 12. Download and Build the Frontend

```bash
# Switch to dspace user
sudo -u dspace bash

# Download the frontend source code
cd /home/dspace
wget https://github.com/DSpace/dspace-angular/archive/refs/tags/dspace-8.1.tar.gz -O angular-frontend.tar.gz

# Create and extract to a specific directory
mkdir -p angular-frontend
tar -xzvf angular-frontend.tar.gz -C angular-frontend --strip-components=1

# Build the frontend
cd angular-frontend
yarn install
yarn build:prod

# Create deployment directory
mkdir -p /home/dspace/dspace-ui-deploy
cp -r dist /home/dspace/dspace-ui-deploy/

# Exit dspace user shell
exit
```

## 13. Configure the Frontend

Create the frontend configuration file:

```bash
# Create config directory
sudo mkdir -p /home/dspace/dspace-ui-deploy/config

# Create config file
sudo nano /home/dspace/dspace-ui-deploy/config/config.prod.yml
```

Add this configuration (adjust as needed):

```yaml
ui:
  ssl: false
  host: 0.0.0.0  # Listen on all interfaces
  port: 4000
  nameSpace: /

rest:
  ssl: false
  host: your-server-ip-or-domain
  port: 8080
  nameSpace: /server
```

## 14. Set Up PM2 for Frontend Management

Create PM2 configuration:

```bash
sudo nano /home/dspace/dspace-ui-deploy/dspace-ui.json
```

Add this content:

```json
{
  "apps": [
    {
      "name": "dspace-ui",
      "cwd": "/home/dspace/dspace-ui-deploy",
      "script": "dist/server/main.js",
      "instances": "max",
      "exec_mode": "cluster",
      "env": {
        "NODE_ENV": "production"
      }
    }
  ]
}
```

Start and configure PM2:

```bash
cd /home/dspace/dspace-ui-deploy
sudo -u dspace pm2 start dspace-ui.json
sudo -u dspace pm2 save
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u dspace --hp /home/dspace
sudo systemctl enable pm2-dspace
```

## 15. Configure Apache as a Reverse Proxy

Install and configure Apache:

```bash
# Install Apache and required modules
sudo apt install apache2 -y
sudo a2enmod ssl headers proxy proxy_http
```

Create a virtual host configuration:

```bash
sudo nano /etc/apache2/sites-available/dspace.conf
```

Add this configuration:

```apache
<VirtualHost *:80>
    ServerName your-server-ip-or-domain
    
    # Preserve Host
    ProxyPreserveHost On
    
    # Proxy backend requests
    ProxyPass /server http://localhost:8080/server
    ProxyPassReverse /server http://localhost:8080/server
    
    # Proxy frontend requests
    ProxyPass / http://localhost:4000/
    ProxyPassReverse / http://localhost:4000/
    
    ErrorLog ${APACHE_LOG_DIR}/dspace_error.log
    CustomLog ${APACHE_LOG_DIR}/dspace_access.log combined
</VirtualHost>
```

Enable the site and restart Apache:

```bash
sudo a2ensite dspace.conf
sudo systemctl restart apache2
```

## 16. Configure Firewall (if enabled)

If you're using the UFW firewall, allow the necessary ports:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 4000/tcp
```

## 17. Set Up Scheduled Tasks

Configure cron jobs for maintenance tasks:

```bash
sudo -u dspace crontab -e
```

Add these tasks:

```cron
# Run the media filter daily at 2:00 AM
0 2 * * * /dspace/bin/dspace filter-media

# Run the indexer daily at 3:00 AM
0 3 * * * /dspace/bin/dspace index-discovery -b

# Run statistics aggregation daily at 4:00 AM
0 4 * * * /dspace/bin/dspace stat-general
```

## 18. Access Your DSpace Installation

You can now access your DSpace installation at:
- Frontend UI: http://your-server-ip-or-domain
- Backend REST API: http://your-server-ip-or-domain/server

## Troubleshooting

### Check Service Status

```bash
# Check backend status
sudo systemctl status dspace-server

# Check frontend status
pm2 status
pm2 logs dspace-ui

# Check Apache status
sudo systemctl status apache2

# Check Solr status
sudo systemctl status solr
```

### Check Log Files

```bash
# DSpace logs
tail -f /dspace/log/dspace.log

# Apache logs
tail -f /var/log/apache2/dspace_error.log
```

### Common Issues

1. **CORS Errors**: Make sure `rest.cors.allowed-origins` in local.cfg includes your frontend URL.

2. **Database Connection Issues**: Verify PostgreSQL is running and the credentials in local.cfg are correct.

3. **Solr Connection Issues**: Ensure Solr is running and the DSpace cores are properly installed.

4. **File Upload Problems**: Check permissions on the assetstore directory:
   ```bash
   sudo chown -R dspace:dspace /dspace/assetstore
   ```

5. **Memory Issues**: If experiencing out-of-memory errors, increase Java heap size in the systemd service file:
   ```bash
   ExecStart=/usr/bin/java -Xmx2048m -jar /dspace/webapps/server-boot.jar
   ```

## Upgrading to HTTPS (Recommended for Production)

For production environments, secure your DSpace installation with HTTPS:

```bash
# Install Certbot for Let's Encrypt SSL certificates
sudo apt install certbot python3-certbot-apache -y

# Obtain certificates (after setting up your domain)
sudo certbot --apache -d your-domain.com

# Update DSpace configurations to use HTTPS
```

## References

- [Official DSpace Documentation](https://wiki.lyrasis.org/display/DSDOC8x/DSpace+8.x+Documentation)
- [DSpace GitHub Repository](https://github.com/DSpace/DSpace)
- [DSpace Angular UI GitHub Repository](https://github.com/DSpace/dspace-angular)

## License

This installation guide is provided under the same license as DSpace: [BSD License](https://github.com/DSpace/DSpace/blob/main/LICENSE).
