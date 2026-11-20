# Juniper Networks® Junos® OS — Implementation & Operations Guide

> **README Utama | v5.1 — Professional Expert Edition (Public-Safe)**
> **Dokumen ini tidak berisi nilai produksi.** Semua IP, AS, password, hostname, dan nama customer diganti placeholder.
> **Nilai aktual hanya di IPAM / vault internal. Jangan publish nilai produksi di README.**

| Atribut | Detail |
| :--- | :--- |
| **Versi** | **v5.1 — Professional Expert Edition (Public-Safe)** |
| **Tanggal** | 20 November 2026 |
| **Menggantikan** | v5.0 (Urutan bab belum sesuai guidelines) |
| **Berlaku untuk** | Junos OS / Junos OS Evolved |
| **Platform** | MX, PTX, ACX, EX, QFX, SRX, vMX / vSRX / vQFX |
| **Audiens** | Network Engineer, NOC, Architect, Pre-Sales |
| **Klasifikasi** | **PUBLIC-SAFE** — Aman dibagikan ke customer / partner / GitHub |
| **Sumber Kebenaran** | File `.conf` di repo ini + IPAM / vault internal |

### Prinsip Dokumen Ini

1. **Placeholder, bukan nilai nyata.** Format: `<PLACEHOLDER>`. Contoh: `<PE-LOOPBACK>`, `<NTP-SERVER-1>`, `<RR-CLUSTER-ID>`.
2. **Lab = contoh, bukan produksi.** Alamat lab di Bab 17 memakai blok dokumentasi / placeholder. Jangan samakan dengan produksi.
3. **Satu sumber kebenaran.** README menjelaskan pola & prosedur. File `.conf` adalah template yang harus diisi dari IPAM sebelum load.
4. **No customer, no incident leak.** Studi kasus di Bab 21 ditulis sebagai pola desain, bukan post-mortem internal.

### Konvensi

* `<>` = wajib diganti dari IPAM / vault. `[]` = opsional.
* `root>` = Operational Mode. `root#` = Configuration Mode. Jangan ketik prompt-nya.
* Struktur: Bagian I Pendahuluan → Bagian II Desain Core → Bagian III Implementasi → Bagian IV Operasi.
* Setiap bab: `Prinsip → Konfigurasi (template) → Verifikasi → Best Practice`.
* Perubahan remote wajib `commit confirmed`. Perubahan tanpa `show | compare` dilarang.

---

## Daftar Isi

> Daftar isi disusun mengikuti standar penulisan dokumen teknis: penomoran berjenjang (Bab — Subbab), judul sesuai heading, dan urutan sesuai isi dokumen — Bagian I Pendahuluan → Bagian II Desain Core → Bagian III Implementasi → Bagian IV Operasi.

1. [Overview & Arsitektur](#1-overview-arsitektur)
2. [CLI & Configuration Lifecycle](#2-cli-configuration-lifecycle)
3. [Design Decision & Trade-off Framework](#3-design-decision-trade-off-framework)
4. [Golden Baseline](#4-golden-baseline)
5. [Interface & LACP](#5-interface-lacp)
6. [Routing: OSPF, IS-IS, Static](#6-routing-ospf-is-is-static)
7. [BGP & Route Reflector](#7-bgp-route-reflector)
8. [Policy Framework](#8-policy-framework)
9. [Firewall Filter & RE Protection](#9-firewall-filter-re-protection)
10. [MPLS Core](#10-mpls-core)
11. [MPLS Services: L2VPN, VPLS, L3VPN](#11-mpls-services-l2vpn-vpls-l3vpn)
12. [EVPN-VXLAN & Migrasi VPLS → EVPN](#12-evpn-vxlan-migrasi-vpls-evpn)
13. [QoS / CoS — Classification, Policer, Scheduler](#13-qos-cos-classification-policer-scheduler)
14. [High Availability & Convergence](#14-high-availability-convergence)
15. [Security Expert: RPKI, RTBH, BGP Flowspec](#15-security-expert-rpki-rtbh-bgp-flowspec)
16. [Template Konfigurasi + Penjelasan](#16-template-konfigurasi-penjelasan)
17. [Lab Topology — Abstract](#17-lab-topology-abstract)
18. [Config Library — Index](#18-config-library-index)
19. [Operasi & Change Management](#19-operasi-change-management)
20. [Troubleshooting Expert: Packet Walk + Traceoptions](#20-troubleshooting-expert-packet-walk-traceoptions)
21. [Studi Kasus Expert — Pola Desain](#21-studi-kasus-expert-pola-desain)
- [Appendix A: Naming & Golden Rules](#appendix-a-naming--golden-rules)
- [Appendix B: Quick Reference](#appendix-b-quick-reference)

---
---

## Bagian I — Pendahuluan & Prinsip Desain

## 1. Overview & Arsitektur

Junos OS adalah single OS di seluruh portfolio. CLI, commit model, dan troubleshooting yang berlaku di EX/QFX berlaku identik di MX/PTX/SRX. Ini mengurangi learning curve dan risiko human error.

| Komponen | Fungsi |
| :--- | :--- |
| **Routing Engine (RE)** | Control Plane: OSPF, IS-IS, BGP, MPLS signalling, management, CLI |
| **Packet Forwarding Engine (PFE)** | Data Plane: Forwarding wire-speed |
| **Service Engine** | Layanan management: SSH, SNMP, NTP, syslog, NETCONF |

**Implikasi desain:**
* PFE tetap forwarding saat RE sibuk / restart (dengan Graceful Restart / NSR).
* Konfigurasi deklaratif. Urutan `set` tidak mengubah hasil akhir.
* Commit bersifat atomik — aktif hanya setelah validasi lolos.

| Kategori | Series | Peran Umum |
| :--- | :--- | :--- |
| Switching | EX, QFX | Access / Aggregation / Data Center |
| Routing | ACX, MX, PTX | Access, Edge, Core P/PE |
| Security | SRX | Firewall, NAT, VPN |
| Virtual | vMX, vSRX, vQFX | Lab, PoC, NFV |

---

## 2. CLI & Configuration Lifecycle

### 2.1 Tiga Mode

| Mode | Prompt | Fungsi |
| :--- | :--- | :--- |
| UNIX Shell | `root@%` | Akses OS, file system |
| Operational | `root>` | Monitoring: `show`, `ping`, `traceroute`, `request` |
| Configuration | `root#` | Ubah candidate configuration |

```bash
root@% cli
root> configure
[edit]
root#
```

> Jangan edit file config langsung dari UNIX Shell kecuali instruksi JTAC. Selalu pakai CLI agar validasi jalan.

### 2.2 Candidate vs Active

| Istilah | Arti | Cara Lihat |
| :--- | :--- | :--- |
| **Candidate** | Draft yang sedang diedit. Belum aktif. | `root# show` |
| **Active** | Konfigurasi yang sedang jalan. | `root> show configuration` |

**SOP Commit (Wajib):**

```bash
root> configure
root# set ...
root# show | compare      # 1. Review diff
root# commit check        # 2. Validasi
root# commit confirmed 5  # 3. Commit dengan auto-rollback (remote / risky)
# ... test ...
root# commit              # 4. Konfirmasi permanen jika stabil
```

| Varian | Kapan Dipakai |
| :--- | :--- |
| `commit check` | Sebelum commit apapun |
| `commit` | Local / low-risk |
| `commit confirmed <menit>` | **Wajib untuk remote / risky** — auto-rollback jika tidak dikonfirmasi |
| `commit at "<YYYY-MM-DD HH:MM:SS>"` | Maintenance window terjadwal |
| `commit comment "<TICKET-ID>: <deskripsi>"` | Semua perubahan produksi — audit trail |
| `commit synchronize` | Dual RE |

```bash
root# rollback 1   # Kembali ke commit sebelumnya
root# rollback 0   # Buang draft, kembali ke active
root# show system commit
```

**Mode Configure:**

| Mode | Perilaku |
| :--- | :--- |
| `configure` (shared) | Default. Semua user edit candidate yang sama |
| `configure private` | Candidate privat per user |
| `configure exclusive` | Lock device untuk perubahan kritis |

---

## 3. Design Decision & Trade-off Framework

> **Kenapa bab ini ada:** Ini yang paling membedakan engineer senior dari junior. Junior bisa config. Senior bisa jelaskan *kenapa* pilih config ini, bukan config lain, dan apa risikonya. Bab ini adalah template untuk setiap design decision di README.

### 3.1 Template Design Decision

Gunakan template ini untuk setiap keputusan desain di dokumen / design review / customer proposal:

```markdown
### Decision: <Judul Keputusan>
**Konteks:** <Masalah / requirement>
**Pilihan yang Dipertimbangkan:**
| Pilihan | Kelebihan | Kekurangan | Biaya / Risiko |
| :--- | :--- | :--- | :--- |
| A. <Pilihan A> | ... | ... | ... |
| B. <Pilihan B> | ... | ... | ... |
**Keputusan:** <Pilihan yang dipilih>
**Alasan:** <Kenapa pilih ini, bukan yang lain>
**Trade-off yang Diterima:** <Apa yang dikorbankan>
**Failure Mode:** <Jika ini gagal, apa yang terjadi? Bagaimana mitigasinya?>
**Rollback:** <Bagaimana kembali ke desain lama?>
```

### 3.2 Contoh Design Decision (Dari README Ini)

**Decision: LDP vs RSVP-TE**

| Pilihan | Kelebihan | Kekurangan |
| :--- | :--- | :--- |
| A. LDP | Sederhana, otomatis, cukup untuk L2VPN/VPLS/L3VPN | Tanpa TE, tanpa bandwidth guarantee |
| B. RSVP-TE | TE, explicit path, FRR <50ms | Kompleks, state per LSP, butuh planning |
| **Keputusan:** LDP untuk lab & core standar, RSVP-TE hanya jika ada requirement TE / FRR ketat | | |
| **Trade-off:** Kehilangan TE, tapi dapat simplicity & operational cost rendah | | |
| **Failure Mode:** Jika butuh TE di masa depan, migrasi LDP → RSVP-TE perlu maintenance window | | |

**Decision: OSPF vs IS-IS**

| Pilihan | Kelebihan | Kekurangan |
| :--- | :--- | :--- |
| A. OSPF | Familiar, mudah troubleshooting, bagus untuk enterprise | LSDB membesar, SPF sering saat skala besar |
| B. IS-IS | Scaling lebih baik, TLV extensible, roadmap SR-MPLS | Kurang familiar untuk tim enterprise, NET addressing perlu IPAM |
| **Keputusan:** OSPF untuk enterprise, IS-IS untuk ISP core besar / roadmap SR | | |

**Decision: VPLS vs EVPN**

| Pilihan | Kelebihan | Kekurangan |
| :--- | :--- | :--- |
| A. VPLS | Legacy, familiar, cukup untuk 3-5 site | Broadcast storm, scaling buruk, single-active |
| B. EVPN | Control-plane MAC, active-active, scaling baik | Kompleks, butuh BGP EVPN, butuh training |
| **Keputusan:** EVPN untuk greenfield, VPLS untuk legacy + migrasi bertahap | | |

**Decision: Dual RR vs Full-Mesh iBGP**

| Pilihan | Kelebihan | Kekurangan |
| :--- | :--- | :--- |
| A. Full-Mesh | Tanpa RR, tanpa single point | Session n*(n-1)/2, tidak scalable >10 PE |
| B. Dual RR | Scalable, PE hanya 2 session | RR adalah critical, butuh failover test |
| **Keputusan:** Dual RR untuk >5 PE, Full-Mesh hanya untuk lab kecil | | |

**Cara pakai:** Setiap kali ada design review, isi template ini. Jangan hanya presentasikan config, presentasikan decision + trade-off + failure mode. Itu yang dinilai principal & auditor.

---

## 4. Golden Baseline

> Template awal untuk setiap device baru. Jangan deploy device tanpa bab ini.

### 4.1 Hostname & Time

Naming: `<SITE>-<ROLE>-<NN>` — Contoh: `<SITE>-PE-01`. Konsisten di seluruh fleet.

```bash
set system host-name <HOSTNAME>
set system time-zone <IANA-TIMEZONE>
```

### 4.2 Authentication

> Gunakan `encrypted-password` atau SSH key. Jangan simpan password plaintext di Git / automation.

```bash
[edit system]
set system root-authentication encrypted-password <ENCRYPTED-PASSWORD>

set system login user <NOC-USER> class operator
set system login user <NOC-USER> authentication encrypted-password <ENCRYPTED-PASSWORD>

set system login user <ADMIN-USER> class super-user
set system login user <ADMIN-USER> authentication ssh-rsa "<SSH-PUBLIC-KEY>"

# Custom class — least privilege
set system login class <READ-CLASS> permissions [ view view-configuration ]
set system login class <READ-CLASS> allow-commands "show .*"
set system login message "Authorized Access Only. All activity is logged."
set system login idle-timeout 10
```

| Built-in Class | Akses |
| :--- | :--- |
| `super-user` | Penuh |
| `operator` | Clear, network, reset, trace, view (tanpa `configure`) |
| `read-only` | View only |
| `unauthorized` | No access |

### 4.3 NTP, DNS, Syslog

NTP wajib — tanpa sinkronisasi waktu, log antar device tidak bisa dikorelasikan.

```bash
[edit system ntp]
set system ntp boot-server <NTP-SERVER-1>
set system ntp server <NTP-SERVER-1> prefer
set system ntp server <NTP-SERVER-2>
set system ntp source-address <LOOPBACK-IP>
# Jika pakai autentikasi: key dari vault, bukan hardcoded di README
# set system ntp authentication-key <KEY-ID> type md5 value <FROM-VAULT>
# set system ntp server <NTP-SERVER-1> key <KEY-ID>
# set system ntp trusted-key <KEY-ID>

set system name-server <DNS-SERVER-1>
set system name-server <DNS-SERVER-2>
set system domain-name <DOMAIN>

set system syslog user * any emergency
set system syslog file messages any notice
set system syslog file interactive-commands interactive-commands any
set system syslog host <SYSLOG-SERVER> any info
set system syslog host <SYSLOG-SERVER> source-address <LOOPBACK-IP>
set system syslog archive size 10m files 10
```

Verifikasi: `show ntp associations`, `show ntp status` — offset stabil, stratum stabil.

### 4.4 Management Services — Hardening

> **Dilarang di produksi:** Telnet, FTP, HTTP plaintext, `xnm-clear-text`. Semua mengirim kredensial plaintext.

```bash
[edit system services]
set system services ssh root-login deny
set system services ssh protocol-version v2
set system services ssh max-sessions-per-connection 5
set system services ssh connection-limit 10
set system services ssh rate-limit 5

# Jika Web dibutuhkan: HTTPS only
# set system services web-management https system-generated-certificate
# set system services web-management https interface <MGMT-INTERFACE>

# NETCONF untuk otomasi
set system services netconf ssh
```

### 4.5 SNMP — SNMPv3 Only

> Jangan pakai community `public` / `private`. Standar produksi adalah SNMPv3 AuthPriv. Kredensial dari vault.

```bash
[edit snmp]
set snmp description "<HOSTNAME> - <ROLE> <SITE>"
set snmp location "<SITE-DESCRIPTION>"
set snmp contact "<NOC-CONTACT>"

set snmp v3 usm local-engine user <SNMP-USER> authentication-sha authentication-password <FROM-VAULT>
set snmp v3 usm local-engine user <SNMP-USER> privacy-aes128 privacy-password <FROM-VAULT>
set snmp v3 vacm security-to-group security-model usm security-name <SNMP-USER> group <RO-GROUP>
set snmp v3 vacm access group <RO-GROUP> default-context-prefix security-model usm security-level authentication-privacy read-view <VIEW>
set snmp v3 vacm view <VIEW> oid .1 include
```

### 4.6 Deactivate / Delete

```bash
delete <hierarchy>           # Hapus permanen dari candidate
deactivate <hierarchy>       # Nonaktifkan sementara (muncul sebagai inactive:)
activate <hierarchy>         # Aktifkan lagi
rename <old> to <new>        # Rename
replace pattern <old> with <new>  # Hati-hati, selalu show | compare sesudahnya
```

---

---

## Bagian II — Desain Jaringan Core

## 5. Interface & LACP

### 5.1 Prinsip

* Setiap interface wajib punya `description`: `to <NEIGHBOR> <PORT> | <CIRCUIT-ID> | <BW>`
* Loopback (`lo0`) adalah identitas device: Router-ID, BGP local-address, LDP endpoint, NTP/SNMP source. Jangan pakai IP fisik untuk ini.
* MTU harus konsisten end-to-end. Mismatch = adjacency flap, MPLS drop diam-diam.

```bash
[edit interfaces]
set interfaces lo0 unit 0 description "Loopback - <HOSTNAME> Router-ID"
set interfaces lo0 unit 0 family inet address <LOOPBACK-IP>/32

set interfaces <CORE-INTERFACE> description "to <NEIGHBOR> <PORT> | <CIRCUIT-ID> | <BW>"
set interfaces <CORE-INTERFACE> mtu <MTU>
set interfaces <CORE-INTERFACE> unit 0 description "to <NEIGHBOR> - P2P Core"
set interfaces <CORE-INTERFACE> unit 0 family inet address <P2P-IP>/30
set interfaces <CORE-INTERFACE> unit 0 family mpls
set interfaces <CORE-INTERFACE> unit 0 family iso
```

Verifikasi: `show interfaces terse`, `show interfaces <IF> extensive`, `ping <P2P-NEIGHBOR> count 5`

### 5.2 LACP (802.3ad)

Pakai LACP `active` di kedua sisi (atau active/passive). Jangan `passive/passive` — LAG tidak terbentuk. MTU & VLAN di `aeX`, bukan di member.

```bash
[edit interfaces]
set interfaces <AE-INTERFACE> description "LAG to <NEIGHBOR> | <BW>"
set interfaces <AE-INTERFACE> flexible-vlan-tagging
set interfaces <AE-INTERFACE> mtu <MTU>
set interfaces <AE-INTERFACE> aggregated-ether-options minimum-links 1
set interfaces <AE-INTERFACE> aggregated-ether-options link-speed <LINK-SPEED>
set interfaces <AE-INTERFACE> aggregated-ether-options lacp active
set interfaces <AE-INTERFACE> aggregated-ether-options lacp periodic fast

set interfaces <AE-INTERFACE> unit <VLAN-ID> description "<VLAN-DESCRIPTION>"
set interfaces <AE-INTERFACE> unit <VLAN-ID> vlan-id <VLAN-ID>
set interfaces <AE-INTERFACE> unit <VLAN-ID> family inet address <IP>/30
set interfaces <AE-INTERFACE> unit <VLAN-ID> family inet filter input <FILTER-NAME>

set interfaces <MEMBER-1> description "Member <AE> to <NEIGHBOR> - Link 1"
set interfaces <MEMBER-1> gigether-options 802.3ad <AE-INTERFACE>
set interfaces <MEMBER-2> description "Member <AE> to <NEIGHBOR> - Link 2"
set interfaces <MEMBER-2> gigether-options 802.3ad <AE-INTERFACE>
```

Verifikasi: `show lacp interfaces <AE>` — Expected: Collecting + Distributing.


---

## 6. Routing: OSPF, IS-IS, Static

### 6.1 Static

Pakai static hanya untuk default route di edge, stub, atau management. Jangan bangun core dengan static.

```bash
[edit routing-options]
set routing-options static route 0.0.0.0/0 next-hop <NEXT-HOP>
set routing-options static route <PREFIX> next-hop <NEXT-HOP> preference 5
```

| Protokol | Default Preference |
| :--- | :--- |
| Direct / Local | 0 |
| Static | 5 |
| OSPF Internal | 10 |
| IS-IS L1 Internal | 15 |
| IS-IS L2 Internal | 18 |
| BGP Internal / External | 170 |

### 6.2 OSPF

**Aturan desain:**
* Area 0 (Backbone) wajib & kontigu.
* Loopback di Area 0 sebagai Router-ID stabil.
* P2P untuk link point-to-point. Passive untuk loopback & interface ke user.
* Autentikasi untuk semua adjacency di produksi. BFD untuk deteksi failure sub-detik.

```bash
[edit]
set routing-options router-id <ROUTER-ID>

set protocols ospf area <AREA-0> interface lo0.0 passive
set protocols ospf area <AREA-0> interface <CORE-IF>.0 interface-type p2p
set protocols ospf reference-bandwidth <REFERENCE-BW>
set protocols ospf traffic-engineering

# Tuning & Hardening (per interface)
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 hello-interval <HELLO>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 dead-interval <DEAD>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 metric <COST>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 authentication md5 <KEY-ID> key <FROM-VAULT>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection minimum-interval <MS>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection multiplier <N>
# set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection full-neighbors-only
```

Verifikasi: `show ospf neighbor` (Full), `show ospf interface`, `show ospf route`, `show bfd session`

**Troubleshooting:** IP/mask identik, MTU identik, Area ID identik, Router-ID unik, hello/dead & auth identik, firewall tidak blokir multicast OSPF.

### 6.3 IS-IS

IS-IS dipilih untuk core besar karena TLV extensible, adjacency tidak terikat IP (CLNS), dan scaling lebih baik.

**NET Address:** `AFI . Area-ID . System-ID . NSEL` — Contoh format: `<AFI>.<AREA>.<SYSTEM-ID>.00`. System-ID diturunkan dari loopback via IPAM. Jangan inventarisasi manual tanpa IPAM.

```bash
[edit]
set interfaces lo0 unit 0 family iso address <NET-ADDRESS>
set interfaces <CORE-IF> unit 0 family iso
set interfaces <CORE-IF> unit 0 family inet address <P2P-IP>/30
set interfaces lo0 unit 0 family inet address <LOOPBACK-IP>/32

set protocols isis interface lo0.0 passive
set protocols isis interface <CORE-IF>.0
# Level control per interface jika perlu:
# set protocols isis interface <CORE-IF>.0 level 2 disable  # L1 only
# set protocols isis interface <CORE-IF>.0 level 1 disable  # L2 only

set protocols isis level 1 wide-metrics-only
set protocols isis level 2 wide-metrics-only
# set protocols isis authentication-key <FROM-VAULT>
# set protocols isis level 1 authentication-key <FROM-VAULT>
```

Verifikasi: `show isis adjacency`, `show isis interface`, `show isis database detail`, `show route protocol isis`

---

## 7. BGP & Route Reflector

### 7.1 Prinsip

* iBGP butuh full-mesh atau Route Reflector. Tanpa itu, iBGP tidak advertise route iBGP ke iBGP lain (split-horizon).
* eBGP butuh `export` policy eksplisit — default Junos: eBGP tidak export apapun tanpa policy.
* Loopback sebagai `local-address` + `router-id` = session stabil.

### 7.2 Route Reflector — Desain

Client hanya peering ke RR (1-2 session), bukan ke semua PE. RR merefleksikan route antar client. **Deploy 2 RR** untuk redundansi. Client peering ke keduanya.

```bash
# RR — Template
[edit]
set routing-options graceful-restart
set routing-options router-id <RR-ROUTER-ID>
set routing-options autonomous-system <AS>

set protocols bgp log-updown
set protocols bgp group <RR-CLIENTS-GROUP> type internal
set protocols bgp group <RR-CLIENTS-GROUP> local-address <RR-LOOPBACK>
set protocols bgp group <RR-CLIENTS-GROUP> family inet-vpn unicast
set protocols bgp group <RR-CLIENTS-GROUP> authentication-key <FROM-VAULT>
set protocols bgp group <RR-CLIENTS-GROUP> cluster <RR-CLUSTER-ID>
set protocols bgp group <RR-CLIENTS-GROUP> peer-as <AS>
set protocols bgp group <RR-CLIENTS-GROUP> neighbor <PE-LOOPBACK-1> description "<PE-1>"
set protocols bgp group <RR-CLIENTS-GROUP> neighbor <PE-LOOPBACK-2> description "<PE-2>"

# Client (PE) — Template
set routing-options graceful-restart
set routing-options router-id <PE-ROUTER-ID>
set routing-options autonomous-system <AS>
set protocols bgp log-updown
set protocols bgp group <TO-RR-GROUP> type internal
set protocols bgp group <TO-RR-GROUP> local-address <PE-LOOPBACK>
set protocols bgp group <TO-RR-GROUP> family inet-vpn unicast
set protocols bgp group <TO-RR-GROUP> authentication-key <FROM-VAULT>
set protocols bgp group <TO-RR-GROUP> peer-as <AS>
set protocols bgp group <TO-RR-GROUP> neighbor <RR-LOOPBACK-1> description "<RR-1>"
set protocols bgp group <TO-RR-GROUP> neighbor <RR-LOOPBACK-2> description "<RR-2>"
```

> **Aturan:** Cluster-ID unik per RR. Biasanya = Router-ID RR. Jangan samakan Cluster-ID antar RR.

Verifikasi: `show bgp summary` (Established), `show bgp neighbor <IP>`, `show route advertising-protocol bgp <IP>`, `show route receive-protocol bgp <IP>`

---

## 8. Policy Framework

Policy Junos adalah daftar term berurutan `from → then`. Evaluasi dari atas ke bawah, berhenti di term pertama yang match. Jika tidak ada yang match → default action.

```bash
[edit policy-options]
set policy-options policy-statement <POLICY-NAME> term 10 from route-filter <PREFIX> orlonger
set policy-options policy-statement <POLICY-NAME> term 10 then accept
set policy-options policy-statement <POLICY-NAME> term 99 then reject   # Default deny eksplisit
```

| Aksi | Arti |
| :--- | :--- |
| `accept` | Terima & berhenti |
| `reject` | Tolak & berhenti |
| `next` | Lanjut ke term berikutnya |
| `community add` | Tambah BGP community |
| `metric`, `preference`, `local-preference` | Manipulasi atribut |

| Protokol | Export (Default) |
| :--- | :--- |
| OSPF | Hanya OSPF + direct |
| IS-IS | Hanya IS-IS |
| BGP | **Export nothing** — harus policy eksplisit |

**Redistribution — Prinsip:** Jangan pernah redistribute tanpa filter. Tanpa filter = loop, suboptimal, flap.

```bash
[edit policy-options]
set policy-options policy-statement <IGP-A-TO-B> term 10 from protocol <IGP-A>
set policy-options policy-statement <IGP-A-TO-B> term 10 from protocol direct
set policy-options policy-statement <IGP-A-TO-B> term 10 then accept
set policy-options policy-statement <IGP-A-TO-B> term 99 then reject

[edit protocols]
set protocols <IGP-B> export <IGP-A-TO-B>
```

> Jangan mutual-redistribute di 2 titik sekaligus tanpa tag / preference control. Pakai `tag` atau `preference` untuk membedakan route yang sudah di-redistribute.

---

## 9. Firewall Filter & RE Protection

### 9.1 Konsep

* Firewall filter Junos bersifat stateless. Inspeksi per-packet.
* Urutan term adalah segalanya. Jika tidak ada term yang match → implicit deny.
* Arah: `input` = masuk interface, `output` = keluar interface.

| Aksi | Arti |
| :--- | :--- |
| `accept` | Forward |
| `discard` | Drop diam-diam |
| `reject` | Drop + ICMP unreachable / TCP RST |
| `next` | Lanjut ke term berikutnya |
| `count`, `log`, `syslog` | Audit |
| `policer` | Rate-limit |

### 9.2 Loopback Protection (RE Protection) — Template Wajib

> Filter ini melindungi Routing Engine. Apply ke `lo0`. Hanya izinkan traffic yang dibutuhkan ke RE, dari source yang sah, dengan rate-limit.

```bash
[edit firewall]
# Template — isi <...> dari IPAM / policy keamanan. Jangan hardcode IP produksi di README.
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from source-address <MGMT-BLOCK>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from protocol tcp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from destination-port 22
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH then policer <SSH-POLICER>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from source-address <BGP-NEIGHBOR-BLOCK>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from protocol tcp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from destination-port 179
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP then policer <BGP-POLICER>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-IGP from protocol ospf
set firewall filter <PROTECT-RE-FILTER> term ALLOW-IGP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP from protocol icmp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP from icmp-type echo-request
set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP from protocol udp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP from destination-port 123
set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP then accept

set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then count <DENY-COUNTER>
set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then log
set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then discard

# Policer — dari policy keamanan, bukan dari README
# set firewall policer <SSH-POLICER> if-exceeding bandwidth-limit <BW>
# set firewall policer <SSH-POLICER> if-exceeding burst-size-limit <BURST>
# set firewall policer <SSH-POLICER> then discard

[edit interfaces lo0]
set interfaces lo0 unit 0 family inet filter input <PROTECT-RE-FILTER>
```

> Yang di-apply adalah **nama filter**, bukan nama term. Verifikasi: `show firewall filter <FILTER>`, `show firewall counter`, `show firewall log`

---

## 10. MPLS Core

**Syarat MPLS jalan:** IGP Full & loopback reachable → `family mpls` aktif di interface core → LDP atau RSVP aktif.

```bash
[edit protocols]
set protocols mpls no-propagate-ttl
set protocols mpls interface lo0.0
set protocols mpls interface <CORE-IF>.0

set protocols ldp interface lo0.0
set protocols ldp interface <CORE-IF>.0
# set protocols ldp session <NEIGHBOR-LOOPBACK> authentication-key <FROM-VAULT>

set protocols lldp interface <CORE-IF>.0
set protocols lldp management-address <LOOPBACK-IP>

# RSVP — hanya jika butuh TE / FRR
# set protocols rsvp interface lo0.0
# set protocols rsvp interface <CORE-IF>.0
# set protocols rsvp interface <CORE-IF>.0 link-protection
```

Verifikasi: `show mpls interface`, `show mpls lsp`, `show ldp neighbor`, `show ldp session`, `show route forwarding-table family mpls`, `show lldp neighbors`

| Pilihan | Kapan Dipakai |
| :--- | :--- |
| **LDP** | Default, sederhana, cukup untuk L2VPN / VPLS / L3VPN |
| **RSVP-TE** | Butuh TE, explicit path, bandwidth guarantee, FRR |
| **Segment Routing** | Modern, controller-based — roadmap |

---

## 11. MPLS Services: L2VPN, VPLS, L3VPN

### 11.1 Matriks Pemilihan

| Service | Layer | Topologi | Kapan Dipakai |
| :--- | :--- | :--- | :--- |
| **L2VPN (L2Circuit)** | L2 Point-to-Point | P2P | 2 site seolah 1 kabel |
| **VPLS** | L2 Multipoint | Multipoint (LAN) | Banyak site seolah 1 switch |
| **L3VPN (VRF)** | L3 | Hub-Spoke / Any-to-Any | Isolasi routing antar customer |

> Jangan pakai VPLS jika hanya 2 site (pakai L2VPN). Jangan pakai L2VPN jika butuh 3+ site (pakai VPLS / EVPN).

### 11.2 L2VPN (L2Circuit)

VC-ID harus identik di kedua PE. MTU & control-word harus identik di kedua PE.

```bash
[edit protocols l2circuit]
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> virtual-circuit-id <VC-ID>
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> description "<DESCRIPTION>"
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> no-control-word
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> ignore-encapsulation-mismatch

[edit interfaces <CE-IF>]
set interfaces <CE-IF> flexible-vlan-tagging
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> encapsulation flexible-ethernet-services
set interfaces <CE-IF> unit <VLAN> description "<DESCRIPTION>"
set interfaces <CE-IF> unit <VLAN> encapsulation vlan-ccc
set interfaces <CE-IF> unit <VLAN> vlan-id <VLAN-ID>
```

Verifikasi: `show l2circuit connections` (Up), `show l2circuit connections extensive`, `ping mpls l2circuit virtual-circuit <VC-ID> neighbor <FAR-PE-LOOPBACK>`

### 11.3 VPLS

```bash
[edit routing-instances <VPLS-INSTANCE>]
set routing-instances <VPLS-INSTANCE> instance-type vpls
set routing-instances <VPLS-INSTANCE> interface <CE-IF>.0
set routing-instances <VPLS-INSTANCE> protocols vpls encapsulation-type ethernet
set routing-instances <VPLS-INSTANCE> protocols vpls no-control-word
set routing-instances <VPLS-INSTANCE> protocols vpls vpls-id <VPLS-ID>
set routing-instances <VPLS-INSTANCE> protocols vpls mtu <MTU>
set routing-instances <VPLS-INSTANCE> protocols vpls ignore-mtu-mismatch
set routing-instances <VPLS-INSTANCE> protocols vpls ignore-encapsulation-mismatch
set routing-instances <VPLS-INSTANCE> protocols vpls neighbor <FAR-PE-LOOPBACK> switchover-delay <MS>
set routing-instances <VPLS-INSTANCE> protocols vpls neighbor <FAR-PE-LOOPBACK> backup-neighbor <BACKUP-PE-LOOPBACK> standby

[edit interfaces <CE-IF>]
set interfaces <CE-IF> unit 0 description "<DESCRIPTION>"
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> encapsulation ethernet-vpls
```

Verifikasi: `show vpls connections`, `show vpls mac-table`

> Untuk greenfield, pertimbangkan **EVPN** sebagai pengganti VPLS. EVPN lebih scalable, mendukung active-active multihoming, dan merupakan standar saat ini. VPLS dipertahankan untuk kompatibilitas legacy.

### 11.4 L3VPN — VRF

**Konsep:**

| Istilah | Arti |
| :--- | :--- |
| **VRF** | Tabel routing terpisah per customer |
| **RD (Route Distinguisher)** | Membuat prefix customer unik di core. Format `<AS>:<NN>`. Harus unik per VRF per PE. |
| **RT (Route Target)** | Community yang menentukan import/export antar PE. Format `target:<AS>:<NN>`. |
| **MP-BGP** | BGP yang membawa VPNv4 (RD + prefix) antar PE via RR |

**L3VPN Dasar:**

```bash
[edit policy-options]
set policy-options policy-statement <VRF>-EXPORT term 10 then community add <RT>
set policy-options policy-statement <VRF>-EXPORT term 10 then accept
set policy-options policy-statement <VRF>-IMPORT term 10 from community <RT>
set policy-options policy-statement <VRF>-IMPORT term 10 then accept
set policy-options policy-statement <VRF>-IMPORT term 99 then reject
set policy-options community <RT> members <RT>

[edit routing-instances <VRF>]
set routing-instances <VRF> instance-type vrf
set routing-instances <VRF> interface <CE-IF>.0
set routing-instances <VRF> route-distinguisher <RD>
set routing-instances <VRF> vrf-import <VRF>-IMPORT
set routing-instances <VRF> vrf-export <VRF>-EXPORT
set routing-instances <VRF> vrf-table-label

[edit interfaces <CE-IF>]
set interfaces <CE-IF> unit 0 description "to <CE>"
set interfaces <CE-IF> encapsulation ethernet
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> unit 0 family inet address <PE-CE-IP>/30
```

**PE-CE BGP dengan AS-Override:**

Masalah: CE di 2 site pakai AS yang sama. BGP akan reject route karena AS-loop. Solusi: `as-override` di PE.

```bash
[edit routing-instances <VRF>]
set routing-instances <VRF> protocols bgp group <TO-CE> type external
set routing-instances <VRF> protocols bgp group <TO-CE> family inet unicast
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> as-override
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> peer-as <CE-AS>
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> local-as <PROVIDER-AS>
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> export <EXPORT-TO-CE>
```

**PE-CE OSPF dengan Sham-Link:**

Masalah: Customer punya backdoor link antar site + MPLS. Tanpa sham-link, OSPF pilih backdoor (intra-area) padahal MPLS adalah jalur utama. Solusi: sham-link membuat MPLS terlihat sebagai link intra-area dengan cost yang bisa di-tune.

```bash
[edit routing-instances <VRF>]
set routing-instances <VRF> interface <CE-IF>.0
set routing-instances <VRF> interface lo0.1
set routing-instances <VRF> protocols ospf export <BGP-TO-OSPF>
set routing-instances <VRF> protocols ospf sham-link local <SHAM-LOCAL>
set routing-instances <VRF> protocols ospf area <CUSTOMER-AREA> sham-link-remote <SHAM-REMOTE> metric <COST>
set routing-instances <VRF> protocols ospf area <CUSTOMER-AREA> interface <CE-IF>.0
set routing-instances <VRF> protocols area <CUSTOMER-AREA> interface lo0.1

[edit interfaces]
set interfaces lo0 unit 1 family inet address <SHAM-LOCAL>/32
```

Verifikasi L3VPN: `show route table <VRF>.inet.0`, `show route forwarding-table vpn <VRF>`, `show bgp summary instance <VRF>`, `ping routing-instance <VRF> <DEST> source <SRC>`, `show ospf neighbor instance <VRF>`


---

---

## 12. EVPN-VXLAN & Migrasi VPLS → EVPN

> **Kenapa bab ini ada:** VPLS di Bab 11 sudah benar untuk legacy, tapi untuk greenfield & roadmap Juniper, standar saat ini adalah **EVPN**. Bab ini menjelaskan kenapa EVPN lebih baik, template EVPN, dan cara migrasi tanpa downtime.

### 12.1 Kenapa EVPN Menggantikan VPLS

| Aspek | VPLS | EVPN |
| :--- | :--- | :--- |
| MAC Learning | Data-plane flooding (broadcast ke semua PE) | Control-plane via BGP (MP-BGP EVPN) |
| Loop / Storm | 1 loop di 1 CE bisa storm ke semua site | Lebih terkontrol, DF election cegah duplikasi |
| Multihoming | Single-active (backup standby) | Active-active (load-balancing) |
| Scaling | Flooding membesar seiring site bertambah | Lebih scalable, MAC via BGP |
| Layanan | L2 saja | L2 + L3 (EVPN + L3VPN terintegrasi) |

**Prinsip:** Jika customer baru, pakai EVPN. Jika customer lama di VPLS, rencanakan migrasi bertahap, bukan flag-day.

### 12.2 Template EVPN-VXLAN (PE)

```bash
[edit]
# 1. BGP EVPN ke RR
set protocols bgp group <TO-RR> family evpn signaling
set protocols bgp group <TO-RR> neighbor <RR-LOOPBACK> description "<RR>"

# 2. EVPN Instance — VLAN-aware
set routing-instances <EVPN-INSTANCE> instance-type evpn
set routing-instances <EVPN-INSTANCE> vlan-id <VLAN-ID>
set routing-instances <EVPN-INSTANCE> interface <CE-IF>.0
set routing-instances <EVPN-INSTANCE> route-distinguisher <RD>
set routing-instances <EVPN-INSTANCE> vrf-target <RT>
set routing-instances <EVPN-INSTANCE> protocols evpn encapsulation vxlan
set routing-instances <EVPN-INSTANCE> protocols evpn extended-vlan-list <VLAN-ID>
set routing-instances <EVPN-INSTANCE> protocols evpn default-gateway no-gateway-community

# 3. VXLAN — VNI harus sama di semua PE untuk 1 layanan
set interfaces <CE-IF> unit <VLAN-ID> family bridge interface-mode access
set interfaces <CE-IF> unit <VLAN-ID> family bridge vlan-id <VLAN-ID>
# set protocols evpn vni <VNI> — dari IPAM, unik per layanan

# 4. Multihoming Active-Active (jika CE dual-homed ke 2 PE)
set interfaces <AE-IF> esi <ESI>
set interfaces <AE-IF> esi all-active
set protocols evpn esi <ESI> auto-derive
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `family evpn signaling` | BGP harus bawa route EVPN (Type 1-5). Tanpa ini, MAC tidak ter-advertise |
| `route-distinguisher` + `vrf-target` | Sama seperti L3VPN: RD unik, RT menentukan import/export |
| `encapsulation vxlan` + `VNI` | VXLAN overlay. VNI harus sama di semua PE untuk 1 layanan |
| `ESI + all-active` | Ethernet Segment Identifier. `all-active` = CE bisa aktif ke 2 PE sekaligus, load-balancing. Tanpa ESI, multihoming jadi single-active |
| `default-gateway` | Untuk EVPN IRB (L3 gateway). Jika hanya L2, tidak perlu IRB |

**Verifikasi:** `show evpn database`, `show evpn mac-table`, `show bgp summary`, `show route table <EVPN>.evpn.0`, `ping <CE-IP>`

### 12.3 Migrasi VPLS → EVPN Tanpa Downtime

1. **Fase 1 — Bangun EVPN paralel.** VPLS tetap jalan, EVPN dibangun di VNI berbeda, tidak ganggu VPLS.
2. **Fase 2 — Pilot 1 site.** Pindahkan 1 CE ke EVPN, verifikasi MAC & ping. Rollback ke VPLS jika gagal (deactivate EVPN, activate VPLS).
3. **Fase 3 — Migrasi per site.** 1 site per maintenance window. Monitor MAC flapping & broadcast.
4. **Fase 4 — Hapus VPLS.** Setelah semua site di EVPN & stabil 1 minggu, hapus VPLS instance.

**Kesalahan umum:** VNI beda antar PE, ESI tidak sama di 2 PE untuk 1 CE dual-homed, lupa `family evpn` di BGP ke RR.

---

## 13. QoS / CoS — Classification, Policer, Scheduler

> **Kenapa bab ini ada:** Customer enterprise selalu tanya: "Gimana VoIP tidak putus saat traffic penuh?" Jawabannya CoS. Tanpa CoS, semua traffic diperlakukan sama — VoIP kalah dengan download besar.

### 13.1 Prinsip CoS Junos

```
CE → PE: Classify (DSCP / 802.1p) → Policer (rate-limit) → Queue → Scheduler (prioritas) → Rewrite (tandai lagi ke core)
```

| Konsep | Arti |
| :--- | :--- |
| **Classifier** | Baca DSCP / 802.1p dari packet, masukkan ke forwarding-class |
| **Forwarding-Class** | Antrian: `voice`, `video`, `data-critical`, `best-effort` |
| **Policer** | Rate-limit per class. Jika melebihi, drop / remark |
| **Scheduler** | Prioritas & bandwidth per antrian. `strict-high` untuk VoIP |
| **Rewrite** | Tulis lagi DSCP / EXP saat keluar PE ke core |

### 13.2 Template CoS (PE)

```bash
[edit class-of-service]
# 1. Forwarding Class
set class-of-service forwarding-classes queue 0 best-effort
set class-of-service forwarding-classes queue 1 data-critical
set class-of-service forwarding-classes queue 2 video
set class-of-service forwarding-classes queue 3 voice priority high

# 2. Classifier — baca DSCP dari CE
set class-of-service classifiers dscp <CLASSIFIER> forwarding-class voice loss-priority low code-points ef
set class-of-service classifiers dscp <CLASSIFIER> forwarding-class video loss-priority low code-points af41
set class-of-service classifiers dscp <CLASSIFIER> forwarding-class data-critical loss-priority low code-points af31
set class-of-service classifiers dscp <CLASSIFIER> forwarding-class best-effort loss-priority high code-points 000000

# 3. Policer — batasi per class (contoh: voice max 10M)
set firewall policer <VOICE-POLICER> if-exceeding bandwidth-limit 10m
set firewall policer <VOICE-POLICER> if-exceeding burst-size-limit 1m
set firewall policer <VOICE-POLICER> then discard

# 4. Scheduler — prioritas & bandwidth
set class-of-service schedulers <VOICE-SCHED> transmit-rate percent 20
set class-of-service schedulers <VOICE-SCHED> priority strict-high
set class-of-service schedulers <DATA-SCHED> transmit-rate percent 50
set class-of-service schedulers <DATA-SCHED> priority low
set class-of-service schedulers <BE-SCHED> transmit-rate percent 30
set class-of-service schedulers <BE-SCHED> priority low

# 5. Scheduler Map + Apply ke interface
set class-of-service scheduler-maps <MAP> forwarding-class voice scheduler <VOICE-SCHED>
set class-of-service scheduler-maps <MAP> forwarding-class data-critical scheduler <DATA-SCHED>
set class-of-service scheduler-maps <MAP> forwarding-class best-effort scheduler <BE-SCHED>
set class-of-service interfaces <CE-IF> unit 0 scheduler-map <MAP>
set class-of-service interfaces <CE-IF> unit 0 classifiers dscp <CLASSIFIER>

# 6. Rewrite — tandai lagi ke core (MPLS EXP)
set class-of-service rewrite-rules exp <REWRITE> forwarding-class voice loss-priority low code-point 5
set class-of-service rewrite-rules exp <REWRITE> forwarding-class best-effort loss-priority high code-point 0
set class-of-service interfaces <CORE-IF> unit 0 rewrite-rules exp <REWRITE>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `forwarding-classes` | Antrian. `voice` harus `priority high` + `queue 3` (paling prioritas) |
| `classifiers dscp` | Baca DSCP dari CE. Jika CE tidak marking, PE harus classify dari IP / port |
| `policer` | Cegah 1 class monopoli bandwidth. Voice dibatasi agar tidak ganggu data, data dibatasi agar tidak ganggu voice |
| `scheduler` | `strict-high` untuk voice = voice selalu didahulukan. Tanpa ini, voice antre sama dengan download |
| `rewrite-rules exp` | MPLS EXP harus konsisten di core agar P router juga prioritaskan voice. Tanpa rewrite, P router tidak tahu mana voice |

**Verifikasi:** `show class-of-service interface <IF>`, `show class-of-service queue`, `show firewall policer`, `ping` + `show interfaces queue` saat load test
**Kesalahan umum:** Classifier tidak match dengan marking CE, scheduler tidak di-apply ke interface, rewrite tidak konsisten di semua PE.

---

## 14. High Availability & Convergence

> **Kenapa bab ini ada:** Design review principal selalu tanya: "Kalau 1 P down, berapa detik traffic pindah? Kalau RE restart, BGP down atau tidak?" Bab ini menjawab dengan angka & mekanisme.

### 14.1 Mekanisme HA Junos

| Mekanisme | Fungsi | Kapan Dipakai |
| :--- | :--- | :--- |
| **Graceful Restart (GR)** | RE restart, PFE tetap forwarding dengan FIB lama, BGP/IGP tidak flap | Single RE, maintenance RE |
| **GRES** | Backup RE takeover tanpa restart PFE, tapi control-plane restart | Dual RE, failover RE |
| **NSR** | Backup RE sudah sync state, failover tanpa flap BGP/IGP sama sekali | Dual RE, SLA ketat (99.999%) |
| **NSB** | NSR untuk bridging / L2 | Dual RE + VPLS / EVPN |
| **BFD** | Deteksi failure sub-detik | Semua link core |
| **FRR (Fast Reroute)** | Backup path MPLS siap, switchover <50ms | RSVP-TE, SLA ketat |

**Trade-off:**

| Pilihan | Kelebihan | Kekurangan |
| :--- | :--- | :--- |
| GR saja | Sederhana, tanpa dual RE | BGP flap sesaat, tapi PFE tetap forwarding |
| GRES + NSR | Zero flap, zero loss | Butuh dual RE, config lebih kompleks, resource lebih besar |
| BFD + FRR | Convergence <1 detik | Butuh tuning, jika BFD terlalu agresif bisa false-positive flap |

### 14.2 Template HA

```bash
[edit]
# Graceful Restart — IGP + BGP
set routing-options graceful-restart
set protocols ospf graceful-restart
set protocols isis graceful-restart
set protocols bgp graceful-restart

# BFD — deteksi failure sub-detik
set protocols ospf area <AREA> interface <IF>.0 bfd-liveness-detection minimum-interval <MS>
set protocols ospf area <AREA> interface <IF>.0 bfd-liveness-detection multiplier <N>

# GRES + NSR — Dual RE only
# set chassis redundancy graceful-switchover
# set system commit synchronize
# set routing-options nonstop-routing

# FRR — RSVP-TE only
# set protocols rsvp interface <IF>.0 link-protection
# set protocols mpls label-switched-path <LSP> link-protection
```

**Penjelasan:** `graceful-restart` di routing-options + per-protocol harus konsisten. `GRES` butuh `graceful-switchover` di chassis. `NSR` butuh `nonstop-routing` + `commit synchronize`. Tanpa `commit synchronize`, backup RE tidak sync config.

**Verifikasi:** `show system switchover`, `show graceful-restart`, `show bfd session`, `show rsvp session`, `show mpls lsp`
**Test Failover (Wajib Berkala):** `request chassis routing-engine master switch` (dual RE) → verifikasi BGP tetap Established, ping tidak loss >1 detik. `deactivate` 1 link core → verifikasi traffic pindah <1 detik.

---

## 15. Security Expert: RPKI, RTBH, BGP Flowspec

> **Kenapa bab ini ada:** `PROTECT-RE` di Bab 9 melindungi RE, tapi tidak melindungi dari BGP hijack, DDoS besar, atau route leak. Bab ini adalah defense in depth untuk ISP / enterprise.

### 15.1 Tiga Lapisan Keamanan BGP

| Lapisan | Fungsi | Contoh |
| :--- | :--- | :--- |
| **RPKI** | Validasi asal prefix. Cegah BGP hijack (attacker klaim prefix orang lain) | ROA: `<PREFIX>` hanya boleh di-originate AS 65000 |
| **RTBH** | Buang traffic serangan ke blackhole (Null route) via BGP community | Serangan DDoS ke 1 IP → advertise /32 dengan community blackhole → upstream buang |
| **BGP Flowspec** | Filter DDoS di edge via BGP (5-tuple: src, dst, port, protocol) | Blokir UDP port 53 dari botnet ke 1 destination, tanpa blokir semua UDP |

### 15.2 Template RPKI + RTBH + Flowspec (Konsep)

```bash
# --- RPKI ---
[edit routing-options]
set routing-options validation group <RPKI-GROUP> session <RPKI-SERVER> local-address <LOOPBACK-IP>
set routing-options validation group <RPKI-GROUP> session <RPKI-SERVER> port <RPKI-PORT>
set routing-options validation group <RPKI-GROUP> session <RPKI-SERVER> refresh-time <SEC>

[edit policy-options]
set policy-options policy-statement <RPKI-POLICY> term VALID from validation-database valid
set policy-options policy-statement <RPKI-POLICY> term VALID then accept
set policy-options policy-statement <RPKI-POLICY> term INVALID from validation-database invalid
set policy-options policy-statement <RPKI-POLICY> term INVALID then reject
set policy-options policy-statement <RPKI-POLICY> term UNKNOWN from validation-database unknown
set policy-options policy-statement <RPKI-POLICY> term UNKNOWN then accept
# Apply: set protocols bgp group <EBGP> import <RPKI-POLICY>

# --- RTBH ---
[edit routing-options]
set routing-options static route <BLACKHOLE-IP>/32 discard
set routing-options static route <BLACKHOLE-IP>/32 community <BLACKHOLE-COMMUNITY>
# Saat serangan: advertise <VICTIM-IP>/32 dengan community blackhole ke upstream
# Upstream akan buang traffic ke <VICTIM-IP> — victim down, tapi core selamat

# --- BGP Flowspec ---
[edit]
set protocols bgp group <FLOWSPEC> family inet flow
set protocols bgp group <FLOWSPEC> neighbor <FLOWSPEC-SERVER> description "Flowspec Controller"
# Contoh rule (dari controller): blokir UDP src <BOTNET> dst <VICTIM> port 53
# set firewall family inet filter <FLOWSPEC-FILTER> term <RULE> from destination-address <VICTIM>/32
# set firewall family inet filter <FLOWSPEC-FILTER> term <RULE> from protocol udp
# set firewall family inet filter <FLOWSPEC-FILTER> term <RULE> from destination-port 53
# set firewall family inet filter <FLOWSPEC-FILTER> term <RULE> then discard
```

**Penjelasan:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `RPKI validation` | Tanpa RPKI, attacker bisa hijack prefix & arahkan traffic ke server jahat. RPKI adalah standar IETF untuk BGP security |
| `ROA valid / invalid / unknown` | `valid` = ROA cocok, `invalid` = ROA tidak cocok (hijack), `unknown` = belum ada ROA. Policy: `invalid` → reject, `valid`/`unknown` → accept |
| `RTBH` | Saat DDoS besar, lebih baik korban 1 IP daripada core down. RTBH adalah circuit breaker |
| `Flowspec` | Filter granular di edge, tanpa harus login ke semua router. Controller kirim Flowspec via BGP, semua PE apply otomatis |

**Verifikasi:** `show validation database`, `show validation session`, `show route <PREFIX> detail` (lihat validation-state), `show firewall filter <FLOWSPEC-FILTER>`
**Kesalahan umum:** RPKI `invalid` di-accept (hijack lolos), RTBH tidak diuji, Flowspec terlalu broad (blokir traffic sah).


---

---

## Bagian III — Implementasi & Validasi

## 16. Template Konfigurasi + Penjelasan

> **Bab ini adalah kumpulan template siap pakai.** Setiap template terdiri dari: Tujuan → Kapan Dipakai → Template (`set` format) → Penjelasan Per Blok → Verifikasi → Kesalahan Umum.
> **Semua nilai adalah placeholder.** Isi dari IPAM / vault sebelum load. Jangan load template mentah ke produksi.
> **Cara pakai:** Copy template → ganti semua `<...>` → `show | compare` → `commit check` → `commit confirmed`.

### 16.1 Template A — Golden Baseline (System & Hardening)

**Tujuan:** Baseline awal untuk setiap device baru. Tanpa baseline ini, device tidak boleh masuk produksi.
**Kapan dipakai:** PE, P, RR, CE, switch — semua device.

```bash
[edit system]
set system host-name <HOSTNAME>
set system time-zone <IANA-TIMEZONE>
set system root-authentication encrypted-password <FROM-VAULT>
set system login user <ADMIN-USER> class super-user
set system login user <ADMIN-USER> authentication ssh-rsa "<SSH-PUBLIC-KEY>"
set system login user <NOC-USER> class operator
set system login user <NOC-USER> authentication encrypted-password <FROM-VAULT>
set system login message "Authorized Access Only. All activity is logged."
set system login idle-timeout 10

set system ntp boot-server <NTP-SERVER-1>
set system ntp server <NTP-SERVER-1> prefer
set system ntp server <NTP-SERVER-2>
set system ntp source-address <LOOPBACK-IP>

set system name-server <DNS-SERVER-1>
set system name-server <DNS-SERVER-2>
set system domain-name <DOMAIN>

set system syslog user * any emergency
set system syslog file messages any notice
set system syslog file interactive-commands interactive-commands any
set system syslog host <SYSLOG-SERVER> any info
set system syslog host <SYSLOG-SERVER> source-address <LOOPBACK-IP>
set system syslog archive size 10m files 10

set system services ssh root-login deny
set system services ssh protocol-version v2
set system services ssh max-sessions-per-connection 5
set system services ssh connection-limit 10
set system services ssh rate-limit 5
set system services netconf ssh
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `host-name` + `time-zone` | Identitas device & korelasi log antar device |
| `root-authentication` + `login user` | Least privilege. Root dibatasi, admin pakai SSH key, NOC hanya operator |
| `login message` + `idle-timeout` | Audit & auto-logout untuk sesi yang ditinggal |
| `ntp` + `source-address` | Log, BGP, troubleshooting butuh waktu sinkron. Source loopback agar stabil |
| `name-server` + `domain-name` | Resolusi nama untuk NTP / syslog / update |
| `syslog` | Audit trail. `interactive-commands` mencatat siapa mengubah apa |
| `services ssh` | Hardening. `root-login deny` + `protocol v2` + rate-limit cegah brute-force. Telnet / FTP dilarang |

**Verifikasi:** `show system uptime`, `show ntp associations`, `show system login`, `show configuration system | display set`
**Kesalahan umum:** Pakai password plaintext di Git, lupa `source-address` NTP, enable Telnet/FTP.

---

### 16.2 Template B — Interface + LACP

**Tujuan:** Interface core & agregasi yang konsisten MTU, description, dan LACP.
**Kapan dipakai:** Link core PE-P, P-P, RR-P, dan link agregasi ke switch / CE.

```bash
[edit interfaces]
# Loopback — identitas device
set interfaces lo0 unit 0 description "Loopback - <HOSTNAME> Router-ID"
set interfaces lo0 unit 0 family inet address <LOOPBACK-IP>/32

# Physical Core
set interfaces <CORE-IF> description "to <NEIGHBOR> <PORT> | <CIRCUIT-ID> | <BW>"
set interfaces <CORE-IF> mtu <MTU>
set interfaces <CORE-IF> unit 0 description "to <NEIGHBOR> - P2P Core"
set interfaces <CORE-IF> unit 0 family inet address <P2P-IP>/30
set interfaces <CORE-IF> unit 0 family mpls
set interfaces <CORE-IF> unit 0 family iso

# LACP — Aggregated Ethernet
set interfaces <AE-IF> description "LAG to <NEIGHBOR> | <BW>"
set interfaces <AE-IF> flexible-vlan-tagging
set interfaces <AE-IF> mtu <MTU>
set interfaces <AE-IF> aggregated-ether-options minimum-links 1
set interfaces <AE-IF> aggregated-ether-options link-speed <LINK-SPEED>
set interfaces <AE-IF> aggregated-ether-options lacp active
set interfaces <AE-IF> aggregated-ether-options lacp periodic fast
set interfaces <AE-IF> unit <VLAN-ID> description "<VLAN-DESCRIPTION>"
set interfaces <AE-IF> unit <VLAN-ID> vlan-id <VLAN-ID>
set interfaces <AE-IF> unit <VLAN-ID> family inet address <IP>/30

set interfaces <MEMBER-1> description "Member <AE-IF> to <NEIGHBOR> - Link 1"
set interfaces <MEMBER-1> gigether-options 802.3ad <AE-IF>
set interfaces <MEMBER-2> description "Member <AE-IF> to <NEIGHBOR> - Link 2"
set interfaces <MEMBER-2> gigether-options 802.3ad <AE-IF>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `lo0` | Router-ID, BGP, LDP, NTP source — jangan pakai IP fisik |
| `description` | Troubleshooting 10x lebih cepat. Tanpa description, NOC butuh 30 menit cari ujung kabel |
| `mtu` | Harus sama end-to-end. Mismatch = OSPF/IS-IS flap, MPLS drop diam-diam |
| `family mpls + iso` | Core butuh MPLS + IS-IS. Tanpa `family mpls`, LDP tidak jalan |
| `aeX + lacp active` | Redundansi + bandwidth. `active` deteksi misconfig & unidirectional link |
| `minimum-links` + `periodic fast` | `minimum-links 1` agar LAG tetap up dengan 1 link. `periodic fast` = deteksi 1 detik (default slow 30 detik) |
| `802.3ad` di member | Member hanya jadi anggota LAG. MTU/VLAN di `aeX`, bukan di member |

**Verifikasi:** `show interfaces terse`, `show lacp interfaces <AE-IF>`, `show interfaces <CORE-IF> extensive`
**Kesalahan umum:** MTU di member beda dengan `aeX`, LACP `passive/passive`, VLAN di member bukan di `aeX`.

---

### 16.3 Template C — OSPF + BFD

**Tujuan:** IGP untuk enterprise / core yang butuh konvergensi cepat dan aman.
**Kapan dipakai:** Core OSPF Area 0, atau PE-CE OSPF.

```bash
[edit]
set routing-options router-id <ROUTER-ID>

[edit protocols ospf]
set protocols ospf reference-bandwidth <REFERENCE-BW>
set protocols ospf traffic-engineering
set protocols ospf area <AREA-0> interface lo0.0 passive
set protocols ospf area <AREA-0> interface <CORE-IF>.0 interface-type p2p
set protocols ospf area <AREA-0> interface <CORE-IF>.0 metric <COST>
set protocols ospf area <AREA-0> interface <CORE-IF>.0 authentication md5 <KEY-ID> key <FROM-VAULT>
set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection minimum-interval <MS>
set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection multiplier <N>
set protocols ospf area <AREA-0> interface <CORE-IF>.0 bfd-liveness-detection full-neighbors-only
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `router-id` | Harus = loopback, unik, stabil. Jangan pakai IP fisik |
| `reference-bandwidth` | Samakan di seluruh domain agar cost konsisten. Jika beda, cost tidak bisa dibandingkan |
| `traffic-engineering` | Wajib jika pakai RSVP-TE |
| `lo0 passive` | Loopback tidak perlu kirim Hello. Tanpa passive, Hello terkirim ke loopback sia-sia |
| `interface-type p2p` | P2P tidak perlu DR/BDR election. Lebih cepat konvergensi |
| `metric` | Cost manual. Bukan MTU. Tune untuk traffic engineering |
| `authentication md5` | Cegah rogue router masuk OSPF. Tanpa auth, attacker bisa inject route |
| `bfd` | Deteksi failure sub-detik (mis. 300ms x 3 = 900ms), jauh lebih cepat dari Dead Interval (40 detik) |

**Verifikasi:** `show ospf neighbor` (Full), `show ospf interface`, `show bfd session`, `show route protocol ospf`
**Kesalahan umum:** Hello/dead & auth beda antar neighbor, MTU beda, lupa `passive` di lo0, `metric` dikira MTU.

---

### 16.4 Template D — IS-IS

**Tujuan:** IGP untuk core besar / ISP yang butuh scaling & roadmap SR-MPLS.
**Kapan dipakai:** Core ISP, atau migrasi dari OSPF ke IS-IS.

```bash
[edit]
set interfaces lo0 unit 0 family iso address <NET-ADDRESS>
set interfaces <CORE-IF> unit 0 family iso
set interfaces <CORE-IF> unit 0 family inet address <P2P-IP>/30
set interfaces lo0 unit 0 family inet address <LOOPBACK-IP>/32

[edit protocols isis]
set protocols isis interface lo0.0 passive
set protocols isis interface <CORE-IF>.0
set protocols isis level 1 wide-metrics-only
set protocols isis level 2 wide-metrics-only
# set protocols isis authentication-key <FROM-VAULT>
# set protocols isis level 1 authentication-key <FROM-VAULT>
# set protocols isis level 2 authentication-key <FROM-VAULT>
# set protocols isis interface <CORE-IF>.0 hello-interval <HELLO>
# set protocols isis interface <CORE-IF>.0 hold-time <HOLD>
# set protocols isis interface <CORE-IF>.0 bfd-liveness-detection minimum-interval <MS>
# set protocols isis interface <CORE-IF>.0 bfd-liveness-detection multiplier <N>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `family iso` + `NET-ADDRESS` | IS-IS adjacency pakai CLNS, bukan IP. Tanpa `family iso`, adjacency tidak terbentuk. NET dari IPAM |
| `lo0 passive` | Loopback tidak perlu Hello IS-IS |
| `wide-metrics-only` | Wajib untuk network modern. Mendukung metric >63 & TLV baru. Tanpa ini, metric dibatasi 63 |
| `authentication-key` | Cegah rogue router. Level 1 & Level 2 bisa beda key |
| `hello-interval` / `hold-time` | Default Hello 9 detik. Tune hanya jika paham dampaknya |

**Verifikasi:** `show isis adjacency` (Up), `show isis interface`, `show isis database detail`, `show route protocol isis`
**Kesalahan umum:** Lupa `family iso`, NET salah format, `wide-metrics-only` tidak di-set, MTU beda.

---

### 16.5 Template E — BGP Route Reflector (RR + PE Client)

**Tujuan:** Skala BGP iBGP tanpa full-mesh. PE hanya peering ke RR.
**Kapan dipakai:** Core dengan >3 PE, atau layanan L3VPN yang butuh VPNv4.

```bash
# --- RR (Route Reflector) ---
[edit]
set routing-options graceful-restart
set routing-options router-id <RR-ROUTER-ID>
set routing-options autonomous-system <AS>

set protocols bgp log-updown
set protocols bgp group <RR-CLIENTS> type internal
set protocols bgp group <RR-CLIENTS> local-address <RR-LOOPBACK>
set protocols bgp group <RR-CLIENTS> family inet-vpn unicast
set protocols bgp group <RR-CLIENTS> authentication-key <FROM-VAULT>
set protocols bgp group <RR-CLIENTS> cluster <RR-CLUSTER-ID>
set protocols bgp group <RR-CLIENTS> peer-as <AS>
set protocols bgp group <RR-CLIENTS> neighbor <PE-LOOPBACK-1> description "<PE-1>"
set protocols bgp group <RR-CLIENTS> neighbor <PE-LOOPBACK-2> description "<PE-2>"

# --- PE Client ---
[edit]
set routing-options graceful-restart
set routing-options router-id <PE-ROUTER-ID>
set routing-options autonomous-system <AS>
set protocols bgp log-updown
set protocols bgp group <TO-RR> type internal
set protocols bgp group <TO-RR> local-address <PE-LOOPBACK>
set protocols bgp group <TO-RR> family inet-vpn unicast
set protocols bgp group <TO-RR> authentication-key <FROM-VAULT>
set protocols bgp group <TO-RR> peer-as <AS>
set protocols bgp group <TO-RR> neighbor <RR-LOOPBACK-1> description "<RR-1>"
set protocols bgp group <TO-RR> neighbor <RR-LOOPBACK-2> description "<RR-2>"
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `graceful-restart` | Upgrade / restart RE tanpa drop BGP |
| `router-id` + `local-address` | Harus = loopback. Session stabil walau 1 link putus |
| `family inet-vpn unicast` | Untuk L3VPN (VPNv4). Tanpa ini, RR tidak reflect route VRF |
| `cluster` | Cluster-ID unik per RR (= Router-ID RR). Duplikat Cluster-ID merusak loop prevention |
| `authentication-key` | Cegah rogue BGP peer. Key dari vault, bukan manual typing |
| `log-updown` | Audit BGP flap di log |
| `2 RR` | Redundancy. PE peering ke keduanya. 1 RR = single point of failure |

**Verifikasi:** `show bgp summary` (Established ke 2 RR), `show bgp group <GROUP>`, `show route advertising-protocol bgp <PE>`, `show route receive-protocol bgp <PE>`
**Kesalahan umum:** Cluster-ID sama antar RR, key beda, hanya 1 RR, lupa `family inet-vpn`.

---

### 16.6 Template F — MPLS + LDP

**Tujuan:** Fondasi untuk L2VPN / VPLS / L3VPN. Distribusi label otomatis.
**Kapan dipakai:** Semua router core (P, PE, RR).

```bash
[edit protocols]
set protocols mpls no-propagate-ttl
set protocols mpls interface lo0.0
set protocols mpls interface <CORE-IF>.0

set protocols ldp interface lo0.0
set protocols ldp interface <CORE-IF>.0
# set protocols ldp session <NEIGHBOR-LOOPBACK> authentication-key <FROM-VAULT>

set protocols lldp interface <CORE-IF>.0
set protocols lldp management-address <LOOPBACK-IP>

# RSVP — hanya jika butuh TE / FRR
# set protocols rsvp interface lo0.0
# set protocols rsvp interface <CORE-IF>.0
# set protocols rsvp interface <CORE-IF>.0 link-protection
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `no-propagate-ttl` | Sembunyikan TTL core dari traceroute user. Best practice ISP — user tidak lihat hop core |
| `mpls interface` | Aktifkan MPLS di interface. Tanpa ini, label tidak di-forward |
| `ldp interface` | Distribusi label otomatis. LDP session terbentuk via IGP |
| `ldp session + auth` | Auth per neighbor untuk keamanan LDP |
| `lldp` | Neighbor discovery untuk dokumentasi & troubleshooting |
| `rsvp + link-protection` | Hanya jika butuh TE / FRR. Jika tidak butuh TE, LDP saja lebih sederhana |

**Verifikasi:** `show mpls interface`, `show mpls lsp`, `show ldp neighbor`, `show ldp session`, `show route forwarding-table family mpls`, `show lldp neighbors`
**Kesalahan umum:** IGP belum Full tapi LDP dipaksa, lupa `family mpls` di interface, MTU tidak cukup untuk label overhead.

---

### 16.7 Template G — L3VPN VRF + PE-CE BGP (AS-Override)

**Tujuan:** Isolasi routing antar customer. Customer bisa pakai IP yang sama tanpa konflik.
**Kapan dipakai:** Layanan L3VPN enterprise, atau customer dengan overlapping IP.

```bash
# 1. Policy — RT Import/Export
[edit policy-options]
set policy-options community <RT> members <RT>
set policy-options policy-statement <VRF>-EXPORT term 10 then community add <RT>
set policy-options policy-statement <VRF>-EXPORT term 10 then accept
set policy-options policy-statement <VRF>-IMPORT term 10 from community <RT>
set policy-options policy-statement <VRF>-IMPORT term 10 then accept
set policy-options policy-statement <VRF>-IMPORT term 99 then reject

# 2. VRF Instance
[edit routing-instances <VRF>]
set routing-instances <VRF> instance-type vrf
set routing-instances <VRF> interface <CE-IF>.0
set routing-instances <VRF> route-distinguisher <RD>
set routing-instances <VRF> vrf-import <VRF>-IMPORT
set routing-instances <VRF> vrf-export <VRF>-EXPORT
set routing-instances <VRF> vrf-table-label

# 3. Interface ke CE
[edit interfaces <CE-IF>]
set interfaces <CE-IF> unit 0 description "to <CE>"
set interfaces <CE-IF> encapsulation ethernet
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> unit 0 family inet address <PE-CE-IP>/30

# 4. PE-CE BGP dengan AS-Override (jika CE di 2 site pakai AS sama)
[edit routing-instances <VRF>]
set routing-instances <VRF> protocols bgp group <TO-CE> type external
set routing-instances <VRF> protocols bgp group <TO-CE> family inet unicast
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> as-override
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> peer-as <CE-AS>
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> local-as <PROVIDER-AS>
set routing-instances <VRF> protocols bgp group <TO-CE> neighbor <CE-IP> export <EXPORT-TO-CE>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `RD` | Membuat prefix unik di core (`<AS>:<NN>`). Harus unik per VRF per PE. Reuse RD = route tercampur |
| `RT` | Menentukan siapa bisa import route siapa. RT salah = customer A bisa lihat route customer B |
| `vrf-table-label` | Per-VRF label. Hemat label, best practice |
| `vrf-import / vrf-export` | Policy yang pakai RT. Tanpa ini, VRF tidak import/export route |
| `as-override` | PE ganti AS CE dengan AS-nya sendiri saat kirim ke CE. Solusi AS-loop saat CE di 2 site pakai AS sama. Tanpa ini, BGP reject karena AS-loop |
| `export <EXPORT-TO-CE>` | Hanya prefix yang diizinkan yang di-advertise ke CE. Tanpa filter, semua route bocor ke CE |

**Verifikasi:** `show route table <VRF>.inet.0`, `show bgp summary instance <VRF>`, `ping routing-instance <VRF> <DEST> source <SRC>`
**Kesalahan umum:** RD sama untuk 2 customer, RT salah, lupa `vrf-table-label`, `as-override` tidak di-set padahal CE AS sama.

---

### 16.8 Template H — L2VPN (L2Circuit)

**Tujuan:** Hubungkan 2 site seolah 1 kabel (Point-to-Point L2).
**Kapan dipakai:** Branch ↔ HQ 1 VLAN, atau 2 site yang butuh L2 murni.

```bash
[edit protocols l2circuit]
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> virtual-circuit-id <VC-ID>
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> description "<DESCRIPTION>"
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> no-control-word
set protocols l2circuit neighbor <FAR-PE-LOOPBACK> interface <CE-IF>.<VLAN> ignore-encapsulation-mismatch

[edit interfaces <CE-IF>]
set interfaces <CE-IF> flexible-vlan-tagging
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> encapsulation flexible-ethernet-services
set interfaces <CE-IF> unit <VLAN> description "<DESCRIPTION>"
set interfaces <CE-IF> unit <VLAN> encapsulation vlan-ccc
set interfaces <CE-IF> unit <VLAN> vlan-id <VLAN-ID>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `virtual-circuit-id` | Harus identik di 2 PE untuk 1 layanan. VC-ID beda = L2Circuit Down |
| `vlan-ccc` | Encapsulation untuk L2Circuit. Bukan `ethernet` biasa |
| `mtu` | Harus >1500 untuk overhead MPLS, dan sama di 2 PE. Mismatch = Down |
| `no-control-word` | Harus sama di 2 PE. Beda = Down |
| `ignore-encapsulation-mismatch` | Toleransi, bukan pengganti samakan encapsulation |

**Verifikasi:** `show l2circuit connections` (Up), `show l2circuit connections extensive`, `ping mpls l2circuit virtual-circuit <VC-ID> neighbor <FAR-PE>`
**Kesalahan umum:** VC-ID beda, VLAN beda, MTU beda, control-word beda — 4 penyebab Down paling sering.

---

### 16.9 Template I — VPLS

**Tujuan:** Banyak site seolah 1 switch (Multipoint L2).
**Kapan dipakai:** 3+ site retail / branch dalam 1 broadcast domain.

```bash
[edit routing-instances <VPLS-INSTANCE>]
set routing-instances <VPLS-INSTANCE> instance-type vpls
set routing-instances <VPLS-INSTANCE> interface <CE-IF>.0
set routing-instances <VPLS-INSTANCE> protocols vpls encapsulation-type ethernet
set routing-instances <VPLS-INSTANCE> protocols vpls no-control-word
set routing-instances <VPLS-INSTANCE> protocols vpls vpls-id <VPLS-ID>
set routing-instances <VPLS-INSTANCE> protocols vpls mtu <MTU>
set routing-instances <VPLS-INSTANCE> protocols vpls ignore-mtu-mismatch
set routing-instances <VPLS-INSTANCE> protocols vpls ignore-encapsulation-mismatch
set routing-instances <VPLS-INSTANCE> protocols vpls neighbor <FAR-PE-LOOPBACK> switchover-delay <MS>
set routing-instances <VPLS-INSTANCE> protocols vpls neighbor <FAR-PE-LOOPBACK> backup-neighbor <BACKUP-PE-LOOPBACK> standby

[edit interfaces <CE-IF>]
set interfaces <CE-IF> unit 0 description "<DESCRIPTION>"
set interfaces <CE-IF> mtu <MTU>
set interfaces <CE-IF> encapsulation ethernet-vpls
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `vpls-id` | Harus identik di semua PE dalam 1 VPLS. Beda = VPLS Down |
| `encapsulation-type ethernet` | Tipe VPLS. Harus sama di semua PE |
| `backup-neighbor standby` | Redundansi. Jika PE primary down, backup takeover |
| `storm-control + MAC limit` | **Wajib dari awal, bukan saat storm.** Batasi broadcast & MAC untuk cegah broadcast storm (lihat Kasus 14.5) |
| `ethernet-vpls` | Encapsulation di interface CE. Bukan `ethernet` biasa |

**Verifikasi:** `show vpls connections` (Up), `show vpls mac-table` (MAC belajar)
**Kesalahan umum:** VPLS-ID beda, lupa `ethernet-vpls`, tidak ada storm-control / MAC limit.

---

### 16.10 Template J — Firewall Filter & RE Protection

**Tujuan:** Lindungi Routing Engine dari traffic yang tidak perlu & serangan.
**Kapan dipakai:** Wajib di semua device produksi. Apply ke `lo0`.

```bash
[edit firewall]
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from source-address <MGMT-BLOCK>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from protocol tcp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH from destination-port 22
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH then policer <SSH-POLICER>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-SSH then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from source-address <BGP-NEIGHBOR-BLOCK>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from protocol tcp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP from destination-port 179
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP then policer <BGP-POLICER>
set firewall filter <PROTECT-RE-FILTER> term ALLOW-BGP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-IGP from protocol ospf
set firewall filter <PROTECT-RE-FILTER> term ALLOW-IGP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP from protocol icmp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP from icmp-type echo-request
set firewall filter <PROTECT-RE-FILTER> term ALLOW-ICMP then accept

set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP from protocol udp
set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP from destination-port 123
set firewall filter <PROTECT-RE-FILTER> term ALLOW-NTP then accept

set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then count <DENY-COUNTER>
set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then log
set firewall filter <PROTECT-RE-FILTER> term DENY-ALL then discard

# Policer — dari policy keamanan
# set firewall policer <SSH-POLICER> if-exceeding bandwidth-limit <BW>
# set firewall policer <SSH-POLICER> if-exceeding burst-size-limit <BURST>
# set firewall policer <SSH-POLICER> then discard

[edit interfaces lo0]
set interfaces lo0 unit 0 family inet filter input <PROTECT-RE-FILTER>
```

**Penjelasan Per Blok:**

| Blok | Kenapa Penting |
| :--- | :--- |
| `source-address` | Hanya neighbor / MGMT yang sah yang boleh BGP / SSH / SNMP ke RE. Tanpa ini, semua IP bisa serang RE |
| `policer` | Rate-limit. Tanpa policer, RE bisa dibanjiri walau dari IP sah |
| `ALLOW-IGP` | OSPF / IS-IS harus diizinkan, sonst IGP Down |
| `DENY-ALL + count + log` | Default deny eksplisit. `count` & `log` untuk audit & early warning DDoS |
| `filter input di lo0` | Yang di-apply adalah **nama filter**, bukan nama term. Salah nama = filter tidak jalan |

**Verifikasi:** `show firewall filter <FILTER>`, `show firewall counter`, `show firewall log`
**Kesalahan umum:** Lupa `source-address`, lupa `policer`, apply nama term bukan nama filter, lupa izinkan IGP/BGP sehingga IGP Down sendiri.

---

### 16.11 Cara Pakai Template

1. **Copy template** → ganti semua `<...>` dari IPAM / vault.
2. **Jangan copy-paste 10 template sekaligus.** Load per template, verifikasi per template.
3. **`show | compare`** sebelum setiap `commit`.
4. **`commit confirmed 5`** untuk perubahan remote.
5. **Verifikasi** sesuai kolom Verifikasi di tiap template.
6. **Simpan** config yang sudah terverifikasi sebagai Golden Config di repo.

---

## 17. Lab Topology — Abstract

> **Tujuan bab ini:** Menunjukkan struktur lab, bukan mempublish addressing produksi.
> **Alamat di bawah ini adalah placeholder / blok contoh.** Alokasi aktual dari IPAM internal. Jangan copy-paste ke produksi tanpa IPAM.

### 17.1 Prinsip Desain Lab

| Prinsip | Penjelasan |
| :--- | :--- |
| Loopback adalah identitas | Semua BGP, LDP, RR, NTP, SNMP pakai loopback |
| Dual-homing | Setiap PE ke 2 P, setiap P ke 2 RR |
| Satu AS, satu IGP | Satu AS, OSPF Area 0 atau IS-IS L2 di core |
| Deterministic | Semua subnet bisa ditebak dari IPAM, tidak ada IP acak |

### 17.2 Node Inventory — Struktur (Bukan Nilai Produksi)

| Role | Jumlah | Fungsi | Catatan |
| :--- | :--- | :--- | :--- |
| **Route Reflector** | 2 | RR Primary + Backup | Cluster-ID unik per RR |
| **P Router** | 2 | Core label switching only | Tanpa VRF, tanpa BGP VPN |
| **PE Router** | 3 | Provider Edge | VRF: L3VPN, L2VPN, VPLS |
| **CE — L3VPN** | 2 | Customer Edge L3VPN | PE-CE BGP |
| **CE — L2VPN** | 2 | Customer Edge L2VPN | L2 murni, VC-ID sama di 2 PE |
| **CE — VPLS** | 3 | Customer Edge VPLS | VPLS-ID sama di semua PE dalam VPLS yang sama |

**Resource Lab (per VM):** 2 vCPU, 4 GB RAM, 3-6 vNIC. Platform: EVE-NG / PNETLab / GNS3 / VMware.

### 17.3 Addressing Plan — Blok (Isi dari IPAM)

| Blok | Penggunaan | Contoh Format |
| :--- | :--- | :--- |
| `<LOOPBACK-BLOCK>` | Router-ID, BGP, LDP, RR, NTP source | `<LOOPBACK-BLOCK>/32` per device |
| `<MGMT-BLOCK>` | OOB, SSH, SNMP, Syslog | `<MGMT-BLOCK>/24` |
| `<P2P-BLOCK>` | P2P PE-P, P-P, RR-P | `<P2P-BLOCK>/30` per link |
| `<PE-CE-BLOCK>` | PE-CE L3VPN | `<PE-CE-BLOCK>/30` per link |
| `<CUSTOMER-LAN-BLOCK>` | LAN customer di CE | `<CUSTOMER-LAN-BLOCK>/24` |

| Parameter Layanan | Aturan |
| :--- | :--- |
| RD | `<AS>:<NN>` — Unik per VRF per PE |
| RT | `target:<AS>:<NN>` — Menentukan import/export |
| VC-ID (L2VPN) | Harus identik di 2 PE untuk 1 layanan |
| VPLS-ID | Harus identik di semua PE dalam 1 VPLS |
| Sham-Link | Endpoint `lo0.1` di VRF, cost di-tune lebih murah dari backdoor jika MPLS adalah primary |

### 17.4 Urutan Lab — Wajib Berurutan

| Lab | Judul | Success Criteria |
| :--- | :--- | :--- |
| LAB 01 | Golden Baseline & Hardening | SSH OK, NTP sync |
| LAB 02 | Interface & LACP | LACP Collecting + Distributing |
| LAB 03 | OSPF + BFD | OSPF Full semua link |
| LAB 04 | IS-IS (Alternatif) | IS-IS adjacency Up |
| LAB 05 | MPLS + LDP + LLDP | LDP Operational |
| LAB 06 | BGP Route Reflector | BGP Established ke 2 RR |
| LAB 07 | Policy & Redistribution | Route sesuai filter |
| LAB 08 | Firewall Filter | Prefix tidak sah ditolak |
| LAB 09 | L2VPN | L2Circuit Up, CE-CE ping |
| LAB 10 | VPLS | VPLS Up, MAC belajar, CE-CE ping |
| LAB 11 | L3VPN Basic | VRF route ada, CE-CE ping |
| LAB 12 | L3VPN AS-Override | Tanpa as-override: ditolak (AS-loop). Dengan: Established |
| LAB 13 | L3VPN Sham-Link | OSPF neighbor via sham-link Full |
| LAB 14 | Failure & Failover | Matikan 1 P → traffic via P lain |
| LAB 15 | Backup / Rollback | Rollback berhasil ke baseline |

**Urutan wajib:** LAB 01 → 03 → 05 → 06 → 09/10/11. Jangan lompat ke L2VPN/L3VPN sebelum IGP + LDP + BGP Established.

---

## 18. Config Library — Index

> **File `.conf` adalah template.** Isi semua `<PLACEHOLDER>` dari IPAM / vault sebelum load. Jangan load template mentah ke produksi.

### 18.1 Inventaris File

| # | File | Role | Isi Utama |
| :--- | :--- | :--- | :--- |
| 1 | `Junos-PE-Golden-Config.conf` | PE Golden | Baseline PE: PROTECT-RE, SNMPv3, OSPF+BFD, BGP TO-RR, MPLS+LDP, VRF template |
| 2 | `Junos-PE-Golden-Config-SET.txt` | PE Golden (SET) | Versi `set` dari file #1, untuk paste via terminal |
| 3 | `Junos-P-01-Golden-Config.conf` | P Core | Core P, OSPF+BFD, MPLS+LDP+RSVP, Tanpa VRF, Tanpa BGP VPN |
| 4 | `Junos-P-02-Golden-Config.conf` | P Core | Mirror P-01 |
| 5 | `Junos-RR-01-Golden-Config.conf` | RR Primary | Cluster unik, BGP RR-CLIENTS → PE, RR-TO-RR → RR-02 |
| 6 | `Junos-RR-02-Golden-Config.conf` | RR Backup | Cluster unik, mirror RR-01 |
| 7 | `Junos-CE-A1-Config.conf` | CE L3VPN | PE-CE BGP, LAN di CE |
| 8 | `Junos-CE-A2-Config.conf` | CE L3VPN | PE-CE BGP, AS sama untuk test AS-Override |
| 9 | `Junos-CE-B1-Config.conf` | CE L2VPN | VLAN, L2 murni, Tanpa BGP/OSPF ke PE |
| 10 | `Junos-CE-B2-Config.conf` | CE L2VPN | VLAN, mirror CE-B1 |
| 11 | `Junos-CE-C1-Config.conf` | CE VPLS | VPLS, L2 multipoint |
| 12 | `Junos-CE-C2-Config.conf` | CE VPLS | VPLS |
| 13 | `Junos-CE-C3-Config.conf` | CE VPLS | VPLS |
| 14 | `Junos-Lab-Topology-IP-Plan.md` / `.pdf` | Dokumen Lab | IP Plan — **Internal Only, bukan untuk README public** |
| 15 | `Junos-OS-Professional-Guide.md` / `.pdf` | Guide ID | Guide lengkap Bahasa Indonesia — **Internal** |
| 16 | `Junos-OS-Professional-Guide-EN.md` / `.pdf` | Guide EN | Guide English — **Customer-ready, tanpa nilai produksi** |

### 18.2 Prinsip Per Role

**PE — Golden Config:** Setiap PE baru. Ganti `<HOSTNAME>`, `<LOOPBACK-IP>`, `<AS>`, `<MGMT-IP>`, `<NTP>`, `<SYSLOG>`, `<RR>`, semua kredensial dari vault.

**P Router:** Label switching ONLY. Tidak punya VRF, tidak punya BGP VPN. Jika P butuh BGP VPN, itu salah desain.

**RR:** PE hanya peering ke RR. RR tidak punya VRF, tidak punya CE. Hanya reflect VPNv4 + IPv4 unicast. RR harus punya IGP + MPLS + LDP seperti P.

**CE — L3VPN:** PE-CE BGP. AS sama di 2 CE sengaja untuk test AS-Override. Blok OSPF Sham-Link tersedia di file (commented) untuk LAB 13.

**CE — L2VPN:** L2 murni. PE tidak routing. CE seolah 1 kabel. Traceroute 1 hop.

**CE — VPLS:** 1 switch virtual. Semua CE dalam 1 subnet yang sama. Broadcast ARP menyebar ke semua site.

### 18.3 Cara Load

```bash
# Device baru — RECOMMENDED
[edit]
# load override /var/tmp/<FILE>.conf
# show | compare
# commit confirmed 10
# commit comment "<TICKET>: <deskripsi>"

# Device existing — merge, tidak delete
[edit]
# load merge /var/tmp/<FILE>.conf
# show | compare
# commit confirmed 10
```

**Verifikasi berurutan (jangan lompat):**
```bash
show ospf neighbor          # Full
show ldp neighbor           # Operational
show bgp summary            # Established ke 2 RR
show l2circuit connections # Up
show vpls connections      # Up
show route table <VRF>.inet.0
ping routing-instance <VRF> <DEST> source <SRC>
```

**Checkpoint:** Setelah LAB 06 sukses (BGP Established), snapshot semua VM.

---

---

## Bagian IV — Operasi & Troubleshooting

## 19. Operasi & Change Management

### 19.1 Backup & Restore

```bash
# Backup lokal
root# save /var/tmp/<HOSTNAME>-<DATE>.conf

# Backup remote (dari Operational Mode)
root> file copy /config/juniper.conf.gz scp://<BACKUP-USER>@<BACKUP-SERVER>:/backup/<HOSTNAME>/

# Restore — HATI-HATI: override = ganti seluruh config
root# load override /var/tmp/<BACKUP>.conf
root# show | compare
root# commit confirmed 5

# Load Merge — merge, tidak delete existing
root# load merge /var/tmp/<PARTIAL>.conf
root# show | compare
root# commit confirmed 5
```

**Automated Backup:** Pakai `system archival configuration transfer-on-commit` + `archive-sites` dari vault. Jangan hardcode kredensial di README.

### 19.2 Troubleshooting Framework

1. **Apa yang berubah?** — `show system commit`, `show | compare`
2. **Layer 1/2** — `show interfaces terse`, `show lacp`
3. **Layer 3 / IGP** — `show ospf neighbor`, `show isis adjacency`, `show route`
4. **MPLS / BGP** — `show mpls lsp`, `show ldp neighbor`, `show bgp summary`
5. **Service** — `show l2circuit`, `show vpls`, `show route table <VRF>`
6. **Log** — `show log messages | last 50`, `show log chassisd`

| Gejala | Penyebab Paling Sering | Fix |
| :--- | :--- | :--- |
| OSPF `Init` | One-way Hello, auth mismatch, MTU mismatch | Cek hello/dead, auth, MTU, firewall multicast |
| OSPF `ExStart` | MTU mismatch, DBD problem | Samakan MTU di core |
| LDP `Non-Existent` | IGP belum Full, loopback tidak ter-reach | Cek `show route <LOOPBACK>` |
| BGP `Active` | TCP 179 diblokir, `local-address` salah, AS salah | Cek PROTECT-RE izinkan 179 |
| BGP `Idle` | `peer-as` salah, `authentication-key` mismatch | Cek peer-as, key dari vault |
| L2VPN `Down` | VC-ID beda, VLAN mismatch, MTU mismatch, control-word beda | Samakan VC-ID, VLAN, MTU |
| VPLS `Down` | VPLS-ID beda, encapsulation salah | Samakan VPLS-ID |
| L3VPN ping gagal | RD/RT mismatch, MP-BGP tidak Established | Cek BGP, VRF, RT |
| AS-Override tidak jalan | CE AS beda, `as-override` tidak di PE | Pastikan CE AS sama untuk test |
| Sham-Link tidak Full | `lo0.1` tidak di VRF, `sham-link-remote` salah | Cek `lo0.1` di VRF |

### 19.3 Prosedur Change Management

**Pre-Change:** Pastikan tiket / CR approved dan maintenance window jelas, config backup terbaru tersimpan, `show system commit` tidak menunjukkan commit yang tidak dijelaskan, rollback plan sudah ditulis (rollback ke commit mana), `commit check` lolos, dan peer / NOC sudah diberitahu.

**During Change:** Gunakan `commit confirmed <5-10>` untuk perubahan remote, lakukan `show | compare` sebelum setiap commit, uji dengan `ping`, `show ospf neighbor`, `show bgp summary`, `show mpls lsp`, `show route table <VRF>`, dan catat waktu serta output verifikasi di tiket.

**Post-Change:** Lakukan `commit` final sebagai konfirmasi `commit confirmed`, verifikasi selama 15 menit untuk memastikan tidak ada flap, alarm, atau error log, pastikan `show system commit` menunjukkan commit comment yang benar, update dokumentasi / diagram jika topologi berubah, dan tutup tiket dengan bukti (before/after).

---


---

## 20. Troubleshooting Expert: Packet Walk + Traceoptions

> **Kenapa bab ini ada:** Troubleshooting junior: cek `show ospf neighbor`, jika Down, restart. Troubleshooting expert: ikuti packet dari CE ke CE, di hop mana hilang, kenapa hilang. Bab ini adalah framework expert.

### 20.1 Packet Walk — L3VPN

```
CE-A1 (10.10.1.10) → PE-01 (VRF) → P-01 (MPLS label) → P-02 → PE-02 (VRF) → CE-A2 (10.10.2.10)

Langkah 1: Di CE-A1, ping & traceroute — apakah sampai PE-01?
Langkah 2: Di PE-01, show route table VRF — apakah route ke 10.10.2.0/24 ada? Via BGP? Via MP-BGP?
Langkah 3: Di PE-01, show route forwarding-table vpn VRF — apakah label ada?
Langkah 4: Di P-01, show route forwarding-table family mpls — apakah label swap benar?
Langkah 5: Di PE-02, show route table VRF — apakah route ter-import? RT benar?
Langkah 6: Di CE-A2, show route — apakah route sampai CE?
```

**Prinsip:** Jangan lompat ke Langkah 6 jika Langkah 2 belum dicek. 80% waktu troubleshooting terbuang karena lompat langkah.

### 20.2 Traceoptions — Debug Protokol

```bash
# OSPF Traceoptions (contoh, jangan enable di produksi tanpa filter)
[edit protocols ospf]
set protocols ospf traceoptions file ospf.log size 10m files 5
set protocols ospf traceoptions flag hello detail
set protocols ospf traceoptions flag error detail
set protocols ospf traceoptions flag lsa-update detail

# BGP Traceoptions
[edit protocols bgp]
set protocols bgp traceoptions file bgp.log size 10m files 5
set protocols bgp traceoptions flag state detail
set protocols bgp traceoptions flag update detail

# LDP Traceoptions
[edit protocols ldp]
set protocols ldp traceoptions file ldp.log size 10m files 5
set protocols ldp traceoptions flag state detail

# Verifikasi & Matikan Setelah Debug
run show log ospf.log | last 50
run show log bgp.log | last 50
# Setelah selesai, deactivate traceoptions — jangan biarkan jalan terus, bisa penuhi disk & CPU
# deactivate protocols ospf traceoptions
# deactivate protocols bgp traceoptions
```

**Penjelasan:**

| Alat | Kapan Dipakai |
| :--- | :--- |
| `show route` + `show route forwarding-table` | Cek control-plane vs data-plane. Route ada di control-plane tapi tidak di forwarding = PFE problem |
| `traceoptions` | Cek Hello, LSA, BGP Update. Untuk adjacency flap yang tidak jelas penyebabnya |
| `monitor traffic interface <IF>` | Capture packet di interface. Untuk MTU, auth, Hello mismatch |
| `show pfe` / `show system resource-monitor` | Cek PFE CPU, RE CPU. Untuk serangan / loop |
| `ping mpls l2circuit` / `ping routing-instance` | Cek MPLS / VRF path. Untuk L2VPN / L3VPN down |

### 20.3 Urutan Troubleshooting 15 Menit

| Menit | Langkah |
| :--- | :--- |
| 0-2 | Apa yang berubah? `show system commit`, `show | compare`, tiket terakhir |
| 2-5 | Layer 1/2: `show interfaces terse`, `show lacp`, `show chassis alarms` |
| 5-8 | IGP: `show ospf neighbor`, `show isis adjacency`, `show route` |
| 8-11 | MPLS/BGP: `show mpls lsp`, `show ldp neighbor`, `show bgp summary` |
| 11-14 | Service: `show l2circuit`, `show vpls`, `show route table <VRF>` |
| 14-15 | Log: `show log messages | last 50`, `show firewall log` |

**Kesalahan umum:** Langsung restart device, langsung `clear` tanpa baca log, langsung ubah config tanpa `show | compare`. Restart menyembunyikan RCA, bukan memperbaiki.

---

## 21. Studi Kasus Expert — Pola Desain

> Bab ini berisi **pola desain & pola failure**, bukan post-mortem internal. Nama customer, skala, angka kerugian, dan detail kerentanan tidak dipublikasikan. Setiap kasus: `Konteks → Pola Failure → RCA (generik) → Desain Solusi → Verifikasi → Lesson`.

---

### 21.1 Migrasi IGP Core Tanpa Downtime

**Konteks:** IGP lama mulai tidak stabil saat skala bertambah. IGP baru dipilih karena scaling & roadmap yang lebih cocok. Constraint: downtime tidak diperbolehkan untuk layanan enterprise.

**Pola Failure:** Migrasi flag-day (shutdown IGP lama, enable IGP baru) berisiko downtime & rollback sulit. Migrasi tanpa persiapan mengungkap hutang teknis (mis. MTU tidak konsisten) yang sebelumnya disembunyikan IGP lama.

**Desain yang Dipilih — Ships-in-the-Night:**
1. Jalankan IGP lama + IGP baru bersamaan. IGP baru dibangun dulu (adjacency Up, database sync), tapi belum dipakai forwarding karena preference dibuat kurang preferred.
2. Pilot di subset router. Balik preference — IGP baru jadi preferred, IGP lama jadi backup.
3. Rollout bertahap per batch, monitor per batch, rollback per-router jika anomaly.
4. Setelah stabil, hapus IGP lama.

**Verifikasi:** `show isis adjacency`, `show ospf neighbor`, `show route protocol <IGP>`, `show bfd session`, `show interfaces | match MTU`

**Lesson:**
* Audit MTU sebelum migrasi IGP. Jangan asumsi semua link sudah konsisten.
* Preference adalah alat migrasi — pahami bahwa preference lebih kecil = lebih preferred.
* Ships-in-the-night lebih kompleks di config, tapi jauh lebih aman daripada flag-day.
* Pilot subset wajib. Rollback plan harus per-router, bukan per-domain.

---

### 21.2 L3VPN Overlapping IP Antar Customer

**Konteks:** Dua customer memakai prefix yang sama di LAN mereka. Tanpa isolasi, route tercampur — pelanggaran isolasi & audit.

**Pola Failure:** RD yang sama untuk 2 VRF membuat VPNv4 yang sama di core. BGP menganggap 1 route, route yang lain hilang. RT yang salah membuat customer A bisa import route customer B.

**Desain:**
* RD unik per VRF per PE. Jangan pernah reuse RD untuk 2 customer.
* RT terpisah per customer. RT menentukan siapa bisa bicara dengan siapa.
* Interface fisik / sub-interface VLAN terpisah per VRF.
* `vrf-table-label` untuk efisiensi label.

**Verifikasi Isolasi (Wajib untuk Audit):**
```bash
show route table <VRF-A>.inet.0
show route table <VRF-B>.inet.0
ping routing-instance <VRF-A> <DEST-A> source <SRC-A>   # Harus sukses
ping routing-instance <VRF-A> <DEST-HANYA-DI-B>          # Harus gagal (no route)
show route protocol bgp table inet.6  # RD unik terlihat di VPNv4
```

**Lesson:** RD = unik. RT = kebijakan. Test isolasi adalah bagian dari change, bukan opsional.

---

### 21.3 L2VPN Down Setelah Change MTU

**Konteks:** Change MTU di 1 PE untuk persiapan jumbo frame, tapi PE far-end belum diubah.

**Pola Failure:** MTU mismatch + control-word mismatch membuat L2Circuit Down. OSPF/BGP mungkin tetap Up, tapi L2Circuit Down — silent failure jika hanya cek IGP.

**Investigasi yang Benar:**
```bash
show l2circuit connections extensive   # Lihat MTU, control-word, encapsulation — bukan hanya Up/Down
show interfaces <CE-IF> extensive | match MTU
ping mpls l2circuit virtual-circuit <VC-ID> neighbor <FAR-PE>
show log messages | match l2circuit
```

**Desain Preventif:**
* Template L2VPN harus menyertakan MTU + control-word + `ignore-mismatch` secara eksplisit di semua PE.
* `ignore-mtu-mismatch` hanya untuk toleransi, bukan pengganti samakan MTU.
* Change harus atomic di kedua PE dalam change yang sama.
* Pakai `commit confirmed` dan tunggu verifikasi penuh sebelum `commit` permanen.

**Lesson:** MTU adalah silent killer L2VPN/VPLS. Cek kedua PE, bukan hanya 1 PE.

---

### 21.4 Route Reflector Redundancy

**Konteks:** PE hanya belajar route dari RR. Jika RR primary down dan RR backup tidak sehat, PE kehilangan semua route VPNv4 — VRF kosong walau IGP & LDP masih Up.

**Pola Failure:** Dual RR tidak ada artinya jika RR backup tidak pernah diuji failover. Cluster-ID duplikat antar RR merusak loop prevention. Kredensial yang tidak sinkron membuat BGP ke RR backup `Idle` tanpa ketahuan selama RR primary Up.

**Desain:**
* 2 RR, Cluster-ID unik per RR (= Router-ID RR).
* Kredensial di-manage via vault / automation, bukan manual typing.
* Failover test berkala: deactivate BGP ke RR-01 di PE pilot, verifikasi route tetap ada via RR-02, activate lagi.
* Monitoring per-neighbor BGP, bukan per-device. Alarm jika ada 1 neighbor `Idle` / `Active`.

**Verifikasi:**
```bash
show bgp summary
show bgp group <RR-CLIENTS>
show route protocol bgp | count
show route table <VRF>.inet.0 | count
```

**Lesson:** Redundancy yang tidak pernah diuji = bukan redundancy. Test failover adalah bagian dari operasi rutin.

---

### 21.5 VPLS Broadcast Storm Karena Loop di CE

**Konteks:** VPLS meniru 1 switch virtual. Loop di 1 CE (kabel patch, STP disabled) menyebar sebagai broadcast ke seluruh VPLS domain.

**Pola Failure:** 1 loop di 1 site bisa berdampak ke semua site dalam VPLS yang sama. VPLS split-horizon mencegah loop antar PE, bukan loop di CE. MAC flapping adalah early warning.

**Desain Pertahanan — 3 Layer:**
1. **Storm-control** di interface VPLS PE — batasi broadcast pps.
2. **MAC limit** di VPLS — drop MAC baru jika melebihi limit.
3. **Jangka panjang — EVPN** — control-plane MAC learning via BGP, bukan flooding. EVPN juga punya DF & aliasing untuk multihoming.

**Verifikasi:**
```bash
show interfaces <CE-IF> extensive | match pps
show vpls mac-table   # Cek MAC flapping
show firewall counter
```

**Lesson:** VPLS = 1 broadcast domain = 1 failure domain. Storm-control & MAC limit harus di-set dari awal, bukan saat storm. Untuk customer besar, pertimbangkan EVPN atau L3VPN.

---

### 21.6 Sham-Link vs Backdoor Link

**Konteks:** Customer punya backdoor link antar site + MPLS L3VPN. OSPF memilih backdoor (intra-area) padahal MPLS adalah jalur utama (lebih besar, latency lebih rendah), karena OSPF hierarki: Intra-area lebih preferred dari External.

**Pola Failure:** Tanpa sham-link, route via MPLS di-advertise sebagai External (O E2) di PE, sedangkan backdoor adalah Intra-area (O). OSPF selalu pilih backdoor.

**Desain — Sham-Link:**
* Sham-link membuat MPLS terlihat sebagai link intra-area dengan cost yang bisa di-tune.
* Cost sham-link harus lebih murah dari backdoor jika MPLS adalah primary. Backdoor tetap dipertahankan sebagai backup.
* Sham-link endpoint (`lo0.1`) harus di VRF, bukan di global.

```bash
# Di VRF:
set routing-instances <VRF> protocols ospf sham-link local <SHAM-LOCAL>
set routing-instances <VRF> protocols ospf area <CUSTOMER-AREA> sham-link-remote <SHAM-REMOTE> metric <COST>
```

**Verifikasi:**
```bash
show ospf neighbor instance <VRF>   # Sham-link neighbor Full
show ospf interface instance <VRF>
show route table <VRF>.inet.0        # Route via sham-link, bukan External
# Test failover: deactivate sham-link → traffic pindah ke backdoor
```

**Lesson:** Sham-link bukan magic — ini cost engineering. Test failover sham-link wajib.

---

### 21.7 RE Protection di Bawah Serangan

**Konteks:** Serangan ke loopback (RE) bisa membanjiri control plane dan membuat IGP/BGP flap, walau forwarding (PFE) masih jalan.

**Pola Failure:** Tanpa RE protection, semua packet ke loopback naik ke RE. Dengan RE protection, packet yang tidak diizinkan di-drop di PFE sebelum mencapai RE.

**Desain:**
* `PROTECT-RE` filter di `lo0` — hanya izinkan traffic yang dibutuhkan, dari source yang sah, dengan policer (rate-limit).
* Loopback tidak di-advertise ke eBGP — hanya di IGP internal.
* Defense in depth: RE filter (device) + upstream filtering + Flowspec (edge).

**Investigasi Saat Serangan:**
```bash
show system resource-monitor   # RE CPU vs PFE CPU
show firewall counter          # Counter DENY naik drastis?
show firewall log              # Source, destination, port, protocol
show bgp summary
show ospf neighbor
show route <LOOPBACK>          # Apakah loopback bocor ke eBGP?
```

**Lesson:**
* RE protection harus punya `source-address` + `policer`, bukan hanya `accept`.
* Monitoring counter DENY adalah early warning.
* Loopback adalah identitas internal — jangan expose ke internet.

---

## Appendix A: Naming & Golden Rules

### Naming Convention

| Objek | Format | Contoh |
| :--- | :--- | :--- |
| Hostname | `<SITE>-<ROLE>-<NN>` | `<SITE>-PE-01` |
| Interface Description | `to <NEIGHBOR> <PORT> \| <CKT> \| <BW>` | `to <NEIGHBOR> <PORT> \| <CKT> \| <BW>` |
| Policy | `UPPER_SNAKE`, prefix fungsi | `<FUNCTION>-<NAME>` |
| Firewall Filter | `UPPER-SNAKE` | `<FILTER-NAME>` |
| VRF / Routing-Instance | `VPN-<CUST>-<NN>` | `VPN-<CUST>-<NN>` |
| BGP Group | `UPPER-SNAKE` | `<GROUP-NAME>` |
| Community | `target:<AS>:<NN>` | `target:<AS>:<NN>` |
| RD | `<AS>:<NN>` | `<AS>:<NN>` |

### Golden Rules

1. Selalu `show | compare` sebelum commit.
2. Selalu `commit confirmed` untuk perubahan remote.
3. Selalu `commit comment` dengan nomor tiket.
4. Jangan pernah enable Telnet / FTP di produksi.
5. Jangan pernah pakai SNMP community plaintext.
6. Loopback adalah identitas — jangan pakai IP fisik untuk BGP / LDP / RR.
7. MTU harus konsisten end-to-end.
8. Default deny di semua policy & firewall — `reject` eksplisit di term terakhir.
9. Backup sebelum change, verifikasi sesudah change.
10. Jika ragu, `deactivate` dulu, jangan `delete`.

---

## Appendix B: Quick Reference

```bash
# Health
show chassis alarms
show system alarms
show system uptime
show system resource-monitor

# Interface & LACP
show interfaces terse
show lacp interfaces

# Routing
show ospf neighbor
show isis adjacency
show bgp summary
show route summary

# MPLS & VPN
show mpls lsp
show ldp neighbor
show l2circuit connections
show vpls connections
show route table <VRF>.inet.0

# Security
show firewall filter <FILTER>
show firewall counter
```

---

> **Disclaimer:** Dokumen ini adalah panduan implementasi. Selalu validasi di lab / staging sebelum produksi dan ikuti change management organisasi kamu. Semua nilai di sini adalah placeholder — isi dari IPAM / vault resmi kamu. Jangan publish nilai produksi di README public.

**© 2026 — Junos OS Implementation & Operations Guide v5.0 | Professional, Public-Safe**

