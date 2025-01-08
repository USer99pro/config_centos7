ขั้นตอนการทำ เครือแม่ข่าย
1.ติดตั้ง centos
	1. เลือก Install Centos7
	2. เลือกภาษา English (United States)
	3. Localization
		Date & time : Asia/Bangkok time zone
	4. SOFTWATE SEKECTION
		เลือก Server with GUI	
	5. INSTALLATION DESTINATION
		เลือกพื้นที่ Disk ต้องการติดตั้ง 
	6. กดปุ่ม Begin Installation
	7. ตั้งค่า ROOTPASSWORD
	8. ตั้งค่า  USER CREATION
		หากต้องการให้ผู้ใช้เป็นสิทธิ์ Administrator ให้ทำการติ๊กถูก Make this user administrator 
	9. ทำการีสตาร์ทเครื่อง (Reboot)
	10. LICENSE INFORMATION
		ติ๊กถูก I accept the license agreement 
	11. กดปุ่ม FINISH CONFIGURATION

2. เช็ครายชื่ออุปกรณ์
df -h
สร้าง drive เพื่อเก็บไฟล์
mkdir -p /mnt/iso
mount -o loop /dev/sdX /mnt/iso  # เปลี่ยน /dev/sdX   เป็น device ของ USB จริง

3. สร้าง localrepo
1. ใช้คำสั่ง nano เพื่อในการแก้ไขไฟล์
nano  /etc/yum.repos.d/local.repo
2. ไฟล์ config local.repo
[localrepo]
name=LocalRepo
baseurl=file:///mnt/iso  # ///mnt/iso หมายถึง โฟลเดอร์ที่ถูก mount 
enabled=1
gpgcheck=0 
 
3. ทำการปิด repo เดิม
yum-config-manager –-disable base
yum-config-manager –-disable updates
yum-config-manager –-disable extras

4. ปิด visual network (ชั่วคราว)
	ใช้คำสั่ง Ifconfig #เพื่อเช็คอุปกรณ์ Network 
ใช้คำสั่ง ip link set <device> down
หรือ
ใช้คำสั่ง ifconfig <device> down

5. ติดตั้ง DHCP
yum install -y dhcp  

1. ทำการแก้ไขไฟล์  nano /etc/dhcp/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {		# Network Id และ Net Mask
range 192.168.1.100 192.168.1.200;  		# Range IP 
option routers 192.168.1.1;	   		# Gateway
option domain-name-servers 192.168.1.1;  	# DNS Server
default-lease-time 600;  				# ระยะเวลาเริ่มต้น
max-lease-time 7200; 				# ระยะเวลาสูงสุด
}
2. ตั้งค่า network ใน setting
		       

3. เริ่มการทำงาน DHCP
systemctl start dhcpd //ให้ service ทำงาน
systemctl enable dhcpd //ให้ service ให้ทำงานอัตโนมัติ

 
6. ติดตั้ง Apache 
yum install -y httpd

1.	เริ่มการทำงาน Apache
systemctl start httpd //ให้ service ทำงาน
systemctl enable httpd //ให้ service ทำงานอัตโนมัติ
systemctl status httpd //เช็คสถานะ service
2.	เพิ่ม service เข้าไปในกฎ Firewall  
	firewall-cmd --permanent --add-service=http //เพิ่ม service ให้กับ firewall
	firewall-cmd --permanent --add-port=53/tcp //เพิ่ม service ให้กับ firewall
		firewall-cmd --permanent --add-port=53/udp //เพิ่ม service ให้กับ firewall
	firewall-cmd –-reload //รีสตาร์ท firewall

3.	แก้ไขไฟล์ /etc/resolv.conf
	nameserver [DNS_Server_IP

7. ติดตั้ง MariaDB
	yum install -y mariadb-server mariadb

	1. เริ่มการทำงาน
systemctl start mariadb //ให้ service ทำงาน
systemctl enable mariadb //ให้ service ทำงานอัตโนมัติ
mysql_secure_installation //ทำการ config mysql ใน สิทธิ์ Root

	2. เรียกใช้ mysql 
		mysql -u <user> -p
	ตัวอย่าง	mysql -u root  -p 
	หมายเหตุ -u หมายถึง user ที่ต้องการเข้าไปใช้งาน 
		 -p หมายถึง รหัสผ่านของ

	3. วิธีการดึงไฟล์ sql เข้าถึง ฐานข้อมูล
		mysql -u root -p < /path/file.sql  //การดึงข้อมูลที่อยู่ในไฟล์เข้าถึงฐานข้อมูลและสามารถดึงข้อมูล user account ของ myqsl ในการล็อกอินฐานข้อมูล
	ตัวอย่าง	Mysql -u root  -p  < /home/user/Downloads/useraccount.sql

8. ติดตั้ง PHP
	yum install -y php php-mysql

1.	แก้ไขไฟล์ php.ini
display_errors = On 	# จาก Off ให้เป็น On
display_startup_errors = On 	# จาก Off ให้เป็น On

9. ติดตั้ง bind (DNS server)
	yum install -y named

	1. คอนฟิก /etc/name.conf  

		options {
			listen-on port 53 { 127.0.0.1; 192.168.122.1; }; //เพิ่ม ip server เช่น 192.168.122.1
    			allow-query     { localhost; 192.168.122.0; }; //เพิ่ม Network (ID)  เช่น 192.168.122.0 สามารถใช้ any แทน Network (ID) เพื่อให้ทุกกลุ่มสามารถเข้าถึงได้
		}

#อันนี้คือการเพิ่มโซน
zone "myweb.com" IN {
        type master;
        file "myweb.com.zone"; //ระบุชื่อไฟล์ที่ใช้เก็บข้อมูล DNS Record สำหรับโดเมน
        allow-update { none; };
};

	2. สร้างไฟล์zoneที่ /var/named/<Domain>.com.zone แล้วทำการแก้ไข
$TTL 86400
@       IN      SOA     <Domain>.com. root.<Domain>.com. (
                        2024031301	; Serial
                        3600		; Refresh
                        1800           	; Retry
                        604800          	; Expire
                        86400 	)    	; Minimum TTL
@       	IN      NS      ns1.<Domain>.com.
@      	IN      A        <ip server>
ns1    	IN      A        <ip server>
www	IN      A        <ip server>

	3. ตรวจสอบไฟล์คอนฟิก
		named-checkconf
		named-checkzone <Domain> /var/named/<Domain>.com.zone
4. เริ่มการทำงาน bind
systemctl start named //ให้ service ทำงาน
systemctl enable named //ให้ service ทำงานอัตโนมัติ


10. ติดตั้ง FTP (vsftpd)
	yum install -y vsftpd

	1. สร้างกรุ๊ป FTP 
		Sudo useradd <user> //สร้าง user
		Sudo passwd <user> //สร้างรหัสผ่าน 
		sudo groupadd ftpgroup //สร้างกลุ่ม
		sudo usermod -aG ftpgroup <ftpuser>  //เพิ่ม FTP user  ในกลุ่ม
		sudo usermod -aG ftpgroup apache     //เพิ่ม apache (httpd) ในกลุ่ม

	2. เปลี่ยนเจ้าของกลุ่มที่อยู่ใน /var/www/html 
		sudo chown -R :ftpgroup /var/www/html

	3. ตั้งค่าสิทธิ์ไดเรกทอรี
		sudo chmod -R 775 /var/www/html  # โดยสามารมให้อ่าน/เขียนได้

	4. สร้าง softlink จาก /var/www/html ไปที่ /home/user99/
		sudo ln -s /var/www/html /home/user99/html






