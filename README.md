# Cisco IOS — Implementation & Operations Guide

## Daftar Isi

- Bagian I — Fondasi: Bab 1-4
- Bagian II — Desain Inti: Bab 5-15
- Bagian III — Implementasi: Bab 16-18
- Bagian IV — Operasi: Bab 19-21
- Appendix A-E, N (Golden Config RR), Studi Kasus A-D

## Bagian I — Fondasi

### Bab 1: Ruang Lingkup

**Prinsip:** Satu domain core, satu kebijakan routing, satu standar konfigurasi.
**Ruang Lingkup:** Routing dan switching enterprise. Di luar lingkup: Segment Routing, EVPN penuh, otomasi.
**Standar:** Hostname, Loopback0, deskripsi antarmuka, dan NTP wajib pada semua perangkat.

### Bab 2: Topologi Referensi

| Peran | Jumlah | Fungsi |
| --- | --- | --- |
| P | 2 | Transit MPLS, tanpa VRF |
| PE | 3 | Terminasi VRF, peering BGP VPNv4 ke RR |
| RR | 2 | Refleksi VPNv4, tanpa forwarding pelanggan |
| CE | 6 | Peering ke PE, tanpa visibilitas core |

**Prinsip:** Kegagalan satu P, PE, atau RR tidak boleh menghentikan layanan. Redundansi harus diuji, bukan diasumsikan.

### Bab 3: Hardening Dasar

**Prinsip:** Akses manajemen merupakan permukaan serangan utama.
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
**Risiko:** VTY tanpa ACL dan tanpa `reload in` menyebabkan kondisi terkunci tanpa mekanisme pengembalian.

### Bab 4: Layer 2 dan STP

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
**Risiko:** Port access tanpa BPDU Guard menerima BPDU tidak sah dan mengubah root.

## Bagian II — Desain Inti

### Bab 5: IP Addressing

**Prinsip:** Loopback merupakan identitas routing. Tautan merupakan transit, bukan identitas.
**Standar:** Loopback0 `/32` untuk router-id OSPF, BGP, dan LDP. Tautan point-to-point `/30` atau `/31`. Blok per situs terdokumentasi, tidak tumpang tindih.
**Verifikasi:** `show ip interface brief` | `show ip route connected`

### Bab 6: OSPF

**Prinsip:** OSPF hanya untuk core. CE tidak termasuk dalam area 0 core.
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
**Risiko:** Router-id ganda, autentikasi tidak cocok, `passive-interface` pada tautan core.

### Bab 7: BGP dan Route Reflector

**Prinsip:** Skalabilitas melalui hierarki. PE hanya peer ke RR, bukan full-mesh. RR hanya melakukan refleksi rute, tidak meneruskan lalu lintas pelanggan, tidak memiliki VRF pelanggan.
**Kondisi Penggunaan:** Wajib jika PE > 2. Tanpa RR, PE wajib full-mesh iBGP VPNv4. Dengan dua RR, satu RR yang tidak aktif tidak menghentikan refleksi.
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

**Konfigurasi Inti — RR1 (RR2 identik, dengan perbedaan pada Loopback dan hostname):**

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
| `no bgp default ipv4-unicast` | RR hanya mengaktifkan VPNv4 secara eksplisit, tidak membawa IPv4 unicast | RR tanpa sengaja menjadi transit IPv4 |
| `bgp cluster-id <CLUSTER-ID>` | Sama pada RR1 dan RR2, satu klaster redundan | Cluster-id yang berbeda menyebabkan proteksi loop gagal dan rute menjadi ganda |
| `route-reflector-client` di VPNv4 | Menandai PE sebagai client, agar RR merefleksikan rute antar client | Tanpa konfigurasi ini, RR tidak merefleksikan rute PE ke PE lain |
| `send-community extended` | Mengirim RT melalui extended community | RT hilang, VRF tidak menerima rute |
| `update-source Loopback0` | Sesi BGP melalui Loopback, tetap bertahan saat satu tautan terputus | Sesi terputus saat tautan fisik terputus |
| Tanpa `next-hop-self` pada RR | Next-hop PE asal dipertahankan, LSP ujung ke ujung tetap utuh | `next-hop-self` pada RR memutus LSP sehingga lalu lintas terbuang |
| Tanpa VRF pada RR | RR tidak meneruskan lalu lintas pelanggan, hanya melakukan kontrol | VRF pada RR membebani RR dan merusak desain |

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
**Risiko / Kesalahan Umum:** Tanpa `send-community extended`, RT tidak terkirim. Tanpa `update-source Loopback0`, sesi bergantung pada satu tautan. `cluster-id` berbeda pada RR1/RR2. RR dikonfigurasi dengan `next-hop-self`. RR diberi VRF pelanggan. RR tidak termasuk dalam OSPF Area 0 atau LDP tidak aktif sehingga Loopback PE tidak terjangkau. `route-reflector-client` hanya pada `address-family ipv4`, bukan `vpnv4`.

### Bab 8: MPLS dan LDP

**Prinsip:** Tanpa label, tidak terdapat penerusan MPLS.
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
**Risiko:** CEF tidak aktif, MPLS hanya global, MTU tidak sama pada kedua ujung.

### Bab 9: MPLS L3VPN dan VRF

**Prinsip:** Isolasi melalui VRF dan RT. Rute hanya masuk ke VRF jika RT cocok.
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
**Risiko:** IP diisi sebelum `ip vrf forwarding` hilang. RT yang tidak cocok menyebabkan rute terdapat pada BGP, tetapi tidak pada VRF.

### Bab 10: VRF Lite

**Prinsip:** Isolasi tanpa MPLS, pada satu perangkat atau antar perangkat kampus.
**Kondisi Penggunaan:** Kampus multi-gedung tanpa core MPLS. Bukan pengganti MPLS L3VPN antar situs.
**Konfigurasi Inti:** Sama seperti Bab 9, tanpa MPLS dan BGP VPNv4. Routing antar VRF hanya melalui firewall atau route leaking yang telah disetujui.
**Verifikasi:** `show ip vrf interfaces` | `show ip route vrf <VRF-NAMA>`
**Risiko:** Route leaking tanpa kontrol merusak isolasi.

### Bab 11: QoS

**Prinsip:** QoS merupakan alokasi kelangkaan, bukan penambah lebar pita.
**Standar:**

| Class | Perlakuan |
| --- | --- |
| Voice | `priority`, policing ketat, antrian terpisah |
| Video | Lebar pita terjamin, prioritas pengguguran |
| Data kritis | Lebar pita minimum |
| Default | Sisa lebar pita, WRED jika didukung |

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
**Risiko:** Priority tanpa policing menyebabkan voice menekan lalu lintas lain. Penandaan yang tidak konsisten pada ingress merusak klasifikasi pada egress.

### Bab 12: Keamanan Control Plane dan Data Plane

**Prinsip:** Control plane dilindungi agar routing tetap beroperasi saat serangan atau lonjakan lalu lintas.
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
**Risiko:** CoPP yang terlalu ketat menjatuhkan adjacency. CoPP yang terlalu longgar tidak melindungi CPU.

### Bab 13: Ketersediaan Tinggi dan Konvergensi Cepat

**Prinsip:** Redundansi tanpa pengujian pengalihan otomatis bukan merupakan redundansi.
**Standar:** Ganda P, ganda PE per situs kritis, ganda RR, HSRP/VRRP pada gateway, BFD pada tautan core dan PE-CE kritis.
**Konfigurasi Inti:**

```
interface GigabitEthernet<SLOT>/<PORT>
 bfd interval <MS> min_rx <MS> multiplier <N>
standby <GROUP> ip <VIP>
standby <GROUP> priority <PRIORITY>
standby <GROUP> preempt
```

**Verifikasi:** `show standby brief` | `show bfd neighbors` | `show ip ospf neighbor`
**Risiko:** Timer BFD yang terlalu agresif pada tautan tidak stabil menyebabkan ketidakstabilan status naik-turun berulang.

### Bab 14: NTP, DNS, DHCP, NAT

**Prinsip:** Log tanpa waktu sinkron tidak dapat dikorelasikan. NTP merupakan prasyarat operasi, bukan pelengkap.
**Konfigurasi Inti:**

```
ntp server <NTP-SERVER-1>
ntp server <NTP-SERVER-2>
ntp authenticate
ip name-server <DNS-SERVER>
ip domain-name <DOMAIN>
```

**Verifikasi:** `show ntp status` | `show ntp associations`
**Risiko:** Satu server NTP, NTP tanpa autentikasi, DHCP tanpa snooping pada access.

### Bab 15: Interkoneksi CE

| Model | Kondisi Penggunaan | Protokol |
| --- | --- | --- |
| eBGP | CE dinamis, banyak prefix | BGP |
| OSPF | CE sudah OSPF | OSPF area terpisah |
| Static | CE satu jalur | Static route |

**Prinsip:** Filter selalu diterapkan pada PE. CE yang salah konfigurasi tidak diperbolehkan membebani VRF secara berlebihan.
**Verifikasi:** `show ip bgp vpnv4 vrf <VRF-NAMA> neighbors` | `show ip route vrf <VRF-NAMA>`

## Bagian III — Implementasi

### Bab 16: Implementasi P dan PE

**Prinsip:** P distabilkan terlebih dahulu, kemudian PE membawa layanan. P tidak menyimpan rute VPN.
**Urutan:** `reload in <MENIT>` > Loopback + OSPF > CEF + MPLS LDP > PE: VRF + BGP VPNv4 + PE-CE.
**Kunci:** `ip vrf forwarding` hapus IP, isi IP setelahnya. `send-community extended` wajib agar RT terkirim.
**Pemeriksaan:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show mpls forwarding-table` | `show ip bgp vpnv4 all summary` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`
**Larangan:** BGP sebelum LDP stabil. IP sebelum `ip vrf forwarding`. Tanpa `reload in`.

### Bab 17: Implementasi RR dan CE

**Prinsip:** RR tidak meneruskan lalu lintas pelanggan. CE tidak mengenal core.
**Urutan:** RR: OSPF + BGP VPNv4, hanya PE sebagai `route-reflector-client`. CE: peering langsung ke PE.
**Kunci:** Filter diterapkan pada PE. Area OSPF PE-CE terpisah dari area 0.
**Pemeriksaan:** `show ip bgp vpnv4 all summary` | `show ip route vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`
**Larangan:** CE sebagai client RR. RR sebagai PE layanan. Peering melalui antarmuka fisik.

### Bab 18: Validasi End-to-End dan Cutover

**Prinsip:** Status Established bukan merupakan bukti penerusan. Rute bukan merupakan bukti label. Ping kecil bukan merupakan bukti MTU.
**Urutan:** OSPF FULL > label terbentuk > rute VPNv4 > rute pada VRF > ping VRF + MTU > pengalihan P/PE/gateway.
**Pemeriksaan:** `show ip ospf neighbor` | `show mpls forwarding-table` | `show ip bgp vpnv4 vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK> size <MTU-TEST> df-bit` | `traceroute vrf <VRF-NAMA> <CE-LOOPBACK>`
**Larangan:** Ping tanpa `vrf`. Tanpa uji MTU, isolasi, dan pengalihan.

## Bagian IV — Operasi

### Bab 19: Operasi Harian

**Prinsip:** Penyimpangan perlu dicari, bukan hanya status tidak aktif. Tanpa garis dasar, anomali tidak terlihat.
**Tindakan:** Pembandingan terhadap garis dasar: adjacency OSPF, neighbor LDP, sesi BGP, rute per VRF, CPU, memori.
**Pemeriksaan:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show cpu` | `show ntp status` | `show archive config`
**Larangan:** Hanya memeriksa status aktif/tidak aktif. Mengabaikan log dan arsip.

### Bab 20: Troubleshooting

**Prinsip:** Penelusuran mengikuti lapisan. BGP tidak diubah jika OSPF belum sehat.
**Alur:** Fisik/L2 > OSPF > LDP/MPLS > BGP VPNv4 > VRF/RT > PE-CE > QoS/CoPP.
**Pemeriksaan:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show ip route vrf <VRF-NAMA>` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>` | `show policy-map control-plane`
**Larangan:** Mengubah RT tanpa memahami kebijakan. Ping tanpa `vrf`. Abaikan CoPP.

### Bab 21: Perubahan, Pencadangan, Siklus Hidup

**Prinsip:** Tanpa titik kembali, perubahan tidak dapat dipertanggungjawabkan.
**Prosedur:**

| Fase | Tindakan |
| --- | --- |
| Sebelum | `reload in <MENIT>` + `archive config` + `copy running-config startup-config` |
| Verifikasi | `show ip ospf neighbor` + `show mpls ldp neighbor` + `show ip bgp vpnv4 all summary` + `ping vrf <VRF-NAMA> <CE-LOOPBACK>` |
| Lolos | `copy running-config startup-config` + `archive config` + `reload cancel` |
| Gagal | `reload` atau `configure replace <ARCHIVE-PATH> force` |

**Siklus Hidup:** `show version` | `show inventory` | `show license feature` | `show archive config`. Pembaruan berkas citra diverifikasi pada Feature Navigator untuk berkas citra yang persis sama.
**Larangan:** Mengubah tanpa `reload in`. Pencadangan tidak pernah diuji. Pembaruan citra tanpa pemeriksaan fitur.

## Appendix

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
| `<MENIT>` | Durasi pengembalian otomatis |

### Appendix C: Batasan Platform

Verifikasi dilakukan pada Feature Navigator sebelum desain final. MPLS, QoS lanjutan, dan fitur keamanan bervariasi antar platform dan berkas citra.

### Appendix D: Studi Kasus

| Kasus | Desain | Hasil yang Dibuktikan |
| --- | --- | --- |
| A. HQ + Branch + DC, MPLS L3VPN | 2 P, 3 PE, 2 RR, VRF per unit bisnis | Isolasi VRF, pengalihan P/PE |
| B. Kampus Multi-Building, VRF Lite | VRF Lite per gedung, tanpa MPLS | Isolasi tanpa core MPLS |
| C. Dual DC + HA + QoS Voice/Video | Dual PE, HSRP, QoS priority voice | Pengalihan gateway, voice tanpa pengguguran |
| D. Migrasi Flat Static ke OSPF + BGP + MPLS | Migrasi bertahap per situs, pengembalian per situs | Cutover tanpa gangguan massal |

### Appendix F: Templat Garis Dasar Golden Config

**Prinsip:** Satu garis dasar per peran. Semua perangkat dimulai dari garis dasar yang sama, ditambah konfigurasi peran.

**Garis dasar Umum — Semua Peran:** Gabungan dari Bab yang sudah ada, bukan konfigurasi baru. Penyalinan ulang dari bagian ini tidak diperlukan jika Bab terkait sudah diterapkan.

| Komponen | Sumber Tunggal |
| --- | --- |
| Hardening, VTY, `enable secret` | Bab 3 |
| NTP, DNS, domain | Bab 14 |
| Archive, pencadangan | Bab 21 |
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

**Verifikasi:** `show running-config` sesuai garis dasar + `show archive config`

### Appendix G: Matriks Platform vs Fitur & Lisensi

**Prinsip:** Verifikasi dilakukan pada Feature Navigator untuk berkas citra yang persis sama. Tabel ini merupakan panduan awal, bukan keputusan final.

| Platform | Peran Cocok | MPLS | Routing Dinamis | Catatan |
| --- | --- | --- | --- | --- |
| 2960-X / WS-C2960 | Access | Tidak | Static / terbatas | Fokus Layer 2, STP, QoS dasar |
| 3560-X / WS-C3560 | Access / Distribution kecil | Terbatas | OSPF / BGP terbatas | Pemeriksaan lisensi IP Services diperlukan |
| 3750-X | Distribution / PE kecil | Terbatas | OSPF / BGP | Stack, pemeriksaan berkas citra dan lisensi diperlukan |
| 3850 | Distribution / PE | Ya, verifikasi citra | OSPF / BGP | Stack, QoS lanjutan |
| ISR 2900 | PE / CE / Branch | Ya, verifikasi citra | OSPF / BGP | Pemeriksaan lisensi Data / Security diperlukan |
| ISR 4000 (4321 / 4331 / 4351) | PE / CE / Branch | Ya, verifikasi citra | OSPF / BGP | Performa dan lisensi per model berbeda |

**Verifikasi:** `show version` | `show license feature` | `show inventory` + Feature Navigator.

### Appendix H: Recovery Darurat

**Prinsip:** Akses console merupakan jalan terakhir. Prosedur wajib diuji sebelum insiden, bukan saat insiden.

**Password Recovery — IOS :**

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
1. Console, periksa rommon> dir flash:
2. Jika citra tersedia: rommon> boot flash:<IOS-IMAGE>
3. Jika tidak tersedia: XMODEM / TFTP melalui ROMMON, atau ganti flash
4. Setelah boot: verifikasi show version, copy running-config startup-config
```

**Config-Register:** `0x2102` normal. `0x2142` bypass startup-config untuk recovery. Kembalikan ke `0x2102` setelah recovery.
**Verifikasi:** `show version | include Configuration register` | `show boot` | `show flash:`

### Appendix I: Matriks Pengujian Pengalihan

**Prinsip:** Redundansi yang tidak diuji dianggap tidak ada.

| Pengujian | Cara | Kriteria Lolos |
| --- | --- | --- |
| P tidak aktif | `shutdown` satu P / cabut tautan core | OSPF konvergensi ulang, LDP dan BGP kembali, ping VRF pulih |
| PE tidak aktif | `shutdown` satu PE | CE berpindah ke PE cadangan, rute VPNv4 melalui PE lain |
| RR tidak aktif | `shutdown` satu RR | Sesi BGP VPNv4 melalui RR kedua tetap Established |
| Gateway tidak aktif | `shutdown` gateway aktif HSRP | VIP berpindah ke standby, ping gateway pulih |
| Tautan core tidak aktif | `shutdown` satu tautan P-PE / P-P | Lalu lintas berpindah ke tautan lain, tanpa kehilangan paket berkepanjangan |
| PE-CE tidak aktif | `shutdown` tautan PE-CE | Sesi PE-CE cadangan aktif, rute VRF pulih |

**Prosedur Uji:** Pengujian dilaksanakan pada jendela yang disepakati. Waktu konvergensi ulang, hasil ping VRF sebelum dan sesudah, serta arsip konfigurasi setelah uji dicatat.
**Verifikasi:** `show ip ospf neighbor` | `show mpls ldp neighbor` | `show ip bgp vpnv4 all summary` | `show standby brief` | `ping vrf <VRF-NAMA> <CE-LOOPBACK>`

### Appendix J: Studi Kasus Perusahaan Perbankan — HO, DRC, Cabang, ATM

**Profil:** Bank dengan Kantor Pusat (HO), Disaster Recovery Center (DRC) sebagai cadangan, 20 Kantor Cabang, dan jaringan ATM.
**Masalah:** Core banking, ATM, corporate, dan guest tercampur. Tidak terdapat DRC yang teruji. Log antar situs tidak terkorelasi. Audit gagal karena isolasi tidak terbukti.
**Prinsip:** Satu core, satu kebijakan. HO aktif, DRC standby teruji. Cabang dan ATM tidak saling mencapai. Replikasi HO-DRC terisolasi pada VRF sendiri.

#### J.1 Arsitektur

| Situs | Peran | Perangkat | VRF |
| --- | --- | --- | --- |
| HO | Aktif, core banking | 2 P, 2 PE, 2 RR | CORE-BANKING, ATM, CORP, GUEST, DRC-REPLIKASI |
| DRC | Cadangan, standby | 1 P, 1 PE, 1 RR standby | CORE-BANKING, ATM, DRC-REPLIKASI |
| Cabang | Layanan cabang | 1 CE per cabang | CORE-BANKING, CORP |
| ATM | Jaringan ATM | 1 CE per cluster ATM / melalui CE Cabang | ATM |

**Topologi:** HO — DRC terhubung melalui dua tautan core. Cabang dan ATM hanya peer ke PE HO / PE DRC, tidak peer antar cabang.

#### J.2 Templat — PE HO

**Tujuan:** PE HO merupakan terminasi utama semua VRF perbankan. Semua lalu lintas cabang dan ATM masuk melalui PE HO saat HO aktif.
**Kondisi Penggunaan:** Digunakan pada dua PE HO. Rollout PE kedua hanya dilakukan setelah PE pertama stabil dan pengalihan teruji.
**Templat:**

```
! === TEMPLAT PE HO — Templat per PE, ganti <...> ===
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

**Penjelasan per Blok:**

| Blok | Fungsi | Jika Salah |
| --- | --- | --- |
| Loopback0 + OSPF + MPLS LDP | Identitas PE, label end-to-end | Loopback tidak terjangkau, VPNv4 tidak terbentuk |
| VRF + RD / RT | Isolasi per layanan | RT salah, VRF menerima rute VRF lain |
| `ip vrf forwarding` sebelum IP | Ikat antarmuka ke VRF | IP hilang jika diisi sebelum VRF |
| BGP VPNv4 ke 2 RR | Redundansi refleksi | Hanya satu RR, RR tidak aktif menyebabkan rute hilang |
| BGP VRF + `prefix-list` | Filter CE, CE tidak membebani VRF secara berlebihan | Tanpa filter, prefix tidak sah masuk ke VRF |
| QoS `BANKING AF31 / ATM AF21 / VOICE EF` | Prioritas perbankan | Tanpa QoS, ATM menekan banking saat kemacetan |
| HSRP + `track` | Gateway berpindah jika tautan P terputus | Tanpa track, VIP tetap bertahan pada PE yang tautan core-nya terputus |
| BFD PE-CE | Deteksi tautan terputus secara cepat | Tanpa BFD, pengalihan lambat, transaksi melampaui batas waktu |

**Verifikasi:** `show ip bgp vpnv4 all summary` | `show ip route vrf CORE-BANKING` | `show ip route vrf ATM` | `ping vrf CORE-BANKING <CE-HO-LOOPBACK>` | `show standby brief` | `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show bfd neighbors`
**Kesalahan Umum:** IP diisi sebelum `ip vrf forwarding`. RT ATM = RT BANKING sehingga isolasi rusak. Hanya peer ke satu RR. HSRP tanpa `preempt` + `track` sehingga VIP tidak berpindah. QoS tanpa `service-policy output`.

#### J.3 Templat — PE DRC

**Tujuan:** PE DRC mengambil alih layanan jika HO tidak aktif. PE DRC bukan sekadar cadangan konfigurasi, PE DRC wajib dapat melayani transaksi.
**Kondisi Penggunaan:** Digunakan pada satu PE DRC. Pengalihan diuji secara berkala pada jendela yang disepakati. Saat HO aktif, DRC standby.
**Templat:**

```
! === TEMPLAT PE DRC — Berbasis PE HO, perbedaan tercantum di bawah ===
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

**Penjelasan per Blok:**

| Blok | Fungsi |
| --- | --- |
| VRF sama dengan HO | DRC dapat menerima rute yang sama saat pengalihan |
| VRF DRC-REPLIKASI terpisah | Replikasi tidak bercampur dengan transaksi |
| HSRP priority DRC lebih rendah | VIP berada pada HO saat HO aktif, berpindah ke DRC saat HO tidak aktif |
| BGP VPNv4 ke 2 RR | DRC tetap memperoleh rute walaupun satu RR tidak aktif |

**Verifikasi:** `show standby brief` | `show ip bgp vpnv4 vrf CORE-BANKING summary` | `ping vrf DRC-REPLIKASI <PE-HO-LOOPBACK>` | uji transaksi core banking dari DRC
**Kesalahan Umum:** RD DRC berbeda dengan HO sehingga replikasi tidak tersambung. HSRP priority DRC = HO sehingga terjadi ketidakstabilan status naik-turun berulang. DRC tidak pernah diuji pengalihannya sehingga saat HO tidak aktif ternyata tidak dapat melayani transaksi.
**Uji Pengalihan:** 1. `shutdown` PE HO, verifikasi VIP berpindah ke DRC, transaksi uji lolos. 2. `no shutdown` PE HO, verifikasi VIP kembali ke HO, replikasi normal.

#### J.4 Templat — CE Cabang

**Tujuan:** Templat baku untuk 20 cabang. Satu templat digunakan 20x dengan penggantian placeholder per cabang.
**Kondisi Penggunaan:** Setiap cabang baru, pilot dua cabang terlebih dahulu, verifikasi transaksi, kemudian rollout 18 cabang sisanya.
**Templat — CE Cabang:**

```
! === TEMPLAT CE CABANG ===
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

**Templat — PE untuk Cabang:**

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

**Penjelasan per Blok:**

| Blok | Fungsi |
| --- | --- |
| CE peer ke PE HO + PE DRC | Cabang tetap beroperasi jika HO tidak aktif |
| `prefix-list` di CE dan PE | Filter dua arah, cabang tidak membebani core secara berlebihan |
| BFD PE-CE | Pengalihan HO ke DRC secara cepat |
| Tanpa peering antar cabang | Isolasi cabang, audit lolos |

**Verifikasi:** `show ip bgp vpnv4 vrf CORE-BANKING neighbors` | `ping vrf CORE-BANKING <CE-CABANG-LOOPBACK>` | uji transaksi cabang | `show bfd neighbors`
**Kesalahan Umum:** CE hanya peer ke PE HO, saat HO tidak aktif cabang ikut tidak aktif. `network` salah sehingga prefix cabang tidak diiklankan. Prefix-list tidak dipasang sehingga prefix tidak sah masuk ke VRF.

#### J.5 Templat — Jaringan ATM

**Tujuan:** ATM merupakan jaringan paling sensitif. ATM wajib terisolasi total dari CORP dan GUEST.
**Kondisi Penggunaan:** Setiap cluster ATM baru. Pilot lima ATM terlebih dahulu, verifikasi isolasi dan transaksi, kemudian rollout seluruhnya.
**Templat — PE untuk ATM:**

```
! === TEMPLAT PE ATM ===
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

**Templat — CE ATM:**

```
! === TEMPLAT CE ATM ===
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

**Penjelasan per Blok:**

| Blok | Fungsi |
| --- | --- |
| VRF ATM + RT ATM saja | Tidak terdapat leaking ke CORP / GUEST |
| `prefix-list` ATM | Hanya prefix ATM yang boleh masuk |
| QoS ATM + policing | ATM tidak menekan banking, ATM dibatasi rate-nya |
| CE ATM peer ke PE HO + DRC | ATM tetap beroperasi saat HO tidak aktif |

**Verifikasi — Bukti Isolasi (Wajib untuk Audit):** `show ip route vrf ATM` tidak berisi rute CORP | `ping vrf ATM <ATM-SERVER>` berhasil | `ping vrf ATM <CORP-SERVER>` wajib gagal | `ping vrf ATM <GUEST-SERVER>` wajib gagal | pengujian transaksi ATM 20x (tarik tunai, pemeriksaan saldo, transfer)
**Kesalahan Umum:** RT ATM = RT CORP sehingga terjadi leaking. Tanpa prefix-list, prefix ATM bercampur. Tanpa policing, ATM membebani tautan secara berlebihan saat serangan. QoS ATM tidak dipasang sehingga transaksi ATM melampaui batas waktu saat kemacetan.

#### J.6 Templat — Keamanan & Operasi Perbankan

**Tujuan:** Hardening, NTP, CoPP, arsip — satu standar untuk semua perangkat perbankan.
**Kondisi Penggunaan:** Diterapkan pada semua PE, P, RR, CE perbankan sebelum cutover.
**Templat:**

```
! === TEMPLAT HARDENING + NTP + CoPP — PERBANKAN ===
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

**Penjelasan per Blok:**

| Blok | Fungsi |
| --- | --- |
| NTP auth + trusted-key | Log terkorelasi, anti-spoofing NTP |
| CoPP | CPU routing terlindungi dari lonjakan lalu lintas |
| VTY + ACL | Hanya SSH dari manajemen |
| Archive + `hidekeys` | Pengembalian tersedia, password tidak terekspos pada log |

**Verifikasi:** `show ntp status` | `show ntp associations` | `show policy-map control-plane` | `show archive config` | `show login`
**Kesalahan Umum:** NTP tanpa auth sehingga log dapat dimanipulasi. CoPP yang terlalu ketat menjatuhkan OSPF / BGP. Archive tanpa `hidekeys` sehingga password terekspos.

**Operasi:** Garis dasar harian: adjacency OSPF, neighbor LDP, sesi BGP VPNv4, rute per VRF, CPU, NTP, arsip. Uji pengalihan HO-DRC berkala pada jendela yang disepakati.
**Verifikasi Akhir:** Transaksi core banking HO dan DRC lolos, transaksi cabang lolos, transaksi ATM lolos, isolasi VRF terbukti, log terkorelasi melalui NTP, arsip terbaru tersedia.

**Pola Implementasi Perbankan:**

| Fase | Tindakan |
| --- | --- |
| 1. Core HO | Garis dasar + OSPF + LDP, verifikasi label |
| 2. DRC | Bangun PE DRC, uji replikasi dan pengalihan HO-DRC |
| 3. Cabang | Templat CE, rollout bertahap, filter pada PE |
| 4. ATM | VRF ATM terisolasi, uji isolasi dan transaksi |
| 5. Operasi | Hardening, NTP, CoPP, QoS, arsip, uji pengalihan berkala |

Parameter class-map

**Sintaks Dasar:**

```
class-map [match-all | match-any] <CLASS-NAMA>
 match <PARAMETER>
```

**Prinsip:** `match-all` = semua kondisi harus terpenuhi (AND). `match-any` = satu kondisi terpenuhi sudah cukup (OR). Default IOS adalah `match-all` jika tidak ditulis.

| Parameter | Sintaks | Fungsi | Contoh | Catatan |
| --- | --- | --- | --- | --- |
| `match-all` / `match-any` | `class-map match-any <NAMA>` | Menentukan logika antar kondisi match | `class-map match-any VOICE` | Default `match-all`. `match-any` digunakan untuk klasifikasi QoS. |
| `match dscp` | `match dscp <NILAI>` | Klasifikasi berdasarkan DSCP | `match dscp ef` | `ef` voice, `af31` banking/WMS, `af21` ATM/POS. Dapat berupa daftar: `match dscp ef af31` |
| `match ip dscp` | `match ip dscp <NILAI>` | Sama dengan `match dscp`, khusus IPv4 | `match ip dscp ef` | `match dscp` digunakan untuk IPv4 + IPv6. |
| `match precedence` | `match precedence <0-7>` | Klasifikasi berdasarkan IP Precedence lama | `match precedence 5` | Legacy. Utamakan DSCP. |
| `match ip precedence` | `match ip precedence <0-7>` | Sama dengan di atas, khusus IPv4 | `match ip precedence 5` | Legacy. |
| `match cos` | `match cos <0-7>` | Klasifikasi berdasarkan CoS Layer 2 (802.1p) | `match cos 5` | Hanya pada trunk / antarmuka Layer 2. |
| `match vlan` | `match vlan <VLAN-ID>` | Klasifikasi berdasarkan VLAN | `match vlan <VLAN-VOICE>` | Untuk QoS per VLAN pada PE / distribution. |
| `match access-group` | `match access-group name <ACL-NAMA>` | Klasifikasi berdasarkan ACL | `match access-group name <ACL-VOICE>` | Paling fleksibel: IP, port, protokol. |
| `match access-group` (nomor) | `match access-group <ACL-NOMOR>` | Sama, dengan ACL bernomor | `match access-group 100` | Nama digunakan agar mudah dibaca. |
| `match protocol` | `match protocol <PROTOKOL>` | Klasifikasi berdasarkan protokol NBAR | `match protocol http` | Bergantung pada NBAR dan berkas citra. Tidak semua platform. |
| `match class-map` | `match class-map <CLASS-LAIN>` | Class-map bersarang | `match class-map <CLASS-INDUK>` | Untuk hierarki QoS. |
| `match qos-group` | `match qos-group <0-99>` | Klasifikasi berdasarkan qos-group dari ingress | `match qos-group 1` | qos-group ditetapkan pada ingress, digunakan pada egress. |
| `match mpls experimental` | `match mpls experimental topmost <0-7>` | Klasifikasi berdasarkan EXP MPLS | `match mpls experimental topmost 5` | Untuk QoS di core P / PE MPLS. |
| `match input-interface` | `match input-interface <INTERFACE>` | Klasifikasi berdasarkan antarmuka masuk | `match input-interface Gi0/1` | Untuk QoS per antarmuka masuk. |
| `match packet length` | `match packet length min <MIN> max <MAX>` | Klasifikasi berdasarkan panjang paket | `match packet length min 100 max 1500` | Jarang digunakan untuk voice / banking. |
| `match ip rtp` | `match ip rtp <PORT-AWAL> <PORT-AKHIR>` | Klasifikasi lalu lintas RTP voice | `match ip rtp 16384 32767` | Alternatif jika DSCP tidak diberi penanda. |

**Contoh Lengkap — Perbankan:** Definisi class-map tercantum pada Bab 11. Nilai lebar pita perbankan tercantum pada Appendix J.2. Kombinasi `match dscp` + `match access-group name <ACL-BANKING>` / `<ACL-ATM>` mengikuti tabel parameter di atas.

**Verifikasi:** `show class-map` | `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show policy-map`
**Kesalahan Umum:** Menggunakan `match-all` saat maksudnya `match-any` sehingga lalu lintas tidak terklasifikasi. ACL salah sehingga class-map kosong. DSCP tidak konsisten pada ingress dan egress.

### Appendix M: Nilai DSCP

**Prinsip:** DSCP 6 bit, nilai 0-63. DSCP menentukan Per-Hop Behavior (PHB) pada setiap hop. Penandaan wajib konsisten pada ingress, agar klasifikasi pada egress benar.

**Struktur AF:** `AFxy` — `x` = class (1-4), `y` = prioritas pengguguran (1 = pengguguran paling rendah, 3 = pengguguran paling tinggi). Contoh: `AF31` lebih diprioritaskan daripada `AF11`, dan `AF31` lebih kecil kemungkinan dibuang daripada `AF33`.

| Nama | DSCP Desimal | DSCP Biner | PHB | Penggunaan Umum | Contoh di Perbankan |
| --- | --- | --- | --- | --- | --- |
| CS0 | 0 | 000000 | Best Effort | Default, tanpa prioritas | GUEST, lalu lintas umum |
| CS1 | 8 | 001000 | Scavenger | Prioritas paling rendah | Pencadangan non-kritis, scavenger |
| AF11 | 10 | 001010 | Assured Forwarding 1, pengguguran rendah | Data kelas 1 | CORP low |
| AF12 | 12 | 001100 | Assured Forwarding 1, pengguguran sedang | Data kelas 1 | CORP medium |
| AF13 | 14 | 001110 | Assured Forwarding 1, pengguguran tinggi | Data kelas 1 | CORP pengguguran tinggi |
| CS2 | 16 | 010000 | Class Selector 2 | OAM, signaling | Monitoring |
| AF21 | 18 | 010010 | Assured Forwarding 2, pengguguran rendah | Data kelas 2 | ATM, POS |
| AF22 | 20 | 010100 | Assured Forwarding 2, pengguguran sedang | Data kelas 2 | ATM, POS |
| AF23 | 22 | 010110 | Assured Forwarding 2, pengguguran tinggi | Data kelas 2 | ATM, POS |
| CS3 | 24 | 011000 | Class Selector 3 | Signaling, broadcast | Signaling, routing |
| AF31 | 26 | 011010 | Assured Forwarding 3, pengguguran rendah | Data kritis kelas 3 | CORE-BANKING, ERP, WMS |
| AF32 | 28 | 011100 | Assured Forwarding 3, pengguguran sedang | Data kritis kelas 3 | CORE-BANKING |
| AF33 | 30 | 011110 | Assured Forwarding 3, pengguguran tinggi | Data kritis kelas 3 | CORE-BANKING |
| CS4 | 32 | 100000 | Class Selector 4 | Video streaming | Video |
| AF41 | 34 | 100010 | Assured Forwarding 4, pengguguran rendah | Video kelas 4 | Video conference, CCTV |
| AF42 | 36 | 100100 | Assured Forwarding 4, pengguguran sedang | Video kelas 4 | Video conference |
| AF43 | 38 | 100110 | Assured Forwarding 4, pengguguran tinggi | Video kelas 4 | Video conference |
| CS5 | 40 | 101000 | Class Selector 5 | Signaling voice | Signaling VoIP |
| VA | 44 | 101100 | Voice Admit | Voice admit | Voice admit, jika digunakan |
| EF | 46 | 101110 | Expedited Forwarding | Voice, tunda dan variasi tunda rendah | VOICE, VoIP |
| CS6 | 48 | 110000 | Class Selector 6 | Network control | OSPF, BGP, LDP |
| CS7 | 56 | 111000 | Class Selector 7 | Network control | OSPF, BGP, LDP |

**Pemetaan yang Digunakan pada Panduan Ini:**

| Lalu lintas | DSCP | Alasan |
| --- | --- | --- |
| VOICE | EF (46) | Tunda dan variasi tunda paling rendah, priority |
| CORE-BANKING / ERP / WMS | AF31 (26) | Data kritis, pengguguran rendah |
| ATM / POS | AF21 (18) | Data penting, di bawah banking |
| Video / CCTV | AF41 (34) | Lebar pita terjamin, toleran tunda |
| CORP / GUEST | CS0 (0) / AF11 (10) | Default, sisa lebar pita |
| Routing / Control | CS6 (48) / CS7 (56) | Tidak boleh dibuang oleh QoS data |

**Pemetaan ke Class-map:** Definisi class-map tercantum pada Bab 11. DSCP pada tabel di atas dipetakan ke class VOICE (EF), BANKING (AF31), ATM (AF21), VIDEO (AF41).

**Verifikasi:** `show policy-map interface GigabitEthernet<SLOT>/<PORT>` | `show class-map`
**Kesalahan Umum:** Penandaan EF untuk semua lalu lintas sehingga priority tidak efektif. DSCP berbeda pada ingress dan egress. AF33 digunakan untuk lalu lintas kritis sehingga mudah dibuang saat kemacetan.

### Appendix N: Golden Config — Route Reflector (RR1 / RR2)

**Prinsip:** RR merupakan perangkat kontrol, bukan data plane pelanggan. Konfigurasi wajib identik kecuali hostname, Loopback, dan IP tautan.

**Konfigurasi Lengkap:** Tercantum pada Bab 7 — Konfigurasi Inti RR1. Penyalinan dari dua tempat tidak diperbolehkan. Di bawah ini hanya selisih RR2 dan uji pengalihan.

**Selisih RR2 — Perbedaan terhadap RR1:**

| Item | RR1 | RR2 |
| --- | --- | --- |
| Hostname | `<RR1-HOSTNAME>` | `<RR2-HOSTNAME>` |
| Loopback / router-id | `<RR1-LOOPBACK>` | `<RR2-LOOPBACK>` |
| IP tautan ke P1 / P2 | `<IP-RR1-P1>` / `<IP-RR1-P2>` | `<IP-RR2-P1>` / `<IP-RR2-P2>` |
| Peer RR | `<RR2-LOOPBACK>` | `<RR1-LOOPBACK>` |
| `cluster-id` | `<CLUSTER-ID>` | Sama — `<CLUSTER-ID>` |

**Uji Pengalihan — RR:**

| Pengujian | Perintah | Harapan |
| --- | --- | --- |
| RR1 tidak aktif | `shutdown` RR1 / cabut tautan | PE tetap Established ke RR2, rute VPNv4 tetap ada |
| RR1 aktif kembali | `no shutdown` | PE Established ke RR1 lagi, tidak terdapat ketidakstabilan status VRF |
| RR1-RR2 tautan tidak aktif | `show ip bgp vpnv4 all summary` di RR1 dan RR2 | Masing-masing RR tetap punya rute dari PE client-nya |

**Verifikasi:** `show ip bgp vpnv4 all summary` | `show running-config | section router bgp` | `show ip ospf neighbor`
**Kesalahan Umum:** Loopback RR2 tidak diganti sehingga router-id ganda. `cluster-id` berbeda. `route-reflector-client` tertinggal pada satu PE. Password BGP tidak sama dengan PE.
