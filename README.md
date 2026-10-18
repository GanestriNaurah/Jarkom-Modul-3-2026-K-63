# Jarkom-Modul-3-2026-k-63

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

## Soal 11

### Konfigurasi dan Implementasi

#### Instalasi BIND9

Instalasi dilakukan pada server Utahime dan Shoko untuk menyediakan layanan DNS beserta utilitas pendukungnya.

**Kode pada Utahime:**

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y bind9 bind9utils dnsutils
```

**Kode pada Shoko:**

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y bind9 bind9utils dnsutils
```

Perintah tersebut mengatur DNS resolver, memperbarui daftar paket, dan memasang BIND9, utilitas administrasi DNS, serta `dig` untuk pengujian.

#### Konfigurasi DNS Master pada Utahime

Pada Utahime, dibuat file zona `/etc/bind/db.jujutsu.k63.com` yang berisi informasi domain, SOA record, NS record, dan alamat IP kedua server DNS.

**Kode file zona:**

```bash
cat > /etc/bind/db.jujutsu.k63.com << 'EOF'
$TTL 86400
@   IN  SOA  ns1.jujutsu.k63.com. admin.jujutsu.k63.com. (
        2024101501 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS   ns1.jujutsu.k63.com.
@       IN  NS   ns2.jujutsu.k63.com.

ns1     IN  A    10.95.1.2
ns2     IN  A    10.95.1.3
EOF
```

SOA record menentukan server DNS utama, alamat administrator, nomor serial, dan interval pembaruan zona. NS record mendefinisikan Utahime dan Shoko sebagai server DNS yang berwenang untuk domain tersebut. A record memetakan nama `ns1` dan `ns2` ke alamat IP masing-masing.

Selanjutnya, konfigurasi zona ditambahkan ke file `/etc/bind/named.conf.local`.

**Kode konfigurasi Master:**

```bash
cat > /etc/bind/named.conf.local << 'EOF'
zone "jujutsu.k63.com" {
    type master;
    file "/etc/bind/db.jujutsu.k63.com";
    allow-transfer { 10.95.1.3; };
    also-notify { 10.95.1.3; };
};
EOF
```

Parameter `type master` menetapkan Utahime sebagai server DNS utama. Parameter `file` menentukan lokasi file zona, sedangkan `allow-transfer` membatasi transfer zona agar hanya dapat dilakukan oleh Shoko dengan alamat IP `10.95.1.3`. Parameter `also-notify` memberi tahu Shoko ketika terdapat perubahan pada zona.

**Validasi konfigurasi:**

```bash
named-checkconf
named-checkzone jujutsu.k63.com /etc/bind/db.jujutsu.k63.com
```

Perintah tersebut digunakan untuk memeriksa sintaks konfigurasi BIND9 dan memastikan file zona memiliki format yang valid. Jika konfigurasi benar, named-checkconf tidak menampilkan pesan kesalahan, sedangkan named-checkzone menampilkan status OK.

**Menjalankan layanan DNS:**

```bash
pkill named 2>/dev/null; sleep 1
named -g &
```

Perintah ini menghentikan proses `named` yang sedang berjalan, menunggu satu detik, kemudian menjalankan BIND9 di latar belakang dengan output log diarahkan ke terminal.

#### Konfigurasi DNS Slave pada Shoko

Shoko dikonfigurasi sebagai DNS Slave agar memperoleh salinan zona dari Utahime melalui mekanisme *zone transfer*.

**Kode konfigurasi Slave:**

```bash
cat > /etc/bind/named.conf.local << 'EOF'
zone "jujutsu.k63.com" {
    type slave;
    file "/etc/bind/db.jujutsu.k63.com";
    masters { 10.95.1.2; };
};
EOF
```

Parameter `type slave` menetapkan Shoko sebagai DNS sekunder, sedangkan `masters` menentukan Utahime dengan alamat IP `10.95.1.2` sebagai sumber zona. File zona pada Shoko diperoleh melalui transfer dari Master dan disimpan oleh BIND9 sebagai salinan zona sekunder.

**Validasi dan menjalankan layanan:**

```bash
named-checkconf
pkill named 2>/dev/null; sleep 1
named -g &
```

Validasi dilakukan untuk memeriksa sintaks konfigurasi. Setelah itu, layanan BIND9 dijalankan agar Shoko dapat meminta dan menerima zona dari Utahime.

#### Pengujian DNS dari Client

Pengujian dilakukan menggunakan perintah `dig` untuk memastikan bahwa kedua server dapat memberikan jawaban atas permintaan DNS untuk domain `jujutsu.k63.com`.

**Pengujian terhadap Utahime:**

```bash
dig @10.95.1.2 jujutsu.k63.com
```

**Pengujian terhadap Shoko:**

```bash
dig @10.95.1.3 jujutsu.k63.com
```

Kedua perintah mengirimkan permintaan DNS secara langsung ke alamat IP server yang dituju. Hasil pengujian digunakan untuk memeriksa apakah domain dapat diresolusikan oleh DNS Master dan DNS Slave.

#### Pengujian Sinkronisasi Nomor Serial

Sinkronisasi diuji dengan menaikkan nomor serial pada file zona di Utahime. Nomor serial digunakan BIND9 untuk mengetahui apakah data zona Master lebih baru daripada salinan zona Slave.

**Kode pada Utahime:**

```bash
sed -i 's/2024101501/2024101502/' /etc/bind/db.jujutsu.k63.com
pkill named && sleep 1 && named -g &
```

Perintah `sed` mengubah nomor serial dari `2024101501` menjadi `2024101502`. Setelah itu, proses BIND9 dimulai kembali agar perubahan zona dimuat. Shoko dapat mengetahui perubahan melalui mekanisme pemberitahuan DNS dan pemeriksaan serial zona secara berkala.

**Kode pemeriksaan serial pada Shoko:**

```bash
dig @10.95.1.3 jujutsu.k63.com SOA
```

Perintah tersebut meminta SOA record dari Shoko. Apabila sinkronisasi berhasil, nomor serial yang ditampilkan akan berubah menjadi `2024101502`, sesuai dengan serial terbaru pada Utahime.

## Soal 12

### Konfigurasi dan Implementasi

#### Update Zone File Forward

File zona forward dibuat untuk menentukan pemetaan domain ke alamat IP, alias domain, dan informasi server DNS.

**Kode pada Utahime:**

```bash
cat > /etc/bind/db.jujutsu.k63.com << 'EOF'
$TTL 86400
@   IN  SOA  ns1.jujutsu.k63.com. admin.jujutsu.k63.com. (
        2024101503 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS   ns1.jujutsu.k63.com.
@       IN  NS   ns2.jujutsu.k63.com.

; A Record - Domain utama ke Gojo
@       IN  A    10.95.1.4

; CNAME - www merujuk ke domain utama
www     IN  CNAME jujutsu.k63.com.

; A Record untuk ns1 dan ns2
ns1     IN  A    10.95.1.2
ns2     IN  A    10.95.1.3
EOF
```

Record A pada simbol `@` mengarahkan domain utama `jujutsu.k63.com` ke alamat IP Gojo, yaitu `10.95.1.4`. Record CNAME membuat `www.jujutsu.k63.com` menjadi alias dari domain utama. Sementara itu, record NS menentukan server DNS yang berwenang mengelola zona tersebut.

#### Konfigurasi Reverse DNS (PTR)

Reverse DNS digunakan untuk mencari nama domain berdasarkan alamat IP. Pada praktikum ini, alamat IP `10.95.1.4` dipetakan kembali ke domain `jujutsu.k63.com`.

**Kode pembuatan file zona Reverse DNS:**

```bash
cat > /etc/bind/db.10.95.1 << 'EOF'
$TTL 86400
@   IN  SOA  ns1.jujutsu.k63.com. admin.jujutsu.k63.com. (
        2024101501 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS   ns1.jujutsu.k63.com.
@       IN  NS   ns2.jujutsu.k63.com.

; PTR Record - 10.95.1.4 ke domain utama
4       IN  PTR  jujutsu.k63.com.
EOF
```

Record PTR dengan angka `4` menunjukkan alamat host `10.95.1.4` pada zona reverse `1.95.10.in-addr.arpa`. Dengan demikian, alamat IP Gojo dapat dicari menggunakan permintaan Reverse DNS.

#### Update Konfigurasi Zona pada BIND9

File `/etc/bind/named.conf.local` digunakan untuk mendaftarkan zona forward dan reverse agar dapat dikelola oleh BIND9.

**Kode konfigurasi:**

```bash
cat > /etc/bind/named.conf.local << 'EOF'
zone "jujutsu.k63.com" {
    type master;
    file "/etc/bind/db.jujutsu.k63.com";
    allow-transfer { 10.95.1.3; };
    also-notify { 10.95.1.3; };
};

zone "1.95.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.10.95.1";
    allow-transfer { 10.95.1.3; };
    also-notify { 10.95.1.3; };
};
EOF
```

Zona forward mengelola pemetaan nama domain ke alamat IP, sedangkan zona reverse mengelola pemetaan alamat IP ke nama domain. Parameter `allow-transfer` mengizinkan Shoko dengan IP `10.95.1.3` menerima transfer zona. Parameter `also-notify` memberi tahu Shoko ketika terdapat perubahan zona.

#### Konfigurasi DNS Forwarder

DNS Forwarder digunakan untuk meneruskan permintaan DNS yang tidak dapat dijawab dari zona lokal ke server DNS lain. Pada konfigurasi ini, permintaan diteruskan ke gateway internet `192.168.122.1`.

**Kode pada Utahime:**

```bash
cat > /etc/bind/named.conf.options << 'EOF'
options {
    directory "/var/cache/bind";

    forwarders {
        192.168.122.1;
    };

    recursion yes;
    allow-query { any; };
};
EOF
```

Parameter `forwarders` menentukan server DNS tujuan penerusan permintaan. Parameter `recursion yes` mengaktifkan resolusi rekursif, sedangkan `allow-query { any; }` mengizinkan permintaan DNS dari seluruh alamat IP. Dalam penerapan nyata, akses ini sebaiknya dibatasi ke jaringan praktikum untuk menghindari penyalahgunaan server DNS.

#### Validasi dan Restart BIND9

Sebelum layanan dijalankan kembali, konfigurasi dan kedua file zona diperiksa untuk memastikan tidak terdapat kesalahan sintaks.

**Kode validasi:**

```bash
named-checkconf
named-checkzone jujutsu.k63.com /etc/bind/db.jujutsu.k63.com
named-checkzone 1.95.10.in-addr.arpa /etc/bind/db.10.95.1
```

**Kode menjalankan ulang layanan:**

```bash
pkill named
sleep 1
named -g &
```

Perintah validasi memeriksa konfigurasi BIND9 dan integritas file zona. Setelah pemeriksaan selesai, proses `named` dijalankan kembali agar perubahan konfigurasi diterapkan.

### Pengujian dan Hasil Output

Output berikut merupakan contoh hasil yang diharapkan, bukan hasil eksekusi aktual.

#### Pengujian A Record

**Perintah:**

```bash
dig @10.95.1.2 jujutsu.k63.com
```

Status `NOERROR` menunjukkan bahwa permintaan berhasil dijawab. Record A menunjukkan bahwa domain `jujutsu.k63.com` diarahkan ke alamat IP Gojo, yaitu `10.95.1.4`.

#### Pengujian CNAME

**Perintah:**

```bash
dig @10.95.1.2 www.jujutsu.k63.com
```

Output menunjukkan bahwa `www.jujutsu.k63.com` merupakan alias dari `jujutsu.k63.com`. Server juga dapat menampilkan record A tujuan sehingga client memperoleh alamat IP `10.95.1.4`.

#### Pengujian Reverse DNS (PTR)

**Perintah:**

```bash
dig @10.95.1.2 -x 10.95.1.4
```

Hasil menunjukkan bahwa alamat IP `10.95.1.4` berhasil dipetakan kembali ke domain `jujutsu.k63.com` melalui record PTR.

#### Pengujian DNS Forwarder

**Perintah:**

```bash
dig @10.95.1.2 google.com
```

Status `NOERROR` dan adanya jawaban A menunjukkan bahwa resolusi domain eksternal berhasil melalui server DNS yang digunakan.

#### Pengujian Koneksi Internet dengan Ping

**Perintah:**

```bash
ping -c 4 google.com
```

Hasil menunjukkan client berhasil menerjemahkan nama domain `google.com` menjadi alamat IP dan menerima balasan ICMP. Namun, ping dapat gagal meskipun DNS berfungsi apabila koneksi jaringan bermasalah atau paket ICMP diblokir oleh tujuan maupun jaringan perantara.

## Soal 13

Replikasi MariaDB: yaga (`10.95.4.9`) sebagai master, inumaki (`10.95.4.8`) sebagai slave, database `db_shibuya_k63`.

```bash
# yaga DAN inumaki: install MariaDB
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y mariadb-server mariadb-client
```
```bash
# yaga (master) - 10.95.4.9

# buat direktori log dan file index
mkdir -p /var/log/mysql
touch /var/log/mysql/mysql-bin.index
chown mysql:mysql /var/log/mysql
chown mysql:mysql /var/log/mysql/mysql-bin.index

# edit konfigurasi MariaDB
cat > /etc/mysql/mariadb.conf.d/50-server.cnf << 'EOF'
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin.log
binlog_do_db = db_shibuya_k63
bind-address = 0.0.0.0
EOF

# restart MariaDB
service mariadb restart

# buat database dan tabel
mysql -u root << 'EOF'
CREATE DATABASE IF NOT EXISTS db_shibuya_k63;
USE db_shibuya_k63;
CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_ip VARCHAR(45),
    endpoint VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
FLUSH PRIVILEGES;
EOF

# buat user replication
mysql -u root << 'EOF'
CREATE USER IF NOT EXISTS 'repl'@'10.95.4.%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.95.4.%';
FLUSH PRIVILEGES;
EOF

# grant remote access untuk Network 4
mysql -u root << 'EOF'
GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.95.4.%' IDENTIFIED BY 'root_password';
FLUSH PRIVILEGES;
EOF

# cek status master
mysql -u root -e "SHOW MASTER STATUS;"
```
```bash
# inumaki (slave) - 10.95.4.8

# buat direktori log
mkdir -p /var/log/mysql
chown mysql:mysql /var/log/mysql

# edit konfigurasi MariaDB
cat > /etc/mysql/mariadb.conf.d/50-server.cnf << 'EOF'
[mysqld]
server-id = 2
bind-address = 0.0.0.0
EOF

# restart MariaDB
service mariadb restart

# buat database dan tabel (untuk menghindari error replikasi)
mysql -u root << 'EOF'
CREATE DATABASE IF NOT EXISTS db_shibuya_k63;
USE db_shibuya_k63;
CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_ip VARCHAR(45),
    endpoint VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
FLUSH PRIVILEGES;
EOF

# hubungkan slave ke master
mysql -u root << 'EOF'
CHANGE MASTER TO
    MASTER_HOST='10.95.4.9',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=940;
START SLAVE;
EOF

# cek status slave
mysql -u root -e "SHOW SLAVE STATUS\G"
```
```bash
# verifikasi replikasi

# yaga (master): masukkan data
mysql -u root db_shibuya_k63 -e "INSERT INTO request_logs (client_ip, endpoint) VALUES ('10.95.2.10', '/api/test-replication');"

# inumaki (slave): cek data
mysql -u root db_shibuya_k63 -e "SELECT * FROM request_logs;"
```
```bash
# verifikasi remote access dari node lain di Network 4 (misal tengen, geto, worker)
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y mariadb-client

# query dari node lain
mysql -h 10.95.4.9 -u root -proot_password -e "SELECT * FROM db_shibuya_k63.request_logs;"
```

## Soal 14

Web server worker: frontend (maki, panda) dan API (nanami, todo, kirara) dengan Nginx + PHP-FPM. Frontend hanya boleh diakses dari gojo, API hanya dari geto.

```bash
# gojo: fix IP (sebelumnya bentrok dengan shoko 10.95.1.3)
ip addr del 10.95.1.3/24 dev eth0
ip addr add 10.95.1.4/24 dev eth0
ip route add default via 10.95.1.1
cat > /etc/network/interfaces << 'ENDOFFILE'
 auto eth0
 iface eth0 inet static
     address 10.95.1.4
     netmask 255.255.255.0
     gateway 10.95.1.1
     dns-nameservers 192.168.122.1
ENDOFFILE
ip addr show eth0 | grep inet    # harus 10.95.1.4/24
```
```bash
# semua worker (maki, panda, nanami, todo, kirara): install Nginx + PHP-FPM
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx php-fpm php-mysql

# start php-fpm (binary di trixie = php-fpm8.4)
/usr/sbin/php-fpm8.4 -D
```
```bash
# API worker (nanami, todo, kirara): direktori dan file API
mkdir -p /var/www/api
cat > /var/www/api/index.php << 'ENDOFFILE'
<?php
header('Content-Type: application/json');

$servername = "10.95.4.9";
$username = "root";
$password = "root_password";
$dbname = "db_shibuya_k63";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die(json_encode(["error" => "Connection failed"]));
}

$client_ip = $_SERVER['REMOTE_ADDR'];
$endpoint = $_SERVER['REQUEST_URI'];

$stmt = $conn->prepare("INSERT INTO request_logs (client_ip, endpoint) VALUES (?, ?)");
$stmt->bind_param("ss", $client_ip, $endpoint);
$stmt->execute();
$stmt->close();

$result = $conn->query("SELECT * FROM request_logs ORDER BY id DESC LIMIT 5");
$logs = [];
while ($row = $result->fetch_assoc()) {
    $logs[] = $row;
}

$conn->close();

echo json_encode([
    "status" => "success",
    "hostname" => gethostname(),
    "client_ip" => $client_ip,
    "recent_logs" => $logs
]);
ENDOFFILE

# validasi syntax
php -l /var/www/api/index.php
```
```bash
# API worker (nanami, todo, kirara): virtual host API (HANYA dari geto 10.95.4.2)
cat > /etc/nginx/sites-available/api << 'ENDOFFILE'
server {
    listen 80;
    server_name api.jujutsu.k63.com;
    root /var/www/api;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    allow 10.95.4.2;
    deny all;
}
ENDOFFILE

ln -sf /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
```
```bash
# frontend worker (maki, panda): direktori dan file PHP
mkdir -p /var/www/frontend
cat > /var/www/frontend/index.php << 'ENDOFFILE'
<?php
echo "<h1>Frontend Worker</h1>";
echo "<p>Hostname: " . gethostname() . "</p>";
ENDOFFILE

php -l /var/www/frontend/index.php
```
```bash
# frontend worker (maki, panda): virtual host frontend (HANYA dari gojo 10.95.1.4)
cat > /etc/nginx/sites-available/frontend << 'ENDOFFILE'
server {
    listen 80;
    server_name jujutsu.k63.com www.jujutsu.k63.com;
    root /var/www/frontend;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    allow 10.95.1.4;
    deny all;
}
ENDOFFILE

ln -sf /etc/nginx/sites-available/frontend /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
```
```bash
# verifikasi

# akses langsung dari worker (HARUS 403)
curl http://localhost/
curl http://10.95.4.3/     # dari maki
curl http://10.95.4.5/     # dari nanami

# geto ke API worker (HARUS BERHASIL)
curl http://10.95.4.5/     # nanami
curl http://10.95.4.6/     # todo
curl http://10.95.4.7/     # kirara

# gojo ke frontend (HARUS BERHASIL, pastikan IP gojo = 10.95.1.4)
curl http://10.95.4.3/     # maki
curl http://10.95.4.4/     # panda

# geto ke frontend worker (HARUS 403 - tidak diizinkan)
curl http://10.95.4.3/     # harus 403
```

## Soal 15

Load balancer API di geto dengan Nginx upstream, weight 3:1 untuk nanami dan todo. Semua perintah di geto.

```bash
# geto: install Nginx dan curl
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx curl
```
```bash
# geto: upstream dan virtual host
cat > /etc/nginx/sites-available/lb << 'ENDOFFILE'
upstream api_cluster {
    server 10.95.4.5 weight=3;
    server 10.95.4.6 weight=1;
}

server {
    listen 80;
    server_name geto.jujutsu.k63.com;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE
```
```bash
# geto: aktifkan dan restart
ln -sf /etc/nginx/sites-available/lb /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
```
```bash
# geto: verifikasi distribusi beban

# satu request berhasil
curl http://localhost/

# loop 20 request - cek rasio
for i in $(seq 1 20); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c

# loop 40 request - hasil lebih akurat
for i in $(seq 1 40); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```
```bash
# geto: verifikasi tambahan

# cek nginx listening
ss -tlnp | grep :80

# cek config upstream
nginx -T | grep -A5 upstream

# cek akses dari node lain di Network 4
curl http://10.95.4.2/
```

## Soal 16

Reverse proxy di gojo dengan path-based routing: `/` ke frontend (maki, panda) secara round robin, `/api` ke geto.

```bash
# gojo: fix IP (bentrok dengan shoko 10.95.1.3)
ip addr del 10.95.1.3/24 dev eth0
ip addr add 10.95.1.4/24 dev eth0
ip route add default via 10.95.1.1

# verifikasi
ip addr show eth0 | grep inet    # HARUS 10.95.1.4/24

# konfigurasi permanen
cat > /etc/network/interfaces << 'ENDOFFILE'
auto eth0
iface eth0 inet static
    address 10.95.1.4
    netmask 255.255.255.0
    gateway 10.95.1.1
    dns-nameservers 192.168.122.1
ENDOFFILE
```
```bash
# gojo: install Nginx
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx
```
```bash
# nanami, todo, DAN kirara: fix API worker supaya path /api tidak 404
# (prasyarat: tanpa ini, request /api menghasilkan 404)
cat > /etc/nginx/sites-available/api << 'ENDOFFILE'
server {
    listen 80;
    server_name api.jujutsu.k63.com;
    root /var/www/api;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    allow 10.95.4.2;
    deny all;
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# gojo: path-based routing
cat > /etc/nginx/sites-available/gojo << 'ENDOFFILE'
upstream frontend_cluster {
    server 10.95.4.3;
    server 10.95.4.4;
}

server {
    listen 80;
    server_name jujutsu.k63.com www.jujutsu.k63.com;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api {
        proxy_pass http://10.95.4.2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE

ln -sf /etc/nginx/sites-available/gojo /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
service nginx restart
```
```bash
# geto: restart (upstream api_cluster sudah ada dari soal 15)
service nginx restart
```
```bash
# verifikasi

# gojo: cek round robin
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o 'Hostname: [a-z]*'; done | sort | uniq -c

# gojo: hasil lebih akurat
for i in $(seq 1 100); do curl -s http://localhost/ | grep -o 'Hostname: [a-z]*'; done | sort | uniq -c

# gojo: cek /api
curl http://localhost/api

# client Divisi A/B (zenin / okkotsu / kugisaki)
echo "nameserver 10.95.1.2" > /etc/resolv.conf
curl http://jujutsu.k63.com/
curl http://www.jujutsu.k63.com/
curl http://jujutsu.k63.com/api
```

## Soal 17

Meneruskan IP client asli lewat header `X-Forwarded-For` dari gojo, geto, sampai API worker, lalu disimpan ke database.

```bash
# ino (Network 3): set IP manual kalau belum dapat DHCP
ip addr add 10.95.3.2/24 dev eth0
ip link set eth0 up
ip route add default via 10.95.3.1
echo "nameserver 10.95.1.2" > /etc/resolv.conf
ip addr show eth0 | grep inet    # HARUS 10.95.3.2/24

# zenin (Network 2): sudah DHCP, cek saja
ip addr show eth0 | grep inet    # HARUS 10.95.2.4
```
```bash
# gojo: teruskan header client IP
cat > /etc/nginx/sites-available/gojo << 'ENDOFFILE'
upstream frontend_cluster {
    server 10.95.4.3;
    server 10.95.4.4;
}

server {
    listen 80;
    server_name jujutsu.k63.com www.jujutsu.k63.com;

    # Percayai informasi IP dari proxy terpercaya
    set_real_ip_from 10.95.0.0/16;
    set_real_ip_from 192.168.122.0/24;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api {
        proxy_pass http://10.95.4.2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# geto: hanya percayai proxy terpercaya (gojo)
cat > /etc/nginx/sites-available/lb << 'ENDOFFILE'
upstream api_cluster {
    server 10.95.4.5 weight=3;
    server 10.95.4.6 weight=1;
}

server {
    listen 80;
    server_name geto.jujutsu.k63.com;

    # HANYA gojo yang dipercayai sebagai proxy
    set_real_ip_from 10.95.1.4;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# nanami, todo, DAN kirara: simpan IP terverifikasi
cat > /var/www/api/index.php << 'ENDOFFILE'
<?php
header('Content-Type: application/json');

$servername = "10.95.4.9";
$username = "root";
$password = "root_password";
$dbname = "db_shibuya_k63";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die(json_encode(["error" => "Connection failed"]));
}

// Daftar proxy terpercaya - IP ini TIDAK boleh dianggap client
$trusted_proxy = array("10.95.4.2", "10.95.1.4");

$client_ip = $_SERVER['REMOTE_ADDR'];

// Ambil IP client asli dari X-Forwarded-For, lewati proxy terpercaya
if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $xff_list = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
    foreach ($xff_list as $ip) {
        $ip = trim($ip);
        if (filter_var($ip, FILTER_VALIDATE_IP) && !in_array($ip, $trusted_proxy)) {
            $client_ip = $ip;
            break;
        }
    }
}

$endpoint = $_SERVER['REQUEST_URI'];

$stmt = $conn->prepare("INSERT INTO request_logs (client_ip, endpoint) VALUES (?, ?)");
$stmt->bind_param("ss", $client_ip, $endpoint);
$stmt->execute();
$stmt->close();

$result = $conn->query("SELECT * FROM request_logs ORDER BY id DESC LIMIT 5");
$logs = [];
while ($row = $result->fetch_assoc()) {
    $logs[] = $row;
}

$conn->close();

echo json_encode([
    "status" => "success",
    "hostname" => gethostname(),
    "client_ip" => $client_ip,
    "recent_logs" => $logs
]);
ENDOFFILE

php -l /var/www/api/index.php
```
```bash
# nanami, todo, DAN kirara: pastikan config nginx API sudah handle /api (soal 16)
cat > /etc/nginx/sites-available/api << 'ENDOFFILE'
server {
    listen 80;
    server_name api.jujutsu.k63.com;
    root /var/www/api;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    allow 10.95.4.2;
    deny all;
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# verifikasi dari client Network 2 dan 3

# zenin (Network 2)
echo "nameserver 10.95.1.2" > /etc/resolv.conf
curl http://jujutsu.k63.com/api
# HARUS: "client_ip":"10.95.2.4"

# ino (Network 3)
echo "nameserver 10.95.1.2" > /etc/resolv.conf
curl http://jujutsu.k63.com/api
# HARUS: "client_ip":"10.95.3.2"
```
```bash
# yaga: cek database
mysql -h 10.95.4.9 -u root -proot_password db_shibuya_k63 -e "SELECT * FROM request_logs ORDER BY id DESC LIMIT 10;"
```

## Soal 18

Failover: kirara dijadikan `backup` di upstream geto, aktif hanya saat nanami dan todo mati.

```bash
# kirara: pastikan API berjalan (prasyarat: kirara sudah dikonfigurasi di soal 14)
ip addr add 10.95.4.7/24 dev eth0 2>/dev/null
ip link set eth0 up
ip route add default via 10.95.4.1 2>/dev/null

# cek status
ip addr show eth0 | grep inet
ps aux | grep nginx | head -2
ps aux | grep php-fpm | head -2
ls -la /run/php/php8.4-fpm.sock
ls -la /var/www/api/index.php

# kalau php-fpm mati
/usr/sbin/php-fpm8.4 -D
service nginx restart

# cek config kirara sudah allow geto
grep -n "allow\|deny" /etc/nginx/sites-available/api
```
```bash
# geto: upstream dengan backup dan health check
cat > /etc/nginx/sites-available/lb << 'ENDOFFILE'
upstream api_cluster {
    server 10.95.4.5 weight=3 max_fails=2 fail_timeout=5s;
    server 10.95.4.6 weight=1 max_fails=2 fail_timeout=5s;
    server 10.95.4.7 backup;
}

server {
    listen 80;
    server_name geto.jujutsu.k63.com;

    set_real_ip_from 10.95.1.4;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# geto: baseline sebelum gangguan

# test kirara langsung (hostname: kirara)
curl http://10.95.4.7/

# distribusi normal (harus nanami + todo saja)
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```
```bash
# simulasi gangguan

# nanami
service nginx stop
ps aux | grep nginx | grep -v grep

# todo
service nginx stop
ps aux | grep nginx | grep -v grep
```
```bash
# geto: buktikan failover ke kirara (tunggu ~5 detik / fail_timeout dulu)
sleep 5
curl http://localhost/
curl http://localhost/api

# loop - semua harus kirara
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c

# dari client (zenin)
curl http://jujutsu.k63.com/api
for i in $(seq 1 10); do curl -s http://jujutsu.k63.com/api | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```
```bash
# pulihkan layanan

# nanami
service nginx start

# todo
service nginx start

# geto: harus kembali ke nanami/todo
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

## Soal 19

Caching halaman frontend dan rate limiting API di gojo.

```bash
# gojo: direktori cache
mkdir -p /var/cache/nginx/frontend
chown -R www-data:www-data /var/cache/nginx
```
```bash
# gojo: konfigurasi
# catatan: proxy_cache_path dan limit_req_zone HARUS di luar blok server{}
cat > /etc/nginx/sites-available/gojo << 'ENDOFFILE'
proxy_cache_path /var/cache/nginx/frontend levels=1:2 keys_zone=frontend_cache:10m max_size=1g inactive=60m;

limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

upstream frontend_cluster {
    server 10.95.4.3;
    server 10.95.4.4;
}

server {
    listen 80;
    server_name jujutsu.k63.com www.jujutsu.k63.com;

    set_real_ip_from 10.95.0.0/16;
    set_real_ip_from 192.168.122.0/24;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    # Halaman web frontend - DENGAN CACHING
    location / {
        proxy_cache frontend_cache;
        proxy_cache_valid 200 10m;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # API - DENGAN RATE LIMITING
    location /api {
        limit_req zone=api_limit burst=5 nodelay;
        limit_req_status 429;
        proxy_pass http://10.95.4.2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
ENDOFFILE

nginx -t && service nginx restart
```
```bash
# verifikasi caching (MISS -> HIT)

# gojo: hapus cache dulu
rm -rf /var/cache/nginx/frontend/*

# client (zenin / ino / kusakabe)
echo "nameserver 10.95.1.2" > /etc/resolv.conf

curl -I http://jujutsu.k63.com/
curl -I http://jujutsu.k63.com/
curl -I http://jujutsu.k63.com/

# HARUS:
# Request 1 -> X-Cache-Status: MISS
# Request 2 -> X-Cache-Status: HIT
# Request 3 -> X-Cache-Status: HIT
```
```bash
# verifikasi rate limiting (client zenin)
for i in $(seq 1 50); do curl -s -o /dev/null -w "%{http_code} " http://jujutsu.k63.com/api; done; echo

# HARUS: muncul beberapa 200 lalu banyak 429
```
```bash
# gojo: utility cek dan clear cache

# cek isi cache
ls -la /var/cache/nginx/frontend/

# cek ukuran cache
du -sh /var/cache/nginx/frontend/

# clear cache (untuk test ulang MISS)
rm -rf /var/cache/nginx/frontend/*

# cek log error rate limit
tail -20 /var/log/nginx/error.log
```

## Soal 20

Benchmark dengan Apache Benchmark (`ab`) dari kusakabe ke gojo, lalu analisis access log.

```bash
# kusakabe (Alpine): persiapan

# set DNS ke utahime
echo "nameserver 10.95.1.2" > /etc/resolv.conf

# cek IP address
ip addr show eth0 | grep inet
# HARUS: inet 10.95.3.3/24

# kalau belum ada IP, set manual
ip addr add 10.95.3.3/24 dev eth0
ip link set eth0 up
ip route add default via 10.95.3.1

# install Apache Benchmark
apk update
apk add apache2-utils

# verifikasi
ab -V
# Output: This is ApacheBench, Version 2.3
```
```bash
# kusakabe: benchmark baseline - halaman web "/"
ab -n 600 -c 30 http://jujutsu.k63.com/
```
```bash
# kusakabe: benchmark dengan rate limiting aktif - "/api"
ab -n 600 -c 30 http://jujutsu.k63.com/api
```
```bash
# kusakabe: benchmark via IP (alternatif / bukti tambahan)
ab -n 600 -c 30 http://10.95.1.4/
ab -n 600 -c 30 http://10.95.1.4/api
```
```bash
# gojo: analisis access log

# hitung response code dari benchmark terakhir
tail -1200 /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# lihat request yang DITOLAK rate limit
grep " 429 " /var/log/nginx/access.log | tail -10

# cek kegagalan APLIKASI (5xx)
grep -E " 5[0-9][0-9] " /var/log/nginx/access.log | tail -10

# cek request ke "/" (harus semua 200)
grep "GET / HTTP" /var/log/nginx/access.log | tail -10 | awk '{print $9}' | sort | uniq -c

# cek kegagalan koneksi di error log
tail -20 /var/log/nginx/error.log
```
