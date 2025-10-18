+++
title = "在OrangePI 3B上安装PXVIRT"
date = "2025-10-18T17:52:16+08:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["PVE"]
keywords = ["PXVIRT"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
# 在OrangePI 3B上安装PXVIRT
PXVIRT是Arm版的PVE，如果有什么问题应该可以参考官方PVE。(大概吧)
## 1. 安装Armbian
(其实这里选择Debian也是可以的)
从[Armbian Build](https://github.com/armbian/build)下载适合OrangePI 3B的镜像，烧录进SD卡中。Armbian第一次初始化需要外接屏幕。
更换国内的镜像源,这里我选的是清华的镜像:[Armbian](https://mirrors.tuna.tsinghua.edu.cn/help/armbian/)。
## 2. 安装和设置PXVIRT
为了让PXVIRT不占用物理网口，这里使用Nginx反代PXVIRT，将PXVIRT绑定在一个环回网口上。
### 安装PXVIRT
需要准备好PXVIRT的证书放在`/root/cert/`下，命名为`certificate.crt`和`private.key`，并确保Nginx可以访问到。
```bash
chown www-data:www-data /root/cert/ -R
chmod 644 /root/cert/certificate.crt
chmod 600 /root/cert/private.key
```
#### 配置网口
```bash
apt update
apt install ifupdown2 resolvconf dhcpcd-base iptables
```
编辑`/etc/network/interfaces`，这里`-o wlan0`根据实际上网网口更改：
```
auto lo
iface lo inet loopback
        up ip address add 172.16.2.1/32 dev lo
        down ip address del 172.16.2.1/32 dev lo

auto dhcp
iface dhcp inet manual

iface end1 inet manual

auto inet
iface inet inet manual

auto vmbr0
iface vmbr0 inet static
        address 172.16.3.1
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '172.16.3.0/24' -o wlan0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '172.16.3.0/24' -o wlan0 -j MASQUERADE

auto wlan0
iface wlan0 inet dhcp
```

编辑`/etc/sysctl.d/pve.conf`：
```
net.ipv4.conf.all.route_localnet=1
```

编辑`/etc/hosts`：
```
127.0.0.1   localhost
172.16.2.1   orangepi3b.local orangepi3b
::1         localhost ip6-localhost ip6-loopback
fe00::0     ip6-localnet
ff00::0     ip6-mcastprefix
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
```
重启网络：
```bash
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable networking
systemctl restart networking
```
#### 配置Nginx
安装Nginx:

```bash
apt update
apt install nginx libnginx-mod-stream
mv /etc/nginx/modules-enabled/50-mod-stream.conf.removed /etc/nginx/modules-enabled/50-mod-stream.conf
mkdir /etc/nginx/streams-available /etc/nginx/streams-enabled
```
编辑`/etc/nginx/sites-available/pve`：
```nginx
upstream proxmox {
        server "orangepi3b";
}
 
server {
        listen 8009 default_server;
        #       listen [::]:80 default_server;
        #rewrite ^(.*) https:// permanent;
        server_name _;
        ssl on;
        proxy_redirect off;
        ssl_certificate /root/cert/certificate.crt;
        ssl_certificate_key /root/cert/private.key;
        location / {
                proxy_ssl_verify off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_pass https://orangepi3b.local:8006;
                proxy_buffering off;
                client_max_body_size 0;
                proxy_connect_timeout  3600s;
                proxy_read_timeout  3600s;
                proxy_send_timeout  3600s;
                send_timeout  3600s;
        }
}
```
启用站点配置：
```bash
ln -s /etc/nginx/sites-available/pve /etc/nginx/sites-enabled/pve
```
修改`/etc/nginx/nginx.conf`：
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}

stream {
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/streams-enabled/*;
}


#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
```
重启Nginx：
```bash
systemctl reload nginx
```
#### 配置PXVIRT
添加Lierfang源：
```bash
curl -L https://mirrors.lierfang.com/pxcloud/lierfang.gpg -o /usr/share/keyrings/pxvirt-lierfang.gpg
```
编辑`/etc/apt/sources.list.d/pxvirt.sources`：
```bash
Types: deb
URIs: https://mirrors.lierfang.com/pxcloud/pxvirt
Suites: bookworm
Components: main
Signed-By: /usr/share/keyrings/pxvirt-lierfang.gpg
```
安装：
```bash
apt update
apt install proxmox-ve pve-manager qemu-server pve-cluster
```
这里如果因为网络问题安装失败，多试几次，好像Lierfang源不太稳定。
登录web页面，`https://目标IP:8009`，用户名为root，密码为你的root密码，领域选择Linux PAM。选择节点`orangepi3b`-`网络`-`创建`-`Linux Bridge`，`IPv4/CIDR`填`172.16.3.1/24`，勾上自动启动。

## 3. 使用PXVIRT
### LXC
访问[Images](https://sgp1lxdmirror01.do.letsbuildthe.cloud/images/)下载LXC模板，选择arm64版本，复制`rootfs.tar.xz`链接。
进入PXVIRT管理面板，选择节点`orangepi3b`-`local`-`CT模板`-`从URL下载`，填入刚刚复制的链接，以apline为例，链接`https://sgp1lxdmirror01.do.letsbuildthe.cloud/images/alpine/3.22/arm64/default/20251016_13:00/rootfs.tar.xz`，名称`alpine-3.22-arm64.tar.xz`，等待下载完成。
选择右上角`创建CT`，填写密码、主机名、ID信息，下一步，模板选择刚刚下载的模板，下一步，分配磁盘，下一步，分配CPU，下一步，分配内存，下一步，设置网络，`设置桥接`选之前创建的网桥，`IPv4`勾选`静态`，`IPv4/CIDR`填`172.16.3.1xx/24`，`网关 (IPv4)`填`172.16.3.1`，下一步一直到完成。
左侧列表应该可以看到刚刚创建的CT，至此一个LXC创建完毕。
### 虚拟机
在OrangePI-3B上还是不适合跑虚拟机，有需要的自行搜索PVE教程。如果要用，选ARM64版本的镜像。

## 4. 配置Wifi
这里我选的是`wpa_supplicant`来配置Wifi：
创建`/etc/wpa_supplicant.conf`：
```bash
wpa_passphrase 网络名称 密码 > /etc/wpa_supplicant.conf
```
### 使用rc.local
编辑`/etc/rc.local`：
```bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

wpa_supplicant -B -Dwext -iwlan0 -c /etc/wpa_supplicant.conf
dhcpcd wlan0

exit 0
```
### 使用systemd
编辑`/etc/systemd/system/wifi-setup.service`：
```
[Unit]
Description=WiFi Setup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/wpa_supplicant -B -Dwext -iwlan0 -c /etc/wpa_supplicant.conf
ExecStartPost=/usr/bin/dhcpcd wlan0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
启用配置：
```bash
systemctl enable wifi-setup.service
```
### 5. 穿透
以ssh为示例：
假设目标IP为`172.16.3.101`
编辑`/etc/nginx/streams-available/101-ssh`
```
server {
        listen 10122;
        proxy_pass 172.16.3.101:22;
}
```
启用配置：
```bash
ln -s /etc/nginx/streams-available/101-ssh /etc/nginx/streams-enabled/101-ssh
```