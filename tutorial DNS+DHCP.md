![image](https://user-images.githubusercontent.com/54473576/226849444-63fd3e45-2bbe-49ce-8f32-65fff0a6172d.png)
# Trên máy gateway:

## Thay đổi IPv4 của interface ens37 

Dùng: `sudo vim /etc/netplan/00-installer-config.yaml`.

```
network:
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      addresses:
      - 10.0.0.1/24
  version: 2
```
Dùng `sudo netplan apply` để lưu lại đã thay đổi

Dùng `ip a` để xem cài đặt đã thay đổi
```
gw1@gateway:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:17 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 172.16.217.137/24 metric 100 brd 172.16.217.255 scope global dynamic ens33
       valid_lft 1191sec preferred_lft 1191sec
    inet6 fe80::20c:29ff:fe28:1317/64 scope link 
       valid_lft forever preferred_lft forever
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:21 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 10.0.0.1/24 brd 10.0.0.255 scope global ens37
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe28:1321/64 scope link 
       valid_lft forever preferred_lft forever
4: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:2b brd ff:ff:ff:ff:ff:ff
    altname enp2s6
    inet 10.0.1.1/24 brd 10.0.1.255 scope global ens38
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe28:132b/64 scope link 
       valid_lft forever preferred_lft forever
```

## Cài đặt isc-dhcp-server

Cài đặt isc-dhcp-server

```
sudo apt install isc-dhcp-server
```
Dùng `sudo vim /etc/dhcp/dhcpd.conf` để sửa file config

```
# To specify the default and maximum lease time 
default-lease-time 600;

max-lease-time 7200;

ddns-update-style none;

authoritative;

subnet 10.0.0.0 netmask 255.255.255.0
{
        range 10.0.0.100 10.0.0.200;
        option routers 10.0.0.1;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 10.0.1.10;
}
```

Dùng `sudo systemctl restart isc-dhcp-server` để khởi động dịch vụ `isc-dhcp-server`

Dùng `sudo systemctl enable isc-dhcp-server` để bật dịch vụ `isc-dhcp-server`


## Enable IPv4_forward:

Để nhận các gói tin từ NIC này sang NIC khác của máy

Uncomment dòng `net.ipv4.ip_forward=1` trong file /etc/sysctl.conf.

Mở file /etc/sysctl.conf dùng `sudo vim /etc/sysctl.conf`. Nhấn `i` để vào chế độ insert và sửa. Sửa xong thì nhấn phím `Esc` và gõ `:wq`

Áp dụng thay đổi tức thì mà không cần reboot dùng: `sudo sysctl -p /etc/sysctl.conf`

## Cài đặt NAT:

Cài đặt iptables-persistent: dùng `sudo apt-get install iptables-persistent`

Dùng `sudo iptables --table nat -L -v` để xem các chain và rule trong bảng `nat`

```
gw1@gateway:~$ sudo iptables --table nat -L -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```
Dùng `sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE` để thêm rule NAT

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

# Trên DNS server

Install bind9:
```
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
```

## Cài đặt DNS Forwarding

Sửa file config DNS dùng:
```
sudo vim /etc/bind/named.conf.options
```
```
acl trustedclients {
        localhost;
        localnets;
        10.0.0.0/24;
        10.0.1.0/24;
        10.0.2.0/24;
};

options {
        directory "/var/cache/bind";

        recursion yes;
        allow-query { trustedclients; };
        allow-query-cache { trustedclients; };
        allow-recursion { trustedclients; };
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

        dnssec-validation auto;

        listen-on-v6 { any; };
        listen-on port 53 { 127.0.0.1; 10.0.1.10; };
};

```

## DNS local

### Define zone files

Dùng `sudo vim named.conf.local` để mở file defile zone

```
zone "good.lab" {
        type master;
        file "/etc/bind/db.good.lab";
};

zone "1.0.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.10.0.1";
};
```
Kiểm tra lại các lỗi với câu lệnh :
```
sudo named-checkconf
```

### Create a forward lookup zone

Tạo 1 file forwarding file bằng cách:
```
sudo cp db.local db.good.lab
```

Trong file `db.good.lab` sau khi tạo có như:
```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1
```
Sửa lại :
```
;
; BIND data file for local good.lab zone
;
$TTL    604800
@       IN      SOA     ns1.good.lab. admin.good.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.good.lab.

ns1     IN      A       10.0.1.10
dhcp1   IN      A       10.0.1.1
```

Kiểm tra lỗi cú pháp với: 
```
sudo named-checkzone good.lab db.good.lab
```

### Create a reverse lookup zone
Để tạo 1 file reverse zone dùng:
```
sudo cp db.127 db.10.0.1
```

Mở file `db.10.0.1` được như:

```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
1.0.0   IN      PTR     localhost.
```

Sửa lại:
```
;
; BIND reverse data file for local good.lab zone
;
$TTL    604800
@       IN      SOA     ns1.good.lab. admin.good.lab. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.good.lab.

10      IN      PTR     ns1.good.lab.
1       IN      PTR     dhcp1.good.lab.
```

Kiểm tra lại cú pháp với:
```
sudo named-checkzone 1.0.10.in-addr.arpa db.10.0.1
```
### Sửa DNS entry của server để tự dùng DNS của chính nó
Sửa file config interface:
```
sudo vim /etc/netplan/00-installer-config.yaml 
```
Sửa lại dòng addresses của nameservers thành địa chỉ loop của server
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens36:
      addresses: [10.0.1.10/24]
      gateway4: 10.0.1.1
      nameservers:
        addresses:
        - 127.0.0.1
        search:
        - good.lab
  version: 2
```

Lưu lại cài đặt đã sửa với netplan:
```
sudo netplan apply
```

### Khởi động dịch vụ bind9:

Khởi động dịch vụ:
```
sudo systemctl start bind9
```

Xem trạng thái của dịch vụ:
```
sudo systemctl status bind9
```

# Trên máy client:

![Screenshot from 2023-03-22 15-51-31](https://user-images.githubusercontent.com/54473576/226849814-fe86f8c8-68ed-48f0-bcf6-e5ad2d6360b5.png)

![Screenshot from 2023-03-22 15-52-46](https://user-images.githubusercontent.com/54473576/226850110-fbe7fb54-e14d-4887-911f-fb6696dc3af0.png)

## Kiểm tra với các máy chủ bên ngoài
![Screenshot from 2023-03-22 15-57-28](https://user-images.githubusercontent.com/54473576/226851450-5f6d3331-253a-4aec-b34d-e3c8133e94ea.png)

## Kiểm tra với DNS local

![Screenshot from 2023-03-23 16-32-23](https://user-images.githubusercontent.com/54473576/227161601-814916ca-9d99-409c-881e-311cfc7d85b0.png)
