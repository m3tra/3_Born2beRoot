# Guide

## OS Install

### Install

- Language: English
- Country: Other->Europe->Portugal
- Local: en_US.UTF-8
- Keymap: American English

### Network

- Hostname: fporto42

### Users

- root
  - password: Quarentae2
- user
  - name: fporto
  - password: Quarentae2

### Clock

- Time Zone: Lisbon

### Partitions

- Manual
  - Create these partitions:
    | Size  | Type    | Location  | Filesystem | Mount Point | Label | Bootable |
    | ----- | ------- | --------- | ---------- | ----------- | ----- | -------- |
    | 500MB | Primary | Beginning | ext4       | /boot       | boot  | yes      |
    | Max   | Logical |           | Do not use |             |       |          |
  - Cofigure encrypted volumes by creating one:
    - Device to encrypt: sda5 (partition previously created with max size)
      - Default setting
      - Data erased
      - Passphrase: Quarentae2
  - Configure LVM:
    - Create LVM group:
      - Name: LVMGroup
      - Device: sda5_crypt (encrypted partition)
    - According to this list of LVM partitions:
      | Name    | Size       | Filesystem | Mount Point | Label   |
      | ------- | ---------- | ---------- | ----------- | ------- |
      | root    | 10G        | ext4       | /           | root    |
      | home    | 8G         | ext4       | /home       | home    |
      | var     | 3G         | ext4       | /var        | var     |
      | srv     | 3G         | ext4       | /srv        | srv     |
      | tmp     | 3G         | ext4       | /tmp        | tmp     |
      | var-log | 4G         | ext4       | /var/log    | var-log |
      | swap    | max (2.8G) | swap       |             |         |
      - Create logical volumes in group LVMGroup with each Name and Size
      - Edit the partitions' filesystems, mount points and labels

### Package manager

- Scan CD: No
- Mirror Country: PT
- Mirror: deb.debian.org
- Proxy: None
- Survey: No
- Software: None

### Bootloader

- grub: install
- Device to boot from: /dev/sda (manual select)

## Configuration

Login as root

### Misc

Run the following command:

```bash
apt install man -y # Install manuals
```

### User

```bash
groupadd user42             # Create new group
usermod -a -G user42 fporto # Make fporto part of the "user42" group
usermod -a -G sudo fporto   # Make fporto part of the "sudo" group
```

### Sudo

Run the following commands:

```bash
apt install sudo -y
mkdir /var/log/sudo
visudo # Edit sudo config
```

Change:

`Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"`

to

`Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"`

Add the following lines:

```conf
Defaults	passwd_tries=3
Defaults	badpass_message="You entered the wrong pass! Try again."
Defaults	logfile="/var/log/sudo/sudo.log"
Defaults	log_input,log_output
Defaults	iolog_dir="/var/log/sudo/io"
Defaults	requiretty
```

### SSH

Run the following command:

```bash
apt install openssh-server -y
```

Add the following lines to `/etc/ssh/sshd_config`

```conf
Port 4242
PermitRootLogin no
X11Forwarding no
```

### UFW

Run the following commands:

```bash
apt install ufw -y
ufw allow 4242/tcp
ufw enable
```

### Password

Set the following values in `/etc/login.defs`:

```conf
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7
```

Run the following commands:

```bash
chage -m 2 fporto # The minimum number of days allowed before the modification of a password will be set to 2
chage -m 2 root
chage -M 30 fporto # The password has to expire every 30 days
chage -M 30 root
chage -W 7 fporto # The user has to receive a warning message 7 days before their password expires
chage -W 7 root
apt install libpam-pwquality -y
```

In `/etc/pam.d/common-password`:

Change:

`password	requisite	pam_pwquality.so retry=3`

to

`password	requisite	pam_pwquality.so retry=3 ucredit=-1 dcredit=-1 difok=7 maxrepeat=3 minlen=10 remember=1 usercheck=1 enforcing=1 enforcing_for_root=1`

### Monitoring Script

Run the following command:

```bash
apt install bc -y
```

Create the following script and save it at `/root/monitoring.sh`:

```bash
TEMP="/tmp/.monitoring"
echo -e "#Architecture: $(uname -a)" > $TEMP
echo "#CPU physical: $(lscpu | grep "^Socket(s):" | awk '{print $2}')"
echo "#vCPU: $(grep processor /proc/cpuinfo | wc -l)"
memAvailable="$(grep MemAvailable /proc/meminfo | awk '{print int(($2 + 512) / 1000)}')"
memTotal="$(grep MemTotal /proc/meminfo | awk '{print int(($2 + 512) / 1000)}')MB"
echo "#Memory(Available-Used): $memAvailable/$memTotal ($(ps -A -o pmem | tail -n+2 | paste -sd+ | bc)%)"
echo "#Disk Usage: $(df --total -BM | grep total | awk '{printf "%.1f/%.1fGB (%.1f%%)", ($3 / 1000000), ($4 / 1000000), $5}')"
echo "#CPU load: $(ps -A -o pmem | tail -n+2 | paste -sd+ | bc)"
echo "#Last boot: $(who -b | awk '{print $3" "$4}')"
echo "#LVM use: $(grep "/dev/mapper" /etc/fstab | awk 'END {if ($1) print "yes"; else print "no";}')"
echo "#Connections (TCP):  $(ss -s | grep "TCP:" | awk -F ' ' '{print $4}' | cut -d ',' -f1) established"
echo "#User count:  $(w | grep "user" |awk -F ',  ' '{print $2}' | awk '{print $1}')"
echo "#Network: IPv4 $(hostname -I)| MAC $(cat /sys/class/net/enp0s3/address)"
echo "#Sudo count: $(cat /var/log/sudo/sudo.log | grep "COMMAND" | wc -l)"
```

Run the following commands:

```bash
chmod +x /root/monitoring.sh # Allow execution of script
crontab -e                   # Edit cron jobs
```

Add the following line:

`*/10 * * * * /root/monitoring.sh`

## Bonus

### WordPress Website

Run the following commands:

```bash
apt install mariadb-server lighttpd php7.3-cgi php7.3-mysql -y
ufw allow 80/tcp
```

#### MariaDB

Run the following command to log into the database:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE wpdb;
CREATE USER 'fporto'@'localhost' IDENTIFIED BY 'Quarentae2';
GRANT ALL ON wpdb.* TO 'fporto'@'localhost' IDENTIFIED BY 'Quarentae2' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

#### Configs

In `/etc/php/7.3/cgi/php.ini`

- Uncomment: `cgi.fix_pathinfo=1`

- Change:
`;date.timezone =`
to
`date.timezone = Europe/Lisbon`

In `/etc/lighttpd/lighttpd.conf`:

- Change:
`server.document-root = "/var/www/html"`
to
`server.document-root = "/var/www/html/wordpress"`

- Add to the list of modules at the top: `mod_rewrite,`

In `/etc/lighttpd/conf-available/15-fastcgi-php.conf`:

- Change:
`"bin-path" => "/usr/bin/php-cgi"`
to
`"bin-path" => "/usr/bin/php-cgi7.3"`

Run the following commands:

```bash
lighttpd-enable-mod fastcgi
lighttpd-enable-mod fastcgi-php
/etc/init.d/lighttpd force-reload
```

#### WordPress

Run the following commands:

```bash
apt install wget
wget -o /tmp/wordpress.tar.gz https://wordpress.org/latest.tar.gz
cd /var/www/html
tar -zxvf /tmp/wordpress.tar.gz
chown -R www-data:www-data wordpress/
chmod -R 755 wordpress/
cd wordpress
cp wp-config-sample.php wp-config.php
```

Change the following lines in `/var/www/html/wordpress/wp-config.php`:

- `define( 'DB_NAME', 'database_name_here' );` to `define( 'DB_NAME', 'wpdb');`
- `define( 'DB_USER', 'username_here' );` to `define( 'DB_USER', 'fporto' );`
- `define( 'DB_USER', 'password_here' );` to `define( 'DB_PASSWORD', 'Quarentae2' );`

## vsftpd

Run the following commands:

```bash
apt install vsftpd -y
ufw allow 20/tcp
ufw allow 21/tcp
```

In `/etc/vsftpd.conf`:

- Change the following lines:
  - `listen=NO` to `listen=YES`
  - `listen_ipv6=YES` to `listen_ipv6=NO`
- Uncomment:
  - `write_enable=YES`
  - `chroot_local_user=YES`
- Add:

  ```conf
  allow_writeable_chroot=YES
  userlist_enable=YES
  userlist_file=/etc/vsftpd.userlist
  userlist_deny=NO
  ```

Run the following commands:

```bash
echo fporto >> /etc/vsftpd.userlist
systemctl restart vsftpd
```

## VM/IP Special

Everytime the server gets assigned a new IP address (for example when you reboot the VM or switch computers at school) you need to run this query

```bash
mariadb
```

```sql
\u wpdb
UPDATE wp_options SET option_value = "http://<current_ip>" WHERE option_id < 3;
COMMIT;
```

## Evaluation

- Simple setup
  - systemctl status

- User
  - (Ask) uname -a
  - groups
  - sudo adduser fporto
  - (Ask) cat /etc/login.defs
  - (Ask) cat /etc/pam.d/common-password
  - sudo groupadd evaluating
  - sudo usermod -a -G evaluating fporto
  - groups fporto

- Hostname and partitions
  - hostname
  - sudo hostnamectl set-hostname fporto42
  - shutdown -r now
  - hostname
  - (Ask) lsblk
  - (Ask) What is LVM?

- Sudo
  - apt list --installed | grep sudo
  - (Ask) sudo usermod -a -G sudo fporto
  - (Ask) sudo visudo
  - sudo cat /var/log/sudo/sudo.log

- UFW
  - apt list --installed | grep ufw
  - sudo systemctl status ufw
  - (Ask) What is UFW?
  - sudo ufw status
  - sudo ufw allow 8080
  - sudo ufw status
  - (Ask) sudo ufw delete allow 8080

- SSH
  - apt list --installed | grep ssh
  - sudo systemctl status 4242
  - (Ask) What is SSH?
  - nano /etc/ssh/sshd_config
    - Port 4242
    - PermitRootLogin no
    - X11Forwarding no

- Monitoring script
  - nano .../monitoring.sh
  - (Ask) What is cron?
  - crontab -e
    - (disable script)
  - shutdown -r now
