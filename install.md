# Deployment Guide to Installing Odoo 14 on Ubuntu 18.04 / 20.04

### Table Of Contents:
1. [Preparing System](#1-preparing-system)
2. [Create Odoo user](#2-create-odoo-user)
3. [Install and Configure PostgreSQL](#3-install-and-configure-postgresql)
4. [Install Wkhtmltopdf](#4-install-wkhtmltopdf)
5. [Install and Configure Odoo](#5-install-and-configure-odoo)
6. [Create a Systemd Unit File](#6-create-a-systemd-unit-file)
7. [Test the Installation](#7-test-the-installation)
8. [Configure Nginx as SSL Termination Proxy](#8-configure-nginx-as-ssl-termination-proxy)
9. [Change the binding interface](#9-change-the-binding-interface)
10. [Enable Multiprocessing](#10-enable-multiprocessing)
11. [General Issues](#11-general-issues)
12. [Conclusion](#12-conclusion)

## 1. Preparing System

Login to your machine as a sudo user and update the system to the latest packages:

```sh
sudo apt update && sudo apt -y upgrade
```

Install Git, PIP, NodeJS and the tools required to build Odoo dependencies:

```sh
sudo apt install git python3-pip build-essential wget python3-dev python3-venv python3-wheel libxslt-dev libzip-dev libldap2-dev libsasl2-dev python3-setuptools node-less virtualenv
```

Install Pillow Dependencies:

```sh
sudo apt-get install libtiff5-dev libjpeg8-dev libopenjp2-7-dev zlib1g-dev libfreetype6-dev liblcms2-dev libwebp-dev tcl8.6-dev tk8.6-dev python3-tk libharfbuzz-dev libfribidi-dev libxcb1-dev libcairo2-dev libjpeg-dev libgif-dev
```

## 2. Create Odoo user

Create a new system user for odoo named odoo17 with home directory /odoo17 using the following command:

```sh
sudo useradd -m -d /odoo17 -U -r -s /bin/bash odoo17
```

Note: you can use any name for your Odoo user as long you create a PostgreSQL user with the same name.

## 3. Install and Configure PostgreSQL

Install the PostgreSQL package from the Ubuntu's default repositories:

```sh
sudo apt install libpq-dev postgresql-client postgresql-client-common python3-psycopg2 postgresql postgresql-contrib
```

Once the installation is completed, create a PostgreSQL user with the same name as the previously created system user, in our case that is odoo17:

```sh
sudo su - postgres -c "createuser -s odoo17"
```

## 4. Install Wkhtmltopdf

The wkhtmktox packages provides a set of open source command line tools which can render HTML into PDF and various image formats. In order to print PDF reports, you will need the wkhtmltopdf tool. The recommended version for Odoo is 0.12.1 which is not available in the official Ubunti 18.04 repositories.

Download the package using the following wget command:

```sh
# Ubuntu 22.04 jammy
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
# Ubuntu 20.04 focal
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6-1/wkhtmltox_0.12.6-1.focal_amd64.deb
# Ubuntu 18.04 bionic
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6-1/wkhtmltox_0.12.6-1.bionic_amd64.deb
```

Once the download is completed install the package by typing:

```sh
sudo apt install ./wkhtmltox_0.12.6-1.*.deb
```

## 5. Install and Configure Odoo

We will install Odoo from the Github repository inside an isolated Python virtual environment.

Before starting with the installation process, change to user "odoo17":

```sh
sudo su - odoo17
```

Start by cloning the Odoo 14 source from the Github	repository:

```sh
git clone https://github.com/odoo/odoo --depth 1 --branch 14.0 odoo17-server/
```

Once the source code is downloaded, create a new Python virtual environment for the Odoo 14 installation:

```sh
cd /odoo17/odoo17-server
virtualenv -p python3 env
```

Next, activate the environment with the following command:

```sh
source env/bin/activate
```

Install all required Python modules with pip:

```sh
pip install wheel && pip install -r requirements.txt
```

> Note: If you encounter any compilation errors during the installation, make sure that you installed all of the required dependencies listed in the Before you begin section.

Deactivate the environment using the following command:

```sh
deactivate
```

create a new directory for the custom addons:

```sh
mkdir /odoo17/odoo17-server/custom
```

Switch back to you sudo user:

```sh
exit
```

Next, create a configuration file, by copying the included sample configuration file:

```sh
sudo cp /odoo17/odoo17-server/odoo/debian/odoo.conf /odoo17/odoo17-server/odoo17-server.conf
```

Open the file and edit it as follows:

```sh
sudo nano /odoo17/odoo17-server/odoo17-server.conf
```

```ini
[options]
; This is the password that allows database operations:
admin_passwd = my_admin_passwd
db_host = localhost
db_port = 5432
db_user = False
db_password = False
addons_path = /odoo17/odoo17-server/addons,/odoo17/odoo17-server/custom
```

> Note: Do not forget to change the my_admin_passwd to something more secure.

## 6. Create a Systemd Unit File

To run Odoo as a service we need to create a service unit file in the /etc/systemd/system/ directory.

Open your text editor and paste the following configuration:

```sh
sudo nano /etc/systemd/system/odoo17.service
```

In directory: `/etc/systemd/system/odoo17.service`

```ini
[Unit]
Description=odoo17 Community Edition
Requires=postgresql.service
After=network.target postgresql.service

[Service]
Type=simple
SyslogIdentifier=odoo17
PermissionsStartOnly=true
User=odoo17
Group=odoo17
ExecStart=/odoo17/odoo17-server/env/bin/python3 /odoo17/odoo17-server/odoo-bin -c /odoo17/odoo17-server/odoo17-server.conf
StandardOutput=journal+console

[Install]
WantedBy=multi-user.target
```

Notify systemd that a new unit file exist and start the Odoo service by running:

```sh
sudo systemctl daemon-reload
sudo systemctl start odoo17
```

Enable the Odoo service to be automatically started at boot time:

```sh
sudo systemctl enable odoo17
```

Check the service status with the following command:

```sh
sudo systemctl status odoo17
```

The output should look something like below indicating that Odoo service is active and running.

Output:

```sh
* odoo17.service - odoo17 Community Edition
   Loaded: loaded (/etc/systemd/system/odoo17.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2021-11-04 12:12:10 PDT; 3s ago
 Main PID: 24334 (python3)
    Tasks: 4 (limit: 2319)
   CGroup: /system.slice/odoo17.service
           `-24334 /odoo17/odoo17-server/env/bin/python3 /odoo17/odoo17-server/odoo-bin -c /odoo17/odoo17-server/odoo17-server.conf
```

If you want to see the messages logged by the Odoo service you can use the command below:

```sh
sudo journalctl -u odoo17
```

## 7. Test the Installation

Open your browser and type: `http://<your_domain_or_IP_address>:8069`

Assuming the installation is successful, a screen similar to the following will appear in browser.

## 8. Configure Nginx as SSL Termination Proxy 

Ensure that you have met the following prerequisites before continuing with this section:

* Domain name pointing to your public server IP. In this tutorial we will use example.com.
* Nginx installed.
* SSL certificate for your domain. You can install a free Let's Encrypt SSL certificate.

The default Odoo web server is serving traffic over HTTP. To make our Odoo deployment more secure we will configure Nginx as a SSL termination proxy that will serve the traffic over HTTPS.

SSL termination proxy is a proxy server which handles the SSL encryption/decryption. This means that our termination proxy (Nginx) will handle and decrypt incoming TLS connections (HTTPS), and it will pass on the unencrypted requests to our internal service (Odoo) so the traffic between Nginx and Odoo will not be encrypted (HTTP).

Using a reverse proxy gives you a lot of benefits such as Load Balancing, SSL Termination, Caching, Compression, Serving Static Content and more.

In this example we will configure SSL Termination, HTTP to HTTPS redirection, WWW to non-WWW redirection, cache the static files and enable GZip compression.

Open your text editor and create the following file:

```sh
sudo touch /etc/nginx/sites-available/example.com
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/site-enabled/example.com
sudo nano /etc/nginx/site-available/example.com
```

```nginx
# Odoo servers
upstream odoo {
    server 127.0.0.1:8069;
}

upstream odoochat {
    server 127.0.0.1:8072;
}

# HTTP -> HTTPS
server {
    listen 80;
    server_name www.example.com example.com;

    include snippets/letsencrypt.conf;
    return 301 https://example.com$request_uri;
}

# WWW -> NON WWW
server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    include snippets/ssl.conf;

    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    # Proxy headers
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    # SSL parameters
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    include snippets/ssl.conf;

    # log files
    access_log /var/log/nginx/odoo.access.log;
    error_log /var/log/nginx/odoo.error.log;

    # Handle longpoll requests
    location /longpolling {
        proxy_pass http://odoochat;
    }

    # Handle / requests
    location / {
       proxy_redirect off;
       proxy_pass http://odoo;
    }

    # Cache static files
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo;
    }

    # Gzip
    gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
    gzip on;
}
```

> Note: Don't forget to replace example.com with your Odoo domain and set the correct path to the SSL certificate files. The snippets used in this configuration are created in this guide.

Once you are done, test and restart the Nginx Service with:

```sh
sudo nginx -t
sudo nginx -s reload 
# sudo serivce nginx restart 
# or sudo systemctl restart nginx
```

Next, we need to tell Odoo that we will use proxy. To do so, open the configuration file `/odoo17/odoo17-server/odoo17-server.conf` and add the following line:

```ini
proxy_mode = True
```

Restart the Odoo service for the changes to take effect:

```sh
sudo systemctl restart odoo17
```

At this point, your server is configured and you can access your Odoo instance at: `https://example.com`

## 9. Change the binding interface

This step is optional, but it is a good security practice.

By default, Odoo server listens to port `8069` on all interfaces. If you want to disable direct access to your Odoo instance you can either block the port `8069` for all public interfaces or force Odoo to listen only on the local interface.

In this guide we will configure Odoo to listen only on `127.0.0.1`. Open the configuration `/odoo17/odoo17-server/odoo17-server.conf` then add the following two lines at the end of the file:

```sh
xmlrpc_interface = 127.0.0.1
netrpc_interface = 127.0.0.1
```

Save the configuration file and restart the Odoo server for the changes to take effect:

```sh
sudo systemctl restart odoo17
```

## 10. Enable Multiprocessing

By default, Odoo is working in multithreading mode. For production deployments, it is recommended to switch to the multiprocessing server as it increases stability, and make better usage of the system resources. In order to enable multiprocessing we need to edit the Odoo configuration and set a non-zero number of worker processes.

The number of workers is calculated based on the number of CPU cores in the system and the available RAM memory.

According to the official Odoo documentation to calculate the workers number and required RAM memory size we will use the following formulas and assumptions:

Worker number calculation

* theoretical maximal number of `worker = (system_cpus * 2) + 1`
* 1 worker can serve `~= 6` concurrent users
* Cron workers also requires CPU

RAM memory size calculation

* We will consider that 20% of all requests are heavy requests, while 80% are lighter ones. Heavy requests are using around 1 GB of RAM while the lighter ones are using around `150 MB` of RAM
* Needed `RAM = number_of_workers * ( (light_worker_ratio * light_worker_ram_estimation) + (heavy_worker_ratio * heavy_worker_ram_estimation) )`

> Note: If you do not know many CPUs you have on your system you can use the following command:

```sh
grep -c ^processor /proc/cpuinfo
```

Let's say we have a system with 4 CPU cores, 8 GB of RAM memory and 30 concurrent Odoo users.

* `30 users / 6 = **5**` (5 is theoretical number of workers needed)
* `(4 * 2) + 1 = **9**` (9 is the theoretical maximum number of workers)

Based on the calculation above we can use 5 workers + 1 worker for the cron worker which is total of 6 workers.
Calculate the RAM memory consumption based on the number of the workers:

* `RAM = 6 * ((0.8 * 150) + (0.2 * 1024)) ~= 2 GB of RAM`

The calculation above show us that our Odoo installation will need around 2 GB of RAM.
To switch to multiprocessing mode, open the configuration file `/odoo17/odoo17-server/odoo17-server.conf` and append the following lines:

```ini
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200
max_cron_threads = 1
workers = 5
```

Restart the Odoo service for the changes to take effect:

```sh
sudo systemctl restart odoo17
```

The rest of the system resources will be used by other services that run on this system. In this guide we installed Odoo along with PostgreSQL and Nginx on a same server and depending on your setup you may also have other services running on your server.

## 11. General Issues

1. If icon disappeared you can do this command:
```sh
sudo -u postgres psql # to connect to postgres
```
```sql
\l
\c YourDB;
DELETE FROM ir_attachment WHERE url LIKE '/web/content/%';
```
for broken css use
```sql
DELETE FROM ir_attachment WHERE datas_fname SIMILAR TO '%.(js|css)';
DELETE FROM ir_attachment WHERE name='web_icon_data';
```
another way to solve this issue:
```sh
./odoo-bin -c odoo17-server.conf -u all # this means upgrade all modules in Odoo
```
connect to shell
```sh
python odoo-bin shell -c odoo17-server.conf -d test_db
```
> `--shell-interface (ipython|ptpython|bpython|python)` Specify a preferred REPL to use in shell mode.

upgrading module from shell, upgrade the the installed modules using orm
```python
>>> xxx = self.env['ir.module.module'].search([]) # to get all modules
>>> yyy = xxx.filtered(lambda a:a.state != 'uninstalled' and not a.author in ['Odoo S.A.', 'Odoo', 'Odoo SA'])
>>> yyy.mapped(lambda a: a.button_immediate_upgrade())
```

uninstall module from shell, remove module using odoo orm
```python
>>> self.env['ir.module.module'].search([('name','=','crm')]).button_immediate_uninstall()
```
2. Long Pooling issues:
When you browse module e.g CRM, is connected with chatter (log messaging), the will appear "Connection Lost" popup.
Just try to upgrade `gevent` using, make sure `gevent` works on different port and check the gevent working in background:
```sh
netstat -plant | grep 8072 # e.g 8072 long pooling port
```
```sh
pip install -U gevent pip && pip install cffi
```
3. URL freezing in odoo:
in System Parameter there is `web.base.url`, it can be changed by calling the browser, to prevent the url from changes add:
```
web.base.url.freeze = True
```
source: [here](https://www.odoo.com/forum/help-1/system-parameters-web-base-url-143575)

4. Exclude your database `MyProductionDB` from database manager list in Odoo configure file:
```
dbfilter = (?!^MyProductionDB$)(^.*$)
```
source: [here](https://stackoverflow.com/a/1395228/1756032)

5. To display Invoice Report as PDF from URL:
```
http://localhost:8069/report/pdf/account.report_invoice/124
```
same as HTML from URL:
```
http://localhost:8069/report/html/account.report_invoice/124
```

## 12. Conclusion

This tutorial walked you through the installation of Odoo 14 on Ubuntu 20.04 in a Python virtual environment using Nginx as a reverse proxy. you also learned how to enable multiprocessing and optimize Odoo for production environment. If you have questions feel free to ask me.