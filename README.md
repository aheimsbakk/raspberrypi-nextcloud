# Installing Nextcloud on Raspberry PI 3

**Using Luks encrypted external disk**

## Prepare external disk

What you need.

* USB disk available to keep your data on.

* USB thumb drive to keep encryption key on.

### Format thumb drive

Format the thumb drive and name drive KEY.

```bash
sudo mkfs.vfat -F32 -n KEY /dev/sdb1
```

Temporary mount disk and create a random key on the drive.

```bash
sudo mount /dev/sdb1 /mnt
sudo dd if=/dev/random of=/mnt/random.key bs=1k count=1
```

### Encrypt the disk

It's just good sense to encrypt your external disk. In case you want to throw it away in the future, you don't need to worry that much if someone get access to it. We're standard Linux using [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup "Linux Unified Key Setup") encryption.

Encrypt and format the disk. Use a strong random password. We'll change that to auto decrypt with an USB drive.

```bash
sudo luksformat -t ext4 /dev/sda
```

Add the USB thumb drive key to the encrypted drive and unmount thumb drive.

```bash
sudo cryptsetup luksAddKey /dev/sda /mnt/random.key
sudo umount /mnt
```

Unencrypt and mount the disk.

```bash
sudo cryptsetup luksOpen /dev/sda cnextcloud
sudo mkdir /media/nextcloud
sudo mount /dev/mapper/cnextcloud /media/nextcloud
```

Prepare for automatic decryption of USB disk on boot.

- Find UUID of USB disk.

    ```bash
    sudo blkid | grep LUKS
    ```

- Rasberian doesn't decrypt disk on boot. Workaround, we do it in `/etc/rc.local`. Edit `/etc/rc.local` and make `/dev/disk/by-uuid` to reflect your UUID.

    ```bash
    vi /etc/rc.local
    ```

    ```bash
    #!/bin/bash -e

    _IP=$(hostname -I) || true
    if [ "$_IP" ]; then
      printf "My IP address is %s\n" "$_IP"
    fi

    cryptsetup --key-file <(/lib/cryptsetup/scripts/passdev /dev/disk/by-label/KEY:/random.key) open /dev/disk/by-uuid/c2acf55a-a40e-4132-b634-a5fe3886d554 cnextcloud
    mount -o noatime,nodiratime /dev/mapper/cnextcloud /media/nextcloud
    exit 0
    ```

## Set up Nextcloud

### Install database, web server and key value server

```bash
sudo apt-get install \
    vim apache2 libnss-mdns mariadb-server redis-server \
    libapache2-mod-php php-zip php-dom php-xmlwriter php-mbstring php-gd \
    php-simplexml php-curl php-mysql php-pgsql php-sqlite3 php-redis php-apcu
```

Stop services from autostart, we're going to configure them to start in `rc.local`.

```bash
systemctl stop apache2 mysql
systemctl disable apache2 mysql
```

Move database, and set permissions on `/media/nextcloud` folder.

```bash
sudo mv /var/lib/mysql /media/nextcloud/; sudo ln -s /media/nextcloud /var/lib/mysql
sudo mkdir /media/nextcloud/html; sudo chown www-data.www-data /media/nextcloud/html
```

Start database and web-server again.

```bash
systemctl start mysql
```

### Configure MySQL

Harden MariaDB, set `root` user password, and log into the database.

```bash
mysql_secure_installation
sudo mysql
```

#### Create the Nextcloud database. 

Change `password` to your super-secret-password.

```mysql
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS nextcloud;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'password';
```

### Configure PHP and webserver

Add recommendations from Nextcloud, see [Database Configuration](https://docs.nextcloud.com/server/12/admin_manual/configuration_database/linux_database_configuration.html).

```bash
cat <<EOF | tee /etc/php/7.0/apache2/conf.d/99-mysql.ini
[mysql]
mysql.allow_local_infile=On
mysql.allow_persistent=On
mysql.cache_size=1000
mysql.max_persistent=-1
mysql.max_links=-1
mysql.default_port=
mysql.default_socket=/var/lib/mysql/mysql.sock  # Debian squeeze: /var/run/mysqld/mysqld.sock
mysql.default_host=
mysql.default_user=
mysql.default_password=
mysql.connect_timeout=60
mysql.trace_mode=Off
EOF

cat <<EOF | sudo tee /etc/php/7.0/apache2/conf.d/99-opcache.ini
opcache.enable=1
opcache.enable_cli=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
EOF

cat <<EOF | sudo tee /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
	ServerAdmin webmaster@raspberrypi.local
	DocumentRoot /media/nextcloud/html/

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined


	<Directory /media/nextcloud/html/>
	  Options +FollowSymlinks
	  AllowOverride All
	  Require all granted

	 <IfModule mod_dav.c>
	  Dav off
	 </IfModule>

	 SetEnv HOME /media/nextcloud/html
	 SetEnv HTTP_HOME /media/nextcloud/html
	</Directory>
</VirtualHost>
EOF

sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo a2enmod setenvif

sudo apache2ctl restart
```


### Get a valid SSL certificate and enable HTTPS

Install software, get certificate and enable HTTPS on the webserver.

```bash
sudo apt-get install certbot
sudo certbot --authenticator webroot --installer apache -w /media/nextcloud/html/ -d example.com
```

### Run the installer Nextcloud installer

Download installer from Nextcloud site.

```bash
wget -O /media/nextcloud/html/setup-nextcloud.php https://download.nextcloud.com/server/installer/setup-nextcloud.php
```

Open your favorite web-browser and run go to [https://raspberrypi.local/setup-nextcloud.php](https://raspberrypi.local/setup-nextcloud.php).

```bash
grep -q Strict-Transport-Security /etc/apache2/sites-enabled/000-default-le-ssl.conf || sudo sed -i '/VirtualHost/a Header always set Strict-Transport-Security "max-age=15552000; includeSubdomains;"' /etc/apache2/sites-enabled/000-default-le-ssl.conf
```

### Add Redis cache to Nextcloud

```bash
cat <<EOF | tee /tmp/redis.txt
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
    'host' => 'localhost',
    'port' => 6379,
  ),
EOF

sudo grep -q Redis /media/nextcloud/html/config/config.php || sudo sed -i '/CONFIG/r /tmp/redis.txt' /media/nextcloud/html/config/config.php
```

### Set up backup of the MariaDB database and Nextcloud cron-job

```bash
sudo mkdir /media/nextcloud/backup
sudo crontab -e
```

```crontab
0 0 * * * mysqldump nextcloud --add-drop-table | gzip > /media/nextcloud/backup/mariadb.sql.gz; savelog -c 30 -l /media/nextcloud/backup/mariadb.sql.gz
*/15 * * * * su -s /bin/bash -c 'php -f /media/nextcloud/html/cron.php' www-data
```
