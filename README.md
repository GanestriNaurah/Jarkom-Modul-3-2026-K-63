# Jarkom-Modul-3-2026-k-63

Prefix IP kelompok: `10.95`

| Node | IP | Peran |
|---|---|---|
| jichi | eth0 DHCP (NAT), eth1 `10.95.1.1`, eth2 `10.95.2.1`, eth3 `10.95.3.1`, eth4 `10.95.4.1` | Router utama dan DHCP Relay |
| tengen | `10.95.4.10` | DHCP Server |

## Soal 1

Konfigurasi jichi: eth0 dari NAT, eth1-eth4 jadi gateway Network 1-4, IPv4 forwarding dan source NAT.

```bash
# jichi: paket yang dibutuhkan (image sudah punya iptables dan nano)
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt update
apt install -y iptables nano
```
```bash
# jichi: /etc/network/interfaces
auto eth0
iface eth0 inet dhcp
    post-up sysctl -w net.ipv4.ip_forward=1
    post-up iptables -t nat -C POSTROUTING -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

auto eth1
iface eth1 inet static
    address 10.95.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 10.95.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 10.95.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 10.95.4.1
    netmask 255.255.255.0
```
```bash
# jichi: /root/setup-jichi.sh (aman dijalankan berulang)
#!/bin/bash
PREFIX="10.95"

echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip link set eth0 up
for i in 1 2 3 4; do
  ip addr replace $PREFIX.$i.1/24 dev eth$i
  ip link set eth$i up
done

sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-forward.conf

iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
```bash
chmod +x /root/setup-jichi.sh
/root/setup-jichi.sh
```
```bash
# bukti jichi
ip -br addr
sysctl net.ipv4.ip_forward
iptables -t nat -L POSTROUTING -n -v      # harus satu baris MASQUERADE
ping -c 3 8.8.8.8
ping -c 3 google.com
```
```bash
# host (debinet): /etc/network/interfaces, ganti N (nomor network) dan H (angka terakhir IP)
auto eth0
iface eth0 inet static
    address 10.95.N.H
    netmask 255.255.255.0
    gateway 10.95.N.1
    dns-nameservers 192.168.122.1

# setelah itu
echo "nameserver 192.168.122.1" > /etc/resolv.conf
ifdown eth0; ifup eth0
```
```bash
# bukti tiap host
ip -br addr
ip route
ping -c 3 10.95.N.1     # gateway
ping -c 3 8.8.8.8       # akses eksternal (host yang relevan)
ping -c 3 google.com
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
apt update
apt install isc-dhcp-relay -y
service isc-dhcp-relay restart
ps aux | grep dhcrelay
cat /etc/default/isc-dhcp-relay
```
```bash
chmod +x /root/setup-dhcp.sh
/root/setup-dhcp.sh
service isc-dhcp-server stop
/etc/init.d/isc-dhcp-server restart
cat /var/lib/dhcp/dhcpd.leases
```
```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apk update
nano /etc/network/interfaces
ip addr flush dev eth0
udhcpc -i eth0
ip route
```

## Soal 4
```bash
# MAC tiap client (jalankan di client)
ip link show eth0

# itadori   02:42:73:bc:7c:00
# fushiguro 02:42:d5:94:24:00
# zenin     02:42:1c:8c:ac:00
# okkotsu   02:42:0c:6b:f2:00

nano /root/dhcpd.conf
```
```bash
default-lease-time 600;
max-lease-time 7200;
authoritative;

class "pool1" {
  match if (substring(hardware, 1, 6) = 02:42:73:bc:7c:00) or
           (substring(hardware, 1, 6) = 02:42:d5:94:24:00);
}

class "pool2" {
  match if (substring(hardware, 1, 6) = 02:42:1c:8c:ac:00) or
           (substring(hardware, 1, 6) = 02:42:0c:6b:f2:00);
}

subnet 10.95.4.0 netmask 255.255.255.0 {
}

# Network 2 (range lama .100-.200 dihapus supaya tidak tumpang tindih)
subnet 10.95.2.0 netmask 255.255.255.0 {
  option routers 10.95.2.1;
  option domain-name-servers 192.168.122.1;

  pool {
    allow members of "pool1";
    range 10.95.2.10 10.95.2.35;
  }

  pool {
    allow members of "pool2";
    range 10.95.2.70 10.95.2.95;
  }
}

# Network 3 (diganti di soal 5)
subnet 10.95.3.0 netmask 255.255.255.0 {
  range 10.95.3.100 10.95.3.200;
  option routers 10.95.3.1;
  option domain-name-servers 192.168.122.1;
}
```
```bash
dhcpd -t -cf /root/dhcpd.conf
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
ps aux | grep dhcpd

# jichi: relay harus jalan (kalau log jichi "Dropping reply received on eth4")
ps aux | grep dhcrelay
pkill dhcrelay
dhcrelay -4 -i eth2 -i eth3 -i eth4 10.95.4.10

# client
ip addr flush dev eth0
udhcpc -i eth0
ip addr show eth0

# tengen
cat /var/lib/dhcp/dhcpd.leases

# hasil: itadori 10.95.2.10, fushiguro 10.95.2.11, okkotsu 10.95.2.70, zenin 10.95.2.71
```

## Soal 5
```bash
# MAC client (jalankan di client)
ip link show eth0

# ino / kusakabe  02:42:16:ac:98:00 dan 02:42:91:bc:97:00
# miwa            02:42:aa:db:27:00

nano /root/dhcpd.conf
```
```bash
# tambahkan class di bawah class pool1 dan pool2
class "short" {
  match if (substring(hardware, 1, 6) = 02:42:16:ac:98:00) or
           (substring(hardware, 1, 6) = 02:42:91:bc:97:00);
}

class "long" {
  match if (substring(hardware, 1, 6) = 02:42:aa:db:27:00);
}

# ganti blok subnet 10.95.3.0 yang lama
subnet 10.95.3.0 netmask 255.255.255.0 {
  option routers 10.95.3.1;
  option domain-name-servers 10.95.1.2, 10.95.1.3;
  option domain-search "K63.com";

  pool {
    allow members of "short";
    range 10.95.3.10 10.95.3.89;
    default-lease-time 1800;
    max-lease-time 1800;
  }

  pool {
    allow members of "long";
    range 10.95.3.100 10.95.3.199;
    default-lease-time 43200;
    max-lease-time 43200;
  }
}
```
```bash
dhcpd -t -cf /root/dhcpd.conf
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
ps aux | grep dhcpd

# kalau lease lama (600 detik) masih nyangkut, bersihkan
service isc-dhcp-server stop
cp /var/lib/dhcp/dhcpd.leases /root/dhcpd.leases.bak
> /var/lib/dhcp/dhcpd.leases
service isc-dhcp-server start

# ino, kusakabe, miwa
ip addr flush dev eth0
udhcpc -i eth0
ip addr show eth0
cat /etc/resolv.conf

# tengen
cat /var/lib/dhcp/dhcpd.leases

# hasil: .3.10 dan .3.11 lease 30 menit, .3.100 (miwa) lease 12 jam
```

## Soal 6
```bash
# MAC kugisaki (jalankan di kugisaki)
ip link show eth0
# 02:42:e2:07:ec:00

nano /root/dhcpd.conf
```
```bash
# tambahkan di DALAM subnet 10.95.3.0, setelah pool long
  host kugisaki {
    hardware ethernet 02:42:e2:07:ec:00;
    fixed-address 10.95.3.99;
  }
```
```bash
# validasi konfigurasi (warning "Host declarations are global" aman)
dhcpd -t -cf /root/dhcpd.conf
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart

# kugisaki: renew
ip addr flush dev eth0
udhcpc -i eth0
ip addr show eth0
# hasil: lease of 10.95.3.99 obtained from 10.95.4.10, inet 10.95.3.99/24

# ulangi renew sekali lagi, IP harus tetap 10.95.3.99

# bukti di tengen
grep -B1 -A4 "host kugisaki" /etc/dhcp/dhcpd.conf

# bukti .99 di luar semua pool dinamis
grep -n "range" /etc/dhcp/dhcpd.conf

# kalau kugisaki hanya "broadcasting discover":
# jichi: cek relay
ps aux | grep dhcrelay
# tengen: log live (stop service dulu, start lagi setelah selesai)
service isc-dhcp-server stop
dhcpd -4 -d -cf /etc/dhcp/dhcpd.conf eth0
service isc-dhcp-server start

# hasil log tengen: DHCPDISCOVER, DHCPOFFER, DHCPREQUEST, DHCPACK on 10.95.3.99 via 10.95.3.1
```

## Soal 7

Opsi jaringan dari server: mask /24, gateway, broadcast, DNS `10.95.1.2` dan `10.95.1.3`, domain-search. Client Alpine (udhcpc BusyBox) hanya menulis baris `search` dari opsi `domain-name`, jadi `domain-name` ditambahkan di samping `domain-search`.

```bash
nano /root/dhcpd.conf
```
```bash
# subnet 10.95.2.0: ganti DNS dan tambah opsi berikut (pool1 dan pool2 tetap)
  option subnet-mask 255.255.255.0;
  option routers 10.95.2.1;
  option broadcast-address 10.95.2.255;
  option domain-name-servers 10.95.1.2, 10.95.1.3;
  option domain-name "K63.com";
  option domain-search "K63.com";

# subnet 10.95.3.0: tambah/ganti opsi berikut (pool short, long, host kugisaki tetap)
  option subnet-mask 255.255.255.0;
  option routers 10.95.3.1;
  option broadcast-address 10.95.3.255;
  option domain-name-servers 10.95.1.2, 10.95.1.3;
  option domain-name "K63.com";
  option domain-search "K63.com";
```
```bash
dhcpd -t -cf /root/dhcpd.conf
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
ps aux | grep dhcpd
grep -n "domain\|routers\|broadcast" /etc/dhcp/dhcpd.conf
```
```bash
# semua client (8 client: itadori, fushiguro, zenin, okkotsu, ino, kusakabe, miwa, kugisaki)
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0
ip addr show eth0          # IP /24 dan brd 10.95.2.255 atau 10.95.3.255
ip route                   # default via 10.95.2.1 atau 10.95.3.1
cat /etc/resolv.conf       # search K63.com, nameserver 10.95.1.2 dan 10.95.1.3
```
```bash
# tengen: cocokkan lease dengan client (tanpa duplikat)
grep -E "^lease|hardware ethernet|binding state" /var/lib/dhcp/dhcpd.leases
awk '/^lease/{ip=$2} /hardware ethernet/{gsub(";","",$3); print ip, $3}' /var/lib/dhcp/dhcpd.leases | sort -u

# hasil (7 pasang, kugisaki tidak muncul karena host reservation):
# 10.95.2.10  02:42:73:bc:7c:00   itadori
# 10.95.2.11  02:42:d5:94:24:00   fushiguro
# 10.95.2.70  02:42:0c:6b:f2:00   okkotsu
# 10.95.2.71  02:42:1c:8c:ac:00   zenin
# 10.95.3.10  02:42:16:ac:98:00   ino / kusakabe
# 10.95.3.11  02:42:91:bc:97:00   kusakabe / ino
# 10.95.3.100 02:42:aa:db:27:00   miwa
```

## Soal 8

Packet capture di tengen saat satu client meminta lease baru.

```bash
# tengen: pasang tcpdump
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt update
apt install -y tcpdump
```
```bash
# tengen: mulai capture (biarkan terbuka)
rm -f /root/dhcp-capture.pcap
tcpdump -i eth0 -n -vvv port 67 or port 68 -w /root/dhcp-capture.pcap
```
```bash
# client (itadori, Network 2): minta lease baru
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0
```
```bash
# tengen: setelah "lease obtained" di client, tekan Ctrl+C, lalu baca hasilnya
tcpdump -n -vvv -r /root/dhcp-capture.pcap | grep -E "DHCP-Message|Server-ID|Your-IP|Lease-Time|Gateway-IP"
```
```bash
# hasil blok itadori:
# Discover  Gateway-IP 10.95.2.1
# Offer     Your-IP 10.95.2.10, Server-ID 10.95.4.10, Lease-Time 464, Gateway-IP 10.95.2.1
# Request   Server-ID 10.95.4.10, Gateway-IP 10.95.2.1
# ACK       Your-IP 10.95.2.10, Server-ID 10.95.4.10, Lease-Time 464, Gateway-IP 10.95.2.1
#
# server        : 10.95.4.10
# alamat client : 10.95.2.10
# lease time    : 464 detik (sisa lease lama, server masih mengingat lease sebelumnya)
# giaddr        : 10.95.2.1 = IP eth2 jichi (interface yang menerima request Network 2)
#
# Baris "Gateway-IP 10.95.4.1" dan NACK pada capture berasal dari renew client lain
# yang ikut tertangkap, bukan dari itadori.
```

## Soal 9

Simulasi gangguan: relay dimatikan, client gagal dapat lease, relay dipulihkan.

```bash
# jichi: matikan relay
service isc-dhcp-relay stop
pkill dhcrelay
ps aux | grep dhcrelay          # tidak boleh ada proses dhcrelay
```
```bash
# client (itadori): paksa lease baru, catat gejalanya
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0
# gejala: "broadcasting discover" berulang, lalu "udhcpc failed to get a DHCP lease"
ip -br addr                     # eth0 tanpa IPv4
ip route                        # kosong
```
```bash
# jichi: pulihkan relay
service isc-dhcp-relay restart
ps aux | grep dhcrelay
# /usr/sbin/dhcrelay -q -i eth2 -i eth3 -i eth4 10.95.4.10
```
```bash
# client (itadori): ulangi, harus pulih
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0
ip -br addr                     # 10.95.2.10/24 (Pool 1)
ip route                        # default via 10.95.2.1
cat /etc/resolv.conf            # search K63.com, nameserver 10.95.1.2 dan 10.95.1.3
```
```bash
# penjelasan: DHCPDISCOVER adalah broadcast dan tidak bisa melewati router.
# Tanpa relay di jichi, paket berhenti di Network 2 dan tidak sampai ke tengen
# di Network 4. Setelah relay hidup, jichi meneruskan sebagai unicast ke 10.95.4.10.
```

## Soal 10
```bash
# tengen: autostart DHCP server (node tanpa systemd, pakai update-rc.d)
cat /etc/default/isc-dhcp-server | grep INTERFACES
cp /root/dhcpd.conf /etc/dhcp/dhcpd.conf
dhcpd -t -cf /etc/dhcp/dhcpd.conf
update-rc.d isc-dhcp-server enable
ls /etc/rc2.d | grep -i dhcp
cp /etc/dhcp/dhcpd.conf /root/dhcpd.conf.final
```
```bash
INTERFACESv4="eth0"
```
```bash
# jichi: autostart DHCP relay
cat /etc/default/isc-dhcp-relay
update-rc.d isc-dhcp-relay enable
ls /etc/rc2.d | grep -i dhcp
```
```bash
SERVERS="10.95.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
```
```bash
# jichi: /etc/network/interfaces, di blok iface eth0 (forwarding dan NAT ikut naik saat boot)
post-up sysctl -w net.ipv4.ip_forward=1
post-up iptables -t nat -C POSTROUTING -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# jichi: di blok iface eth4 (cadangan kalau update-rc.d tidak menaikkan relay)
post-up service isc-dhcp-relay restart

# tengen: di blok iface eth0 (cadangan kalau update-rc.d tidak menaikkan dhcpd)
post-up touch /var/lib/dhcp/dhcpd.leases
post-up service isc-dhcp-server restart

# cek tersimpan
grep post-up /etc/network/interfaces
```
```bash
# restart node jichi dulu, lalu tengen dari GNS3 (stop lalu start), jangan jalankan perintah manual

# tengen
service isc-dhcp-server status
ps aux | grep dhcpd
ip addr show eth0
ip route

# jichi
service isc-dhcp-relay status
ps aux | grep dhcrelay
sysctl net.ipv4.ip_forward
iptables -t nat -L POSTROUTING -n -v      # satu baris MASQUERADE
ip addr

# hasil proses setelah restart (jam START harus baru):
# /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf eth0
# /usr/sbin/dhcrelay -q -i eth2 -i eth3 -i eth4 10.95.4.10

# semua client
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0
ip addr show eth0
ip route
cat /etc/resolv.conf
ping -c 3 10.95.2.1     # client Network 2
ping -c 3 10.95.3.1     # client Network 3

# tengen
cat /var/lib/dhcp/dhcpd.leases
awk '/^lease/{ip=$2} /hardware ethernet/{gsub(";","",$3); print ip, $3}' /var/lib/dhcp/dhcpd.leases | sort -u
```
```bash
# /etc/dhcp/dhcpd.conf final (sudah termasuk perubahan soal 7)
default-lease-time 600;
max-lease-time 7200;
authoritative;

class "pool1" {
  match if (substring(hardware, 1, 6) = 02:42:73:bc:7c:00) or
           (substring(hardware, 1, 6) = 02:42:d5:94:24:00);
}

class "pool2" {
  match if (substring(hardware, 1, 6) = 02:42:1c:8c:ac:00) or
           (substring(hardware, 1, 6) = 02:42:0c:6b:f2:00);
}

class "short" {
  match if (substring(hardware, 1, 6) = 02:42:16:ac:98:00) or
           (substring(hardware, 1, 6) = 02:42:91:bc:97:00);
}

class "long" {
  match if (substring(hardware, 1, 6) = 02:42:aa:db:27:00);
}

subnet 10.95.4.0 netmask 255.255.255.0 {
}

subnet 10.95.2.0 netmask 255.255.255.0 {
  option subnet-mask 255.255.255.0;
  option routers 10.95.2.1;
  option broadcast-address 10.95.2.255;
  option domain-name-servers 10.95.1.2, 10.95.1.3;
  option domain-name "K63.com";
  option domain-search "K63.com";

  pool {
    allow members of "pool1";
    range 10.95.2.10 10.95.2.35;
  }

  pool {
    allow members of "pool2";
    range 10.95.2.70 10.95.2.95;
  }
}

subnet 10.95.3.0 netmask 255.255.255.0 {
  option subnet-mask 255.255.255.0;
  option routers 10.95.3.1;
  option broadcast-address 10.95.3.255;
  option domain-name-servers 10.95.1.2, 10.95.1.3;
  option domain-name "K63.com";
  option domain-search "K63.com";

  pool {
    allow members of "short";
    range 10.95.3.10 10.95.3.89;
    default-lease-time 1800;
    max-lease-time 1800;
  }

  pool {
    allow members of "long";
    range 10.95.3.100 10.95.3.199;
    default-lease-time 43200;
    max-lease-time 43200;
  }

  host kugisaki {
    hardware ethernet 02:42:e2:07:ec:00;
    fixed-address 10.95.3.99;
  }
}
```
