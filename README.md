# Cisco IOS — Implementation & Operations Guide

## Daftar Isi

- [Modul 1: Ruang Lingkup](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-1-ruang-lingkup)
- [Modul 2: Topologi Referensi](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-2-topologi-referensi)
- [Modul 3: Hardening Dasar](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-3-hardening-dasar)
- [Modul 4: Layer 2 dan STP](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-4-layer-2-dan-stp)
- [Modul 5: IP Addressing](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-5-ip-addressing)
- [Modul 6: OSPF](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-6-ospf)
- [Modul 7: BGP dan Route Reflector](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-7-bgp-dan-route-reflector)
- [Modul 8: MPLS dan LDP](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-8-mpls-dan-ldp)
- [Modul 9: MPLS L3VPN dan VRF](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-9-mpls-l3vpn-dan-vrf)
- [Modul 10: VRF Lite](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-10-vrf-lite)
- [Modul 11: QoS](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-11-qos)
- [Modul 12: Keamanan Control Plane dan Data Plane](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-12-keamanan-control-plane-dan-data-plane)
- [Modul 13: High Availability dan Fast Convergence](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-13-high-availability-dan-fast-convergence)
- [Modul 14: NTP, DNS, DHCP, NAT](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-14-ntp-dns-dhcp-nat)
- [Modul 15: Interkoneksi CE](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-15-interkoneksi-ce)
- [Modul 16: Implementasi P dan PE](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-16-implementasi-p-dan-pe)
- [Modul 17: Implementasi RR dan CE](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-17-implementasi-rr-dan-ce)
- [Modul 18: Validasi End-to-End dan Cutover](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-18-validasi-end-to-end-dan-cutover)
- [Modul 19: Operasi Harian](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-19-operasi-harian)
- [Modul 20: Troubleshooting](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-20-troubleshooting)
- [Modul 21: Change, Backup, Lifecycle](https://github.com/anggrdwjy/Library-Cisco-IOS-Enterprise/blob/main/README.md#modul-21-change-backup-lifecycle)

### Modul 1: Ruang Lingkup

**Prinsip:** Satu domain core, satu kebijakan routing, satu standar konfigurasi.
**Ruang Lingkup:** Routing dan switching enterprise. Di luar lingkup: Segment Routing, EVPN penuh, otomasi.
**Standar:** Hostname, Loopback0, deskripsi antarmuka, dan NTP wajib pada semua perangkat.

### Modul 2: Topologi Referensi

| Peran | Jumlah | Fungsi |
| --- | --- | --- |
| P | 2 | Transit MPLS, tanpa VRF |
| PE | 3 | Terminasi VRF, peering BGP VPNv4 ke RR |
| RR | 2 | Refleksi VPNv4, tanpa forwarding pelanggan |
| CE | 6 | Peering ke PE, tanpa visibilitas core |

**Prinsip:** Kegagalan satu P, PE, atau RR tidak boleh menghentikan layanan. Redundansi harus diuji, bukan diasumsikan.

### Modul 3: Hardening Dasar

**Prinsip:** Akses manajemen adalah permukaan serangan utama.
**Konfigurasi Inti:**

```
enable secret <FROM-VAULT>
service password-encryption
no ip http server
no ip http secure-server
line vty 0 4
 transport input ssh
 access-class <ACL-VTY-IN> in
 exec-timeout <TIMEOUT>
login block-for <DETIK> attempts <JUMLAH> within <DETIK>
```

**Verifikasi:** `show ip ssh` | `show access-lists` | `show login`
**Risiko:** VTY tanpa ACL dan tanpa `reload in` menyeModulkan lockout tanpa rollback.

### Modul 4: Layer 2 dan STP

**Prinsip:** Loop Layer 2 menghentikan core lebih cepat daripada kegagalan routing.
**Konfigurasi Inti:**

```
spanning-tree mode rapid-pvst
spanning-tree vlan <VLAN-ID> root primary
interface GigabitEthernet<SLOT>/<PORT>
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport mode trunk
 switchport trunk allowed vlan <VLAN-LIST>
```

**Verifikasi:** `show spanning-tree vlan <VLAN-ID> brief` | `show interfaces status`
**Risiko:** Port access tanpa BPDU Guard menerima BPDU liar dan mengubah root.

### Modul 5: IP Addressing

**Prinsip:** Loopback adalah identitas routing. Link adalah transit, bukan identitas.
**Standar:** Loopback0 `/32` untuk router-id OSPF, BGP, dan LDP. Link point-to-point `/30` atau `/31`. Blok per site terdokumentasi, tidak tumpang tindih.
**Verifikasi:** `show ip interface brief` | `show ip route connected`

### Modul 6: OSPF

**Prinsip:** OSPF hanya untuk core. CE tidak masuk area 0 core.
**Konfigurasi Inti:**

```
router ospf <PROCESS>
 router-id <LOOPBACK>
 passive-interface default
 no passive-interface GigabitEthernet<SLOT>/<PORT>
 network <NETWORK-LOOPBACK> <WILDCARD> area 0
 network <NETWORK-LINK> <WILDCARD> area 0
```

**Verifikasi:** `show ip ospf neighbor` | `show ip ospf interface brief`
**Risiko:** Router-id duplikat, autentikasi tidak cocok, `passive-interface` pada link core.

### Modul 7: BGP dan Route Reflector

**Prinsip:** Skalabilitas melalui hierarki. PE hanya peer ke RR, bukan full-mesh. RR hanya refleksi rute, tidak forwarding trafik pelanggan, tidak punya VRF pelanggan.
**Kapan Dipakai:** Wajib jika PE > 2. Tanpa RR, PE harus full-mesh iBGP VPNv4. Dengan 2 RR, satu RR down tidak menghentikan refleksi.
**Desain RR di Panduan Ini:** 2 RR, `cluster-id` sama untuk keduanya agar PE melihat satu cluster yang redundan. RR1 dan RR2 peer satu sama lain. RR tidak `next-hop-self` agar next-hop PE asal dipertahankan.

**Konfigurasi Inti — PE (ke 2 RR):**

```
router bgp <BGP-AS>
 bgp router-id <PE-LOOPBACK>
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor <RR1-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR1-LOOPBACK> description RR1
 neighbor <RR1-LOOPBACK> update-source Loopback0
 neighbor <RR1-LOOPBACK> password <FROM-VAULT>
 neighbor <RR2-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR2-LOOPBACK> description RR2
 neighbor <RR2-LOOPBACK> update-source Loopback0
 neighbor <RR2-LOOPBACK> password <FROM-VAULT>
 !
 address-family vpnv4
  neighbor <RR1-LOOPBACK> activate
  neighbor <RR1-LOOPBACK> send-community extended
  neighbor <RR2-LOOPBACK> activate
  neighbor <RR2-LOOPBACK> send-community extended
 exit-address-family
```

**Konfigurasi Inti — RR1 (RR2 identik, ganti Loopback dan hostname):**

```
hostname <RR1-HOSTNAME>
!
ip cef
!
interface Loopback0
 description Router-ID RR1
 ip address <RR1-LOOPBACK> 255.255.255.255
!
interface GigabitEthernet<SLOT>/<PORT>
 description to P1
 ip address <IP-RR1-P1> <MASK-RR1-P1>
 mpls ip
 ip ospf <PROCESS> area 0
 no shutdown
!
interface GigabitEthernet<SLOT>/<PORT>
 description to P2
 ip address <IP-RR1-P2> <MASK-RR1-P2>
 mpls ip
 ip ospf <PROCESS> area 0
 no shutdown
!
router ospf <PROCESS>
 router-id <RR1-LOOPBACK>
 passive-interface default
 no passive-interface GigabitEthernet<SLOT>/<PORT>
 network <NETWORK-RR1-LOOPBACK> <WILDCARD> area 0
 network <NETWORK-RR1-P1> <WILDCARD> area 0
 network <NETWORK-RR1-P2> <WILDCARD> area 0
!
mpls ip
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
router bgp <BGP-AS>
 bgp router-id <RR1-LOOPBACK>
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 bgp cluster-id <CLUSTER-ID>
 !
 neighbor <PE1-LOOPBACK> remote-as <BGP-AS>
 neighbor <PE1-LOOPBACK> description PE1-HO
 neighbor <PE1-LOOPBACK> update-source Loopback0
 neighbor <PE1-LOOPBACK> password <FROM-VAULT>
 neighbor <PE2-LOOPBACK> remote-as <BGP-AS>
 neighbor <PE2-LOOPBACK> description PE2-DRC
 neighbor <PE2-LOOPBACK> update-source Loopback0
 neighbor <PE2-LOOPBACK> password <FROM-VAULT>
 neighbor <PE3-LOOPBACK> remote-as <BGP-AS>
 neighbor <PE3-LOOPBACK> description PE3-CABANG
 neighbor <PE3-LOOPBACK> update-source Loopback0
 neighbor <PE3-LOOPBACK> password <FROM-VAULT>
 neighbor <RR2-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR2-LOOPBACK> description RR2-PEER
 neighbor <RR2-LOOPBACK> update-source Loopback0
 neighbor <RR2-LOOPBACK> password <FROM-VAULT>
 !
 address-family vpnv4
  neighbor <PE1-LOOPBACK> activate
  neighbor <PE1-LOOPBACK> send-community extended
  neighbor <PE1-LOOPBACK> route-reflector-client
  neighbor <PE2-LOOPBACK> activate
  neighbor <PE2-LOOPBACK> send-community extended
  neighbor <PE2-LOOPBACK> route-reflector-client
  neighbor <PE3-LOOPBACK> activate
  neighbor <PE3-LOOPBACK> send-community extended
  neighbor <PE3-LOOPBACK> route-reflector-client
  neighbor <RR2-LOOPBACK> activate
  neighbor <RR2-LOOPBACK> send-community extended
 exit-address-family
```

**Per-Blok Penjelasan — RR:**

| Blok | Fungsi | Jika Salah |
| --- | --- | --- |
| `no bgp default ipv4-unicast` | RR hanya aktifkan VPNv4 secara eksplisit, tidak bawa IPv4 unicast | RR tanpa sengaja jadi transit IPv4 |
| `bgp cluster-id <CLUSTER-ID>` | Sama di RR1 dan RR2, satu cluster redundan | Cluster-id beda membuat loop proteksi gagal, rute duplikat |
| `route-reflector-client` di VPNv4 | Tandai PE sebagai client, agar RR refleksikan rute antar client | Tanpa ini, RR tidak refleksikan rute PE ke PE lain |
| `send-community extended` | Kirim RT via extended community | RT hilang, VRF tidak terima rute |
| `update-source Loopback0` | Sesi BGP via Loopback, tahan satu link down | Sesi down saat link fisik down |
| Tanpa `next-hop-self` di RR | Next-hop PE asal dipertahankan, LSP end-to-end utuh | `next-hop-self` di RR memutus LSP, trafik blackhole |
| Tanpa VRF di RR | RR tidak forwarding pelanggan, hanya kontrol | VRF di RR membebani RR dan merusak desain |

**Verifikasi — RR:**

```
show ip bgp vpnv4 all summary
show ip bgp vpnv4 all neighbors <PE-LOOPBACK> advertised-routes
show ip bgp vpnv4 all neighbors <PE-LOOPBACK> received-routes
show ip bgp vpnv4 all | include <RD>
show ip ospf neighbor
show mpls ldp neighbor
show ip route <PE-LOOPBACK>
```

**Verifikasi — PE (rute dari RR):** `show ip bgp vpnv4 all summary` | `show ip bgp vpnv4 vrf <VRF-NAMA> | include <PREFIX-CE>`
**Risiko / Kesalahan Umum:** Tanpa `send-community extended`, RT tidak terkirim. Tanpa `update-source Loopback0`, sesi bergantung pada satu link. `cluster-id` beda di RR1/RR2. RR dikonfig `next-hop-self`. RR diberi VRF pelanggan. RR tidak masuk OSPF Area 0 atau LDP down sehingga Loopback PE tidak reachable. `route-reflector-client` hanya di `address-family ipv4` bukan `vpnv4`.

### Modul 8: MPLS dan LDP

**Prinsip:** Tanpa label, tidak ada forwarding MPLS.
**Konfigurasi Inti:**

```
ip cef
mpls ip
mpls label protocol ldp
mpls ldp router-id Loopback0 force
interface GigabitEthernet<SLOT>/<PORT>
 mpls ip
 mpls mtu <MPLS-MTU>
```

**Verifikasi:** `show mpls ldp neighbor` | `show mpls forwarding-table`
**Risiko:** CEF nonaktif, MPLS hanya global, MTU tidak sama pada kedua ujung.

### Modul 9: MPLS L3VPN dan VRF

**Prinsip:** Isolasi melalui VRF dan RT. Rute hanya masuk VRF jika RT cocok.
**Konfigurasi Inti:**

```
ip vrf <VRF-NAMA>
 rd <RD>
 route-target export <RT>
 route-target import <RT>
interface GigabitEthernet<SLOT>/<PORT>
 ip vrf forwarding <VRF-NAMA>
 ip address <IP-PE-CE> <MASK-PE-CE>
```

**Verifikasi:** `show ip vrf` | `show ip route vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`
**Risiko:** IP diisi sebelum `ip vrf forwarding` hilang. RT tidak cocok membuat rute ada di BGP, tidak di VRF.

### Modul 10: VRF Lite

**Prinsip:** Isolasi tanpa MPLS, pada satu perangkat atau antar perangkat campus.
**Kapan Dipakai:** Campus multi-building tanpa core MPLS. Bukan pengganti MPLS L3VPN antar site.
**Konfigurasi Inti:** Sama seperti Modul 9, tanpa MPLS dan BGP VPNv4. Routing antar VRF hanya melalui firewall atau route leaking yang disetujui.
**Verifikasi:** `show ip vrf interfaces` | `show ip route vrf <VRF-NAMA>`
**Risiko:** Leaking tanpa kontrol merusak isolasi.

### Modul 11: QoS

**Prinsip:** QoS adalah alokasi kelangkaan, bukan penambah bandwidth.
**Standar:**

| Class | Perlakuan |
| --- | --- |
| Voice | `priority`, policing ketat, antrian terpisah |
| Video | Bandwidth terjamin, drop precedence |
| Data kritis | Bandwidth minimum |
| Default | Sisa bandwidth, WRED jika didukung |

**Konfigurasi Inti:**

```
class-map match-any <CLASS-VOICE>
 match dscp ef
policy-map <POLICY-OUT>
 class <CLASS-VOICE>
  priority <KBPS>
 class class-default
  fair-queue
interface GigabitEthernet<SLOT>/<PORT>
 service-policy output <POLICY-OUT>
```

**Verifikasi:** `show policy-map interface GigabitEthernet<SLOT>/<PORT>`
**Risiko:** Priority tanpa policing membuat voice menekan trafik lain. Marking tidak konsisten di ingress merusak klasifikasi di egress.

### Modul 12: Keamanan Control Plane dan Data Plane

**Prinsip:** Control plane dilindungi agar routing tetap hidup saat serangan atau storm.
**Konfigurasi Inti:**

```
ip access-list extended <ACL-COPP>
 permit ospf any any
 permit tcp any any eq bgp
 permit udp any any eq 646
 deny ip any any
class-map match-all <CLASS-COPP>
 match access-group name <ACL-COPP>
policy-map <POLICY-COPP>
 class <CLASS-COPP>
  police <RATE> conform-action transmit exceed-action drop
control-plane
 service-policy input <POLICY-COPP>
```

**Verifikasi:** `show policy-map control-plane` | `show access-lists`
**Risiko:** CoPP terlalu ketat menjatuhkan adjacency. Terlalu longgar tidak melindungi CPU.

### Modul 13: High Availability dan Fast Convergence

**Prinsip:** Redundansi tanpa pengujian failover bukan redundansi.
**Standar:** Dual P, dual PE per site kritis, dual RR, HSRP/VRRP pada gateway, BFD pada link core dan PE-CE kritis.
**Konfigurasi Inti:**

```
interface GigabitEthernet<SLOT>/<PORT>
 bfd interval <MS> min_rx <MS> multiplier <N>
standby <GROUP> ip <VIP>
standby <GROUP> priority <PRIORITY>
standby <GROUP> preempt
```

**Verifikasi:** `show standby brief` | `show bfd neighbors` | `show ip ospf neighbor`
**Risiko:** Timer BFD terlalu agresif pada link tidak stabil menyeModulkan flapping.

### Modul 14: NTP, DNS, DHCP, NAT

**Prinsip:** Log tanpa waktu sinkron tidak dapat dikorelasikan. NTP adalah prasyarat operasi, bukan pelengkap.
**Konfigurasi Inti:**

```
ntp server <NTP-SERVER-1>
ntp server <NTP-SERVER-2>
ntp authenticate
ip name-server <DNS-SERVER>
ip domain-name <DOMAIN>
```

**Verifikasi:** `show ntp status` | `show ntp associations`
**Risiko:** Satu NTP server, NTP tanpa autentikasi, DHCP tanpa snooping di access.

### Modul 15: Interkoneksi CE

| Model | Kapan Dipakai | Protokol |
| --- | --- | --- |
| eBGP | CE dinamis, banyak prefix | BGP |
| OSPF | CE sudah OSPF | OSPF area terpisah |
| Static | CE satu jalur | Static route |

**Prinsip:** Filter selalu di PE. CE yang salah konfigurasi tidak boleh membanjiri VRF.
**Verifikasi:** `show ip bgp vpnv4 vrf <VRF-NAMA> neighbors` | `show ip route vrf <VRF-NAMA>`

### Modul 16: Implementasi P dan PE

**Prinsip:** P stabil dulu, baru PE bawa layanan. P tidak simpan rute VPN.
**Urutan:** `reload in <MENIT>` > Loopback + OSPF > CEF + MPLS LDP > PE: VRF + BGP VPNv4 + PE-CE.
**Kunci:** `ip vrf forwarding` hapus IP, isi IP setelahnya. `send-community extended` wajib agar RT terkirim.
**Cek:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show mpls forwarding-table` | `show ip bgp vpnv4 all summary` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`
**Jangan:** BGP sebelum LDP stabil. IP sebelum `ip vrf forwarding`. Tanpa `reload in`.

### Modul 17: Implementasi RR dan CE

**Prinsip:** RR tidak forward trafik pelanggan. CE tidak kenal core.
**Urutan:** RR: OSPF + BGP VPNv4, hanya PE sebagai `route-reflector-client`. CE: peering langsung ke PE.
**Kunci:** Filter di PE. Area OSPF PE-CE terpisah dari area 0.
**Cek:** `show ip bgp vpnv4 all summary` | `show ip route vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`
**Jangan:** CE sebagai client RR. RR sebagai PE layanan. Peering via antarmuka fisik.

### Modul 18: Validasi End-to-End dan Cutover

**Prinsip:** Established bukan bukti forwarding. Rute bukan bukti label. Ping kecil bukan bukti MTU.
**Urutan:** OSPF FULL > label terbentuk > rute VPNv4 > rute di VRF > ping VRF + MTU > failover P/PE/gateway.
**Cek:** `show ip ospf neighbor` | `show mpls forwarding-table` | `show ip bgp vpnv4 vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK> size <MTU-TEST> df-bit` | `traceroute vrf <VRF-NAMA> <CE-LOOPBACK>`
**Jangan:** Ping tanpa `vrf`. Tanpa uji MTU, isolasi, failover.

## Bagian IV — Operasi

### Modul 19: Operasi Harian

**Prinsip:** Cari penyimpangan, bukan hanya down. Tanpa baseline, anomali tidak terlihat.
**Lakukan:** Bandingkan ke baseline: adjacency OSPF, neighbor LDP, sesi BGP, rute per VRF, CPU, memori.
**Cek:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show cpu` | `show ntp status` | `show archive config`
**Jangan:** Hanya cek up/down. Abaikan log dan arsip.

### Modul 20: Troubleshooting

**Prinsip:** Ikut lapisan. Jangan ubah BGP jika OSPF belum sehat.
**Alur:** Fisik/L2 > OSPF > LDP/MPLS > BGP VPNv4 > VRF/RT > PE-CE > QoS/CoPP.
**Cek:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show ip route vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>` | `show policy-map control-plane`
**Jangan:** Ubah RT tanpa pahami kebijakan. Ping tanpa `vrf`. Abaikan CoPP.

### Modul 21: Change, Backup, Lifecycle

**Prinsip:** Tanpa titik kembali, perubahan tidak bisa dipertanggungjawabkan.
**Prosedur:**

| Fase | Tindakan |
| --- | --- |
| Sebelum | `reload in <MENIT>` + `archive config` + `copy running-config startup-config` |
| Verifikasi | `show ip ospf neighbor` + `show mpls ldp neighbor` + `show ip bgp vpnv4 all summary` + `ping vrf <VRF-NAMA> <CE-LOOPBACK>` |
| Lolos | `copy running-config startup-config` + `archive config` + `reload cancel` |
| Gagal | `reload` atau `configure replace <ARCHIVE-PATH> force` |

**Lifecycle:** `show version` | `show inventory` | `show license feature` | `show archive config`. Pembaruan citra diverifikasi di Feature Navigator untuk citra yang persis sama.
**Jangan:** Ubah tanpa `reload in`. Backup tidak pernah diuji. Update citra tanpa cek fitur.

### Appendix A: Perintah Verifikasi Inti

```
show ip ospf neighbor
show mpls ldp neighbor
show mpls forwarding-table
show ip bgp vpnv4 all summary
show ip route vrf <VRF-NAMA>
ping vrf <VRF-NAMA> <CE-LOOPBACK>
show policy-map interface GigabitEthernet<SLOT>/<PORT>
show standby brief
show bfd neighbors
show ntp status
show archive config
```

### Appendix B: Placeholder

| Placeholder | Arti |
| --- | --- |
| `<LOOPBACK>` | Loopback0 perangkat |
| `<BGP-AS>` | AS BGP core |
| `<VRF-NAMA>` | Nama VRF |
| `<RT>` / `<RD>` | Route Target / Route Distinguisher |
| `<FROM-VAULT>` | Nilai dari vault, tidak ditulis di dokumen |
| `<MENIT>` | Durasi rollback otomatis |

### Appendix C: Batasan Platform

Verifikasi di Feature Navigator sebelum desain final. MPLS, QoS lanjutan, dan fitur keamanan bervariasi antar platform dan citra.

### Appendix D: Studi Kasus

| Kasus | Desain | Hasil yang Dibuktikan |
| --- | --- | --- |
| A. HQ + Branch + DC, MPLS L3VPN | 2 P, 3 PE, 2 RR, VRF per unit bisnis | Isolasi VRF, failover P/PE |
| B. Campus Multi-Building, VRF Lite | VRF Lite per gedung, tanpa MPLS | Isolasi tanpa core MPLS |
| C. Dual DC + HA + QoS Voice/Video | Dual PE, HSRP, QoS priority voice | Failover gateway, voice tanpa drop |
| D. Migrasi Flat Static ke OSPF + BGP + MPLS | Migrasi bertahap per site, rollback per site | Cutover tanpa gangguan massal |

### Appendix F: Template Baseline Golden Config

**Prinsip:** Satu baseline per peran. Semua perangkat mulai dari baseline yang sama, ditambah konfigurasi peran.

**Baseline Umum — Semua Peran:** Gabungan dari Modul yang sudah ada, bukan konfig baru. Jangan salin ulang dari sini jika sudah terapkan Modul terkait.

| Komponen | Sumber Tunggal |
| --- | --- |
| Hardening, VTY, `enable secret` | Modul 3 |
| NTP, DNS, domain | Modul 14 |
| Archive, backup | Modul 21 |
| Logging | Baris unik di bawah |

```
hostname <HOSTNAME>
logging host <SYSLOG-SERVER>
```

**Tambahan per Peran:**

| Peran | Tambahan |
| --- | --- |
| P | Loopback0 + OSPF area 0 + CEF + MPLS LDP |
| PE | P + VRF + BGP VPNv4 ke RR + PE-CE |
| RR | Loopback0 + OSPF area 0 + BGP VPNv4, `route-reflector-client` ke PE |
| CE | Loopback0 + peering ke PE, eBGP / OSPF / static |

**Verifikasi:** `show running-config` sesuai baseline + `show archive config`

### Appendix G: Matriks Platform vs Fitur & Lisensi

**Prinsip:** Verifikasi di Feature Navigator untuk citra yang persis sama. Tabel ini panduan awal, bukan keputusan final.

| Platform | Peran Cocok | MPLS | Routing Dinamis | Catatan |
| --- | --- | --- | --- | --- |
| 2960-X / WS-C2960 | Access | Tidak | Static / terbatas | Fokus Layer 2, STP, QoS dasar |
| 3560-X / WS-C3560 | Access / Distribution kecil | Terbatas | OSPF / BGP terbatas | Cek lisensi IP Services |
| 3750-X | Distribution / PE kecil | Terbatas | OSPF / BGP | Stack, cek citra dan lisensi |
| 3850 | Distribution / PE | Ya, verifikasi citra | OSPF / BGP | Stack, QoS lanjutan |
| ISR 2900 | PE / CE / Branch | Ya, verifikasi citra | OSPF / BGP | Cek lisensi Data / Security |
| ISR 4000 (4321 / 4331 / 4351) | PE / CE / Branch | Ya, verifikasi citra | OSPF / BGP | Performa dan lisensi per model berbeda |

**Verifikasi:** `show version` | `show license feature` | `show inventory` + Feature Navigator.

### Appendix H: Recovery Darurat

**Prinsip:** Akses console adalah jalan terakhir. Prosedur harus diuji sebelum insiden, bukan saat insiden.

**Password Recovery — IOS  :**

```
1. Console, reboot, kirim Break saat boot
2. rommon> confreg 0x2142
3. rommon> reset
4. Router# copy startup-config running-config
5. Router(config)# enable secret <FROM-VAULT>
6. Router(config)# config-register 0x2102
7. Router# copy running-config startup-config
8. Router# reload
```

**IOS Corrupt / Boot Gagal:**

```
1. Console, cek rommon> dir flash:
2. Jika citra tersedia: rommon> boot flash:<IOS-IMAGE>
3. Jika tidak tersedia: XMODEM / TFTP via ROMMON, atau ganti flash
4. Setelah boot: verifikasi show version, copy running-config startup-config
```

**Config-Register:** `0x2102` normal. `0x2142` bypass startup-config untuk recovery. Kembalikan ke `0x2102` setelah recovery.
**Verifikasi:** `show version | include Configuration register` | `show boot` | `show flash:`

### Appendix I: Matriks Pengujian Failover

**Prinsip:** Redundansi yang tidak diuji dianggap tidak ada.

| Pengujian | Cara | Kriteria Lolos |
| --- | --- | --- |
| P down | `shutdown` satu P / cabut link core | OSPF reconverge, LDP dan BGP kembali, ping VRF pulih |
| PE down | `shutdown` satu PE | CE pindah ke PE cadangan, rute VPNv4 via PE lain |
| RR down | `shutdown` satu RR | Sesi BGP VPNv4 via RR kedua tetap Established |
| Gateway down | `shutdown` gateway aktif HSRP | VIP pindah ke standby, ping gateway pulih |
| Link core down | `shutdown` satu link P-PE / P-P | Trafik pindah ke link lain, tanpa loss berkepanjangan |
| PE-CE down | `shutdown` link PE-CE | Sesi PE-CE cadangan aktif, rute VRF pulih |

**Prosedur Uji:** Lakukan pada jendela yang disepakati. Catat waktu reconverge, hasil ping VRF sebelum dan sesudah, dan arsip konfigurasi setelah uji.
**Verifikasi:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show standby brief` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`

### Appendix J: Studi Kasus Perusahaan Perbankan — HO, DRC, Cabang, ATM

**Profil:** Bank dengan Kantor Pusat (HO), Disaster Recovery Center (DRC) sebagai backup, 20 Kantor Cabang, dan jaringan ATM.
**Masalah:** Core banking, ATM, corporate, dan guest tercampur. Tidak ada DRC yang teruji. Log antar site tidak terkorelasi. Audit gagal karena isolasi tidak terbukti.
**Prinsip:** Satu core, satu kebijakan. HO aktif, DRC standby teruji. Cabang dan ATM tidak saling mencapai. Replikasi HO-DRC terisolasi di VRF sendiri.

#### J.1 Arsitektur

| Site | Peran | Perangkat | VRF |
| --- | --- | --- | --- |
| HO | Aktif, core banking | 2 P, 2 PE, 2 RR | CORE-BANKING, ATM, CORP, GUEST, DRC-REPLIKASI |
| DRC | Backup, standby | 1 P, 1 PE, 1 RR standby | CORE-BANKING, ATM, DRC-REPLIKASI |
| Cabang | Layanan cabang | 1 CE per cabang | CORE-BANKING, CORP |
| ATM | Jaringan ATM | 1 CE per cluster ATM / via CE Cabang | ATM |

**Topologi:** HO — DRC terhubung via 2 link core. Cabang dan ATM hanya peer ke PE HO / PE DRC, tidak peer antar cabang.

#### J.2 Template — PE HO

**Purpose:** PE HO adalah terminasi utama semua VRF perbankan. Semua trafik cabang dan ATM masuk lewat PE HO saat HO aktif.
**When Used:** Dipakai di 2 PE HO. Rollout PE kedua hanya setelah PE pertama stabil dan failover teruji.
**Template:**

```
! === TEMPLATE PE HO — Salin per PE, ganti <...> ===
hostname <PE-HO>
!
interface Loopback0
 description Router-ID PE HO
 ip address <PE-HO-LOOPBACK> 255.255.255.255
!
interface GigabitEthernet<SLOT>/<PORT>
 description to P1
 ip address <IP-PE-HO-P1> <MASK>
 mpls ip
 ip ospf <PROCESS> area 0
 no shutdown
!
interface GigabitEthernet<SLOT>/<PORT>
 description to P2
 ip address <IP-PE-HO-P2> <MASK>
 mpls ip
 ip ospf <PROCESS> area 0
 no shutdown
!
router ospf <PROCESS>
 router-id <PE-HO-LOOPBACK>
 passive-interface default
 no passive-interface GigabitEthernet<SLOT>/<PORT>
 network <NETWORK-PE-HO-LOOPBACK> <WILDCARD> area 0
 network <NETWORK-PE-HO-P1> <WILDCARD> area 0
 network <NETWORK-PE-HO-P2> <WILDCARD> area 0
!
mpls ip
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
ip vrf CORE-BANKING
 rd <RD-BANKING>
 route-target export <RT-BANKING>
 route-target import <RT-BANKING>
ip vrf ATM
 rd <RD-ATM>
 route-target export <RT-ATM>
 route-target import <RT-ATM>
ip vrf CORP
 rd <RD-CORP>
 route-target export <RT-CORP>
 route-target import <RT-CORP>
ip vrf GUEST
 rd <RD-GUEST>
 route-target export <RT-GUEST>
 route-target import <RT-GUEST>
ip vrf DRC-REPLIKASI
 rd <RD-DRC>
 route-target export <RT-DRC>
 route-target import <RT-DRC>
!
interface GigabitEthernet<SLOT>/<PORT>
 description <PE-HO-CE-BANKING>
 ip vrf forwarding CORE-BANKING
 ip address <IP-PE-HO-BANKING> <MASK>
 bfd interval <MS> min_rx <MS> multiplier <N>
 no shutdown
!
interface GigabitEthernet<SLOT>/<PORT>
 description Gateway CORE-BANKING HSRP
 ip address <IP-PE-HO-GW> <MASK>
 standby <GROUP> ip <VIP-BANKING>
 standby <GROUP> priority <PRIORITY-HO>
 standby <GROUP> preempt
 standby <GROUP> track <TRACK-P1>
 no shutdown
!
router bgp <BGP-AS>
 bgp router-id <PE-HO-LOOPBACK>
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor <RR1-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR1-LOOPBACK> update-source Loopback0
 neighbor <RR1-LOOPBACK> password <FROM-VAULT>
 neighbor <RR2-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR2-LOOPBACK> update-source Loopback0
 neighbor <RR2-LOOPBACK> password <FROM-VAULT>
 !
 address-family vpnv4
  neighbor <RR1-LOOPBACK> activate
  neighbor <RR1-LOOPBACK> send-community extended
  neighbor <RR2-LOOPBACK> activate
  neighbor <RR2-LOOPBACK> send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CORE-BANKING
  neighbor <CE-HO-IP> remote-as <CE-AS>
  neighbor <CE-HO-IP> password <FROM-VAULT>
  neighbor <CE-HO-IP> prefix-list <PL-BANKING-IN> in
  neighbor <CE-HO-IP> prefix-list <PL-BANKING-OUT> out
 exit-address-family
 !
 address-family ipv4 vrf ATM
  neighbor <CE-ATM-IP> remote-as <CE-AS>
  neighbor <CE-ATM-IP> password <FROM-VAULT>
  neighbor <CE-ATM-IP> prefix-list <PL-ATM-IN> in
  neighbor <CE-ATM-IP> prefix-list <PL-ATM-OUT> out
 exit-address-family
!
class-map match-any BANKING
 match dscp af31
class-map match-any ATM
 match dscp af21
class-map match-any VOICE
 match dscp ef
policy-map <POLICY-OUT-HO>
 class VOICE
  priority <KBPS-VOICE>
 class BANKING
  bandwidth <KBPS-BANKING>
 class ATM
  bandwidth <KBPS-ATM>
 class class-default
  fair-queue
!
interface GigabitEthernet<SLOT>/<PORT>
 description <PE-HO-CE-BANKING>
 service-policy output <POLICY-OUT-HO>
```

**Per-Block Explanation:**

| Blok | Fungsi | Jika Salah |
| --- | --- | --- |
| Loopback0 + OSPF + MPLS LDP | Identitas PE, label end-to-end | Loopback tidak reachable, VPNv4 tidak terbentuk |
| VRF + RD / RT | Isolasi per layanan | RT salah, VRF terima rute VRF lain |
| `ip vrf forwarding` sebelum IP | Ikat antarmuka ke VRF | IP hilang jika diisi sebelum VRF |
| BGP VPNv4 ke 2 RR | Redundansi refleksi | Hanya 1 RR, RR down = rute hilang |
| BGP VRF + `prefix-list` | Filter CE, CE tidak banjiri VRF | Tanpa filter, prefix liar masuk VRF |
| QoS `BANKING AF31 / ATM AF21 / VOICE EF` | Prioritas perbankan | Tanpa QoS, ATM menekan banking saat congestion |
| HSRP + `track` | Gateway pindah jika link P down | Tanpa track, VIP diam di PE yang link core-nya down |
| BFD PE-CE | Deteksi link down cepat | Tanpa BFD, failover lambat, transaksi timeout |

**Verification:** `show ip bgp vpnv4 all summary` | `show ip route vrf CORE-BANKING` | `show ip route vrf ATM` | `ping vrf CORE-BANKING <CE-HO-LOOPBACK>` | `show standby brief` | `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show bfd neighbors`
**Common Errors:** IP diisi sebelum `ip vrf forwarding`. RT ATM = RT BANKING sehingga isolasi rusak. Hanya peer ke 1 RR. HSRP tanpa `preempt` + `track` sehingga VIP tidak pindah. QoS tanpa `service-policy output`.

#### J.3 Template — PE DRC

**Purpose:** PE DRC mengambil alih layanan jika HO down. Bukan sekadar backup konfigurasi — harus bisa transaksi.
**When Used:** Dipakai di 1 PE DRC. Diuji failover berkala pada jendela yang disepakati. Saat HO aktif, DRC standby.
**Template:**

```
! === TEMPLATE PE DRC — Berbasis PE HO, bedanya di bawah ===
hostname <PE-DRC>
!
interface Loopback0
 ip address <PE-DRC-LOOPBACK> 255.255.255.255
!
router ospf <PROCESS>
 router-id <PE-DRC-LOOPBACK>
 network <NETWORK-PE-DRC-LOOPBACK> <WILDCARD> area 0
!
mpls ip
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
ip vrf CORE-BANKING
 rd <RD-BANKING>
 route-target export <RT-BANKING>
 route-target import <RT-BANKING>
ip vrf ATM
 rd <RD-ATM>
 route-target export <RT-ATM>
 route-target import <RT-ATM>
ip vrf DRC-REPLIKASI
 rd <RD-DRC>
 route-target export <RT-DRC>
 route-target import <RT-DRC>
!
interface GigabitEthernet<SLOT>/<PORT>
 description <PE-DRC-HO-REPLIKASI>
 ip vrf forwarding DRC-REPLIKASI
 ip address <IP-PE-DRC-REPLIKASI> <MASK>
 no shutdown
!
interface GigabitEthernet<SLOT>/<PORT>
 description Gateway CORE-BANKING HSRP DRC
 ip address <IP-PE-DRC-GW> <MASK>
 standby <GROUP> ip <VIP-BANKING>
 standby <GROUP> priority <PRIORITY-DRC>
 standby <GROUP> preempt
 no shutdown
!
router bgp <BGP-AS>
 bgp router-id <PE-DRC-LOOPBACK>
 no bgp default ipv4-unicast
 neighbor <RR1-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR1-LOOPBACK> update-source Loopback0
 neighbor <RR1-LOOPBACK> password <FROM-VAULT>
 neighbor <RR2-LOOPBACK> remote-as <BGP-AS>
 neighbor <RR2-LOOPBACK> update-source Loopback0
 neighbor <RR2-LOOPBACK> password <FROM-VAULT>
 !
 address-family vpnv4
  neighbor <RR1-LOOPBACK> activate
  neighbor <RR1-LOOPBACK> send-community extended
  neighbor <RR2-LOOPBACK> activate
  neighbor <RR2-LOOPBACK> send-community extended
 exit-address-family
 !
 address-family ipv4 vrf DRC-REPLIKASI
  neighbor <PE-HO-REPLIKASI-IP> remote-as <BGP-AS>
  neighbor <PE-HO-REPLIKASI-IP> prefix-list <PL-DRC-IN> in
 exit-address-family
```

**Per-Block Explanation:**

| Blok | Fungsi |
| --- | --- |
| VRF sama dengan HO | DRC bisa terima rute yang sama saat failover |
| VRF DRC-REPLIKASI terpisah | Replikasi tidak tercampur dengan transaksi |
| HSRP priority DRC lebih rendah | VIP di HO saat HO aktif, pindah ke DRC saat HO down |
| BGP VPNv4 ke 2 RR | DRC tetap dapat rute walau 1 RR down |

**Verification:** `show standby brief` | `show ip bgp vpnv4 vrf CORE-BANKING summary` | `ping vrf DRC-REPLIKASI <PE-HO-LOOPBACK>` | uji transaksi core banking dari DRC
**Common Errors:** RD DRC beda dengan HO sehingga replikasi tidak nyambung. HSRP priority DRC = HO sehingga flapping. DRC tidak pernah diuji failover sehingga saat HO down ternyata tidak bisa transaksi.
**Failover Test:** 1. `shutdown` PE HO, verifikasi VIP pindah ke DRC, transaksi uji lolos. 2. `no shutdown` PE HO, verifikasi VIP kembali ke HO, replikasi normal.

#### J.4 Template — CE Cabang

**Purpose:** Template baku untuk 20 cabang. Satu template, dipakai 20x dengan ganti placeholder per cabang.
**When Used:** Setiap cabang baru, pilot 2 cabang dulu, verifikasi transaksi, baru rollout 18 sisa.
**Template — CE Cabang:**

```
! === TEMPLATE CE CABANG ===
hostname <CE-CABANG>
!
interface Loopback0
 description Router-ID CE Cabang
 ip address <CE-CABANG-LOOPBACK> 255.255.255.255
!
interface GigabitEthernet<SLOT>/<PORT>
 description <CE-CABANG-PE>
 ip address <IP-CE-CABANG> <MASK-CE-PE>
 bfd interval <MS> min_rx <MS> multiplier <N>
 no shutdown
!
router bgp <CE-AS>
 bgp router-id <CE-CABANG-LOOPBACK>
 bgp log-neighbor-changes
 neighbor <PE-HO-IP> remote-as <BGP-AS>
 neighbor <PE-HO-IP> description PE-HO
 neighbor <PE-HO-IP> password <FROM-VAULT>
 neighbor <PE-HO-IP> prefix-list <PL-CABANG-OUT> out
 neighbor <PE-DRC-IP> remote-as <BGP-AS>
 neighbor <PE-DRC-IP> description PE-DRC
 neighbor <PE-DRC-IP> password <FROM-VAULT>
 neighbor <PE-DRC-IP> prefix-list <PL-CABANG-OUT> out
 network <NETWORK-CABANG-BANKING> mask <MASK-CABANG>
 network <NETWORK-CABANG-CORP> mask <MASK-CABANG>
```

**Template — PE untuk Cabang:**

```
router bgp <BGP-AS>
 address-family ipv4 vrf CORE-BANKING
  neighbor <CE-CABANG-IP> remote-as <CE-AS>
  neighbor <CE-CABANG-IP> prefix-list <PL-CABANG-IN> in
  neighbor <CE-CABANG-IP> prefix-list <PL-CABANG-OUT> out
 exit-address-family
 address-family ipv4 vrf CORP
  neighbor <CE-CABANG-IP> remote-as <CE-AS>
  neighbor <CE-CABANG-IP> prefix-list <PL-CORP-IN> in
 exit-address-family
```

**Per-Block Explanation:**

| Blok | Fungsi |
| --- | --- |
| CE peer ke PE HO + PE DRC | Cabang tetap jalan jika HO down |
| `prefix-list` di CE dan PE | Filter dua arah, cabang tidak banjiri core |
| BFD PE-CE | Failover HO ke DRC cepat |
| Tanpa peering antar cabang | Isolasi cabang, audit lolos |

**Verification:** `show ip bgp vpnv4 vrf CORE-BANKING neighbors` | `ping vrf CORE-BANKING <CE-CABANG-LOOPBACK>` | uji transaksi cabang | `show bfd neighbors`
**Common Errors:** CE hanya peer ke PE HO, saat HO down cabang ikut down. `network` salah sehingga prefix cabang tidak diiklankan. Prefix-list tidak dipasang sehingga prefix liar masuk VRF.

#### J.5 Template — Jaringan ATM

**Purpose:** ATM adalah jaringan paling sensitif. Harus terisolasi total dari CORP dan GUEST.
**When Used:** Setiap cluster ATM baru. Pilot 5 ATM dulu, verifikasi isolasi + transaksi, baru rollout semua.
**Template — PE untuk ATM:**

```
! === TEMPLATE PE ATM ===
ip vrf ATM
 rd <RD-ATM>
 route-target export <RT-ATM>
 route-target import <RT-ATM>
!
interface GigabitEthernet<SLOT>/<PORT>
 description <PE-ATM>
 ip vrf forwarding ATM
 ip address <IP-PE-ATM> <MASK-PE-ATM>
 no shutdown
!
router bgp <BGP-AS>
 address-family ipv4 vrf ATM
  neighbor <CE-ATM-IP> remote-as <CE-AS>
  neighbor <CE-ATM-IP> password <FROM-VAULT>
  neighbor <CE-ATM-IP> prefix-list <PL-ATM-IN> in
  neighbor <CE-ATM-IP> prefix-list <PL-ATM-OUT> out
 exit-address-family
!
class-map match-any ATM
 match dscp af21
policy-map <POLICY-OUT-ATM>
 class ATM
  bandwidth <KBPS-ATM>
  police <RATE-ATM> conform-action transmit exceed-action drop
 class class-default
  fair-queue
!
interface GigabitEthernet<SLOT>/<PORT>
 description <PE-ATM>
 service-policy output <POLICY-OUT-ATM>
```

**Template — CE ATM:**

```
! === TEMPLATE CE ATM ===
hostname <CE-ATM>
!
interface Loopback0
 ip address <CE-ATM-LOOPBACK> 255.255.255.255
!
interface GigabitEthernet<SLOT>/<PORT>
 description <CE-ATM-PE>
 ip address <IP-CE-ATM> <MASK-CE-ATM>
 no shutdown
!
router bgp <CE-AS>
 bgp router-id <CE-ATM-LOOPBACK>
 neighbor <PE-HO-IP> remote-as <BGP-AS>
 neighbor <PE-HO-IP> password <FROM-VAULT>
 neighbor <PE-DRC-IP> remote-as <BGP-AS>
 neighbor <PE-DRC-IP> password <FROM-VAULT>
 network <NETWORK-ATM> mask <MASK-ATM>
```

**Per-Block Explanation:**

| Blok | Fungsi |
| --- | --- |
| VRF ATM + RT ATM saja | Tidak ada leaking ke CORP / GUEST |
| `prefix-list` ATM | Hanya prefix ATM yang boleh masuk |
| QoS ATM + policing | ATM tidak menekan banking, ATM dibatasi rate |
| CE ATM peer ke PE HO + DRC | ATM tetap jalan saat HO down |

**Verification — Bukti Isolasi (Wajib untuk Audit):** `show ip route vrf ATM` tidak berisi rute CORP | `ping vrf ATM <ATM-SERVER>` berhasil | `ping vrf ATM <CORP-SERVER>` harus gagal | `ping vrf ATM <GUEST-SERVER>` harus gagal | uji transaksi ATM 20x (tarik tunai, cek saldo, transfer)
**Common Errors:** RT ATM = RT CORP sehingga leaking. Tanpa prefix-list, prefix ATM tercampur. Tanpa policing, ATM membanjiri link saat serangan. QoS ATM tidak dipasang sehingga transaksi ATM timeout saat congestion.

#### J.6 Template — Keamanan & Operasi Perbankan

**Purpose:** Hardening, NTP, CoPP, arsip — satu standar untuk semua perangkat perbankan.
**When Used:** Diterapkan di semua PE, P, RR, CE perbankan sebelum cutover.
**Template:**

```
! === TEMPLATE HARDENING + NTP + CoPP — PERBANKAN ===
enable secret <FROM-VAULT>
service password-encryption
no ip http server
no ip http secure-server
!
ntp authenticate
ntp authentication-key <KEY-ID> md5 <FROM-VAULT>
ntp trusted-key <KEY-ID>
ntp server <NTP-SERVER-1> key <KEY-ID>
ntp server <NTP-SERVER-2> key <KEY-ID>
!
ip access-list extended <ACL-COPP>
 permit ospf any any
 permit tcp any any eq bgp
 permit udp any any eq 646
 deny ip any any
class-map match-all <CLASS-COPP>
 match access-group name <ACL-COPP>
policy-map <POLICY-COPP>
 class <CLASS-COPP>
  police <RATE> conform-action transmit exceed-action drop
control-plane
 service-policy input <POLICY-COPP>
!
line vty 0 4
 transport input ssh
 access-class <ACL-VTY-IN> in
 exec-timeout <TIMEOUT>
!
logging host <SYSLOG-SERVER>
!
archive
 log config
  logging enable
  hidekeys
 path <ARCHIVE-PATH>
 write-memory
```

**Per-Block Explanation:**

| Blok | Fungsi |
| --- | --- |
| NTP auth + trusted-key | Log terkorelasi, anti spoofing NTP |
| CoPP | CPU routing terlindungi dari storm |
| VTY + ACL | Hanya SSH dari manajemen |
| Archive + `hidekeys` | Rollback tersedia, password tidak terekspos di log |

**Verification:** `show ntp status` | `show ntp associations` | `show policy-map control-plane` | `show archive config` | `show login`
**Common Errors:** NTP tanpa auth sehingga log bisa dimanipulasi. CoPP terlalu ketat menjatuhkan OSPF / BGP. Archive tanpa `hidekeys` sehingga password terekspos.

**Operasi:** Baseline harian: adjacency OSPF, neighbor LDP, sesi BGP VPNv4, rute per VRF, CPU, NTP, arsip. Uji failover HO-DRC berkala pada jendela yang disepakati.
**Verifikasi Akhir:** Transaksi core banking HO dan DRC lolos, transaksi cabang lolos, transaksi ATM lolos, isolasi VRF terbukti, log terkorelasi via NTP, arsip terbaru tersedia.

**Pola Implementasi Perbankan:**

| Fase | Tindakan |
| --- | --- |
| 1. Core HO | Baseline + OSPF + LDP, verifikasi label |
| 2. DRC | Bangun PE DRC, uji replikasi dan failover HO-DRC |
| 3. Cabang | Template CE, rollout bertahap, filter di PE |
| 4. ATM | VRF ATM terisolasi, uji isolasi dan transaksi |
| 5. Operasi | Hardening, NTP, CoPP, QoS, arsip, uji failover berkala |

Parameter class-map

**Sintaks Dasar:**

```
class-map [match-all | match-any] <CLASS-NAMA>
 match <PARAMETER>
```

**Prinsip:** `match-all` = semua kondisi harus terpenuhi (AND). `match-any` = satu kondisi terpenuhi sudah cukup (OR). Default IOS   adalah `match-all` jika tidak ditulis.

| Parameter | Sintaks | Fungsi | Contoh | Catatan |
| --- | --- | --- | --- | --- |
| `match-all` / `match-any` | `class-map match-any <NAMA>` | Menentukan logika antar kondisi match | `class-map match-any VOICE` | Default `match-all`. Gunakan `match-any` untuk klasifikasi QoS. |
| `match dscp` | `match dscp <NILAI>` | Klasifikasi berdasarkan DSCP | `match dscp ef` | `ef` voice, `af31` banking/WMS, `af21` ATM/POS. Bisa list: `match dscp ef af31` |
| `match ip dscp` | `match ip dscp <NILAI>` | Sama dengan `match dscp`, khusus IPv4 | `match ip dscp ef` | Gunakan `match dscp` untuk IPv4 + IPv6. |
| `match precedence` | `match precedence <0-7>` | Klasifikasi berdasarkan IP Precedence lama | `match precedence 5` | Legacy. Utamakan DSCP. |
| `match ip precedence` | `match ip precedence <0-7>` | Sama dengan di atas, khusus IPv4 | `match ip precedence 5` | Legacy. |
| `match cos` | `match cos <0-7>` | Klasifikasi berdasarkan CoS Layer 2 (802.1p) | `match cos 5` | Hanya pada trunk / antarmuka Layer 2. |
| `match vlan` | `match vlan <VLAN-ID>` | Klasifikasi berdasarkan VLAN | `match vlan <VLAN-VOICE>` | Untuk QoS per VLAN di PE / distribution. |
| `match access-group` | `match access-group name <ACL-NAMA>` | Klasifikasi berdasarkan ACL | `match access-group name <ACL-VOICE>` | Paling fleksibel: IP, port, protokol. |
| `match access-group` (nomor) | `match access-group <ACL-NOMOR>` | Sama, dengan ACL bernomor | `match access-group 100` | Gunakan nama agar mudah dibaca. |
| `match protocol` | `match protocol <PROTOKOL>` | Klasifikasi berdasarkan protokol NBAR | `match protocol http` | Tergantung NBAR dan citra. Tidak semua platform. |
| `match class-map` | `match class-map <CLASS-LAIN>` | Class-map bersarang | `match class-map <CLASS-INDUK>` | Untuk hierarki QoS. |
| `match qos-group` | `match qos-group <0-99>` | Klasifikasi berdasarkan qos-group dari ingress | `match qos-group 1` | qos-group diset di ingress, dipakai di egress. |
| `match mpls experimental` | `match mpls experimental topmost <0-7>` | Klasifikasi berdasarkan EXP MPLS | `match mpls experimental topmost 5` | Untuk QoS di core P / PE MPLS. |
| `match input-interface` | `match input-interface <INTERFACE>` | Klasifikasi berdasarkan antarmuka masuk | `match input-interface Gi0/1` | Untuk QoS per antarmuka masuk. |
| `match packet length` | `match packet length min <MIN> max <MAX>` | Klasifikasi berdasarkan panjang paket | `match packet length min 100 max 1500` | Jarang dipakai untuk voice / banking. |
| `match ip rtp` | `match ip rtp <PORT-AWAL> <PORT-AKHIR>` | Klasifikasi trafik RTP voice | `match ip rtp 16384 32767` | Alternatif jika DSCP tidak ter-marking. |

**Contoh Lengkap — Perbankan:** Definisi class-map: Lihat Modul 11. Nilai bandwidth perbankan: Lihat Appendix J.2. Kombinasi `match dscp` + `match access-group name <ACL-BANKING>` / `<ACL-ATM>` mengikuti tabel parameter di atas.

**Verifikasi:** `show class-map` | `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show policy-map`
**Kesalahan Umum:** Menggunakan `match-all` saat maksudnya `match-any` sehingga trafik tidak terklasifikasi. ACL salah sehingga class-map kosong. DSCP tidak konsisten di ingress dan egress.

### Appendix M: Nilai DSCP

**Prinsip:** DSCP 6 bit, nilai 0-63. Menentukan Per-Hop Behavior (PHB) di setiap hop. Marking harus konsisten di ingress, agar klasifikasi di egress benar.

**Struktur AF:** `AFxy` — `x` = class (1-4), `y` = drop precedence (1 = drop paling rendah, 3 = drop paling tinggi). Contoh: `AF31` lebih diprioritaskan daripada `AF11`, dan `AF31` lebih kecil kemungkinan di-drop daripada `AF33`.

| Nama | DSCP Desimal | DSCP Biner | PHB | Penggunaan Umum | Contoh di Perbankan |
| --- | --- | --- | --- | --- | --- |
| CS0 | 0 | 000000 | Best Effort | Default, tanpa prioritas | GUEST, traffic umum |
| CS1 | 8 | 001000 | Scavenger | Prioritas paling rendah | Backup non-kritis, scavenger |
| AF11 | 10 | 001010 | Assured Forwarding 1, drop rendah | Data kelas 1 | CORP low |
| AF12 | 12 | 001100 | Assured Forwarding 1, drop sedang | Data kelas 1 | CORP medium |
| AF13 | 14 | 001110 | Assured Forwarding 1, drop tinggi | Data kelas 1 | CORP high-drop |
| CS2 | 16 | 010000 | Class Selector 2 | OAM, signaling | Monitoring |
| AF21 | 18 | 010010 | Assured Forwarding 2, drop rendah | Data kelas 2 | ATM, POS |
| AF22 | 20 | 010100 | Assured Forwarding 2, drop sedang | Data kelas 2 | ATM, POS |
| AF23 | 22 | 010110 | Assured Forwarding 2, drop tinggi | Data kelas 2 | ATM, POS |
| CS3 | 24 | 011000 | Class Selector 3 | Signaling, broadcast | Signaling, routing |
| AF31 | 26 | 011010 | Assured Forwarding 3, drop rendah | Data kritis kelas 3 | CORE-BANKING, ERP, WMS |
| AF32 | 28 | 011100 | Assured Forwarding 3, drop sedang | Data kritis kelas 3 | CORE-BANKING |
| AF33 | 30 | 011110 | Assured Forwarding 3, drop tinggi | Data kritis kelas 3 | CORE-BANKING |
| CS4 | 32 | 100000 | Class Selector 4 | Video streaming | Video |
| AF41 | 34 | 100010 | Assured Forwarding 4, drop rendah | Video kelas 4 | Video conference, CCTV |
| AF42 | 36 | 100100 | Assured Forwarding 4, drop sedang | Video kelas 4 | Video conference |
| AF43 | 38 | 100110 | Assured Forwarding 4, drop tinggi | Video kelas 4 | Video conference |
| CS5 | 40 | 101000 | Class Selector 5 | Signaling voice | Signaling VoIP |
| VA | 44 | 101100 | Voice Admit | Voice admit | Voice admit, jika dipakai |
| EF | 46 | 101110 | Expedited Forwarding | Voice, delay dan jitter rendah | VOICE, VoIP |
| CS6 | 48 | 110000 | Class Selector 6 | Network control | OSPF, BGP, LDP |
| CS7 | 56 | 111000 | Class Selector 7 | Network control | OSPF, BGP, LDP |

**Pemetaan yang Dipakai di Panduan Ini:**

| Trafik | DSCP | Alasan |
| --- | --- | --- |
| VOICE | EF (46) | Delay dan jitter paling rendah, priority |
| CORE-BANKING / ERP / WMS | AF31 (26) | Data kritis, drop rendah |
| ATM / POS | AF21 (18) | Data penting, di bawah banking |
| Video / CCTV | AF41 (34) | Bandwidth terjamin, toleran delay |
| CORP / GUEST | CS0 (0) / AF11 (10) | Default, sisa bandwidth |
| Routing / Control | CS6 (48) / CS7 (56) | Tidak boleh di-drop oleh QoS data |

**Pemetaan ke Class-map:** Definisi class-map: Lihat Modul 11. DSCP di tabel di atas dipetakan ke class VOICE (EF), BANKING (AF31), ATM (AF21), VIDEO (AF41).

**Verifikasi:** `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show class-map`
**Kesalahan Umum:** Marking EF untuk semua trafik sehingga priority tidak efektif. DSCP berbeda di ingress dan egress. AF33 dipakai untuk trafik kritis sehingga mudah di-drop saat congestion.

### Appendix N: Golden Config — Route Reflector (RR1 / RR2)

**Prinsip:** RR adalah perangkat kontrol, bukan data plane pelanggan. Konfigurasi harus identik kecuali hostname, Loopback, dan IP link.

**Konfig Lengkap:** Lihat Modul 7 — Konfigurasi Inti RR1. Jangan salin dari dua tempat. Di bawah ini hanya delta RR2 dan uji failover.

**Delta RR2 — Yang Berbeda dari RR1:**

| Item | RR1 | RR2 |
| --- | --- | --- |
| Hostname | `<RR1-HOSTNAME>` | `<RR2-HOSTNAME>` |
| Loopback / router-id | `<RR1-LOOPBACK>` | `<RR2-LOOPBACK>` |
| IP link ke P1 / P2 | `<IP-RR1-P1>` / `<IP-RR1-P2>` | `<IP-RR2-P1>` / `<IP-RR2-P2>` |
| Peer RR | `<RR2-LOOPBACK>` | `<RR1-LOOPBACK>` |
| `cluster-id` | `<CLUSTER-ID>` | Sama — `<CLUSTER-ID>` |

**Failover Test — RR:**

| Test | Perintah | Harapan |
| --- | --- | --- |
| RR1 down | `shutdown` RR1 / cabut link | PE tetap Established ke RR2, rute VPNv4 tetap ada |
| RR1 up kembali | `no shutdown` | PE Established ke RR1 lagi, tidak ada flap VRF |
| RR1-RR2 link down | `show ip bgp vpnv4 all summary` di RR1 dan RR2 | Masing-masing RR tetap punya rute dari PE client-nya |

**Verifikasi:** `show ip bgp vpnv4 all summary` | `show running-config | section router bgp` | `show ip ospf neighbor`
**Kesalahan Umum:** Lupa ganti Loopback RR2 sehingga router-id duplikat. `cluster-id` beda. `route-reflector-client` tertinggal di satu PE. Password BGP tidak sama dengan PE.



