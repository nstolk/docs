# **Gitea setup**

## **Table of contents**
1. Prerequisites
2. Database setup
3. Gitea environment setup
4. Install Gitea
5. Create service to automatically start Gitea
6. Configure Nginx
7. Gitea Web Installation
8. Miscellaneous

## **Prerequisites**

* A (virtual private) server
* Nginx
* Git
* MySQL/MariaDB
* Certbot

## **Database setup**

Gitea will require a database to work. In this example we will create a database named `gitea` and create the user `giteauser` for it.

    CREATE DATABASE gitea;

    CREATE USER 'giteauser'@'localhost' IDENTIFIED BY 'password';

    GRANT ALL ON gitea.* TO 'giteauser'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;

    FLUSH PRIVILEGES;

    EXIT;

## **Gitea environment setup**

On our server we will create a user named `git` which will run Gitea. You can choose to skip this step if you're complacent with running commands as root.

    sudo adduser \
    --system \
    --shell /bin/bash \
    --gecos 'Git Version Control' \
    --group \
    --disabled-password \
    --home /home/git \
    git

Create the required directory strcuture and permissions:

    sudo mkdir -p /var/lib/gitea/{custom,data,indexers,public,log}
    sudo chown git:git /var/lib/gitea/{data,indexers,log}
    sudo chmod 750 /var/lib/gitea/{data,indexers,log}
    sudo mkdir /etc/gitea
    sudo chown root:git /etc/gitea
    sudo chmod 770 /etc/gitea

Since we've created a user with the disabled password flag, we can only authenticate with this user through SSH.
Below we'll take the necessary steps to ensure we can authenticate with SSH.

In /home/git, create .ssh directory:

    mkdir .ssh

Create authorized_keys file in the .ssh directory:

    touch authorized_keys

On your **own** machine, go to your .ssh directory and copy the contents of your public SSH key:

    cat id_rsa.pub

Open your authorized_keys file on the server:

    nano authorized_keys

And paste the contents.

Set correct permissions for the created user:

    sudo chown -R git:git /home/git/.ssh
    sudo chmod 0700 /home/git/.ssh
    sudo chmod 0600 /home/git/.ssh/authorized_keys

## **Install Gitea**

    wget -O gitea https://dl.gitea.io/gitea/1.9.3/gitea-1.9.3-linux-amd64
Or get the newest package from Gitea's website
https://docs.gitea.io/en-us/install-from-binary/

Make it executable:

    chmod +x gitea

Copy the gitea binary to a global location:

    sudo cp gitea /usr/local/bin/gitea

## **Create service to automatically start Gitea**

Create a service file:

    sudo nano /etc/systemd/system/gitea.service

Paste the following contents in the file:

    [Unit]
    Description=Gitea (Git with a cup of tea)
    After=syslog.target
    After=network.target
    #After=mysqld.service
    #After=postgresql.service
    #After=memcached.service
    #After=redis.service

    [Service]
    # Modify these two values and uncomment them if you have
    # repos with lots of files and get an HTTP error 500 because
    # of that
    ###
    #LimitMEMLOCK=infinity
    #LimitNOFILE=65535
    RestartSec=2s
    Type=simple
    User=git
    Group=git
    WorkingDirectory=/var/lib/gitea/
    ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
    Restart=always
    Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
    # If you want to bind Gitea to a port below 1024 uncomment
    # the two values below
    ###
    #CapabilityBoundingSet=CAP_NET_BIND_SERVICE
    #AmbientCapabilities=CAP_NET_BIND_SERVICE

    [Install]
    WantedBy=multi-user.target

Enable and start Gitea

    sudo systemctl daemon-reload
    sudo systemctl enable gitea
    sudo systemctl start gitea

Check Gitea's status

    sudo systemctl status gitea

## **Configure Nginx**

We'll assume you've already setup a basic Nginx config for your domain. In this example we've already configured the domain `nicks.computer` and now we want to add a subdomain where Gitea will run. The fully qualified domain name will be `git.nicks.computer`.

Open the Nginx configuration file

    sudo nano /etc/nginx/sites-available/nicks.computer

Paste the following contents in a new server block

    upstream gitea {
        server 127.0.0.1:3000;
    }

    server {
        listen 80;
        server_name git.nicks.computer;
        root /var/lib/gitea/public;
        access_log off;
        error_log off;

        location / {
                try_files maintain.html $uri $uri/index.html @node;
        }

        location @node {
        client_max_body_size 0;
        proxy_pass http://localhost:3000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
        proxy_redirect off;
        proxy_read_timeout 120;
        }
    }

Navigate to the VPS website of your choice and add the A (and AAAA) records.

![DNS Record](/gitea/img/DNS-record.jpg "DNS Record")

We'll assume that SSL was already previously configured for the domain `nicks.computer`. We'll refresh the certificate and add the new subdomain.

    sudo certbot --nginx -d nicks.computer -d git.nicks.computer

Select Expand (E), then select 1 (no further changes to the webserver configuration)

Certbot should automatically include the necessary links to the certificates in the subdomain server block.


## **Gitea Web Installation**
Navigate to `git.nicks.computer`, you should see this page
![Gitea](/gitea/img/gitea-homepage.png "Gitea Homepage")

Go to `git.nicks.computer/install` and work your way through the installation process.

Use the previously created database name, username and password here:

![Gitea Database Settings](/gitea/img/gitea-db-settings.png "Database Settings")

Here's an overview of what the general settings might look:

![Gitea General Settings](/gitea/img/gitea-general-settings.png "General Settings")

Make sure to take a glance at the optional settings, which I consider the most important settings of your Gitea server.

I would strongly recommend to check the '**Disable Self-Registration**' box here, unless you don't mind strangers being able to create accounts and host their Git projects on your server :-)

It's also recommended to make an administrator account before installing. As an administrator you'll be able to manage accounts, repositories and other settings.

![Gitea Optional Settings](/gitea/img/gitea-optional-settings.png "Optional Settings")

Now let the installation finish and login!

## **Miscellaneous**

Let's create a new repository:

![Gitea New Repository](/gitea/img/gitea-new-repo.png)

After creating a new repository you'll see the familiar layout of GitHub.

Gitea will make a copy of all your repositories in

    /home/git/gitea-repositories

You can of course clone any repository on your Gitea instance to any directory on any machine you have access to.

Gitea also comes with powerful CLI tools. We can create a new user:

    gitea admin create-user --username test --password test --email test@example.com

This will create user `test` with password `test`.

![Gitea User Management](/gitea/img/gitea-admin-account-management.png "User Management")

![Gitea Edit User](/gitea/img/gitea-edit-user.png "Edit User")


