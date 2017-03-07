# Libki Server

## Contributing

If you discover issues, please, add them to the GitHub Libki server [bug tracker](https://github.com/Libki/libki-server/issues).

GitHub is currently the canonical source for the Libki source code.
Please, make [pull requests](https://help.github.com/articles/about-pull-requests/) through GitHub.

## Getting started

### A Docker-based quick setup

* [Install Docker Engine.](https://docs.docker.com/engine/installation/)
* [Install Docker Compose.](https://docs.docker.com/compose/install/)

```bash
git clone https://github.com/Libki/libki-server.git
cd libki-server/
cp libki_local.conf.example libki_local.conf
patch <docker/libki_local.conf.patch
docker-compose up -d
docker-compose exec libki perl /libki/installer/update_db.pl
docker-compose exec libki perl /libki/script/administration/create_user.pl -u libkiadmin -p some_password -s -m 999
```

Visit `localhost:8080/administration` in your browser to log in as `libkiadmin` with the password `some_password`!

#### To start Libki later

If Libki stops for any reason, just re-launch the Docker containers.

```bash
cd libki-server/
docker-compose up -d
```

After the initial install, this command will only take a couple seconds.

## The more complete Docker-based setup

For this setup, the Libki source code will be copied into a global folder.

```
sudo mkdir -p /var/libki/app
sudo chown $USER:$USER /var/libki/app
git clone https://github.com/Libki/libki-server.git /var/libki/app
```

Before building the containers, update the configuration files.

In `libki_local.conf`, point to the Docker-ized MySQL database.

```
<connect_info>
  dsn  dbi:mysql:database=libki;host=libki_mysql
  ...
```

Make sure the MySQL credentials in `docker-compose.yml` match those in `libki_local.conf`.

Then launch the container and go get some coffee while it builds.

```
docker-compose up -d
```

If you're importing an existing database, do that now.

```
docker cp ./dump-of-libki.sql libki_mysql:/tmp/dbdump.sql
docker-compose exec mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -h localhost libki < /tmp/dbdump.sql'
```

Update the database, whether you imported or not.

```
docker-compose exec libki perl /libki/installer/update_db.pl
```

If you didn't import a database, create an admin user.

```
docker-compose exec libki perl /libki/script/administration/create_user.pl -u libkiadmin -p some_password -s -m 999
```

Now we are going to fine-tune our setup.

First setup Libki as a service, so we can use, `service libki [status|start|stop|restart]` to interact with Libki.
This init script expects the Libki project to be at `/var/libki/app`.

```
sudo cp docker/libki /etc/init.d/
sudo chmod 755 /etc/init.d/libki
sudo chown root:root /etc/init.d/libki
```

Next, have the system check every minute to make sure Libki is running, and start it if needed.
This cron script also expects the Libki project to be at `/var/libki/app`.

```
sudo cp docker/ensure_libki_running /etc/cron.d/
sudo chown root:root /etc/cron.d/ensure_libki_running
```

Finally, set an automatic backup to run each day at noon.
The backup files will be stored in `/var/libki`.

```
sudo cp docker/backup_libki /etc/cron.d/
sudo chown root:root /etc/cron.d/backup_libki
```

## Manual Installation

### Create a user

For this basic, single instance installation, we will create a user named 'libki'.

```bash
sudo adduser libki
```

### Install needed packages

```bash
sudo apt-get install curl perl git make build-essential unzip
```

### Download and install Libki and needed Perl modules

We will use local::lib for our installation, this means that all the Perl modules we need will be installed for the 'libki' user and will not affect any Debian installed Perl modules!

* Log in as that user

```bash
su libki
```

* Install cpanminus and local::lib

```bash
curl -L http://cpanmin.us | perl - -l ~/perl5 App::cpanminus local::lib
eval `perl -I ~/perl5/lib/perl5 -Mlocal::lib=~/perl5`
echo 'eval $(perl -I$HOME/perl5/lib/perl5 -Mlocal::lib)' >> ~/.bashrc
```

* Clone the Libki server git repository

```bash
git clone https://github.com/Libki/libki-server.git
```

* Enter the libki-server directory

```bash
cd libki-server
```

* Install Libki's Perl dependencies from CPAN

```bash
cpanm -n --installdeps .
```

### Install and configure MySQL for Libki

* Switch back to your admin account ( i.e. the one you created the libki user with ) to set up your database.
* Install the mysql-server Debian package

```bash
sudo apt-get install mysql-server
```

* Create the Libki database

```sql
mysql -uroot -p
CREATE DATABASE libki;
CREATE USER 'libki'@'localhost' IDENTIFIED BY 'YOURPASSWORDHERE';
GRANT ALL PRIVILEGES ON libki.* TO 'libki'@'localhost';
FLUSH PRIVILEGES;
exit
```

* Now switch back to the 'libki' user again for the following steps.
* Copy of the example config file and edit it

```bash
cp libki_local.conf.example libki_local.conf
nano libki_local.conf
```

Change the database name, user and password to what you previously have specified. Enable or disable SIP by changing the parity bit.

* Run the database installer/updater

```bash
./installer/update_db.pl
```

NOTE: If this fails with an error including "Perhaps the DBD::mysql perl module hasn't been fully installed,"
then install the Perl Mysql package using:

```bash
sudo apt-get install libdbd-mysql-perl
```


This fills the Libki database with the necessary tables and other information.

* Create a superadmin user to log in to Libki as

```bash
./script/administration/create_user.pl -u *LIBKIADMINNAME* -p *LIBKIADMINPASSWORD* -s -m 999
```

This creates the Libki administrator. It also sets a password and minutes the user has. The login minutes must be given, but are irrelevant for an admin.

### Set up your init script

* Copy the init script template to init.d

```bash
sudo cp /home/libki/libki-server/init-script-template /etc/init.d/libki
sudo chmod +x /etc/init.d/libki
sudo update-rc.d libki defaults
```

The first line copies the file, the second makes it executable, and the final line tells your server to start at boot.
If you’ve followed the instructions closely so far, the init script should work out of the box for you.
By default, Libki is run via Starman as the backend, but you can switch to Gazelle by commenting the line enabling Starman and uncommenting the line enabling Gazelle.
With a little modification it would be possible to use other backends as well, such as Starlet.
Starman is a very mature PSGI backend, whereas Gazelle is newer but appears to be higher performance.

### Set up your reverse proxy

* Install Apache

```bash
sudo apt-get install apache2
```

* Remove the default config

```bash
sudo rm /etc/apache2/sites-enabled/000-default.conf
```

* Create the file /etc/apache2/sites-available/libki.conf

```bash
sudo nano /etc/apache2/sites-available/libki.conf
```

Copy and paste the following into it:

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
<VirtualHost *:80>
    ServerName server.libki.org
    DocumentRoot /home/libki/libki-server
    <Proxy *>
        Order deny,allow
        Allow from allow
    </Proxy>
    ProxyPass        / http://localhost:3000/ retry=0
    ProxyPassReverse / http://localhost:3000/ retry=0
</VirtualHost>
```

Change server.libki.org to your actual domain

* Save and close the file
* Enable your new Libki config file

```bash
sudo a2ensite libki
```

* Enable mod_proxy and mod_proxy_http

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
```

* Restart apache

```bash
sudo service apache2 restart
```

### Set up the Libki cron jobs

```bash
sudo su - libki
crontab -e
```

Add in the following lines:

```
* * * * * perl ~/libki-server/script/cronjobs/libki.pl
0 0 * * * perl ~/script/cronjobs/libki_nightly.pl
```
On _Ubuntu 16.04_ I had to provide full paths to everything because _cron_ has a restricted environment:

```
* * * * * /usr/bin/perl -I /home/libki/perl5/lib/perl5 /home/libki/libki-server/script/cronjobs/libki.pl > /home/libki/libki-crontab-minute.log 2>&1
0 0 * * * /usr/bin/perl -I /home/libki/perl5/lib/perl5 /home/libki/libki-server/script/cronjobs/libki_nightly.pl > /home/libki/libki-crontab-day.log 2>&1
```

**NOTE** The "_> /home/libki/libki-crontab-minute.log 2>&1_" is to redirect all output to a file in the libki home directory, so that I could see why it wasn't working.
I figured it made sense to leave that in there for future troubleshooting.
You can remove it if you don't want that.

The script libki.pl is run every minute and decrements a logged in user's time (among other tasks).
The second script, libki_nightly.pl, is run once per night to reset each user's allotted time for the day (as well as other cleanup tasks).

**Note** The Libki log file needs to be writable by both root and the libki user.
If the libki user cannot write to it, the cron scripts will fail to run correctly and the client timers will never count down!

```bash
touch ~/libki_server.log
sudo chown libki:root ~/libki_server.log
sudo chmod 664 ~/libki_server.log
```

This creates an empty log file, sets the owner and group, then sets the correct permissions on the file.

* Set up automatic restarter

If you wish to have the Libki server restart itself in the case it dies for some reason, we can add an automatic restarter to the root user’s crontab.

```bash
sudo su -
crontab -e
```

Add the following lines

```
* * * * * /etc/init.d/libki start
```

### Start Libki

```bash
sudo /etc/init.d/libki start
```

You can check to see if the daemon is running via

```bash
ps aux | grep libki
```

You should see a line similar to the following:

```bash
root     28326  0.0  0.3  10932  7132 ?        S    09:56   0:00 /usr/bin/perl /home/libki/perl5/bin/start_server --daemonize --port 3000 --pid-file /home/libki/libki.pid --status-file /home/libki/libki.status --log-file /home/libki/libki_server.log -- /home/libki/perl5/bin/plackup -I /home/libki/Libki/lib -I /home/libki/perl5/lib/perl5/ -s Starman --workers 2 --max-requests 50000 -E production -a /home/libki/Libki/libki.psgi
```
You should now be able to go to the IP (or hostname if your network's DNS is configured correctly) of your server.
For a browser on the server machine you could use: http://172.0.0.1
To access the administrator area it would be: http://127.0.0.1/administration

### Troubleshooting

You can now test to see if your server is running by using the cli web browser 'links'.
If you don't have links installed you can installed it via the command

```bash
sudo apt-get install links
```

Now, open the Libki public page via:

```bash
links 127.0.0.1:80
```

If this loads the Libki login page, congrats!
If you get an error, you can try bypassing the proxy and access the server directly on port 3000.

```bash
links 127.0.0.1:3000
```

If this works, then you'll want to check your Apache error logs for the failure reason.
If it does not work, you'll want to check the Libki server error log instead.
It can be found at /home/libki/libki_server.log if you've followed this tutorial closely.

### Documentation Authors

* Kyle M Hall kyle@kylehall.info
* Christopher Vella chris@calyx.net.au
* Luke Fritz ldfritz@gmail.com
