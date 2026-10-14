# Jarkom-Modul-3-2026-k-63

## Soal 1
```bash
```
```bash
```
```bash
```

## Soal 2
```bash
Restart dengan
- sysctl net.ipv4.ip_forward
- iptables -t nat -L POSTROUTING -n -v
- /root/setup-jichi.sh

echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt update
apt install -y isc-dhcp-server nano
nano /etc/default/isc-dhcp-server
nano /etc/dhcp/dhcpd.conf

cp /etc/dhcp/dhcpd.conf /root/dhcpd.conf
cp /etc/default/isc-dhcp-server /root/isc-dhcp-server.default
cp /etc/network/interfaces /root/interfaces.bak
chmod +x /root/setup-dhcp.sh

ip -br addr
cat /etc/dhcp/dhcpd.conf
dhcpd -t -cf /etc/dhcp/dhcpd.conf
ps aux | grep dhcpd

service isc-dhcp-server stop
dhcpd -4 -d -cf /etc/dhcp/dhcpd.conf eth0

service isc-dhcp-server restart
```
```bash
INTERFACESv4="eth0"
```
```bash
default-lease-time 600;
max-lease-time 7200;
authoritative;

# Network 4: subnet lokal tanpa pool dinamis
subnet 10.95.4.0 netmask 255.255.255.0 {
}

# Network 2
subnet 10.95.2.0 netmask 255.255.255.0 {
    range 10.95.2.100 10.95.2.200;
    option routers 10.95.2.1;
    option domain-name-servers 192.168.122.1;
}

# Network 3
subnet 10.95.3.0 netmask 255.255.255.0 {
    range 10.95.3.100 10.95.3.200;
    option routers 10.95.3.1;
    option domain-name-servers 192.168.122.1;
}
```
```bash
#!/bin/bash

echo "nameserver 192.168.122.1" > /etc/resolv.conf
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
cp /root/isc-dhcp-server.default /etc/default/isc-dhcp-server
touch /var/lib/dhcp/dhcpd.leases
service isc-dhcp-server restart
```
## Soal 3

```bash
service isc-dhcp-relay-restart
ps aux | grep dhcrelay
apt update
apt install isc-dhcp-relay -y
service isc-dhcp-relay restart
cat /etc/default/isc-dhcp-relay
```
```bash
chmod +x /root/setup-dhcp.sh
/root/setup-dhcp.sh
service isc-dhcp-server stop
service isc-dhcp-server stop
/etc/init.d/isc-dhcp-server restart
cat /var/lib/dhcpd.leases
```
```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apk update
nano /etc/network/interfaces
apk update rc-service
rc-service networking start
ip addr flush dev eth0
udhcpc -i eth0
ip route
```

## Soal 4
```bash
default-lease-time 600;
max-lease-time 7200;
authoritative;

class "pool1" {
   match if (substring(hardware, 1, 6) = 02:42;73:bc:7c:00) or
            (substring(hardware, 1, 6) = 02:42:d5:94:24:00);
}

class "pool2" {
   match if (substring(hardware, 1, 6) = 02:42:1c:8c:ac:00) or
            (substring(hardware, 1, 6) = 02:42:0c:6b:f2:00);
}

subnet 10.95.4.0 netmask 255.255.255.0 {
}

subnet 10.95.2.0 netmask 255.255.255.0 {
  range 10.95.2.100 10.95.2.200;
  option routers 10.95.2.1;
  option domain-name-servers 192.168.122.1;

pool {
  allow members of "pool1";
  range 10.95.2.10 10.95.2.35;
 }

pool {
  allow members of "pool2";
  range 10.95.2.10 10.95.2.35;
 }
}

subnet 10.95.3.0 netmask 255.255.255.0 {
  range 10.95.3.100 10.95.3.200;
  option routers 10.95.3.1;
  option domain-name-servers 192.168.122.1;
}
```
