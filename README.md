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

### Konfigurasi dan Implementasi

#### Instalasi MariaDB

MariaDB dipasang pada Yaga dan Inumaki sebagai layanan database yang mendukung replikasi.

**Kode pada Yaga dan Inumaki:**

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y mariadb-server mariadb-client
```

Perintah tersebut mengatur DNS resolver, memperbarui daftar paket, dan memasang server serta client MariaDB.

#### Konfigurasi DB Master pada Yaga

Yaga menggunakan alamat IP `10.95.4.9` dan bertugas menyimpan data utama serta mencatat perubahan database melalui binary log.

**Membuat direktori log:**

```bash
mkdir -p /var/log/mysql
touch /var/log/mysql/mysql-bin.index
chown mysql:mysql /var/log/mysql
chown mysql:mysql /var/log/mysql/mysql-bin.index
```

Direktori tersebut digunakan untuk menyimpan file binary log yang mencatat perubahan data agar dapat dikirim dan diterapkan oleh Slave.

**Konfigurasi MariaDB Master:**

```bash
cat > /etc/mysql/mariadb.conf.d/50-server.cnf << 'EOF'
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin.log
binlog_do_db = db_shibuya_k63
bind-address = 0.0.0.0
EOF
```

Parameter `server-id = 1` memberikan identitas unik kepada Master. Parameter `log-bin` mengaktifkan binary logging, sedangkan `binlog_do_db` membatasi database yang dicatat untuk replikasi. `bind-address = 0.0.0.0` memungkinkan MariaDB menerima koneksi melalui alamat jaringan yang tersedia, dengan tetap memerlukan pengaturan hak akses dan firewall yang sesuai.

**Menjalankan ulang MariaDB:**

```bash
service mariadb restart
```

**Membuat database dan tabel:**

```bash
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
```

Database `db_shibuya_k63` berfungsi sebagai penyimpanan log aktivitas. Tabel `request_logs` memiliki kolom `id` sebagai primary key otomatis, `client_ip` untuk menyimpan alamat IP client, `endpoint` untuk mencatat endpoint yang diakses, dan `timestamp` untuk mencatat waktu pembuatan data.

**Membuat user replikasi:**

```bash
mysql -u root << 'EOF'
CREATE USER IF NOT EXISTS 'repl'@'10.95.4.%'
IDENTIFIED BY 'repl_password';

GRANT REPLICATION SLAVE ON *.*
TO 'repl'@'10.95.4.%';

FLUSH PRIVILEGES;
EOF
```

User `repl` diberikan hak replikasi untuk koneksi dari alamat IP yang sesuai dengan pola `10.95.4.%`. Akun ini digunakan oleh Inumaki untuk mengambil perubahan data dari Master.

**Memberikan akses remote untuk Network 4:**

```bash
mysql -u root << 'EOF'
GRANT ALL PRIVILEGES ON *.*
TO 'root'@'10.95.4.%'
IDENTIFIED BY 'root_password';

FLUSH PRIVILEGES;
EOF
```

Perintah tersebut memberikan hak akses database kepada akun remote `root` dari jaringan `10.95.4.0/24` sesuai pola host yang digunakan. Untuk penerapan yang lebih aman, sebaiknya dibuat akun khusus dengan hak akses minimum yang diperlukan daripada memberikan seluruh hak istimewa kepada akun `root`.

**Memeriksa status Master:**

```bash
mysql -u root -e "SHOW MASTER STATUS;"
```

Perintah `SHOW MASTER STATUS` digunakan untuk melihat nama file binary log dan posisi pencatatan terakhir yang menjadi acuan konfigurasi replikasi Slave.

#### Konfigurasi DB Slave pada Inumaki

Inumaki menggunakan alamat IP `10.95.4.8` dan bertugas menerima serta menerapkan perubahan data dari Yaga.

**Membuat direktori log:**

```bash
mkdir -p /var/log/mysql
chown mysql:mysql /var/log/mysql
```

**Konfigurasi MariaDB Slave:**

```bash
cat > /etc/mysql/mariadb.conf.d/50-server.cnf << 'EOF'
[mysqld]
server-id = 2
bind-address = 0.0.0.0
EOF
```

Parameter `server-id = 2` memberikan identitas unik kepada Slave agar tidak sama dengan Master. Parameter `bind-address` memungkinkan koneksi jaringan ke layanan MariaDB sesuai dengan konfigurasi firewall dan hak akses.

**Menjalankan ulang MariaDB:**

```bash
service mariadb restart
```

**Membuat database dan tabel:**

```bash
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
```

Database dan tabel disiapkan agar struktur data tersedia pada Slave. Dalam penerapan replikasi, struktur dan isi awal database harus konsisten dengan Master agar tidak terjadi konflik ketika perubahan mulai diterapkan.

**Menghubungkan Slave ke Master:**

```bash
mysql -u root << 'EOF'
CHANGE MASTER TO
    MASTER_HOST='10.95.4.9',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=940;

START SLAVE;
EOF
```

Perintah `CHANGE MASTER TO` menetapkan alamat server Master, akun replikasi, nama file binary log, dan posisi log awal. Nilai `MASTER_LOG_FILE` dan `MASTER_LOG_POS` harus disesuaikan dengan hasil `SHOW MASTER STATUS` dari Yaga, bukan diasumsikan selalu sama.

Perintah `START SLAVE` memulai proses replikasi sehingga Inumaki dapat mengambil perubahan dari binary log Master dan menerapkannya ke database lokal.

**Memeriksa status Slave:**

```bash
mysql -u root -e "SHOW SLAVE STATUS\G"
```

Status Slave digunakan untuk memeriksa koneksi ke Master, proses pembacaan binary log, dan proses penerapan perubahan pada database.

#### Verifikasi Replikasi Data

Pengujian dilakukan dengan memasukkan data baru ke tabel `request_logs` pada Yaga, kemudian memeriksa apakah data tersebut muncul pada Inumaki.

**Memasukkan data pada Master (Yaga):**

```bash
mysql -u root db_shibuya_k63 -e \
"INSERT INTO request_logs (client_ip, endpoint)
VALUES ('10.95.2.10', '/api/test-replication');"
```

Perintah tersebut menambahkan satu catatan aktivitas dengan alamat IP client `10.95.2.10` dan endpoint `/api/test-replication`.

**Memeriksa data pada Slave (Inumaki):**

```bash
mysql -u root db_shibuya_k63 -e \
"SELECT * FROM request_logs;"
```

Jika replikasi berjalan dengan baik, data yang dimasukkan pada Yaga akan muncul pada hasil query di Inumaki tanpa perlu melakukan INSERT ulang secara manual.

#### Verifikasi Akses Remote dari Network 4

Pengujian dilakukan dari node lain di Network 4, misalnya Tengen, Geto, atau worker, untuk memastikan koneksi remote ke database Master berfungsi.

**Instalasi client pada node penguji:**

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
apt-get update
apt-get install -y mariadb-client
```

**Melakukan query ke Master:**

```bash
mysql -h 10.95.4.9 -u root -proot_password \
-e "SELECT * FROM db_shibuya_k63.request_logs;"
```

Perintah tersebut menghubungkan client ke MariaDB pada Yaga menggunakan alamat IP `10.95.4.9` dan menampilkan data dari tabel `request_logs`. Keberhasilan koneksi menunjukkan bahwa akses remote dan jalur jaringan berfungsi sesuai konfigurasi.

## Soal 14

### Konfigurasi dan Implementasi

#### Perbaikan Alamat IP Gojo

Gojo dikonfigurasikan menggunakan alamat IP `10.95.1.4` sebagai jalur proxy web menuju frontend worker.

**Kode pada Gojo:**

```bash
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

ip addr show eth0 | grep inet
```

Perintah tersebut mengubah alamat IP Gojo menjadi `10.95.1.4/24`, menetapkan gateway, dan menyimpan konfigurasi jaringan statis agar alamat tetap digunakan setelah konfigurasi jaringan diterapkan.

#### Instalasi Nginx dan PHP-FPM

Instalasi dilakukan pada Maki, Panda, Nanami, Todo, dan Kirara. Nginx bertugas menerima permintaan HTTP, sedangkan PHP-FPM menjalankan kode PHP. Paket `php-mysql` digunakan agar aplikasi PHP dapat terhubung ke MariaDB.

**Kode instalasi pada seluruh worker:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx php-fpm php-mysql
```

**Menjalankan PHP-FPM:**

```bash
/usr/sbin/php-fpm8.4 -D
```

Perintah tersebut menjalankan proses PHP-FPM versi 8.4 di latar belakang sesuai lingkungan praktikum.

#### Konfigurasi API Worker pada Nanami, Todo, dan Kirara

API worker bertugas menerima permintaan HTTP, mencatat alamat IP client dan endpoint yang diakses ke database Yaga, serta mengembalikan respons dalam format JSON.

**Membuat file API:**

```bash
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
```

Aplikasi menggunakan koneksi MariaDB ke Yaga dengan alamat IP `10.95.4.9`. Setiap request yang berhasil diproses akan dicatat ke tabel `request_logs`, kemudian API mengembalikan status, hostname worker, alamat IP sumber, dan lima log terbaru dalam format JSON.

**Validasi sintaks PHP:**

```bash
php -l /var/www/api/index.php
```

Perintah tersebut memeriksa sintaks file PHP sebelum aplikasi dijalankan.

#### Konfigurasi Virtual Host API

Virtual host API dibuat agar worker merespons domain `api.jujutsu.k63.com`. Akses dibatasi hanya untuk alamat IP Geto, yaitu `10.95.4.2`.

**Kode konfigurasi pada Nanami, Todo, dan Kirara:**

```bash
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

Parameter `server_name` menentukan domain layanan API. Direktif `fastcgi_pass` meneruskan pemrosesan file PHP ke PHP-FPM. Direktif `allow` dan `deny` membatasi akses HTTP berdasarkan alamat IP sumber sehingga permintaan dari alamat selain Geto ditolak.

#### Konfigurasi Frontend Worker pada Maki dan Panda

Maki dan Panda menyediakan halaman frontend sederhana yang menampilkan hostname masing-masing worker.

**Membuat file frontend:**

```bash
mkdir -p /var/www/frontend

cat > /var/www/frontend/index.php << 'ENDOFFILE'
<?php
echo "<h1>Frontend Worker</h1>";
echo "<p>Hostname: " . gethostname() . "</p>";
ENDOFFILE
```

**Validasi sintaks PHP:**

```bash
php -l /var/www/frontend/index.php
```

File tersebut menggunakan fungsi `gethostname()` untuk menampilkan identitas hostname dari mesin yang menjalankan aplikasi.

#### Konfigurasi Virtual Host Frontend

Virtual host frontend menggunakan domain `jujutsu.k63.com` dan `www.jujutsu.k63.com`. Akses dibatasi hanya untuk Gojo dengan alamat IP `10.95.1.4`.

**Kode konfigurasi pada Maki dan Panda:**

```bash
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

Direktif `server_name` menentukan domain yang dilayani oleh frontend worker. Pembatasan akses menggunakan `allow 10.95.1.4` dan `deny all` memastikan bahwa hanya permintaan dari Gojo yang diizinkan oleh aturan Nginx tersebut.

### Pengujian 

#### Pengujian Akses Langsung ke Worker

Pengujian dilakukan dari client yang tidak termasuk alamat IP yang diizinkan.

**Perintah pengujian:**

```bash
curl http://localhost/
curl -i http://10.95.4.3/
curl -i http://10.95.4.5/
```

Status `403 Forbidden` menunjukkan bahwa permintaan ditolak oleh aturan pembatasan akses Nginx. Hasil aktual bergantung pada alamat IP sumber yang digunakan saat pengujian.

#### Pengujian Akses API melalui Geto

Pengujian dilakukan dari Geto dengan alamat IP `10.95.4.2` untuk memastikan bahwa jalur akses API diizinkan.

**Perintah pada Geto:**

```bash
curl http://10.95.4.5/     # nanami
curl http://10.95.4.6/     # todo
curl http://10.95.4.7/     # kirara
```

Respons menunjukkan API worker berhasil memproses permintaan, mencatat aktivitas ke database, dan mengembalikan data dalam format JSON. Nilai aktual hostname, ID, waktu, dan isi log bergantung pada hasil eksekusi.

#### Pengujian Akses Frontend melalui Gojo

Pengujian dilakukan dari Gojo untuk memastikan bahwa akses ke Maki dan Panda diizinkan.

**Perintah pada Gojo:**

```bash
curl http://10.95.4.3/     # maki
curl http://10.95.4.4/     # panda
```

Hasil menunjukkan frontend worker dapat diakses melalui jalur yang diizinkan dan menampilkan hostname masing-masing mesin.

#### Pengujian Akses Frontend dari Geto

Pengujian ini dilakukan untuk memastikan bahwa Geto tidak memperoleh akses langsung ke frontend worker.

**Perintah pada Geto:**

```bash
curl -i http://10.95.4.3/
```

Status `403 Forbidden` menunjukkan bahwa permintaan ditolak karena alamat IP Geto tidak termasuk alamat yang diizinkan untuk mengakses frontend worker.

## Soal 15

### Konfigurasi dan Implementasi

#### Instalasi Nginx dan Curl

Instalasi dilakukan pada Geto. Nginx digunakan sebagai Load Balancer, sedangkan Curl digunakan untuk mengirimkan request HTTP dan menguji distribusi beban.

**Kode instalasi:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx curl
```

#### Konfigurasi Upstream dan Virtual Host

Konfigurasi upstream dibuat pada file `/etc/nginx/sites-available/lb`. Upstream `api_cluster` berisi dua server backend, yaitu Nanami dan Todo.

**Kode konfigurasi:**

```nginx
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
```

Penjelasan konfigurasi:

- `upstream api_cluster` mendefinisikan kelompok server backend yang menerima request dari Load Balancer.
- `server 10.95.4.5 weight=3` memberikan bobot 3 kepada Nanami.
- `server 10.95.4.6 weight=1` memberikan bobot 1 kepada Todo.
- `proxy_pass http://api_cluster` meneruskan request ke salah satu server dalam kelompok upstream.
- `proxy_set_header` meneruskan informasi host dan alamat IP client ke backend.

Nginx menggunakan Weighted Round Robin secara default ketika beberapa server upstream diberi bobot. Dengan bobot 3:1, distribusi request normal yang diharapkan adalah sekitar 75% menuju Nanami dan 25% menuju Todo.

#### Mengaktifkan Konfigurasi

Setelah file konfigurasi dibuat, konfigurasi diaktifkan melalui symbolic link. Konfigurasi default dinonaktifkan agar tidak berbenturan dengan virtual host Load Balancer.

**Kode aktivasi:**

```bash
ln -sf /etc/nginx/sites-available/lb /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

nginx -t
service nginx restart
```

Perintah `nginx -t` digunakan untuk memeriksa validitas sintaks konfigurasi sebelum Nginx dijalankan ulang.

#### Pengujian Satu Request

Pengujian awal dilakukan dari Geto untuk memastikan bahwa Load Balancer dapat meneruskan request ke backend.

**Kode pengujian:**

```bash
curl http://localhost/
```

Respons dapat berasal dari Nanami atau Todo, bergantung pada urutan distribusi request.

#### engujian Distribusi 20 Request

Pengujian dilakukan dengan mengirimkan 20 request secara berulang ke Geto. Nama hostname pada setiap respons diambil menggunakan `grep`, kemudian jumlah respons dari masing-masing backend dihitung menggunakan `sort` dan `uniq`.

**Kode pengujian:**

```bash
for i in $(seq 1 20); do
    curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'
done | sort | uniq -c
```

#### Pengujian Distribusi 40 Request

Untuk memperoleh sampel pengujian yang lebih besar, jumlah request ditingkatkan menjadi 40.

**Kode pengujian:**

```bash
for i in $(seq 1 40); do
    curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'
done | sort | uniq -c
```

Perbandingan jumlah request adalah 30:10 atau 3:1.

Persentase distribusi dihitung sebagai berikut:

- Nanami: \(30/40 \times 100\% = 75\%\)
- Todo: \(10/40 \times 100\% = 25\%\)

Hasil tersebut sesuai dengan bobot yang ditentukan dalam konfigurasi upstream.

#### Verifikasi Layanan dan Konfigurasi

**Memeriksa port 80:**

```bash
ss -tlnp | grep :80
```

Perintah tersebut digunakan untuk memastikan Nginx mendengarkan koneksi HTTP pada port 80.

**Memeriksa konfigurasi upstream:**

```bash
nginx -T | grep -A5 upstream
```

Perintah tersebut menampilkan bagian konfigurasi upstream yang sedang digunakan oleh Nginx.

**Menguji akses dari node lain di jaringan Data Center:**

```bash
curl http://10.95.4.2/
```

Pengujian ini memastikan bahwa Geto dapat diakses melalui alamat IPnya dari node lain. Keberhasilan pengujian bergantung pada konfigurasi jaringan dan aturan firewall yang berlaku.

## Soal 16

### Konfigurasi dan Implementasi

#### Perbaikan Alamat IP Gojo

Gojo menggunakan alamat IP `10.95.1.4/24` agar tidak berbenturan dengan Shoko yang menggunakan alamat `10.95.1.3`.

**Kode konfigurasi pada Gojo:**

```bash
ip addr del 10.95.1.3/24 dev eth0
ip addr add 10.95.1.4/24 dev eth0
ip route add default via 10.95.1.1

ip addr show eth0 | grep inet
```

Konfigurasi permanen jaringan dituliskan pada file `/etc/network/interfaces`.

```bash
cat > /etc/network/interfaces << 'ENDOFFILE'
auto eth0
iface eth0 inet static
    address 10.95.1.4
    netmask 255.255.255.0
    gateway 10.95.1.1
    dns-nameservers 192.168.122.1
ENDOFFILE
```

Perintah verifikasi digunakan untuk memastikan bahwa Gojo telah menggunakan alamat IP yang benar.

#### Instalasi Nginx pada Gojo

Nginx digunakan sebagai reverse proxy publik yang menerima request dari client dan meneruskannya ke backend sesuai path URL.

**Kode instalasi:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
apt-get update
apt-get install -y nginx
```

#### Penyesuaian Virtual Host API Worker

Penyesuaian dilakukan pada Nanami, Todo, dan Kirara agar request dengan path yang tidak cocok dengan file fisik tetap diarahkan ke `index.php`. Hal ini diperlukan supaya request seperti `/api` dapat diproses oleh aplikasi PHP.

**Kode konfigurasi pada setiap API worker:**

```nginx
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
```

Simpan konfigurasi tersebut pada `/etc/nginx/sites-available/api`, kemudian jalankan:

```bash
nginx -t && service nginx restart
```

Direktif `try_files` memastikan request yang tidak menunjuk ke file atau direktori fisik diarahkan ke `index.php` beserta query string. Aturan `allow` dan `deny` membatasi akses API worker agar hanya menerima request dari Geto dengan alamat IP `10.95.4.2`.

#### Konfigurasi Path-Based Routing pada Gojo

Konfigurasi ini merupakan bagian utama praktikum. Gojo menggunakan dua jalur routing, yaitu frontend cluster untuk path root dan Geto untuk path `/api`.

**Kode konfigurasi pada `/etc/nginx/sites-available/gojo`:**

```nginx
upstream frontend_cluster {
    server 10.95.4.3;
    server 10.95.4.4;
}

server {
    listen 80;
    server_name jujutsu.k63.com www.jujutsu.k63.com;

    location /api {
        proxy_pass http://10.95.4.2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://frontend_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Penjelasan konfigurasi:

- `upstream frontend_cluster` mengelompokkan Maki (`10.95.4.3`) dan Panda (`10.95.4.4`).
- `location /api` meneruskan request API ke Geto (`10.95.4.2`), yang selanjutnya mendistribusikan request ke Nanami dan Todo.
- `location /` meneruskan request selain path `/api` ke frontend cluster.
- `proxy_set_header` meneruskan informasi host dan alamat IP client ke backend.

Nginx memilih lokasi `/api` karena memiliki prefix yang lebih spesifik dibandingkan lokasi `/`. Dengan demikian, request API tidak ikut diarahkan ke frontend worker.

**Mengaktifkan konfigurasi:**

```bash
ln -sf /etc/nginx/sites-available/gojo /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

nginx -t
service nginx restart
```

#### Restart Load Balancer Internal Geto

Geto bertugas menerima request API dari Gojo dan mendistribusikannya ke Nanami serta Todo menggunakan upstream `api_cluster` dengan algoritma Weighted Round Robin berbobot 3:1.

**Kode pada Geto:**

```bash
service nginx restart
```

Perintah ini menerapkan konfigurasi Nginx yang telah tersedia pada Geto. Konfigurasi upstream harus sudah dibuat dan diuji sebelumnya.

### Pengujian

#### Pengujian Round Robin Frontend

Pengujian dilakukan pada Gojo dengan mengirimkan request ke root sebanyak 10 kali.

**Kode pengujian:**

```bash
for i in $(seq 1 10); do
    curl -s http://localhost/ | grep -o 'Hostname: [a-z]*'
done | sort | uniq -c
```

Hasil menunjukkan request root berhasil didistribusikan ke Maki dan Panda. Karena kedua server memiliki bobot default yang sama, distribusi yang mendekati seimbang diharapkan terjadi.

Pengujian dapat diulang dengan 100 request:

```bash
for i in $(seq 1 100); do
    curl -s http://localhost/ | grep -o 'Hostname: [a-z]*'
done | sort | uniq -c
```

#### Pengujian Path API dari Gojo

**Kode pengujian:**

```bash
curl http://localhost/api
```

Respons dapat berasal dari Nanami atau Todo karena Geto meneruskan request ke upstream `api_cluster`. Field `hostname` menunjukkan backend yang memproses request. Isi `recent_logs` bergantung pada data yang tersimpan di database.

#### Pengujian dari Client Divisi A atau B

Pengujian dilakukan dari salah satu client, misalnya Zenin, Okkotsu, atau Kugisaki.

**Menentukan DNS resolver:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
```

**Menguji domain utama:**

```bash
curl http://jujutsu.k63.com/
curl http://www.jujutsu.k63.com/
```

Hasil menunjukkan request ke domain utama diteruskan oleh Gojo ke frontend cluster.

**Menguji layanan API:**

```bash
curl http://jujutsu.k63.com/api
```

Respons API dapat berasal dari Nanami atau Todo. Hal ini menunjukkan bahwa request dari client melewati Gojo terlebih dahulu, kemudian diteruskan ke Geto untuk didistribusikan ke API worker.

## Soal 17

### Konfigurasi dan Implementasi

#### Konfigurasi Alamat IP Client

Alamat IP client perlu dipastikan sebelum pengujian agar alamat yang tersimpan di database dapat dibandingkan dengan alamat asli masing-masing client.

**Pada Ino (Network 3):**

```bash
ip addr add 10.95.3.2/24 dev eth0
ip link set eth0 up
ip route add default via 10.95.3.1
echo "nameserver 10.95.1.2" > /etc/resolv.conf
ip addr show eth0 | grep inet
```

**Pada Zenin (Network 2):**

```bash
ip addr show eth0 | grep inet
```

Zenin diharapkan menggunakan IP `10.95.2.4`, sedangkan Ino menggunakan IP `10.95.3.2`.

#### Konfigurasi Gojo untuk Meneruskan Header Client

Gojo meneruskan request ke frontend worker atau Geto. Header `X-Real-IP` digunakan untuk membawa alamat IP client, sedangkan `X-Forwarded-For` menyimpan rantai alamat IP yang dilewati request.

**Kode konfigurasi `/etc/nginx/sites-available/gojo`:**

```nginx
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
```

Direktif `proxy_set_header X-Real-IP $remote_addr` meneruskan alamat client yang terlihat oleh Gojo. Sementara itu, `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` menambahkan alamat tersebut ke rantai header `X-Forwarded-For`.

**Validasi dan restart Nginx pada Gojo:**

```bash
nginx -t && service nginx restart
```

#### Konfigurasi Geto sebagai Load Balancer Internal

Geto hanya mempercayai Gojo dengan alamat IP `10.95.1.4` sebagai proxy yang berwenang memberikan informasi IP client. Geto kemudian meneruskan informasi tersebut ke API worker.

**Kode konfigurasi `/etc/nginx/sites-available/lb`:**

```nginx
upstream api_cluster {
    server 10.95.4.5 weight=3;
    server 10.95.4.6 weight=1;
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
```

Direktif `set_real_ip_from 10.95.1.4` menentukan proxy terpercaya. Modul Real IP Nginx menggunakan header `X-Forwarded-For` dari proxy tersebut untuk menentukan alamat client yang sebenarnya. Direktif `real_ip_recursive on` membantu memilih alamat nontepercaya yang tepat dari rantai header.

Setelah alamat client diverifikasi, `$remote_addr` digunakan untuk meneruskan alamat tersebut ke API worker melalui `X-Real-IP`. Header `X-Forwarded-For` tetap diteruskan untuk menyimpan informasi rantai proxy.

**Validasi dan restart Nginx pada Geto:**

```bash
nginx -t && service nginx restart
```

#### Konfigurasi API Worker untuk Mencatat IP Client

Konfigurasi berikut diterapkan pada Nanami, Todo, dan Kirara. API worker menerima request, menentukan alamat client, lalu menyimpan alamat tersebut ke database Yaga.

**Kode `/var/www/api/index.php`:**

```php
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
```

Kode tersebut memeriksa format alamat IP melalui `FILTER_VALIDATE_IP` dan hanya menggunakan `X-Real-IP` jika koneksi berasal dari alamat proxy yang tercantum pada daftar kepercayaan. Jika tidak memenuhi syarat, aplikasi menggunakan alamat koneksi langsung dari `REMOTE_ADDR`.

**Validasi sintaks PHP:**

```bash
php -l /var/www/api/index.php
```

#### Konfigurasi Virtual Host API Worker

API worker perlu mengarahkan request yang tidak menunjuk ke file fisik ke `index.php`, sehingga endpoint `/api` dapat diproses oleh aplikasi PHP.

**Kode konfigurasi pada Nanami, Todo, dan Kirara:**

```nginx
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
```

Simpan konfigurasi tersebut pada `/etc/nginx/sites-available/api`, kemudian jalankan:

```bash
nginx -t && service nginx restart
```

Aturan akses mengizinkan request langsung ke API worker hanya dari Geto (`10.95.4.2`). Karena itu, API worker tidak seharusnya menerima request langsung dari Zenin atau Ino.

### Pengujian 

#### Pengujian dari Zenin (Network 2)

**Perintah:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
curl http://jujutsu.k63.com/api
```

Alamat `10.95.2.4` menunjukkan IP Zenin yang diharapkan tercatat sebagai client, bukan IP Gojo atau Geto.

#### Pengujian dari Ino (Network 3)

**Perintah:**

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf
curl http://jujutsu.k63.com/api
```

Alamat `10.95.3.2` menunjukkan IP Ino yang diharapkan tercatat sebagai client. Hostname dapat berupa Nanami atau Todo karena request didistribusikan oleh Geto.

#### Verifikasi Data pada Database Yaga

Pemeriksaan dilakukan dengan menjalankan query pada database `db_shibuya_k63`.

**Perintah:**

```bash
mysql -h 10.95.4.9 -u root -proot_password db_shibuya_k63 \
-e "SELECT id, client_ip, endpoint, timestamp FROM request_logs ORDER BY id DESC LIMIT 10;"
```

Hasil menunjukkan alamat IP client Network 2 dan Network 3 tersimpan pada kolom `client_ip`. 

## Soal 18

### Konfigurasi dan Implementasi

#### Memastikan Layanan API pada Kirara Berjalan

Memastikan Kirara memiliki alamat IP yang sesuai serta layanan Nginx dan PHP-FPM berjalan dengan baik.

Konfigurasi alamat IP dan routing pada Kirara:

```bash
ip addr add 10.95.4.7/24 dev eth0 2>/dev/null
ip link set eth0 up
ip route add default via 10.95.4.1 2>/dev/null
```

Pemeriksaan status jaringan dan layanan:

```bash
ip addr show eth0 | grep inet
ps aux | grep nginx | head -2
ps aux | grep php-fpm | head -2
ls -la /run/php/php8.4-fpm.sock
ls -la /var/www/api/index.php
```

Apabila PHP-FPM belum berjalan, jalankan perintah berikut:

```bash
/usr/sbin/php-fpm8.4 -D
service nginx restart
```

Selanjutnya, periksa aturan akses pada konfigurasi API:

```bash
grep -n "allow\|deny" /etc/nginx/sites-available/api
```

Pemeriksaan ini dilakukan untuk memastikan Kirara memiliki berkas API, socket PHP-FPM, dan konfigurasi akses yang diperlukan agar dapat melayani request seperti worker API utama.

#### Konfigurasi Upstream pada Geto

Konfigurasi upstream pada Geto diubah agar Nanami dan Todo tetap menjadi worker utama, sedangkan Kirara menjadi worker cadangan.

Berkas konfigurasi `/etc/nginx/sites-available/lb`:

```nginx
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
```

Penjelasan konfigurasi:

- **Nanami (`10.95.4.5`)** menjadi worker utama dengan bobot `3`, sehingga memperoleh proporsi request lebih besar dibandingkan Todo ketika keduanya tersedia.
- **Todo (`10.95.4.6`)** menjadi worker utama dengan bobot `1`.
- **Kirara (`10.95.4.7`)** ditetapkan sebagai server `backup`. Server ini digunakan ketika worker utama tidak tersedia menurut mekanisme pemilihan upstream Nginx.
- `max_fails=2` menentukan jumlah kegagalan yang diperhitungkan sebelum server dianggap tidak tersedia.
- `fail_timeout=5s` menentukan periode penghitungan kegagalan sekaligus waktu server dianggap tidak tersedia setelah mencapai batas kegagalan.
- `set_real_ip_from 10.95.1.4` menetapkan alamat Gojo sebagai proxy yang dipercaya untuk informasi IP klien.
- `real_ip_header X-Forwarded-For` dan `real_ip_recursive on` digunakan untuk memproses alamat klien dari header yang diteruskan proxy.

Setelah konfigurasi selesai, lakukan pemeriksaan dan restart Nginx:

```bash
nginx -t && service nginx restart
```

#### Pengujian Kondisi Normal

Sebelum melakukan simulasi gangguan, Kirara diuji secara langsung dari Geto:

```bash
curl http://10.95.4.7/
```

Selanjutnya, pengujian distribusi request dilakukan melalui upstream:

```bash
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

Pengujian ini bertujuan untuk memeriksa hostname worker yang memberikan respons ketika Nanami dan Todo masih aktif. Dalam kondisi normal, request seharusnya dilayani oleh worker utama, sedangkan Kirara tetap menjadi cadangan.

#### Simulasi Gangguan pada Worker Utama

Untuk menguji mekanisme failover, layanan Nginx pada Nanami dihentikan:

```bash
service nginx stop
ps aux | grep nginx | grep -v grep
```

Lakukan langkah yang sama pada Todo:

```bash
service nginx stop
ps aux | grep nginx | grep -v grep
```

Dengan kedua worker utama dihentikan, Geto tidak dapat menggunakan Nanami dan Todo untuk melayani request. Oleh karena itu, Kirara diharapkan mengambil alih layanan sebagai worker cadangan.

#### Pengujian Failover ke Kirara

Setelah layanan worker utama dihentikan, tunggu sekitar lima detik sesuai parameter `fail_timeout`, lalu uji endpoint dari Geto:

```bash
sleep 5
curl http://localhost/
curl http://localhost/api
```

Kemudian, kirim beberapa request untuk memeriksa hostname worker yang merespons:

```bash
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

Pengujian dari klien Zenin pada Network 2 dilakukan menggunakan:

```bash
curl http://jujutsu.k63.com/api
for i in $(seq 1 10); do curl -s http://jujutsu.k63.com/api | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

Pengujian ini digunakan untuk memastikan request dari klien tetap mendapatkan respons dari Kirara ketika kedua worker utama tidak tersedia.

#### Pemulihan Layanan

Setelah pengujian selesai, layanan Nginx pada Nanami dan Todo dijalankan kembali.

Pada Nanami:

```bash
service nginx start
```

Pada Todo:

```bash
service nginx start
```

Selanjutnya, periksa kembali distribusi request melalui Geto:

```bash
for i in $(seq 1 10); do curl -s http://localhost/ | grep -o '"hostname":"[^"]*"'; done | sort | uniq -c
```

Langkah ini bertujuan untuk memastikan worker utama dapat kembali melayani request setelah pulih.

## Soal 19

### Konfigurasi dan Implementasi

#### Membuat Direktori Cache

Langkah pertama adalah membuat direktori untuk menyimpan cache halaman frontend pada server Gojo.

```bash
mkdir -p /var/cache/nginx/frontend
chown -R www-data:www-data /var/cache/nginx
```

Perintah `mkdir -p` digunakan untuk membuat direktori cache jika belum tersedia. Sementara itu, `chown` digunakan untuk memberikan kepemilikan direktori kepada pengguna dan grup `www-data`, sehingga Nginx dapat mengakses direktori tersebut.

#### Konfigurasi Nginx pada Gojo

Konfigurasi Nginx pada Gojo diubah untuk menerapkan proxy caching pada halaman frontend dan rate limiting pada endpoint `/api`.

Berkas konfigurasi `/etc/nginx/sites-available/gojo`:

```nginx
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
```

Penjelasan konfigurasi:

- `proxy_cache_path` menentukan lokasi penyimpanan cache, nama zona cache, ukuran maksimum, dan periode tidak aktif cache.
- `proxy_cache_valid 200 10m` menentukan masa berlaku cache selama 10 menit untuk respons HTTP `200`.
- `proxy_cache_key` menentukan identitas cache berdasarkan skema, metode request, host, dan URI.
- `add_header X-Cache-Status $upstream_cache_status` menambahkan header yang menunjukkan status cache dari upstream.
- `limit_req_zone` membuat zona pembatasan request berdasarkan `$binary_remote_addr` dengan batas rata-rata 10 request per detik.
- `burst=5` mengizinkan lonjakan hingga lima request tambahan yang menunggu atau diproses sesuai konfigurasi.
- `nodelay` membuat request dalam batas burst yang tersedia tidak mengalami penundaan akibat mekanisme antrean.
- `limit_req_status 429` menetapkan kode HTTP `429` untuk request yang ditolak karena melampaui batas.
- `proxy_set_header X-Real-IP` dan `proxy_set_header X-Forwarded-For` meneruskan informasi alamat klien ke server tujuan.

Setelah konfigurasi selesai, lakukan pengujian sintaks dan restart Nginx:

```bash
nginx -t && service nginx restart
```

#### Verifikasi Proxy Caching

Sebelum pengujian, cache dibersihkan terlebih dahulu pada Gojo:

```bash
rm -rf /var/cache/nginx/frontend/*
```

Selanjutnya, dari klien Zenin, Ino, atau Kusakabe, atur DNS dan kirimkan request ke halaman frontend:

```bash
echo "nameserver 10.95.1.2" > /etc/resolv.conf

curl -I http://jujutsu.k63.com/
curl -I http://jujutsu.k63.com/
curl -I http://jujutsu.k63.com/
```

Header `X-Cache-Status` digunakan untuk memeriksa apakah respons diambil dari upstream atau dari cache Nginx.

HASIL HARUS:
- Request 1 -> X-Cache-Status: MISS
- Request 2 -> X-Cache-Status: HIT
- Request 3 -> X-Cache-Status: HIT

#### Verifikasi Rate Limiting

Pengujian rate limiting dilakukan dari klien Zenin dengan mengirimkan 50 request secara beruntun ke endpoint `/api`:

```bash
for i in $(seq 1 50); do curl -s -o /dev/null -w "%{http_code} " http://jujutsu.k63.com/api; done; echo
```

Perintah tersebut menampilkan kode HTTP dari setiap request tanpa mencetak isi respons. Dengan demikian, kode `200` dan `429` dapat diamati secara langsung.

#### Pemeriksaan dan Pembersihan Cache

Untuk memeriksa file cache yang tersimpan, jalankan pada Gojo:

```bash
ls -la /var/cache/nginx/frontend/
```

Untuk melihat ukuran direktori cache:

```bash
du -sh /var/cache/nginx/frontend/
```

Untuk membersihkan cache agar pengujian dapat dimulai kembali dari kondisi awal:

```bash
rm -rf /var/cache/nginx/frontend/*
```

Pemeriksaan log error Nginx dilakukan dengan perintah:

```bash
tail -20 /var/log/nginx/error.log
```

Log tersebut dapat membantu mengidentifikasi request yang ditolak oleh mekanisme rate limiting.

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
