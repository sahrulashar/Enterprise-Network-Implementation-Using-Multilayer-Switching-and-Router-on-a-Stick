# Lab Jaringan: Multilayer Switching + Guest Wifi (VLAN 40)

Dokumentasi topologi, addressing, dan konfigurasi untuk lab Packet Tracer yang mengimplementasikan VLSM, VLAN, trunking (native VLAN), multilayer switching (inter-VLAN routing), dan jaringan tamu (guest wifi) dengan DHCP terpisah.

## 1. Topologi

```
                         Internet
                             |
                      +------------+
                      |   R1       |   (Router 2911 - default gateway ke internet)
                      | Gi0/0      |
                      +------------+
                             |  172.20.0.240/30  (routed point-to-point link)
                      +------------------+
                      | MULTILAYERSWITCH |   (3650-24PS, ip routing enabled)
                      |  SVI Vlan10/20/  |
                      |  30/40           |
                      +------------------+
                        |              \
                        | trunk         \ access (Gi1/0/24)
                        | native vlan99  \
                      +-------+       +------------+
                      |  SW1  |       |  Server0   |
                      | (2960)|       | DHCP Guest |
                      +-------+       +------------+
                 /     |      \
          VLAN10/     VLAN20   VLAN30   VLAN40 (Fa0/10)
          HR (Fa0/1-3) FINANCE  IT         |
          3 PC         (Fa0/4-6)(Fa0/7-9)  AP GuestWifi
                        3 PC     3 PC       |
                                        4 Smartphone (wireless)
```

**Catatan desain penting:**
- Link `MULTILAYERSWITCH Gi1/0/2 <-> R1 Gi0/0` adalah **routed port point-to-point** (`no switchport`), jadi tidak bisa dipakai sebagai trunk ROAS. Karena itu, VLAN 40 (Guest) **tidak** dirutekan lewat router — inter-VLAN routing untuk semua VLAN (10, 20, 30, 40) dilakukan langsung oleh SVI di `MULTILAYERSWITCH`. Router hanya berperan sebagai default gateway ke internet.
- Trunk antara `SW1` dan `MULTILAYERSWITCH` menggunakan **native VLAN 99** (bukan VLAN 1 default) sebagai praktik keamanan standar.
- Server DHCP untuk Guest Wifi ditempatkan **di VLAN yang sama (40)**, langsung di port access `Gi1/0/24` milik `MULTILAYERSWITCH` — jadi tidak perlu `ip helper-address`.

## 2. Tabel VLAN & Addressing (VLSM dari blok 172.20.0.0/23)

| VLAN | Nama | Subnet | Range usable | Gateway (SVI) | Device |
|---|---|---|---|---|---|
| 10 | HR | 172.20.0.128/26 | .129 – .190 | .190 | SW1 Fa0/1-3 (3 PC) |
| 20 | FINANCE | 172.20.0.192/27 | .193 – .222 | .222 | SW1 Fa0/4-6 (3 PC) |
| 30 | IT | 172.20.0.224/28 | .225 – .238 | .238 | SW1 Fa0/7-9 (3 PC) |
| 40 | GUEST_WIFI | 172.20.0.0/25 | .2 – .126 | .126 | SW1 Fa0/10 → AP GuestWifi (4 smartphone) + Server0 DHCP |
| 99 | Native (trunk) | — (belum ada SVI) | — | — | native VLAN di semua trunk, dicadangkan untuk management |
| — | WAN (routed link) | 172.20.0.240/30 | .241 – .242 | — | MULTILAYERSWITCH Gi1/0/2 (.241) ↔ R1 Gi0/0 (.242) |

**DHCP pool `GUEST` di Server0:**
- Default gateway: `172.20.0.126`
- DNS server: `8.8.8.8`
- Start IP: `172.20.0.2`
- Subnet mask: `255.255.255.128`
- Max users: `100`
- Server0 sendiri pakai IP statis `172.20.0.2`... *(pastikan start IP pool digeser di atas IP statis server agar tidak bentrok, mis. mulai dari `.3` jika `.2` dipakai Server0)*

## 3. Peran & Status Tiap Device

| Device | Model | Peran | Status |
|---|---|---|---|
| R1 | Cisco 2911 | Edge router, default gateway ke internet | ✅ Selesai |
| MULTILAYERSWITCH | 3650-24PS | Distribution layer, inter-VLAN routing (SVI 10/20/30/40), trunk ke SW1 | ✅ Selesai |
| SW1 | Catalyst 2960 | Access layer, port access tiap VLAN, trunk ke distribution | ✅ Selesai |
| Server0 | Server-PT | DHCP server khusus VLAN 40 (Guest) | ✅ Selesai |
| AccessPoint GuestWifi | Access Point-PT | Menghubungkan 4 smartphone secara wireless ke VLAN 40 | ✅ Selesai |

## 4. Verifikasi yang Sudah Dilakukan

```
show vlan brief                 -> semua VLAN (10,20,30,40) aktif di SW1 & MULTILAYERSWITCH
show interfaces trunk           -> native vlan 99, allowed vlan 10,20,30,40 di kedua trunk
show ip interface brief         -> SVI Vlan10/20/30/40 up/up dengan IP sesuai tabel addressing
```
Smartphone sudah berhasil mendapat IP dari DHCP pool `GUEST` melalui Desktop → IP Configuration → DHCP.

## 5. Daftar File

```
configs/R1.cfg                  - running-config router edge
configs/MULTILAYERSWITCH.cfg    - running-config distribution switch
configs/SW1.cfg                 - running-config access switch
```

Import ulang ke Packet Tracer via copy-paste ke `configure terminal` masing-masing device, atau gunakan sebagai referensi dokumentasi.
