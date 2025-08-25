# README — ขั้นตอนติดตั้งและตั้งค่า **เครือแม่ข่าย** (CentOS 7 + DHCP + Apache + MariaDB + PHP + BIND DNS + FTP)

> เอกสารนี้สรุปขั้นตอนทีละขั้นสำหรับตั้งค่าเครื่องแม่ข่ายบน **CentOS 7** โดยใช้สื่อภายในเครื่อง (Local Repository จาก ISO/USB) และติดตั้งบริการหลัก: DHCP, Web (Apache), Database (MariaDB), PHP, DNS (BIND), และ FTP (vsftpd)

> ⚠️ หมายเหตุเรื่องเวอร์ชัน: CentOS 7 สิ้นสุดระยะซัพพอร์ตแล้ว การใช้งานเพื่อการผลิตควรพิจารณา RHEL/AlmaLinux/RockyLinux เวอร์ชันปัจจุบัน หรือแยกเครือข่ายทดสอบให้ชัดเจน

---

## 0) สิ่งที่ต้องเตรียมก่อนเริ่ม
- ไฟล์ ISO: `CentOS-7-x86_64-DVD-xxxx.iso` (หรือแผ่น/USB ที่มีแพ็กเกจครบ)
- สิทธิ์ผู้ดูแลระบบ (root) บนเครื่อง
- ข้อมูลเครือข่ายที่ต้องใช้ เช่น Network ID, Gateway, DNS, ช่วงแจก IP สำหรับ DHCP
- หากต้องใช้ `ifconfig` ให้ติดตั้ง `net-tools` เพิ่มเติม (CentOS 7 ค่าเริ่มต้นจะมี `ip` มากกว่า)

---

## 1) ติดตั้ง CentOS 7 (กรณีติดตั้งใหม่)
1. บูตเครื่องจากสื่อที่เตรียมไว้ → เลือก **Install CentOS 7**
2. เลือกภาษา **English (United States)**
3. เมนู **Localization** → **Date & Time** : เลือกเขตเวลา **Asia/Bangkok**
4. **Software Selection**: เลือก **Server with GUI**
5. **Installation Destination**: เลือกดิสก์ปลายทางที่จะติดตั้ง
6. กด **Begin Installation**
7. ตั้งค่า **Root Password**
8. ตั้งค่า **User Creation**  
   - ถ้าต้องการสิทธิ์ผู้ดูแลระบบให้ติ๊ก **Make this user administrator**
9. กด **Reboot** เพื่อรีสตาร์ทหลังติดตั้งเสร็จ
10. **License Information** → ติ๊ก **I accept the license agreement**
11. กด **Finish Configuration**

---

## 2) เตรียมสื่อและสร้าง Local Repository (ออฟไลน์)
> เลือกวิธีติดตั้ง repo ภายในเครื่องจาก ISO/USB เพื่อให้ `yum` ใช้งานได้แม้ไม่มีอินเทอร์เน็ต

### 2.1 ตรวจสอบดิสก์และสร้างจุดเมานต์
```bash
df -h
mkdir -p /mnt/iso
```

### 2.2 เมานต์สื่อ (เลือกอย่างใดอย่างหนึ่ง)
- กรณีไฟล์ ISO ในเครื่อง:
```bash
mount -o loop /path/to/CentOS-7-x86_64-DVD-xxxx.iso /mnt/iso
```
- กรณีเป็นอุปกรณ์บล็อก (USB/DVD): แทนที่ `/dev/sdX` ด้วยอุปกรณ์จริง (เช่น `/dev/sdb1` หรือ `/dev/sr0`)
```bash
mount -o ro /dev/sdX /mnt/iso
```

### 2.3 สร้างไฟล์ repo
```bash
cat >/etc/yum.repos.d/local.repo << 'EOF'
[localrepo]
name=LocalRepo
baseurl=file:///mnt/iso
enabled=1
gpgcheck=0
EOF
```

### 2.4 ปิด repo ออนไลน์ (แนะนำติดตั้ง `yum-utils` ก่อน)
```bash
yum install -y yum-utils
yum-config-manager --disable base
yum-config-manager --disable updates
yum-config-manager --disable extras

yum clean all
yum repolist
```

> ถ้าใช้เสร็จและต้องการยกเลิก ให้ `umount /mnt/iso` แล้วลบไฟล์ `/etc/yum.repos.d/local.repo`

---

## 3) ปิดการเชื่อมต่อเครือข่าย **ชั่วคราว** (ถ้าจำเป็น)
> ระวัง: การปิด network จะตัดการเชื่อมต่อระยะไกลทั้งหมด

ตรวจหาอุปกรณ์เครือข่าย:
```bash
ip addr   # หรือ yum install -y net-tools && ifconfig
```

ปิดอุปกรณ์ชั่วคราว (แทน `<device>` ด้วยชื่อจริง เช่น `ens33`/`enp0s3`):
```bash
ip link set <device> down
# หรือ
ifconfig <device> down
```

---

## 4) ติดตั้งและตั้งค่า DHCP Server
### 4.1 ติดตั้งแพ็กเกจ
```bash
yum install -y dhcp
```

### 4.2 แก้ไขไฟล์คอนฟิก `/etc/dhcp/dhcpd.conf`
ตัวอย่าง (แก้ให้ตรงกับเครือข่ายจริง):
```conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;  # ช่วง IP ที่จะแจก
  option routers 192.168.1.1;         # Gateway
  option domain-name-servers 192.168.1.1;  # DNS Server
  default-lease-time 600;
  max-lease-time 7200;
}
```

> (เลือกทำ) ระบุอินเทอร์เฟซที่ให้บริการในไฟล์ `/etc/sysconfig/dhcpd`  
> เพิ่มบรรทัด: `DHCPDARGS="ens33"`

### 4.3 เปิดไฟร์วอลล์และเริ่มบริการ
```bash
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload

systemctl enable dhcpd
systemctl start dhcpd
systemctl status dhcpd --no-pager
```

---

## 5) ติดตั้งและตั้งค่า Web Server (Apache)
### 5.1 ติดตั้งและเปิดบริการ
```bash
yum install -y httpd

systemctl enable httpd
systemctl start httpd
systemctl status httpd --no-pager
```

### 5.2 เปิดไฟร์วอลล์พอร์ตเว็บ
```bash
firewall-cmd --permanent --add-service=http
# ถ้าต้องใช้ HTTPS ด้วย:
# firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

ทดสอบ:
```bash
curl -I http://127.0.0.1/
```

---

## 6) ติดตั้งและตั้งค่า MariaDB
### 6.1 ติดตั้งและเปิดบริการ
```bash
yum install -y mariadb-server mariadb

systemctl enable mariadb
systemctl start mariadb
systemctl status mariadb --no-pager
```

### 6.2 ตั้งค่าความปลอดภัยเบื้องต้น
```bash
mysql_secure_installation
```

### 6.3 ทดสอบเข้าใช้งาน และการนำเข้าไฟล์ `.sql`
```bash
mysql -u root -p
# หรือ นำเข้าข้อมูล (ตัวอย่าง):
mysql -u root -p < /home/user/Downloads/useraccount.sql
```

---

## 7) ติดตั้งและตั้งค่า PHP
```bash
yum install -y php php-mysql
```

แก้ไขไฟล์ `/etc/php.ini` (ใช้เพื่อดีบักในสภาพแวดล้อมทดสอบเท่านั้น):
```ini
display_errors = On
display_startup_errors = On
```

รีสตาร์ท Apache ให้โหลด PHP:
```bash
systemctl restart httpd
```

(เลือกทำ) สร้างไฟล์ทดสอบ PHP:
```bash
echo '<?php phpinfo(); ?>' > /var/www/html/info.php
```

---

## 8) ติดตั้งและตั้งค่า DNS Server (BIND)
### 8.1 ติดตั้งแพ็กเกจ
```bash
yum install -y bind bind-utils
```

### 8.2 แก้ไขไฟล์คอนฟิกหลัก `/etc/named.conf`
ตัวอย่าง (ปรับ IP/Network ตามจริง):
```conf
options {
  listen-on port 53 { 127.0.0.1; 192.168.122.1; };
  allow-query     { localhost; 192.168.122.0/24; };
  recursion yes;
};


// เพิ่ม zone ภายใน (ตัวอย่างโดเมน myweb.com)
zone "myweb.com" IN {
  type master;
  file "myweb.com.zone";   // ไฟล์เก็บ DNS record
  allow-update { none; };
};
```

> หมายเหตุ: เดิมผู้ใช้พิมพ์เป็น `/etc/name.conf` ที่ถูกต้องคือ `/etc/named.conf`

### 8.3 สร้างไฟล์โซนที่ `/var/named/myweb.com.zone`
```zone
$TTL 86400
@       IN      SOA     myweb.com. root.myweb.com. (
                        2024031301 ; Serial (แก้เลขทุกครั้งที่ปรับไฟล์)
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400 )    ; Minimum
@       IN      NS      ns1.myweb.com.
@       IN      A       <SERVER_IP>
ns1     IN      A       <SERVER_IP>
www     IN      A       <SERVER_IP>
```

ตั้งสิทธิ์ไฟล์และ SELinux context (กรณีเปิดใช้งาน):
```bash
chown root:named /var/named/myweb.com.zone
chmod 640 /var/named/myweb.com.zone
restorecon -v /var/named/myweb.com.zone  # ถ้า SELinux เปิด
```

### 8.4 ตรวจสอบคอนฟิกและเปิดบริการ
```bash
named-checkconf
named-checkzone myweb.com /var/named/myweb.com.zone

firewall-cmd --permanent --add-service=dns
# หรือเฉพาะพอร์ต:
# firewall-cmd --permanent --add-port=53/tcp
# firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload

systemctl enable named
systemctl start named
systemctl status named --no-pager
```

### 8.5 ตั้งค่าให้เครื่องนี้เป็น DNS ในระบบ
แก้ไข `/etc/resolv.conf` (บนโฮสต์ที่ต้องการใช้ DNS ภายใน):
```
nameserver <SERVER_IP>
```

ทดสอบ:
```bash
dig @<SERVER_IP> myweb.com A
dig @<SERVER_IP> www.myweb.com A
```

---

## 9) ติดตั้งและตั้งค่า FTP Server (vsftpd)
### 9.1 ติดตั้ง เปิดบริการ และเปิดไฟร์วอลล์
```bash
yum install -y vsftpd

systemctl enable vsftpd
systemctl start vsftpd
systemctl status vsftpd --no-pager

firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload
```

### 9.2 (เลือกทำ) ปรับคอนฟิกให้เขียนไฟล์ได้ในโฮม/เว็บ
แก้ไขไฟล์ `/etc/vsftpd/vsftpd.conf` ให้แน่ใจว่ามีค่า:
```conf
anonymous_enable=NO
local_enable=YES
write_enable=YES
umask=022
# ถ้าจะ chroot ผู้ใช้ไว้ในโฮม (ขึ้นกับดีไซน์ระบบของคุณ):
# chroot_local_user=YES
# allow_writeable_chroot=YES
```

รีสตาร์ทบริการ:
```bash
systemctl restart vsftpd
```

### 9.3 จัดการผู้ใช้/กลุ่มสำหรับอัปโหลดไฟล์เว็บ
```bash
# สร้างผู้ใช้ (ตัวอย่างชื่อผู้ใช้: ftpuser)
useradd ftpuser
passwd ftpuser

# สร้างกลุ่มสำหรับงาน FTP/Web
groupadd ftpgroup

# เพิ่มผู้ใช้ ftp และ Apache (httpd) เข้าไปในกลุ่ม
usermod -aG ftpgroup ftpuser
usermod -aG ftpgroup apache

# เปลี่ยนกลุ่มเจ้าของของโฟลเดอร์เว็บ และกำหนดสิทธิ์
chown -R :ftpgroup /var/www/html
chmod -R 775 /var/www/html

# (เลือกทำ) สร้างลิงก์จากโฮมผู้ใช้ไปยังเว็บรูท
ln -s /var/www/html /home/user99/html
```

ทดสอบการเข้าสู่ระบบ FTP ด้วย `ftp` หรือไคลเอนต์กราฟิก

---

## 10) การเปิด/รีโหลดไฟร์วอลล์สรุป
```bash
# Web
firewall-cmd --permanent --add-service=http
# HTTPS (ถ้าใช้)
# firewall-cmd --permanent --add-service=https

# DHCP
firewall-cmd --permanent --add-service=dhcp

# DNS
firewall-cmd --permanent --add-service=dns

# FTP
firewall-cmd --permanent --add-service=ftp

# ใช้งานจริง
firewall-cmd --reload
```

---

## 11) เช็กลิสต์ทดสอบ End-to-End
1. **DHCP**: เครื่องลูกข่ายได้รับ IP ในช่วงที่กำหนด (`192.168.1.100–192.168.1.200`) และตั้งค่า Gateway/DNS ถูกต้อง
2. **DNS**: ใช้คำสั่ง `dig`/`nslookup` ตรวจสอบว่า `myweb.com` และ `www.myweb.com` ชี้ไปยัง `<SERVER_IP>`
3. **Web**: เปิด `http://<SERVER_IP>/` หรือ `http://myweb.com/`
4. **MariaDB**: ล็อกอิน `mysql -u root -p` และทดสอบ query/นำเข้าข้อมูล
5. **PHP**: เปิด `http://<SERVER_IP>/info.php` เพื่อตรวจสอบว่า PHP ทำงาน
6. **FTP**: ล็อกอินด้วยผู้ใช้ `ftpuser` อัปโหลด/ดาวน์โหลดไฟล์ใน `/var/www/html` ได้ตามสิทธิ์

---

## ภาคผนวก: ไฟล์ตัวอย่างแบบย่อ

**`/etc/dhcp/dhcpd.conf`**
```conf
subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.100 10.0.0.200;
  option routers 10.0.0.1;
  option domain-name-servers 10.0.0.1;
  default-lease-time 600;
  max-lease-time 7200;
}
```

**`/etc/named.conf` (เฉพาะส่วนสำคัญ)**
```conf
options {
  listen-on port 53 { 127.0.0.1; 10.0.0.1; };
  allow-query { localhost; 10.0.0.0/24; };
};

zone "myweb.com" IN {
  type master;
  file "myweb.com.zone";
};
```

**`/var/named/myweb.com.zone`**
```zone
$TTL 86400
@   IN  SOA myweb.com. root.myweb.com. (2024031301 3600 1800 604800 86400)
@   IN  NS  ns1.myweb.com.
@   IN  A   <SERVER_IP>
ns1 IN  A   <SERVER_IP>
www IN  A   <SERVER_IP>
```

---

### เคล็ดลับแก้ปัญหาเบื้องต้น
- `yum repolist` ไม่เจอ repo → ตรวจสอบ `mount` และ `baseurl=file:///mnt/iso`
- `named` อ่านไฟล์โซนไม่ได้ → ตรวจสอบสิทธิ์, SELinux (`restorecon`), และ `named-checkzone`
- `dhcpd` ไม่แจก IP → ตรวจสอบ `firewalld`, อินเทอร์เฟซ (`DHCPDARGS`), และว่ามีเซิร์ฟเวอร์ DHCP อื่นชนกันหรือไม่
- `FTP` เขียนไฟล์ไม่ได้ → ตรวจสอบ `write_enable=YES`, สิทธิ์กลุ่ม/โฟลเดอร์, และ umask

---
