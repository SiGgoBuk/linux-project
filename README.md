# 🐧 Linux Server Administration Project
> Rocky Linux 9.7 환경에서 서버 운영에 필요한 핵심 명령어 및 서비스 구성 실습

<br>

## 개요

| 항목 | 내용 |
|------|------|
| **기간** | 2026.01 |
| **OS** | Rocky Linux 9.7 |
| **환경** | VMware Workstation Pro 17 |
| **구성** | VM 3대 (Project / Project-B / Project-C) |
| **역할** | 조장 |

### VM 사양

| VM | IP | RAM | DISK |
|----|----|-----|------|
| Project (Main) | 192.168.111.100/24 | 4GB | 80GB |
| Project-B | 192.168.111.150/24 | 2GB | 80GB |
| Project-C | 192.168.111.200/24 | 2GB | 80GB |

<br>

## 목차

1. [환경 구성](#1-환경-구성)
2. [사용자 및 그룹 관리](#2-사용자-및-그룹-관리)
3. [디스크 추가 및 LVM 구성](#3-디스크-추가-및-lvm-구성)
4. [디스크 쿼터 설정](#4-디스크-쿼터-설정)
5. [서버 구성](#5-서버-구성)

<br>

---

## 1. 환경 구성

Rocky Linux 9.7을 3대 설치 후 고정 IP 설정 (CLI / GUI 두 방식 모두 실습)

### CLI 방식
```bash
# 네트워크 설정 파일 직접 편집
vi /etc/NetworkManager/system-connections/ens160.nmconnection

# [ipv4] 섹션 설정값
address1=192.168.111.100/24
dns=192.168.111.2;
gateway=192.168.111.2
method=manual

# 설정 적용
nmcli connection reload
nmcli connection up ens160

# IP 확인
ip address
```

### TUI 방식 (nmtui)
```bash
nmtui  # TUI 환경에서 네트워크 설정
```

### GUI 방식 (nmtui)
```bash
# 우분투 GUI 환경에서 네트워크 설정
```

<br>

---

## 2. 사용자 및 그룹 관리

### 사용자 생성
```bash
# 사용자 8명 생성
adduser sonhm
adduser leegi
adduser kimmj
adduser hwanghc
adduser kdj
adduser lyh
adduser jsh
adduser jgh

# 사용자 확인 (/etc/passwd 마지막 8행)
tail -8 /etc/passwd

# 패스워드 설정
passwd sonhm
```

> 일반 사용자의 UID는 1000번부터 할당됨
> 
> `/etc/passwd` 구성: `계정명:암호:UID:GID:설명:홈디렉터리:로그인쉘`
> 
> `/etc/shadow` 구성: `계정명:암호:최종변경일:MIN:MAX:WARNING:INACTIVE:EXPIRE:Flag`

### 그룹 생성 및 사용자 포함
```bash
# 그룹 생성
groupadd eusoccer
groupadd krsoccer

# 기본 그룹 변경
usermod -g eusoccer sonhm
usermod -g eusoccer leegi
usermod -g eusoccer kimmj
usermod -g eusoccer hwanghc
usermod -g krsoccer kdj
usermod -g krsoccer lyh
usermod -g krsoccer jsh
usermod -g krsoccer jgh

# 그룹 멤버로 추가
gpasswd -a sonhm eusoccer
gpasswd -a leegi eusoccer
gpasswd -a kimmj eusoccer
gpasswd -a hwanghc eusoccer
gpasswd -a kdj krsoccer
gpasswd -a lyh krsoccer
gpasswd -a jsh krsoccer
gpasswd -a jgh krsoccer

# 그룹 확인
tail /etc/group
```

> `/etc/group` 구성: `그룹명:그룹암호:GID:그룹멤버`

<br>

---

## 3. 디스크 추가 및 LVM 구성

물리 디스크 3개(20G/30G/50G)를 추가하여 LVM으로 논리 볼륨 구성

```
[LVM 구조]
Physical Volume (PV)         Volume Group (VG)     Logical Volume (LV)
  /dev/sdb1 (20G)  ──┐
  /dev/sdc1 (30G)  ──┼──▶  DATA (100G)  ──▶  VIDEO (40G)
  /dev/sdd1 (50G)  ──┘                  └──▶  AUDIO (나머지)
```

### 디스크 확인
```bash
lsblk
fdisk -l | grep sd
```

### 파티션 생성
```bash
fdisk /dev/sdb  # 20G 디스크
fdisk /dev/sdc  # 30G 디스크
fdisk /dev/sdd  # 50G 디스크
# 각각 파티션 타입 8e (Linux LVM) 으로 설정
```

### PV → VG → LV 순서로 구성
```bash
# PV 생성
pvcreate /dev/sdb1 /dev/sdc1 /dev/sdd1
pvscan  # PV 확인

# VG 생성
vgcreate DATA /dev/sdb1 /dev/sdc1 /dev/sdd1
vgdisplay  # VG 확인

# LV 생성
lvcreate --size 40G --name VIDEO DATA
lvcreate --extents 100%FREE --name AUDIO DATA
lvscan  # LV 확인
```

### 파일시스템 생성 및 마운트
```bash
# ext4 파일시스템 생성
mkfs.ext4 /dev/DATA/VIDEO
mkfs.ext4 /dev/DATA/AUDIO

# 마운트 포인트 생성 및 마운트
mkdir /lvm1 /lvm2
mount /dev/DATA/VIDEO /lvm1
mount /dev/DATA/AUDIO /lvm2

# 영구 마운트 설정
vi /etc/fstab
# 추가:
# /dev/DATA/VIDEO  /lvm1  ext4  defaults  0 0
# /dev/DATA/AUDIO  /lvm2  ext4  defaults  0 0
```

<br>

---

## 4. 디스크 쿼터 설정

10G 디스크를 추가하여 사용자별 디스크 사용량 제한 설정

### 사용자별 쿼터 할당값

| 사용자 | Soft Limit | Hard Limit |
|--------|-----------|-----------|
| aespa | 700M | 1G |
| IVE | 700M | 1G |
| NewJeans | 700M | 1G |

### 구성 절차
```bash
# 1. 파티션 생성 및 마운트
fdisk /dev/sdb
mkfs.ext4 /dev/sdb1
mkdir /userHome
mount /dev/sdb1 /userHome

# 2. 쿼터 전용 사용자 생성
useradd -d /userHome/aespa aespa
useradd -d /userHome/IVE IVE
useradd -d /userHome/NewJeans NewJeans

passwd aespa
passwd IVE
passwd NewJeans

# 3. fstab에 쿼터 옵션 추가
vi /etc/fstab
# /dev/sdb1  /userHome  ext4  defaults,usrjquota=aquota.user,jqfmt=vfsv0  0 0

# 4. remount (재부팅 없이 fstab 적용)
mount --options remount /userHome

# 5. 쿼터 DB 생성
cd /userHome
quotaoff -avug
quotacheck -augmn
rm -rf aquota.*
touch aquota.user aquota.group
chmod 600 aquota.*
quotacheck -augmn
quotaon -avug

# 6. 사용자별 쿼터 할당
edquota -u aespa   # soft: 700, hard: 1000 (단위: 1K block)
edquota -u IVE
edquota -u NewJeans

# 타 사용자에게 동일 쿼터 복사
edquota -p aespa IVE
edquota -p IVE NewJeans

# 7. 쿼터 확인
repquota /userHome
quota  # 사용자 전환 후 개인 쿼터 확인
```

<br>

---

## 5. 서버 구성

Main 서버(192.168.111.100)를 중심으로 9종 서버 구성

### 5-1. SSH Server

```bash
# 설치 확인 및 설치
rpm -qa openssh-server
dnf install openssh-server

# 환경설정 (필요 시)
vi /etc/ssh/sshd_config

# 서비스 시작 및 자동시작 등록
systemctl restart sshd
systemctl enable sshd

# 방화벽 허가
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
firewall-cmd --list-services
```

---

### 5-2. XRDP Server (원격 데스크톱)

```bash
# XRDP 설치 여부 확인 및 설치
rpm -qa xrdp
dnf -y install epel-release

# 서비스 시작
systemctl start xrdp
systemctl enable xrdp

# 방화벽 허가 (포트 3389)
firewall-cmd --permanent --add-port=3389/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```

---

### 5-3. DNS Server

Project(100)을 DNS 서버로 구성, `history.com` 도메인 운영

```bash
# 설치
rpm -qa bind bind-chroot
dnf -y install bind bind-chroot

# 환경설정
vi /etc/named.conf
# listen-on port 53 { any; };
# listen-on-v6 port 53 { none; };
# allow-query { any; };
# dnssec-validation no;

# 존 선언 추가 (named.conf 하단)
# zone "history.com" IN {
#     type master;
#     file "history.com.db";
#     allow-update { none; };
# };

# 존 파일 작성
vi /var/named/history.com.db
```

```
$TTL    3H
@       SOA     @       root. (2 1D 1H 1W 1H)
        IN      NS      @
        IN      A       192.168.111.100

project     IN  A   192.168.111.100
project-b   IN  A   192.168.111.150
project-c   IN  A   192.168.111.200

www         IN  CNAME   project-b
ftp         IN  CNAME   project-c
```

```bash
# 문법 검사
named-checkconf
named-checkzone history.com /var/named/history.com.db

# 서비스 시작
systemctl start named
systemctl enable named

# 방화벽 허가
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload

# 동작 확인
nslookup
> server 192.168.111.100
> www.history.com
> ftp.history.com
```

---

### 5-4. Web Server (Apache httpd)

Project-B(150)에 Web Server 구성

```bash
# 설치
rpm -qa httpd
dnf install -y httpd

# 서비스 시작
systemctl start httpd
systemctl enable httpd

# 방화벽 허가
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# index.html 수정
vi /var/www/html/index.html

# 접속 확인
curl www.history.com
```

---

### 5-5. FTP Server (vsftpd)

Project-C(200)에 익명 FTP 서버 구성

```bash
# 설치
rpm -qa vsftpd
dnf -y install vsftpd

# 환경설정 (익명 로그인 활성화)
vi /etc/vsftpd/vsftpd.conf
# anonymous_enable=YES

# 공유 디렉터리에 파일 배치
cd /var/ftp/pub
cp /boot/vmlinuz-5.* file1

# 서비스 시작
systemctl restart vsftpd
systemctl enable vsftpd

# 방화벽 허가
firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload
```

```bash
# 클라이언트에서 접속 테스트
ftp ftp.history.com
# Name: anonymous
# Password: (아무값)
ftp> cd pub
ftp> ls
ftp> get file1
```

---

### 5-6. NFS Server

Project(100)에서 `/share` 디렉터리를 Project-B(150)와 공유

```bash
# 설치 확인
rpm -qa nfs-utils

# exports 파일 설정
vi /etc/exports
# /share  *(rw,sync)

# 공유 디렉터리 생성 및 권한
mkdir /share
chmod 707 /share

# 서비스 시작
systemctl restart nfs-server
systemctl enable nfs-server

# 방화벽 허가
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```

```bash
# 클라이언트(Project-B)에서 마운트
mkdir ~/myShare
su -c 'mount -t nfs 192.168.111.100:/share myShare'

# 영구 마운트 (fstab)
vi /etc/fstab
# 192.168.111.100:/share  /home/rocky/myShare  nfs  defaults  0 0
```

---

### 5-7. Samba Server

Linux ↔ Windows 파일 공유 (`/sbhistory` 디렉터리)

```bash
# 설치
dnf -y install samba samba-common samba-client

# 공유 디렉터리 구성
mkdir /sbhistory
groupadd sambaGroup
chgrp sambaGroup /sbhistory
chmod 770 /sbhistory
usermod -G sambaGroup rocky
smbpasswd -a rocky

# 환경설정
vi /etc/samba/smb.conf
```

```ini
[global]
    workgroup = WORKGROUP
    security = user
    unix charset = UTF-8
    map to guest = Bad User

[sbhistory]
    path = /sbhistory
    writable = yes
    guest ok = no
    create mode = 0777
    directory mode = 0777
    valid users = @sambaGroup
```

```bash
# 설정 문법 확인
testparm

# 서비스 시작
systemctl restart smb nmb
systemctl enable smb nmb

# 방화벽 허가
firewall-cmd --permanent --add-service=samba
firewall-cmd --permanent --add-service=samba-client

# SELinux 설정
setsebool -P samba_enable_home_dirs on
chcon -R -t samba_share_t /sbhistory

# Windows에서 접속: \\192.168.111.100\sbhistory
```

---

### 5-8. DHCP Server

Project(100)에서 DHCP 서버 구성, Project-C(200)가 IP 자동 수신

```bash
# 설치
dnf -y install dhcp-server

# 환경설정
vi /etc/dhcp/dhcpd.conf
```

```
ddns-update-style interim;
subnet 192.168.111.0 netmask 255.255.255.0 {
    option routers 192.168.111.2;
    option subnet-mask 255.255.255.0;
    range dynamic-bootp 192.168.111.201 192.168.111.210;
    option domain-name-servers 8.8.8.8;
    default-lease-time 10000;
    max-lease-time 50000;
}
```

```bash
# 서비스 시작
systemctl restart dhcpd
systemctl enable dhcpd

# 방화벽 허가
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --reload

# 결과: Project-C에 192.168.111.201 자동 배정
```

---

### 5-9. Mail Server (Sendmail + Dovecot)

```bash
# 설치
dnf -y install sendmail sendmail-cf dovecot

# 호스트명 변경
hostnamectl set-hostname mail.history.com
vi /etc/hosts
# 192.168.111.100  mail.history.com
vi /etc/sysconfig/network
# HOSTNAME=mail.history.com

# DNS 존 파일에 MX 레코드 추가
vi /var/named/history.com.db
# IN  MX  10  mail.history.com
# mail  IN  A  192.168.111.100

# 오류 검사
named-checkconf
named-checkzone history.com history.com.db
# OK

# sendmail.cf 설정
vi /etc/mail/sendmail.cf
# 85: Cwhistory.com
# 87: Fw/etc/mail/local-host-names
# 268: DaemonPortOptions=Port=smtp, Name=MTA

# access 파일 설정
vi /etc/mail/access
# 192.168.111  RELAY
# history.com  RELAY

# Makemap hash
makemap hash /etc/mail/access < /etc/mail/access

# Dovecot 설정
vi /etc/dovecot/dovecot.conf
# protocols = imap pop3 lmtp submission
# listen = *, ::
# base_dir = /var/run/dovecot/

# mail 수정
vi /etc/dovecot/conf.d/10-mail.conf
# mail_location = mbox:~/mail:INBOX=/var/mail/%u
# mail_access_groups = mail
# lock_method = fcnt1

# ssl 수정
vi /etc/dovecot/conf.d/10-ssl.conf
# ssl = yes

# 서비스 시작
systemctl restart named sendmail dovecot
systemctl enable named sendmail dovecot

# 방화벽 허가
firewall-cmd --permanent --add-service=smtp
firewall-cmd --permanent --add-service=pop3
firewall-cmd --permanent --add-service=imap
firewall-cmd --reload
```

---

### 5-10. MariaDB Server

```bash
# 설치
dnf install mariadb-server

# 서비스 시작
systemctl restart mariadb
systemctl enable mariadb

# 방화벽 허가
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload

# root 패스워드 설정
mysqladmin -u root password '1234'

# DB 접속
mysql -h localhost -u root -p

# 접속 권한 설정 (원격 접속 허용)
MariaDB> grant all on *.* to root@'%' identified by '1234';
MariaDB> FLUSH PRIVILEGES;

# DB 생성
MariaDB> CREATE DATABASE history CHARACTER SET utf8;
MariaDB> show databases;
```

```bash
# 클라이언트(Project-B)에서 원격 접속
dnf -y install mariadb
mysql -h 192.168.111.100 -u root -p
MariaDB> show databases;  # history DB 확인
```

<br>

---

## 방화벽 서비스 최종 목록

```bash
firewall-cmd --list-services
# cockpit dhcp dhcpv6-client dns ftp http imap mysql nfs pop3 samba samba-client smtp ssh
```

<br>

---

## 참고

- Rocky Linux 9.7 공식 문서: https://docs.rockylinux.org
- 실습 환경: VMware Workstation Pro 17 (NAT 네트워크)
- 팀 구성: 조장 김동진 / 조원 장규혁, 이영훈, 정성현
