# CentOS 7 Configuration

## Installation

1. Install CentOS 7:
   1. Select "Install CentOS 7".
   2. Select the language "English (United States)".
   3. In Localization, set the Date & Time to "Asia/Bangkok" time zone.
   4. In Software Selection, choose "Server with GUI".
   5. In Installation Destination, select the desired disk for installation.
   6. Click "Begin Installation".
   7. Set the ROOT PASSWORD.
   8. In User Creation, tick "Make this user administrator" if you want the user to have administrator privileges.
   9. Reboot the system.
   10. In License Information, tick "I accept the license agreement".
   11. Click "FINISH CONFIGURATION".

2. Check the list of devices:
   ```
   df -h
   ```
   Create a drive to store files:
   ```
   mkdir -p /mnt/iso
   mount -o loop /dev/sdX /mnt/iso  # Replace /dev/sdX with the actual USB device
   ```

3. Create a local repository:
   1. Use the `nano` command to edit the file:
      ```
      nano /etc/yum.repos.d/local.repo
      ```
   2. Configure the `local.repo` file:
      ```
      [localrepo]
      name=LocalRepo
      baseurl=file:///mnt/iso  # /mnt/iso is the mounted folder
      enabled=1
      gpgcheck=0
      ```
   3. Disable the existing repositories:
      ```
      yum-config-manager --disable base
      yum-config-manager --disable updates
      yum-config-manager --disable extras
      ```

4. Temporarily disable the visual network:
   ```
   ifconfig  # Check the network device
   ip link set <device> down
   # or
   ifconfig <device> down
   ```

5. Install and configure DHCP:
   1. Install DHCP:
      ```
      yum install -y dhcp
      ```
   2. Edit the DHCP configuration file:
      ```
      nano /etc/dhcp/dhcpd.conf
      ```
      Add the following configuration:
      ```
      subnet 192.168.1.0 netmask 255.255.255.0 {
          range 192.168.1.100 192.168.1.200;
          option routers 192.168.1.1;
          option domain-name-servers 192.168.1.1;
          default-lease-time 600;
          max-lease-time 7200;
      }
      ```
   3. Start and enable the DHCP service:
      ```
      systemctl start dhcpd
      systemctl enable dhcpd
      ```

6. Install and configure Apache:
   1. Install Apache:
      ```
      yum install -y httpd
      ```
   2. Start and enable the Apache service:
      ```
      systemctl start httpd
      systemctl enable httpd
      systemctl status httpd
      ```
   3. Add the HTTP and DNS services to the firewall:
      ```
      firewall-cmd --permanent --add-service=http
      firewall-cmd --permanent --add-port=53/tcp
      firewall-cmd --permanent --add-port=53/udp
      firewall-cmd --reload
      ```
   4. Edit the `/etc/resolv.conf` file and add the DNS server IP address.

7. Install and configure MariaDB:
   1. Install MariaDB:
      ```
      yum install -y mariadb-server mariadb
      ```
   2. Start and enable the MariaDB service:
      ```
      systemctl start mariadb
      systemctl enable mariadb
      mysql_secure_installation
      ```
   3. Connect to the MySQL server:
      ```
      mysql -u <user> -p
      ```
   4. Import a SQL file into the database:
      ```
      mysql -u root -p < /path/file.sql
      ```

8. Install and configure PHP:
   1. Install PHP:
      ```
      yum install -y php php-mysql
      ```
   2. Edit the `php.ini` file:
      ```
      display_errors = On
      display_startup_errors = On
      ```

9. Install and configure BIND (DNS server):
   1. Install BIND:
      ```
      yum install -y named
      ```
   2. Edit the `/etc/named.conf` file:
      ```
      options {
          listen-on port 53 { 127.0.0.1; 192.168.122.1; };
          allow-query     { localhost; 192.168.122.0; };
      }

      zone "myweb.com" IN {
          type master;
          file "myweb.com.zone";
          allow-update { none; };
      }
      ```
   3. Create the zone file `/var/named/myweb.com.zone` and configure it.
   4. Verify the configuration and start the BIND service:
      ```
      named-checkconf
      named-checkzone myweb.com /var/named/myweb.com.zone
      systemctl start named
      systemctl enable named
      ```

10. Install and configure FTP (vsftpd):
    1. Create an FTP user group and add users:
       ```
       sudo useradd <user>
       sudo passwd <user>
       sudo groupadd ftpgroup
       sudo usermod -aG ftpgroup <ftpuser>
       sudo usermod -aG ftpgroup apache
       ```
    2. Change the ownership of the `/var/www/html` directory:
       ```
       sudo chown -R :ftpgroup /var/www/html
       ```
    3. Set the permissions for the `/var/www/html` directory:
       ```
       sudo chmod -R 775 /var/www/html
       ```
    4. Create a symbolic link from `/var/www/html` to `/home/user99/html`:
       ```
       sudo ln -s /var/www/html /home/user99/html
       ```

## Usage

This configuration script sets up a CentOS 7 server with the following components:

- CentOS 7 operating system
- Local repository for package installation
- DHCP server for IP address management
- Apache web server
- MariaDB database server
- PHP support
- BIND DNS server
- vsftpd FTP server

After following the installation steps, you can use the configured services to host your web applications, manage your database, and provide DNS and FTP services.

## Contributing

If you find any issues or have suggestions for improvements, please feel free to submit a pull request or open an issue on the project's repository.
