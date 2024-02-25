# Linux 

A series of setup papers centered around networking, automation usign Ansible and Linux administration. 

Distros used
- Ubuntu Server
- CentOS Stream 9

> WARNING: May contain wrong setup instructions (but somehow it works so i do not care)

## Network setup
```
                 ----------
                 | ROUTER |
                 ----------
                     |
     -------------------------------------
    |          |           |              |
-------  -----------  ------------  ------------
| web |  | ansible |  | client-1 |  | client-2 |
-------  -----------  ------------  ------------
```

| DNS         | IP            | OS                    |  |
| ------- | ----------- | ------------- | --------------------- |
| router         | 192.168.1.1            | Ubuntu Server              | DNS, DHCP server                           |
| web            | 192.168.1.2            | Ubuntu Server              | Web server                                 |
| ansible        | 192.168.1.3            | Ubuntu Server              | Administration server                      |
|                | 192.168.1.4            | CentOS 9                   | Client 1                                   |
|                | 192.168.1.5            | CentOS 9                   | Client 2                                   |

Clients and servers have access to the internet through the **router**, the **web** and **admin** have static addresses **192.168.1.2** and **192.168.1.3** router has address of 192.168.1.1, the rest are clients controlled by Ansible installed on the admin server.

# Router
Ubuntu Server as Router
### Assign IP address to interface
```
sudo vim /etc/netplan/00-installey-config.yaml
__________________________________
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.168.1.1/24]
      routes:
        - to: default
          via: 192.168.10.220
  version: 2
__________________________________

sudo netplan apply
```
### DHCP server
```
sudo apt install isc-dhcp-server
sudo vim /etc/dhcp/dhcpd.conf
__________________________________

default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;

subnet 192.168.1.0 netmask 255.255.255.0 {
        range 192.168.1.2 192.168.1.100;
        option routers 192.168.1.1;
        option domain-name-servers 192.168.1.1;
        option domain-name "foxnetwork.com";
}

host WebServer {
        hardware ethernet 08:00:27:b7:ef:6c;
        fixed-address 192.168.1.2;
}

host AnsibleServer {
        hardware ethernet 08:00:27:fa:fd:91;
        fixed-address 192.168.1.3;
}
```

Assign interface for DHCP service
```
sudo vim /etc/default/isc-dhcp-server
__________________________________

INTERFACESv4="enp0s8"
```

Restart/start DHCP server
```
sudo systemctl start isc-dhcp-server //START SERVICE 
sudo systemctl status isc-dhcp-server //CHECK STATUS
sudo systemctl restart isc-dhcp-server //RESTART SERVICE
```

To check DHCP data
> sudo journalctl -xe | grep dhcp

Or **/var/log/syslog**

## DNS Server
### Install bind9
```
sudo apt install bind9
```

### Modify options
```
sudo vim /etc/bin/named.conf.options
__________________________________

options {
        directory "/var/cache/bind";

        forwarders {
            8.8.8.8;
            8.8.4.4;
        };

        dnssec-validation auto;
        listen-on { 192.168.1.1; };
};

```
### Set up zones
```
sudo vim /etc/bind/named.conf.local
__________________________________

zone "foxnetwork.com" {
        type master;
        file "/etc/bind/zones/foxnetwork.com";
};

zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zone/networkfox.com";
};

```

### Create zones:
#### 1. Forward zone: db.foxnetwork.com
```
sudo vim /etc/bind/zones/foxnetwork.com
__________________________________
;       BIND data file
;
;
$TTL    86400
@       IN      SOA     router.foxnetwork.com. admin.foxnetwork.com. (
                        1       ;       SWERIAL
                        604800  ;       REFRESH
                        86400   ;       RETRY
                        2419200 ;       EXPIRE
                        86400 ) ;       NEGATIVE CACHE TTL
;
@           IN      NS      router.foxnetwork.com.
router      IN      A       192.168.10.220
web         IN      A       192.168.1.2
ansible     IN      A       192.168.1.3

```

#### 2. Reverse zone: db.reverse
```
sudo vim /etc/bind/zones/networkfox.com
__________________________________
;       BIND REVERSE ZONES
;
;
$TTL    86400
@       IN      SOA     router.foxnetwork.com.     admin.foxnetwork.com. (
                        1       ;       SERIAL
                        604800  ;       REFRESH
                        86400   ;       RETRY
                        2419200 ;       EXPIRE
                        86400 ) ;       NEGATIVE CACHE TTL
;
@               NS      router.foxnetwork.com.
1       IN      PTR     router
2       IN      PTR     web
3       IN      PTR     ansible
```
### Restart service
```
sudo systemctl start bind9.service
sudo systemctl restart bind9.service
sudo systemctl status bind9.service
```

## Enable internet access for clients
### Enable IP Forwarding
```
sudo vim /etc/sysctl.conf
__________________________________
...
net.ipv4.ip_forward=1 //Uncomment
...
__________________________________
sudo sysctl -p
```

### Set Up NAT (Network Address Translation):
```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

If necessary restart services
```
sudo systemctl restart isc-dhcp-server
```

## Change resolve.conf configuration
Install **resolvconf** package
```
sudo apt install resolvconf
```

Modify file **/etc/resolvconf/resolv.conf.d/head**
```
sudo vim /etc/resolvconf/resolv.conf.d/head
__________________________________
nameserver 127.0.0.1
nameserver 192.168.1.1
```

Update **/etc/resolve.conf**
```
sudo resolvconf --enable-updates
sudo resolvconf -u
```

Restart services
```
sudo systemctl restart isc-dhcp-server
sudo systemctl restart bind9.service
```

