# How to install Oracle 11g R2 Express

## ความต้องการระบบ
- CPU: 1 CPU
- RAM: 4 GB

## สร้างไฟล์ Swap ขนาด 8Gb:
```sh
sudo swapoff -a
sudo dd if=/dev/zero of=/swapfile bs=1G count=8

sudo chmod 600 /swapfile

sudo mkswap /swapfile

sudo swapon /swapfile
```

 ## ตรวจสอบข้อมูล Swapfile
 ```sh
 grep SwapTotal /proc/meminfo
 ```
 
 ## สร้างกลุ่มผู้ใช้งานสำหรับ Oracle
 
 ### Create the Oracle Inventory group:
```sh
sudo -i

groupadd oinstall

groupadd dba

mkdir /home/oracle/

mkdir -p /u01/app/oracle

useradd -g oinstall -G dba -d /home/oracle -s /bin/bash oracle
```

### ตั้ง Password ของ Oracle User
```sh
passwd oracle
```

### Set สิทธิ์
```sh
chown -R oracle:oinstall /home/oracle

chown -R oracle:oinstall /u01/app/oracle
```

### สร้าง Folder
```sh
mkdir -p /u01/app/oraInventory
```

### ทำการ Set สิทธิ์
```sh
chown -R oracle:oinstall /u01/app/oraInventory
```

### สร้างไฟล์ `/etc/sysctl.conf`
```sh
# ============================

# Oracle 11g

# ============================

# semaphores: semmsl, semmns, semopm, semmni

kernel.sem = 250 32000 100 128

kernel.shmall = 2097152

kernel.shmmni = 4096

# Replace kernel.shmmax with the half of your memory size in bytes

# if lower than 4 GB minus 1

# 2147483648 is 2 GigaBytes (4 GB of RAM / 2)

kernel.shmmax=2147483648

#

# Max number of network connections. Use sysctl -a | grep ip_local_port_range to check.

net.ipv4.ip_local_port_range = 9000  65500

#

net.core.rmem_default = 262144

net.core.rmem_max = 4194304

net.core.wmem_default = 262144

net.core.wmem_max = 1048576

#

# The maximum allowed value, set to avoid overhead and input/output errors

fs.aio-max-nr = 1048576

# 512 * Processes

fs.file-max = 6815744

fs.suid_dumpable = 1

#

# To allow dba to allocate hugetlbfs pages

# 1001 is your oinstall group, you can check this id with the grep oinstall /etc/group command

vm.hugetlb_shm_group = 1001
```

### ทำการ Apply `/etc/sysctl.conf`
```sh
sysctl -p
```

### ทำการแก้ไข `/etc/security/limits.conf`.
```sh
# Oracle

oracle           soft    nproc   2047

oracle           hard    nproc   16384

oracle           soft    nofile  1024

oracle           hard    nofile  65536

oracle           soft    stack   10240
```

### ทำการเช็ค `/etc/pam.d/login`

```sh
cat /etc/pam.d/login | grep pam_limits.so

#Output
$ session    required   pam_limits.so
```
ถ้าไม่มีผลตาม `Output` ให้ทำการเติมคำสั่งต่อไปนี้ไป
```sh
session    required   pam_limits.so
```

```sh
cat /etc/pam.d/login | grep pam_limits.so
```

### แก้ไขไฟล์ /etc/profile
```sh
if [ $USER = "oracle" ]; then

        if [ $SHELL = "/bin/ksh" ]; then

              ulimit -p 16384

              ulimit -n 65536

        else

              ulimit -u 16384 -n 65536

        fi

fi
```

### ติดตั้ง Package ต่อไปนี้
```sh
sudo apt-get update && sudo apt-get install -y libaio1 libaio-dev expat sysstat \
libelf-dev elfutils libstdc++5 ksh alien \
gcc gawk binutils gawk x11-utils rpm alien openssh-server \
lib32z1 libxm4 libuil4 libmrm4 libmotif-common lib32ncurses5 lib32tinfo5
```

### ทำการดาวน์โหลดและติดตั้ง
- [Download](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html) ไฟล์ติดตั้ง Oracle Database Express Edition.
- unzip ไฟล์
```sh
unzip oracle-xe-11.2.0-1.0.x86_64.rpm.zip
```

### ติดตั้ง
```sh
cd Disk1
sudo alien --scripts -d oracle-xe-11.2.0-1.0.x86_64.rpm
```

### ตั้งค่าไฟล์ `chkconfig`
```sh
sudo pico /sbin/chkconfig
```

```sh
#!/bin/bash
# Oracle 11gR2 XE installer chkconfig hack for Ubuntu
file=/etc/init.d/oracle-xe
if [[ ! `tail -n1 $file | grep INIT` ]]; then
  echo >> $file
  echo '### BEGIN INIT INFO' >> $file
  echo '# Provides: OracleXE' >> $file
  echo '# Required-Start: $remote_fs $syslog' >> $file
  echo '# Required-Stop: $remote_fs $syslog' >> $file
  echo '# Default-Start: 2 3 4 5' >> $file
  echo '# Default-Stop: 0 1 6' >> $file
  echo '# Short-Description: Oracle 11g Express Edition' >> $file
  echo '### END INIT INFO' >> $file
fi update-rc.d oracle-xe defaults 80 01
```

### ตั้งค่า Permission ให้กับไฟล์ `/sbin/chkconfig`
```sh
sudo chmod 755 /sbin/chkconfig
```

### ทำการ Set Parameter kernel
```sh
sudo pico /etc/sysctl.d/60-oracle.conf
```

### คัดลอกคำสั่งต่อไปนี้ลงในไฟล์
```sh
# File: /etc/sysctl.d/60-oracle.conf
# Oracle 11g XE kernel parameters 
fs.file-max=6815744  
net.ipv4.ip_local_port_range=9000 65000  
kernel.sem=250 32000 100 128 
kernel.shmmax=536870912 
```

### ทำการ Verify ไฟล์
```sh
sudo cat /etc/sysctl.d/60-oracle.conf 

sudo service procps start

sudo sysctl -q fs.file-max
```

### ทำการ Mount /dev/shm สำหรับ Oracle
```sh
sudo pico /etc/rc2.d/S01shm_load
```

### คัดลอกคำสั่งต่อไปนี้ลงในไฟล์ `/etc/rc2.d/S01shm_load`
```sh
#!/bin/sh
case "$1" in
start)
    mkdir /var/lock/subsys 2>/dev/null
    touch /var/lock/subsys/listener
    rm /dev/shm 2>/dev/null
    mkdir /dev/shm 2>/dev/null
*)
    echo error
    exit 1
    ;;

esac 
```

### ตั้งค่า Permission ให้กับไฟล์ `/etc/rc2.d/S01shm_load`
```sh
sudo chmod 755 /etc/rc2.d/S01shm_load
```

### จากนั้นรันคำสั่งต่อไปนี้
```sh
sudo ln -s /usr/bin/awk /bin/awk

sudo mkdir /var/lock/subsys

sudo touch /var/lock/subsys/listener
```

### ทำการ restart เครื่อง
```sh
sudo reboot
```

## ติดตั้ง Oracle

### ติดตั้ง DBMS
```sh
sudo dpkg --install oracle-xe_11.2.0-2_amd64.deb
```

### ตั้งค่า Service ของ Oracle
```sh
sudo /etc/init.d/oracle-xe configure
```

### Set Envorionment
```sh
pico ~/.bashrc
```

```sh
export ORACLE_HOME=/u01/app/oracle/product/11.2.0/xe
export ORACLE_SID=XE
export NLS_LANG=`$ORACLE_HOME/bin/nls_lang.sh`
export ORACLE_BASE=/u01/app/oracle
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
export PATH=$ORACLE_HOME/bin:$PATH
```

```sh
. ~/.profile
```

### ทำกัน Start Service oracle
```sh
sudo service oracle-xe start
```

### ทำการ Add User ของคุณเข้าใสกลุ่มผู้ดูแลระบบ DBA
```sh
sudo usermod -a -G dba YOURUSERNAME
```

## Oracle Command line

### Start Service
```sh
sudo service oracle-xe start
```

### Using SQLPlus
```sh
sqlplus sys as sysdba
```

### User DBA on SQLPlus
```sh
# Create User
create user USERNAME identified by PASSWORD;

# Alter reset log
alter database open resetlogs;

# Grant role
grant connect, resource to USERNAME;
```
