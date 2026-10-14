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
nano /etc/dhcp.conf
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
