# Panduan Tugas Besar KDI — Kelompok 4
## Rencong Digital Health (RDH) dan Rencong CSIRT

> **Versi 2.** Disusun untuk **Kelompok 4 (X = 04)**. Semua nilai (hostname, IP, port, VPN) sudah dihitung.
> Panduan ini ditulis dari sudut pandang **anggota kelompok yang ikut mengambil keputusan**, jadi di bagian awal kamu akan menemukan *keputusan tim* beserta alasan logisnya. Kalau ada yang ingin diubah, ubah di awal supaya semua bagian tetap konsisten.

**Aturan keras dari dokumen tugas (jangan dilanggar):**

| Aturan | Artinya buat kita |
|---|---|
| Uji hanya di VM sendiri | Sniffing, nmap, login gagal hanya antar `kdi-kel04-srv` dan `kdi-kel04-cli`. Jangan menyentuh WiFi kampus/publik. |
| Data dummy saja | Nama, email, telepon pasien semuanya fiktif. Jangan pakai data anggota kelompok. |
| Hostname harus terlihat | Setiap screenshot dan demo menampilkan prompt `admin04@kdi-kel04-srv` atau `...@kdi-kel04-cli`. |
| Sensor rahasia | Password, passphrase, private key, QR TOTP, kode cadangan wajib ditutup (blok hitam). |
| A2 dan D2 kata-kata sendiri | Panduan ini hanya memberi kerangka dan fakta pegangan, bukan teks jadi. |
| Sebut tool AI | Tulis di laporan tool AI yang dipakai dan untuk apa (hanya untuk memahami konsep/mengatasi error). |

---

## Daftar Isi

0. [Keputusan tim dan cerita besar RDH](#0-keputusan-tim-dan-cerita-besar-rdh)
1. [Identitas organisasi (isi BAB 1)](#1-identitas-organisasi-rdh-isi-bab-1)
2. [Parameter final Kelompok 4](#2-parameter-final-kelompok-4)
3. [Persiapan lab](#3-persiapan-lab)
4. [Bagian A: Konsep, etika, privasi](#4-bagian-a--konsep-etika-dan-privasi)
5. [Bagian B: Enkripsi, integritas, backup](#5-bagian-b--enkripsi-integritas-dan-backup)
6. [Bagian C: Akses, SSH/MFA, HTTPS, firewall, VPN](#6-bagian-c--autentikasi-kontrol-akses-dan-jaringan)
7. [Bagian D: Manajemen insiden](#7-bagian-d--manajemen-insiden)
8. [Peta logika: ancaman → kontrol → bukti](#8-peta-logika-ancaman--kontrol--bukti)
9. [Menyusun laporan PDF](#9-menyusun-laporan-pdf)
10. [Merekam video presentasi](#10-merekam-video-presentasi)
11. [Pengumpulan dan checklist akhir](#11-pengumpulan-dan-checklist-akhir)
12. [Jadwal 7 hari](#12-jadwal-7-hari)
13. [Troubleshooting](#13-troubleshooting)
14. [Sumber rujukan](#14-sumber-rujukan)

---

## 0. Keputusan tim dan cerita besar RDH

### 0.1 Keputusan yang kita ambil (beserta alasannya)

| # | Keputusan | Alasan logis |
|---|---|---|
| 1 | **Organisasi: Rencong Digital Health (RDH)**, klinik digital di Banda Aceh dengan layanan pendaftaran online, telekonsultasi, rekam medis elektronik, dan pembayaran online. | Data kesehatan dan keuangan pribadi termasuk **data pribadi yang bersifat spesifik** menurut UU PDP, jadi tuntutan keamanannya nyata. Setiap kontrol yang kita pasang punya alasan bisnis yang jelas, bukan sekadar menuruti tugas. |
| 2 | **Tim keamanan: Rencong CSIRT (RCSIRT)** dengan fungsi pemantauan ala SOC ringan. | Tugas meminta kita berperan sebagai CSIRT. Dengan nama dan struktur tetap, laporan D2 dan video terasa seperti dokumen organisasi sungguhan. |
| 3 | **Staf RDH bekerja jarak jauh lewat VPN.** | Ini menghubungkan kasus A2 (**VPN Fortinet**, sisa 4 mod 4 = 0) dengan praktik C5 (WireGuard). Cerita menjadi satu rangkaian: *VPN adalah pintu masuk, jadi pintunya harus kuat.* |
| 4 | **Server memakai tiga zona data** (`/data/public`, `/data/internal`, `/data/confidential`) dan satu subfolder `rekam_medis` yang hanya boleh dibuka admin. | Memetakan klasifikasi A1 langsung ke struktur folder C1, jadi prediksi akses bisa dijelaskan dari *kebijakan*, bukan asal. |
| 5 | **Hanya `admin04` yang boleh login SSH.** `staf04` dan `auditor04` ada sebagai akun peran, diuji lewat `sudo -u`. | Prinsip *least privilege*. Staf dan auditor bekerja lewat aplikasi/file share, bukan shell server. |
| 6 | **Insiden D1 = brute force SSH dari akun VPN yang diduga bocor.** Klien (`10.10.4.2`) berperan sebagai penyerang yang sudah punya akses VPN. | Sesuai pelajaran Fortinet: kredensial VPN yang bocor memberi penyerang pintu masuk, lalu ia mencoba masuk ke server. Kita tunjukkan lapisan pertahanan berikutnya (MFA + fail2ban). |
| 7 | **Snapshot bernama** sebelum setiap perubahan berisiko (lihat 3.3), dan **change log** ditulis sambil bekerja. | Praktik organisasi nyata: setiap perubahan bisa dilacak dan dibatalkan. Ini juga jadi bahan "Preparation" di laporan NIST. |
| 8 | **Evidence dikumpulkan sambil bekerja** dengan penamaan baku (`B1-01-enkripsi.png`). | Menghindari kepanikan di hari terakhir dan memudahkan penomoran gambar di laporan. |

### 0.2 Cerita besar (satu alur untuk semua bagian)

Pakai alur ini di pembukaan video dan di BAB 1, supaya semua bagian terasa satu cerita:

> **RDH** baru membuka layanan klinik digital. Datanya sensitif (identitas pasien, rekam medis, pembayaran), stafnya sebagian bekerja dari rumah. Pimpinan meminta **Rencong CSIRT** memastikan data aman. Tim bekerja dalam empat fase:
>
> 1. **Fondasi (Bagian A):** *Data apa yang kita pegang, seberapa rahasia, dan apa kewajiban hukumnya?* Kita belajar dari kasus VPN Fortinet.
> 2. **Lindungi data (Bagian B):** *Kalau data dicuri atau rusak, apa yang tetap aman?* Enkripsi, tanda tangan, hash, backup.
> 3. **Kunci pintu (Bagian C):** *Siapa boleh masuk, lewat mana, dan bagaimana kita memastikan itu benar dia?* RBAC, SSH + MFA, HTTPS, firewall, VPN.
> 4. **Uji ketahanan (Bagian D):** *Kalau tetap ada yang mencoba menjebol, apakah kita sanggup mendeteksi, menahan, dan belajar?* fail2ban dan laporan NIST.

### 0.3 Pembagian peran (8 anggota; jika 7 orang, gabungkan No. 6 dan 3)

| No | Nama | NIM | Peran di RDH | Padanan dunia nyata | Tugas | Tampil di video |
|---|---|---|---|---|---|---|
| 1 | | | **Ketua / CISO** | Security program manager | Jadwal, BAB 1, A1, A3, A2 (bersama No. 8), submit | Pembukaan, penutup, konsep |
| 2 | | | **Kriptografi** | Cryptography engineer | B1, B3 | Demo enkripsi simetris + integritas |
| 3 | | | **Backup dan DR** | Backup/DR engineer | B2, B4 | Demo GPG, backup, restore |
| 4 | | | **Akses dan identitas** | IAM engineer | C1, C2 | Demo uji akses + SSH TOTP |
| 5 | | | **Web dan firewall** | Network security engineer | C3, C4 | Demo Wireshark + nmap/UFW |
| 6 | | | **VPN** | VPN engineer | C5, topologi | Demo VPN |
| 7 | | | **Analis insiden** | SOC analyst / incident handler | D1, D2 tahap 1–4 | Demo fail2ban |
| 8 | | | **GRC dan dokumentasi** | Governance, risk, compliance | A2 (bersama No. 1), D2 tahap 5–6, CSIRT, UU PDP vs GDPR, editing laporan dan video | Ringkasan NIST |

> Satu orang boleh memegang lebih dari satu peran, tetapi **setiap orang wajib berbicara di video**.

---

## 1. Identitas organisasi RDH (isi BAB 1)

BAB 1 menuntut: profil organisasi, spesifikasi host, topologi lab, dan tabel parameter. Kita tambah beberapa dokumen kecil supaya RDH terlihat seperti organisasi siber sungguhan.

### 1.1 Profil singkat

| Item | Isi (contoh, boleh disesuaikan) |
|---|---|
| Nama | Rencong Digital Health (RDH) |
| Bidang | Klinik digital: pendaftaran online, telekonsultasi, rekam medis elektronik, pembayaran |
| Lokasi (fiktif) | Banda Aceh |
| Data yang dikelola | Identitas pasien, kontak, riwayat kunjungan, rekam medis, data pembayaran, akun staf |
| Pengguna sistem | Admin sistem, staf operasional, auditor kepatuhan, pasien (lewat portal) |
| Tim keamanan | Rencong CSIRT (RCSIRT) |
| Prinsip keamanan | Defense in depth, least privilege, *secure by default*, bukti dulu baru klaim |

### 1.2 Kebijakan keamanan ringkas (10 pasal)

Satu halaman cukup. Ini yang menjadi "landasan" semua kontrol di Bab 3.

1. Semua data diklasifikasikan (Public, Internal, Confidential, Highly Confidential).
2. Akses diberikan menurut peran dan kebutuhan (RBAC, *need-to-know*).
3. Login administrator wajib memakai kunci SSH **dan** TOTP.
4. Login root dan login password lewat SSH dinonaktifkan.
5. Lalu lintas yang membawa kredensial wajib terenkripsi (HTTPS/VPN).
6. Akses administrasi hanya melalui VPN.
7. Firewall *default deny*, hanya membuka yang perlu.
8. Backup terenkripsi, terjadwal, disimpan terpisah, dan diuji pulihnya.
9. Insiden ditangani mengikuti tahapan NIST, dan kegagalan pelindungan data diberitahukan sesuai UU PDP.
10. Hanya data dummy di lingkungan lab, dan setiap perubahan dicatat di *change log*.

### 1.3 Register aset

| ID | Aset | Lokasi | Pemilik | Klasifikasi |
|---|---|---|---|---|
| A-01 | Server utama `kdi-kel04-srv` | VM server | admin04 | Highly Confidential |
| A-02 | Laptop staf remote `kdi-kel04-cli` | VM klien | staf/analis | Internal |
| A-03 | Berkas pelanggan `data_pelanggan_kel04.csv` | `/data/confidential` | admin04 | Confidential |
| A-04 | Rekam medis | `/data/confidential/rekam_medis` | admin04 | Highly Confidential |
| A-05 | Backup terenkripsi | `/var/backups/rencong` + klien | admin04 | Highly Confidential |
| A-06 | Portal internal (Nginx) | server, port 443 | admin04 | Confidential |
| A-07 | Kunci SSH, GPG, WireGuard | home admin04, `/etc/wireguard` | admin04 | Highly Confidential |

### 1.4 Register risiko (pegangan "kenapa kontrol ini ada")

| ID | Risiko | Dampak | Kontrol RDH | Dibuktikan di |
|---|---|---|---|---|
| R1 | Brute force/penyalahgunaan login SSH | Pengambilalihan server (C, I, A) | Kunci SSH + TOTP, port non-standar, `AllowUsers`, fail2ban | C2, D1 |
| R2 | Penyadapan kredensial di jaringan | Pencurian akun (C) | HTTPS, VPN | C3, C5 |
| R3 | Data hilang/dienkripsi pelaku (ransomware) | Layanan lumpuh (A) | Backup terenkripsi, salinan terpisah, uji restore | B4 |
| R4 | Akses berlebihan oleh pengguna internal | Kebocoran/perubahan data (C, I) | RBAC + ACL, klasifikasi data | C1 |
| R5 | Kredensial VPN bocor / gateway VPN tak ter-patch | Pintu masuk penyerang (C, I, A) | MFA, VPN minim permukaan serangan, firewall, pemantauan | A2, C4, C5, D1 |
| R6 | Data cadangan atau data kiriman diubah/dicuri | Pemalsuan data (I, C) | Enkripsi, hash, tanda tangan digital | B1–B3 |

### 1.5 Topologi lab (buat gambar di draw.io/PowerPoint, atau pakai ini sebagai dasar)

```
                  Laptop host (VirtualBox)
        ┌────────────────────────────────────────────────┐
        │  Host-only 192.168.56.0/24                     │
        │                                                │
        │  kdi-kel04-cli ───────────────── kdi-kel04-srv │
        │  192.168.56.154                  192.168.56.104│
        │  (laptop staf / "penyerang")     (server RDH)  │
        │       │      WireGuard 10.10.4.0/24     │      │
        │       └── 10.10.4.2 ══ UDP 51820 ══ 10.10.4.1  │
        │                                                │
        │  Adapter NAT (internet) di kedua VM            │
        └────────────────────────────────────────────────┘
        Wireshark: berjalan di kdi-kel04-cli (interface Host-only)
```

### 1.6 Spesifikasi host

Isi dengan laptop/PC yang dipakai: tipe, prosesor, RAM (≥ 8 GB), versi VirtualBox, sistem operasi host. Lampirkan juga ringkasan VM (server: 1 vCPU, 2 GB, 20 GB; klien: 1–2 vCPU, 2 GB).

---

## 2. Parameter final Kelompok 4

| Parameter | Aturan | **Nilai Kelompok 4** |
|---|---|---|
| Hostname server | `kdi-kelX-srv` | **`kdi-kel04-srv`** |
| Hostname klien | `kdi-kelX-cli` | **`kdi-kel04-cli`** |
| IP server (Host-only) | `192.168.56.(100+X)/24` | **`192.168.56.104/24`** |
| IP klien (Host-only) | `192.168.56.(150+X)/24` | **`192.168.56.154/24`** |
| Port SSH baru | `2200+X` | **`2204`** |
| Subnet VPN WireGuard | `10.10.X.0/24` (X tanpa nol depan) | **`10.10.4.0/24`** (server `10.10.4.1`, klien `10.10.4.2`) |
| User Linux | `adminX`, `stafX`, `auditorX` | **`admin04`, `staf04`, `auditor04`** |
| File data dummy | `data_pelanggan_kelX.csv` (min. 20 baris) | **`data_pelanggan_kel04.csv`** |
| Studi kasus | sisa X mod 4 | 4 mod 4 = **0 → VPN Fortinet** |
| Port WireGuard (keputusan tim) | bebas | `51820/udp` |

> Hati-hati: subnet VPN ditulis `10.10.4.0/24` (**bukan** `10.10.04.0/24`). Sedangkan hostname memakai dua digit (`kel04`).

---

## 3. Persiapan lab

### 3.1 Siapkan VirtualBox dan dua VM

| | VM server | VM klien |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS | Ubuntu Desktop 24.04 LTS |
| Spesifikasi | 1 vCPU, RAM 2 GB, disk 20 GB | 1–2 vCPU, RAM 2 GB |
| Adapter 1 | NAT | NAT |
| Adapter 2 | Host-only (192.168.56.0/24) | Host-only |
| User saat instalasi | **`admin04`** | bebas (misal `klien04`) |
| Pilihan saat instal | centang **Install OpenSSH server** | |

Langkah:

1. VirtualBox → **File → Tools → Network Manager → Host-only Networks**. Pastikan ada jaringan `192.168.56.1/24`. Disarankan matikan DHCP server di jaringan itu supaya tidak bentrok dengan IP statis kita.
2. Tiap VM: **Settings → Network**. Adapter 1 = NAT, Adapter 2 = Host-only Adapter.
3. Instal Ubuntu pada keduanya.

### 3.2 Hostname, zona waktu, dan IP

**Di server:**

```bash
sudo hostnamectl set-hostname kdi-kel04-srv
sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1 kdi-kel04-srv/' /etc/hosts
sudo timedatectl set-timezone Asia/Jakarta     # supaya log langsung berwaktu WIB
ip a                                           # cek nama interface host-only (biasanya enp0s8)
```

IP statis di server:

```bash
sudo nano /etc/netplan/60-hostonly.yaml
```
```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses: [192.168.56.104/24]
```
```bash
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
ip a        # pastikan 192.168.56.104 muncul di enp0s8
```

**Di klien (Ubuntu Desktop memakai NetworkManager):**

```bash
sudo hostnamectl set-hostname kdi-kel04-cli
sudo timedatectl set-timezone Asia/Jakarta
sudo nmcli con add type ethernet ifname enp0s8 con-name hostonly ip4 192.168.56.154/24
sudo nmcli con up hostonly
ping -c 3 192.168.56.104        # harus berbalas
```

Buka terminal baru supaya prompt menampilkan `...@kdi-kel04-srv` / `...@kdi-kel04-cli`.

> **Logika zona waktu:** Timeline insiden D2 harus cocok antara log server, Wireshark, dan laporan. Menyamakan zona waktu sekarang menghemat banyak kebingungan nanti.

### 3.3 Snapshot bernama dan change log

Ambil snapshot kedua VM dengan nama berikut pada saat yang tepat:

| Snapshot | Kapan |
|---|---|
| `s0-bersih` | Setelah instal OS dan jaringan jalan |
| `s1-sebelum-ssh` | Sebelum C2 (ubah SSH/MFA) |
| `s2-sebelum-ufw` | Sebelum C4 (aktifkan firewall) |
| `s3-sebelum-vpn` | Sebelum C5 |
| `s4-sebelum-insiden` | Sebelum D1 |

Buat **change log** (tabel di lampiran laporan), isi sambil kerja:

| Waktu | Perubahan | Oleh | Alasan | Snapshot |
|---|---|---|---|---|
| 2026-11-xx 19:30 | SSH port 22 → 2204, matikan login root | Anggota 4 | Kebijakan #4 | s1-sebelum-ssh |

### 3.4 Instal alat bantu

```bash
# SERVER
sudo apt update
sudo apt install -y acl nginx apache2-utils ufw wireguard fail2ban libpam-google-authenticator

# KLIEN
sudo apt update
sudo apt install -y nmap wireguard wireshark
# jawab "Yes" bila ditanya apakah non-superuser boleh capture paket
sudo usermod -aG wireshark $USER      # lalu logout dan login lagi
```

> Jangan aktifkan UFW dulu. Firewall baru dipasang di C4.

### 3.5 Buat group, user, dan struktur `/data`

Kita buat sekarang karena B2 butuh user `auditor04`, dan B4 butuh folder `/data` untuk dibackup. **Hak akses final** baru dipasang di C1.

```bash
# Group peran
sudo groupadd admin
sudo groupadd staf
sudo groupadd auditor

# User peran (tanpa password, karena hanya admin04 yang login SSH)
sudo adduser --disabled-password --gecos "" staf04
sudo adduser --disabled-password --gecos "" auditor04

# Masukkan ke group
sudo usermod -aG admin admin04
sudo usermod -aG staf staf04
sudo usermod -aG auditor auditor04
id admin04 staf04 auditor04          # bukti keanggotaan

# Struktur data (sementara dimiliki admin04 agar bisa diisi)
sudo mkdir -p /data/public /data/internal /data/confidential/rekam_medis
sudo chown -R admin04:admin /data
```

Logout lalu login lagi sebagai `admin04` supaya keanggotaan group `admin` aktif.

### 3.6 Isi data dummy (≥ 20 baris, semua fiktif)

```bash
cd /data/confidential
echo "id_pasien,nama,email,telepon,alamat,layanan,tanggal_kunjungan,status_bayar" > data_pelanggan_kel04.csv
for i in $(seq 1 25); do
  printf "P%03d,Pasien Fiktif %02d,pasien%02d@example.test,0812000%05d,Jl. Contoh No.%d Banda Aceh,Telekonsultasi,2026-10-%02d,%s\n" \
    $i $i $i $i $i $(( (i % 28) + 1 )) $( [ $((i % 3)) -eq 0 ] && echo Belum || echo Lunas ) \
    >> data_pelanggan_kel04.csv
done
wc -l data_pelanggan_kel04.csv       # harus 26 (1 header + 25 baris)
head -5 data_pelanggan_kel04.csv
```

Isi berkas pendukung agar tiap zona punya contoh:

```bash
echo "Brosur layanan RDH - jam praktik 08.00-20.00 WIB (dummy)"      > /data/public/brosur_layanan.txt
echo "SOP-01: Prosedur reset akun staf (dummy)"                      > /data/internal/sop_reset_akun.txt
echo "2026-10-01 10:00 login admin04 sukses (dummy)"                 > /data/confidential/log_audit_dummy.txt
echo "P001 | Diagnosa: flu ringan | Catatan dokter: istirahat (DUMMY)" > /data/confidential/rekam_medis/rm_dummy.txt
find /data -type f
```

> Tips: variasikan nama dan layanan di CSV supaya terlihat nyata, tetapi tetap fiktif (`example.test`, nomor `0812000xxxxx`).

Catat **hash dasar** sebagai bukti integritas awal (dipakai lagi di B4 untuk membuktikan restore sempurna):

```bash
sha256sum /data/confidential/data_pelanggan_kel04.csv | tee ~/hash_dasar_kel04.txt
```

---

## 4. Bagian A — Konsep, Etika, dan Privasi

Bagian ini masuk **BAB 2** laporan. Tidak perlu VM. Kuncinya adalah **alasan**: setiap pilihan harus bisa dijelaskan.

### A1. Klasifikasi data RDH

**Logika klasifikasi:**
- *Public*: boleh dilihat siapa saja, tidak merugikan kalau terbaca.
- *Internal*: untuk staf, merugikan kalau bocor ke luar tapi dampaknya terbatas.
- *Confidential*: data pribadi umum atau data bisnis sensitif.
- *Highly Confidential*: data pribadi **spesifik** (kesehatan, keuangan pribadi menurut UU PDP) dan rahasia kriptografis. Bocor = dampak berat bagi pasien.

Tabel (min. 8; di bawah 12 contoh yang sudah konsisten dengan C1):

| No | Jenis data | Kategori | CIA paling kritis | Peran yang boleh mengakses | Folder |
|---|---|---|---|---|---|
| 1 | Brosur layanan dan jam praktik | Public | Availability (harus selalu bisa diakses) | Semua | `/data/public` |
| 2 | Jadwal praktik dokter | Public | Integrity (jadwal salah merugikan pasien) | Semua (tulis: admin) | `/data/public` |
| 3 | SOP dan pedoman internal | Internal | Integrity | admin (tulis), staf (tulis), auditor (baca) | `/data/internal` |
| 4 | Inventaris obat dan stok | Internal | Integrity + Availability | admin, staf; auditor baca | `/data/internal` |
| 5 | Identitas dan kontak pasien (nama, email, telepon, alamat) | Confidential | Confidentiality | admin (tulis); auditor (baca) | `/data/confidential` |
| 6 | Log audit dan log keamanan | Confidential | Integrity (log tak boleh diubah) | admin (tulis); auditor (baca) | `/data/confidential` |
| 7 | Kode sumber portal (kekayaan intelektual) | Confidential | Integrity | admin | `/data/confidential` |
| 8 | Laporan keuangan organisasi | Confidential | Integrity | admin; auditor baca | `/data/confidential` |
| 9 | Rekam medis (diagnosa, hasil lab) | Highly Confidential | Confidentiality + Integrity | admin saja | `/data/confidential/rekam_medis` |
| 10 | Data keuangan pribadi pasien (rekening/kartu) | Highly Confidential | Confidentiality | admin saja | `/data/confidential/rekam_medis` |
| 11 | Hash kata sandi, kunci SSH/GPG/VPN | Highly Confidential | Confidentiality | admin sistem | home admin / `/etc/wireguard` |
| 12 | Backup terenkripsi | Highly Confidential | Availability + Confidentiality | admin | `/var/backups/rencong` |

> Penjelasan wajib: *kenapa* tiap data masuk kategori itu dan *kenapa* satu aspek CIA lebih kritis. Contoh logika: log audit paling kritis di **integrity**, karena log yang bisa diubah penyerang tidak bisa dipercaya sebagai bukti.
>
> Catat juga: staf layanan pelanggan mengakses data pelanggan **lewat aplikasi**, bukan lewat file share server. Inilah alasan `staf` tidak punya akses langsung ke `/data/confidential`.

### A2. Studi kasus: VPN Fortinet (karena 4 mod 4 = 0)

**Keputusan tim:** kita fokus pada **kebocoran kredensial VPN FortiGate (Nov 2020)** yang berakar pada kerentanan **CVE-2018-13379**, dan menyebut kerentanan VPN Fortinet lain sebagai konteks pembanding.

**Fakta pegangan** (sudah dicek dari pemberitaan; tetap baca sumbernya lalu tulis ulang dengan kata-katamu):

- CVE-2018-13379 adalah celah *path traversal* pada portal web SSL-VPN FortiOS yang memungkinkan penyerang **tanpa autentikasi** mengunduh berkas sistem lewat permintaan HTTP yang dirancang khusus.
- Fortinet merilis tambalan pada Mei 2019 dan mengingatkan pelanggan beberapa kali setelahnya.
- Pada November 2020, seorang peretas menyebarkan daftar sekitar 50.000 alamat IP perangkat FortiGate yang masih rentan, lalu basis data besar berisi berkas sesi `sslvpn_websession` dengan username dan password (teks biasa) dari perangkat-perangkat itu. Disebutkan di antara korbannya institusi perbankan, telekomunikasi, dan pemerintah.
- Meski perangkat kemudian ditambal, **kredensial yang sudah bocor tetap berisiko** dipakai untuk *credential stuffing*. Karena itu pengguna diminta mengganti password.
- Fortinet sebelumnya juga memperingatkan bahwa kelompok ancaman tingkat negara memanfaatkan celah ini.

**Kerangka jawaban (isi dengan kalimatmu sendiri):**

1. **Kronologi singkat.** Celah ditemukan, tambalan dirilis, perangkat banyak yang tidak ditambal, daftar IP dan kredensial bocor, dampaknya.
2. **Analisis pelanggaran** (tabel di bawah). Jelaskan *siapa* pengendali data di kasus ini (organisasi pemilik perangkat), *siapa* subjek datanya (pengguna VPN).
3. **Regulasi:** UU PDP (UU No. 27 Tahun 2022) dan GDPR; sebutkan juga satu kalimat perbandingan dengan CCPA.
4. **Ancaman teknis:** eksploitasi kerentanan, *unauthenticated file read*, pencurian dan penyalahgunaan kredensial, *credential stuffing*, kegagalan *patch management*, penyimpanan kredensial yang terbaca, dan potensi pergerakan lateral/ransomware setelah masuk lewat VPN.
5. **Tiga rekomendasi pencegahan**, lalu tunjukkan bahwa RDH menerapkannya di lab (ini nilai tambah).

**Tabel analisis (kerangka):**

| Aspek | Pertanyaan yang kamu jawab | Hubungan regulasi (verifikasi pasalnya) |
|---|---|---|
| Persetujuan | Apakah pengguna VPN menyetujui datanya terpapar? Apa dasar pemrosesan kredensial mereka? | UU PDP Pasal 20 (dasar pemrosesan); GDPR Art. 6 |
| Transparansi | Apakah organisasi pemilik perangkat memberi tahu pengguna dan regulator dengan jelas dan tepat waktu? | UU PDP Pasal 21 dan 46; GDPR Art. 12–14, 33–34 |
| Akuntabilitas | Tambalan sudah ada sejak 2019, mengapa belum dipasang? Siapa yang bertanggung jawab? | UU PDP Pasal 47 (pembuktian pemenuhan kewajiban) dan pasal keamanan (Pasal 35, 39); GDPR Art. 5(2), 32 |
| Keadilan | Pengguna biasa tidak bersalah tetapi menanggung risiko akun dibajak. Adil? | Prinsip pemrosesan yang sah dan adil (GDPR Art. 5(1)(a)) |

**Tiga rekomendasi yang masuk akal (jadikan contoh arah, jangan disalin):**

| Rekomendasi | Mencegah apa | Diterapkan RDH di |
|---|---|---|
| **Patch management** terjadwal untuk gateway VPN dan server (pemindaian kerentanan, SLA tambalan) | Eksploitasi celah yang sudah diketahui | Kebijakan #10; tambahkan `apt update/upgrade` ke SOP (opsional) |
| **MFA** untuk setiap akses jarak jauh dan **reset kredensial** setelah dugaan bocor | Penyalahgunaan kredensial yang dicuri | C2 (key + TOTP), D1 |
| **Perkecil permukaan serangan dan pantau**: akses admin hanya lewat VPN, firewall default deny, log dipantau, segmentasi | Penyerang yang berhasil masuk lewat VPN bergerak bebas | C4, C5, D1 |

**Perbandingan singkat tiga regulasi** (untuk menjawab capaian pembelajaran nomor 2):

| Aspek | UU PDP (Indonesia) | GDPR (UE) | CCPA (California) |
|---|---|---|---|
| Dasar pemrosesan | Persetujuan eksplisit dan dasar lain | Persetujuan **opt-in** dan dasar sah lain | Pendekatan **opt-out** untuk penjualan/berbagi data |
| Hak subjek | Akses, koreksi, penghapusan, tarik persetujuan, dll. | Akses, koreksi, hapus, portabilitas, keberatan, dll. | Hak tahu, hapus, koreksi, opt-out |
| Notifikasi kebocoran | Tertulis ≤ 3×24 jam ke subjek data dan lembaga (Pasal 46) | ≤ 72 jam ke otoritas; ke subjek jika risiko tinggi | Lewat undang-undang notifikasi kebocoran negara bagian |

> Karena dosen menilai "tajam dan dikaitkan regulasi", usahakan setiap kalimat analisis mengikuti pola **klaim → bukti dari kasus → pasal/prinsip → rekomendasi**.

### A3. Kebijakan privasi RDH (maksimal 1 halaman)

Draf siap-edit (sesuaikan kalimatnya; jangan lebih dari satu halaman A4):

```
KEBIJAKAN PRIVASI — RENCONG DIGITAL HEALTH (RDH)

1. Data yang kami kumpulkan
   Nama, kontak (email dan telepon), alamat, riwayat kunjungan, keluhan/rekam medis,
   dan status pembayaran. Data dikumpulkan seperlunya untuk layanan klinik.

2. Tujuan
   Pendaftaran dan penjadwalan, pelayanan medis, penagihan, serta pemenuhan
   kewajiban hukum dan audit keamanan.

3. Persetujuan (opt-in)
   Kami hanya memproses data setelah Anda MENCENTANG sendiri kotak persetujuan
   (tidak pernah tercentang otomatis). Persetujuan untuk tujuan di luar layanan
   diminta terpisah. Anda dapat menarik persetujuan kapan saja.

4. Hak Anda
   Mengakses data Anda, memperbaiki data yang keliru, dan meminta penghapusan
   data (kecuali yang wajib kami simpan menurut hukum). Permintaan ke DPO;
   kami menanggapi dalam [..] hari kerja.

5. Masa retensi
   Data identitas dan kunjungan: [..] tahun sejak kunjungan terakhir.
   Rekam medis: sesuai ketentuan hukum yang berlaku. Log keamanan: [..] bulan.
   Setelah itu data dihapus/dimusnahkan secara aman.

6. Keamanan
   Enkripsi, kontrol akses berbasis peran, autentikasi berlapis, firewall/VPN,
   dan backup terenkripsi.

7. Jika terjadi kebocoran
   Kami memberi tahu Anda dan lembaga berwenang secara tertulis maksimal
   3x24 jam sejak kegagalan pelindungan data diketahui, memuat data yang
   terungkap, kapan/bagaimana terjadi, dan langkah penanganan.

8. Kontak penanggung jawab data (DPO)
   [Nama fiktif], dpo@rdh.example.test, [jam layanan].
```

> Isi tanda `[..]` dengan keputusan kelompok. Pastikan angka retensi **masuk akal dan konsisten** dengan klasifikasi A1 dan kebijakan keamanan.

---

## 5. Bagian B — Enkripsi, Integritas, dan Backup

> **Cerita fase ini:** *"Kalau data dicuri atau rusak, apa yang tetap aman?"*
> Dikerjakan di **VM server** sebagai `admin04`. Pastikan prompt memperlihatkan `admin04@kdi-kel04-srv`.

Siapkan folder kerja agar tidak mengacak data asli:

```bash
mkdir -p ~/lab_kripto
cp /data/confidential/data_pelanggan_kel04.csv ~/lab_kripto/
cd ~/lab_kripto
ls -l
```

### B1. Enkripsi simetris AES-256

**Logika:** satu kunci (passphrase) mengunci *dan* membuka. Cepat dan cocok untuk berkas besar.

```bash
# 1) Enkripsi (diminta passphrase dua kali; SENSOR di laporan)
openssl enc -aes-256-cbc -pbkdf2 -salt -in data_pelanggan_kel04.csv -out data_pelanggan_kel04.csv.enc

# 2) Bukti: isi berkas .enc tidak terbaca
file data_pelanggan_kel04.csv.enc
head -c 120 data_pelanggan_kel04.csv.enc | xxd | head -5
# bandingkan dengan aslinya:
head -3 data_pelanggan_kel04.csv

# 3) Dekripsi
openssl enc -d -aes-256-cbc -pbkdf2 -in data_pelanggan_kel04.csv.enc -out hasil_dekripsi.csv

# 4) Buktikan hasilnya identik dengan aslinya
diff data_pelanggan_kel04.csv hasil_dekripsi.csv && echo "IDENTIK: dekripsi sukses"
```

**Apa yang harus terlihat di screenshot:** (1) perintah enkripsi, (2) isi `.enc` berupa karakter acak (dibanding CSV asli yang terbaca), (3) dekripsi + pesan `IDENTIK`.

**Penjelasan "tantangan distribusi kunci"** (tulis dengan bahasamu, ide intinya):

- Pengirim dan penerima harus memegang **kunci yang sama**. Bagaimana mengirim kunci itu tanpa disadap? Kalau dikirim lewat jalur yang sama dengan data, penyadap mendapat keduanya.
- **Skalabilitas:** n pihak yang saling berkomunikasi butuh n(n−1)/2 kunci unik.
- **Rotasi dan pencabutan:** kalau kunci bocor, semua data yang memakai kunci itu terbuka dan semua pihak harus berganti kunci.
- Ini alasan kita butuh enkripsi asimetris (B2), dan di dunia nyata keduanya dipakai bersama (hibrida).

### B2. Enkripsi asimetris dan tanda tangan digital (GPG)

**Skenario RDH:** `admin04` mengirim berkas pelanggan ke `auditor04` untuk audit tahunan, dengan persetujuan DPO. Berkas harus **rahasia** di jalan (dienkripsi untuk auditor) dan **asli** (ditandatangani admin).

**Pengaturan jalur pertukaran** (folder bersama yang bisa ditulis siapa saja, seperti "kotak surat"):

```bash
sudo mkdir -p /srv/pertukaran
sudo chmod 1777 /srv/pertukaran      # sticky bit: tiap user hanya bisa hapus berkasnya sendiri
```

**1) Buat key pair masing-masing.**

```bash
# sebagai admin04 (user yang sedang login)
export GPG_TTY=$(tty)
gpg --full-generate-key
#  Pilih: (1) RSA and RSA → 3072 atau 4096 bit → masa berlaku 1y
#  Nama: admin04 RDH, Email: admin04@rdh.example.test, lalu passphrase (SENSOR)
gpg --list-keys
```
```bash
# pindah menjadi auditor04
sudo -iu auditor04
export GPG_TTY=$(tty)
gpg --full-generate-key
#  Nama: auditor04 RDH, Email: auditor04@rdh.example.test
gpg --list-keys
exit            # kembali ke admin04
```

**2) Tukar public key** (public key aman dibagikan, private key tidak pernah keluar):

```bash
# admin04: ekspor public key sendiri
gpg --armor --export admin04@rdh.example.test > /srv/pertukaran/admin04.pub

# auditor04: ekspor public key sendiri
sudo -iu auditor04 bash -c 'gpg --armor --export auditor04@rdh.example.test > /srv/pertukaran/auditor04.pub'

# admin04: impor public key auditor
gpg --import /srv/pertukaran/auditor04.pub

# auditor04: impor public key admin
sudo -iu auditor04 gpg --import /srv/pertukaran/admin04.pub
```

**3) Verifikasi fingerprint dan percayai key.** Ini langkah anti-pemalsuan: kalau penyerang menukar public key di jalan (MITM), fingerprint-nya akan berbeda.

```bash
gpg --fingerprint auditor04@rdh.example.test            # catat fingerprint
sudo -iu auditor04 gpg --fingerprint auditor04@rdh.example.test   # bandingkan: harus sama
gpg --edit-key auditor04@rdh.example.test
#   gpg> trust    → pilih 5 (ultimate) → y → quit

sudo -iu auditor04 gpg --edit-key admin04@rdh.example.test
#   gpg> trust    → 5 → y → quit
```

**4) Pengirim mengenkripsi sekaligus menandatangani:**

```bash
gpg --encrypt --sign -r auditor04@rdh.example.test \
    -o /srv/pertukaran/data_pelanggan_kel04.csv.gpg \
    ~/lab_kripto/data_pelanggan_kel04.csv
ls -l /srv/pertukaran
```

**5) Penerima mendekripsi dan memverifikasi:**

```bash
sudo -iu auditor04
export GPG_TTY=$(tty)
gpg -o hasil_gpg.csv --decrypt /srv/pertukaran/data_pelanggan_kel04.csv.gpg
# Harus muncul: gpg: Good signature from "admin04 RDH <admin04@rdh.example.test>"
head -3 hasil_gpg.csv
exit
```

**6) Bukti kerahasiaan: pihak lain tidak bisa membuka** (nilai tambah dan mudah didemokan):

```bash
sudo -iu staf04 gpg --decrypt /srv/pertukaran/data_pelanggan_kel04.csv.gpg
# Hasil: gpg: decrypt_message failed: No secret key  → staf04 tidak punya kunci privat penerima
```

**Perbandingan B1 vs B2** (masukkan ke laporan):

| Aspek | Simetris (B1) | Asimetris + tanda tangan (B2) |
|---|---|---|
| Kunci | Satu kunci rahasia dipakai bersama | Sepasang kunci: public (dibagi) + private (rahasia) |
| Distribusi kunci | Sulit, harus lewat kanal aman | Mudah, public key boleh disebar (tetap verifikasi fingerprint) |
| Kecepatan | Cepat | Lebih lambat |
| Yang dijamin | Confidentiality | Confidentiality (enkripsi) + Integrity dan keaslian pengirim (tanda tangan) + non-repudiation |
| Cocok untuk | Berkas besar, penyimpanan | Pertukaran antar pihak, verifikasi asal |

> Catatan akurat: GPG sebenarnya memakai skema **hibrida**: data dienkripsi dengan kunci sesi simetris, lalu kunci sesi itu dienkripsi dengan public key penerima.

### B3. Integritas

**Logika:** hash adalah "sidik jari" data. Berubah satu karakter, sidik jarinya berubah total (*avalanche effect*).

```bash
cd ~/lab_kripto

# 1) Hash berkas asli
sha256sum data_pelanggan_kel04.csv | tee hash_asli.txt

# 2) Ubah SATU karakter pada salinan, hitung ulang
cp data_pelanggan_kel04.csv uji_integritas.csv
sed -i '2s/./X/' uji_integritas.csv          # ganti karakter pertama baris 2
diff <(sed -n '2p' data_pelanggan_kel04.csv) <(sed -n '2p' uji_integritas.csv)
sha256sum uji_integritas.csv                 # hash sama sekali berbeda
```

**Verifikasi tanda tangan GPG gagal setelah file diubah:**

```bash
cp data_pelanggan_kel04.csv uji_ttd.csv
gpg --armor --detach-sign uji_ttd.csv        # menghasilkan uji_ttd.csv.asc (tanda tangan terpisah)
gpg --verify uji_ttd.csv.asc uji_ttd.csv     # → Good signature

sed -i '2s/./X/' uji_ttd.csv                 # ubah 1 karakter
gpg --verify uji_ttd.csv.asc uji_ttd.csv     # → BAD signature
```

**Kaitkan dengan integrity:** hash membuktikan *berkas tidak berubah*; tanda tangan digital mengikat hash itu ke *identitas penanda tangan*, jadi pihak lain tidak bisa mengubah data lalu mengaku data itu asli. Ini menjawab risiko R6 dan aspek **Integrity** pada CIA Triad.

### B4. Backup terenkripsi dan pemulihan

**Logika (dan alasan kasus PDN/ransomware):** backup yang tidak teruji, tidak terpisah, atau tidak terenkripsi sama saja tidak ada. Kita buktikan empat hal: *terjadwal*, *terenkripsi*, *terpisah*, *bisa dipulihkan*.

**1) Simpan passphrase backup di berkas terlindungi** (hanya untuk lab dummy):

```bash
sudo bash -c 'echo "PassBackupDummy-Kel04" > /root/.backup_pass_kel04'
sudo chmod 600 /root/.backup_pass_kel04
sudo mkdir -p /var/backups/rencong
sudo chown root:admin /var/backups/rencong
sudo chmod 750 /var/backups/rencong
```

**2) Buat skrip `backup.sh`:**

```bash
sudo nano /usr/local/bin/backup.sh
```
```bash
#!/bin/bash
# backup.sh - Rencong Digital Health (Kelompok 4)
# Arsip folder /data -> enkripsi AES-256 -> nama berstempel waktu
set -euo pipefail

SRC_DIR="/data"
DEST="/var/backups/rencong"
PASSFILE="/root/.backup_pass_kel04"
TS=$(date +%Y%m%d_%H%M%S)
ARSIP="/tmp/backup_kel04_${TS}.tar.gz"
HASIL="${DEST}/backup_kel04_${TS}.tar.gz.enc"

# 1. Arsipkan (sertakan ACL dan atribut agar izin ikut pulih)
tar --acls --xattrs -czf "$ARSIP" -C / "${SRC_DIR#/}"

# 2. Enkripsi
openssl enc -aes-256-cbc -pbkdf2 -salt -pass file:"$PASSFILE" -in "$ARSIP" -out "$HASIL"

# 3. Bersihkan arsip tak terenkripsi, atur izin
rm -f "$ARSIP"
chown root:admin "$HASIL"
chmod 640 "$HASIL"

# 4. Retensi: hapus backup lebih dari 7 hari
find "$DEST" -name 'backup_kel04_*.enc' -mtime +7 -delete

# 5. Catat di log
echo "[$(date '+%F %T')] Backup selesai: $HASIL" >> /var/log/backup_kel04.log
```
```bash
sudo chmod +x /usr/local/bin/backup.sh
sudo /usr/local/bin/backup.sh                   # uji manual
ls -l /var/backups/rencong
sudo tail /var/log/backup_kel04.log
```

**3) Jadwalkan dengan cron:**

```bash
sudo crontab -e
```
Tambahkan **satu** baris (untuk demo, tiap 2 menit):
```
*/2 * * * * /usr/local/bin/backup.sh
```
> Jelaskan di laporan: untuk produksi RDH memakai jadwal harian, contoh `0 2 * * * /usr/local/bin/backup.sh` (jam 02.00, saat layanan sepi). Tunggu ≥ 4 menit, tunjukkan bahwa file baru muncul dengan stempel waktu berbeda (`ls -l /var/backups/rencong`). **Setelah screenshot, beri tanda `#` pada baris cron** agar disk tidak penuh.

**4) Salin ke VM klien sebagai salinan terpisah** (dijalankan dari klien, SSH masih port 22 pada tahap ini):

```bash
# di kdi-kel04-cli
mkdir -p ~/backup_rdh
scp admin04@192.168.56.104:/var/backups/rencong/backup_kel04_*.enc ~/backup_rdh/
ls -l ~/backup_rdh       # bukti salinan terpisah (prompt harus kdi-kel04-cli)
```

**5) Simulasi bencana dan pemulihan:**

```bash
# di server: bukti data ada, lalu "bencana"
ls -R /data | head
sudo rm -rf /data                         # data terhapus (simulasi ransomware/salah hapus)
ls /data                                  # "No such file or directory"
```
```bash
# di klien: kirim salinan backup terbaru kembali ke server
FILE=$(ls -t ~/backup_rdh/*.enc | head -1)
scp "$FILE" admin04@192.168.56.104:/tmp/restore.tar.gz.enc
```
```bash
# di server: dekripsi dan pulihkan
sudo openssl enc -d -aes-256-cbc -pbkdf2 -pass file:/root/.backup_pass_kel04 \
     -in /tmp/restore.tar.gz.enc -out /tmp/restore.tar.gz
sudo tar --acls --xattrs -xzf /tmp/restore.tar.gz -C /
sudo rm -f /tmp/restore.tar.gz /tmp/restore.tar.gz.enc

# bukti pulih sempurna: hash sama dengan hash dasar di 3.6
sha256sum /data/confidential/data_pelanggan_kel04.csv
cat ~/hash_dasar_kel04.txt
```
Kedua hash harus sama, itulah bukti integritas hasil restore.

**Kaitkan dengan DRP dan kasus PDN** (kerangka argumen):

| Konsep | Penjelasan | Contoh di RDH |
|---|---|---|
| **RPO** (data maksimal yang boleh hilang) | Menentukan frekuensi backup | Harian di produksi; 2 menit hanya untuk demo |
| **RTO** (waktu maksimal pemulihan) | Menentukan kesiapan prosedur | Restore di lab selesai dalam beberapa menit |
| **Aturan 3-2-1** | 3 salinan, 2 media, 1 di luar lokasi | Data asli + backup di server + salinan di klien (terpisah) |
| **Enkripsi backup** | Melindungi confidentiality jika backup dicuri | AES-256 |
| **Uji restore** | Backup yang tak pernah diuji adalah asumsi | Kita membuktikannya dengan hash |
| **Pelajaran PDN** | Serangan ransomware melumpuhkan layanan; dampak parah ketika backup tidak memadai/terpisah | Latar alasan salinan di klien dan uji restore |

> Catatan jujur untuk bagian lessons learned: backup otomatis ke klien lewat SSH + TOTP tidak bisa berjalan tanpa interaksi. Perbaikan di dunia nyata adalah akun backup khusus dengan kunci terbatas atau sistem backup *pull* yang terpisah.
>
> Setelah C1 selesai, jalankan `sudo /usr/local/bin/backup.sh` sekali lagi supaya backup terbaru memuat ACL final.

---

## 6. Bagian C — Autentikasi, Kontrol Akses, dan Pengamanan Jaringan

> **Cerita fase ini:** *"Siapa boleh masuk, lewat mana, dan bagaimana memastikan itu benar dia?"*
> **Urutan aman:** C1 → C2 → C3 → C4 → C5. Ambil snapshot sesuai tabel 3.3 sebelum C2, C4, dan C5.

### C1. Kontrol akses berbasis peran (RBAC)

**Logika desain:** hak akses diturunkan dari klasifikasi A1.

| Direktori | Kategori | `admin` | `staf` | `auditor` | Lainnya |
|---|---|---|---|---|---|
| `/data/public` | Public | rwx | r-x | r-x | r-x |
| `/data/internal` | Internal | rwx | rwx | r-x | --- |
| `/data/confidential` | Confidential | rwx | --- | r-x | --- |
| `/data/confidential/rekam_medis` | Highly Confidential | rwx | --- | --- | --- |

Alasan tiap baris (tulis di laporan):
- **Public:** semua boleh membaca; hanya admin mengubah (menjaga integrity brosur/jadwal).
- **Internal:** staf bekerja di sini (SOP, inventaris) jadi boleh menulis; auditor hanya memeriksa.
- **Confidential:** auditor perlu membaca untuk audit, staf tidak perlu (mereka memakai aplikasi), hanya admin menulis.
- **Rekam medis:** *need-to-know* paling ketat; auditor memeriksa lewat **log akses**, bukan membaca isi rekam medis.

**1) Pasang kepemilikan dan izin:**

```bash
sudo apt install -y acl        # jika belum

# Pemilik: root, group: admin
sudo chown -R root:admin /data
sudo chmod 755 /data

# PUBLIC
sudo find /data/public -type d -exec chmod 775 {} +
sudo find /data/public -type f -exec chmod 664 {} +
sudo setfacl -R -m g:staf:rX,g:auditor:rX /data/public

# INTERNAL
sudo find /data/internal -type d -exec chmod 770 {} +
sudo find /data/internal -type f -exec chmod 660 {} +
sudo setfacl -R -m g:staf:rwX,g:auditor:rX /data/internal
sudo setfacl -d -m g:admin:rwx,g:staf:rwx,g:auditor:rx /data/internal    # default utk berkas baru

# CONFIDENTIAL
sudo find /data/confidential -type d -exec chmod 770 {} +
sudo find /data/confidential -type f -exec chmod 660 {} +
sudo setfacl -R -m g:auditor:rX /data/confidential
sudo setfacl -d -m g:admin:rwx,g:auditor:rx /data/confidential

# REKAM MEDIS (lebih ketat: auditor dicabut)
sudo setfacl -R -m g:auditor:--- /data/confidential/rekam_medis
sudo setfacl -d -m g:admin:rwx,g:auditor:--- /data/confidential/rekam_medis

# Bukti konfigurasi
ls -ld /data /data/*
getfacl /data/internal
getfacl /data/confidential/rekam_medis
```

> Catatan: setelah ACL aktif, kolom *group* di `ls -l` menampilkan **mask** ACL, bukan group pemilik. Itu normal, jelaskan jika muncul di screenshot.

**2) Isi tabel PREDIKSI dulu, sebelum menguji** (dosen menilai prediksi vs hasil):

| User | Direktori | Prediksi baca | Prediksi tulis | Hasil baca | Hasil tulis | Sesuai? |
|---|---|---|---|---|---|---|
| admin04 | /data/public | Ya | Ya | | | |
| admin04 | /data/internal | Ya | Ya | | | |
| admin04 | /data/confidential | Ya | Ya | | | |
| staf04 | /data/public | Ya | Tidak | | | |
| staf04 | /data/internal | Ya | Ya | | | |
| staf04 | /data/confidential | Tidak | Tidak | | | |
| auditor04 | /data/public | Ya | Tidak | | | |
| auditor04 | /data/internal | Ya | Tidak | | | |
| auditor04 | /data/confidential | Ya | Tidak | | | |
| *(tambahan)* auditor04 | /data/confidential/rekam_medis | Tidak | Tidak | | | |

**3) Uji dengan `sudo -u`:**

```bash
# BACA
sudo -u admin04   ls /data/confidential
sudo -u staf04    ls /data/confidential              # → Permission denied
sudo -u auditor04 ls /data/confidential              # → berhasil
sudo -u auditor04 ls /data/confidential/rekam_medis  # → Permission denied

# TULIS
sudo -u staf04    touch /data/internal/uji_staf.txt        # berhasil
sudo -u auditor04 touch /data/internal/uji_auditor.txt     # → Permission denied
sudo -u admin04   touch /data/confidential/uji_admin.txt   # berhasil
sudo -u staf04    touch /data/public/uji_staf.txt          # → Permission denied
```

Ulangi sampai semua baris tabel tercakup, screenshot tiap kelompok pengujian, lalu isi kolom hasil. **Jika hasil tidak sesuai prediksi, jangan disembunyikan.** Perbaiki konfigurasi atau jelaskan penyebabnya; analisis itu bernilai.

Bersihkan berkas uji: `sudo rm -f /data/internal/uji_*.txt /data/confidential/uji_*.txt`

**Model kontrol akses:** kombinasi **RBAC** (hak mengikuti *peran*/group, bukan individu), **DAC** (pemilik/admin menentukan izin lewat `chown`/`chmod`), dan **ACL** (`setfacl` untuk izin granular per group). Prinsip: **least privilege** dan **need-to-know**. Label klasifikasi data menentukan struktur folder, mirip semangat MAC, tetapi di Linux penegakannya tetap DAC + ACL.

### C2. Pengamanan SSH dan MFA (TOTP)

> ⚠️ **Wajib:** ambil snapshot `s1-sebelum-ssh`, **jangan tutup sesi SSH yang sedang terbuka** sampai login baru terbukti berhasil, dan ingat konsol VirtualBox sebagai jalur darurat.

**Tujuan dan logika:** menutup R1. Penyerang butuh **tiga hal**: kunci privat (dimiliki), passphrase kunci (diketahui), dan kode TOTP dari ponsel (dimiliki).

**1) Buat kunci SSH di klien dan salin ke server** (masih port 22):

```bash
# di kdi-kel04-cli
ssh-keygen -t ed25519 -C "admin04@kdi-kel04-cli"      # isi passphrase (SENSOR)
ssh-copy-id admin04@192.168.56.104
ssh admin04@192.168.56.104                             # uji masuk dengan kunci
```

**2) Aktifkan TOTP untuk `admin04` di server:**

```bash
google-authenticator
```

| Pertanyaan | Jawab |
|---|---|
| Time-based tokens? | `y` |
| Update `.google_authenticator`? | `y` |
| Disallow multiple uses (cegah pemakaian ulang kode)? | `y` |
| Increase time window? | `n` |
| Rate-limiting? | `y` |

Pindai QR dengan Google Authenticator/Aegis/Authy. **QR dan kode cadangan (*emergency scratch codes*) wajib disensor di laporan**; simpan kode cadangan di tempat aman.

**3) Konfigurasi PAM:**

```bash
sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
sudo nano /etc/pam.d/sshd
```
Beri tanda `#` pada baris `@include common-auth` (agar password tidak ditanya), lalu tambahkan:
```
auth required pam_google_authenticator.so
```

**4) Konfigurasi `sshd`:**

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```
Pastikan baris berikut aktif dan tidak tertimpa:
```
Port 2204
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
AllowUsers admin04
```

Cek agar tidak ada berkas lain yang menimpa, lalu uji sintaks:

```bash
sudo grep -rn "PasswordAuthentication\|^Port" /etc/ssh/sshd_config.d/
sudo sshd -t
sudo sshd -T | grep -E "^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|authenticationmethods|allowusers)"
```

**5) Restart SSH.** Ubuntu 24.04 memakai *socket activation*, sehingga mengganti `Port` sering tidak berlaku jika hanya `restart`. Matikan socket-nya:

```bash
sudo systemctl daemon-reload
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh
sudo systemctl restart ssh
ss -tlnp | grep 2204          # harus listening di port 2204
```

**6) Uji dari klien (sesi lama tetap terbuka):**

```bash
ssh -p 2204 admin04@192.168.56.104
#   (passphrase kunci jika ada) → "Verification code:" → kode 6 digit dari ponsel
```

Bukti yang diminta:
1. Login berhasil setelah memasukkan kunci **dan** kode TOTP.
2. `ssh -p 2204 root@192.168.56.104` ditolak.
3. `ssh -p 2204 -o PubkeyAuthentication=no admin04@192.168.56.104` ditolak (tanpa kunci tidak bisa masuk).
4. `ssh -p 22 admin04@192.168.56.104` gagal (port lama sudah tidak dipakai).

**Faktor autentikasi yang dipakai:**

| Faktor | Wujud |
|---|---|
| *Something you know* | Passphrase kunci SSH |
| *Something you have* | File kunci privat di klien + ponsel pemilik TOTP |

> Jelaskan juga: TOTP (RFC 6238) berbasis waktu, kode berganti tiap 30 detik, sehingga kode yang tersadap cepat kedaluwarsa. Password SSH dimatikan karena rentan brute force.

### C3. HTTP vs HTTPS (sniffing)

**Cerita:** portal internal RDH menyimpan halaman terlindungi Basic Auth. Kita buktikan betapa berbahayanya HTTP, lalu perbaiki dengan HTTPS (menutup R2).

> **Etika:** sniffing hanya pada adapter Host-only antar-VM milik kelompok.

**1) Siapkan Nginx dengan Basic Auth (server):**

```bash
sudo mkdir -p /var/www/portal/secure
echo "<h1>Portal Internal RDH (kdi-kel04-srv)</h1>" | sudo tee /var/www/portal/secure/index.html
sudo htpasswd -c /etc/nginx/.htpasswd userdummy        # password dummy, mis. PassDummy-04
sudo rm -f /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-available/portal-rdh
```
```nginx
server {
    listen 80;
    server_name kdi-kel04-srv;
    root /var/www/portal;
    index index.html;

    location /secure/ {
        auth_basic "Portal Internal RDH";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```
```bash
sudo ln -s /etc/nginx/sites-available/portal-rdh /etc/nginx/sites-enabled/portal-rdh
sudo nginx -t && sudo systemctl reload nginx
curl -i -u userdummy:PassDummy-04 http://192.168.56.104/secure/     # uji dari klien (harus 200 OK)
```

**2) Rekam lalu lintas HTTP di Wireshark (klien):**

1. Buka Wireshark, pilih interface **enp0s8** (Host-only), klik **Start**.
2. Di terminal klien: `curl -u userdummy:PassDummy-04 http://192.168.56.104/secure/` (atau login lewat browser).
3. Isi filter `http.authbasic` (atau `http`).
4. Klik paket `GET /secure/` → perluas **Hypertext Transfer Protocol → Authorization: Basic ...**. Wireshark menampilkan **Credentials: userdummy:...** dalam teks terbaca.
5. Alternatif bukti yang kuat: klik kanan paket → **Follow → TCP Stream**. Seluruh percakapan HTTP terbaca.
6. Screenshot (tampilkan bahwa kredensial terbaca; sesuai aturan sensor, tutup bagian password dan tunjukkan username serta *field* `Authorization`). Simpan sebagai `http_capture.pcapng`.

> Jelaskan bahwa Base64 adalah *encoding*, **bukan enkripsi**, makanya mudah dibalik.

**3) Buat sertifikat self-signed dan aktifkan HTTPS (server):**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/kdi-kel04.key \
  -out /etc/ssl/certs/kdi-kel04.crt \
  -subj "/C=ID/ST=Aceh/L=Banda Aceh/O=Rencong Digital Health/CN=kdi-kel04-srv" \
  -addext "subjectAltName=DNS:kdi-kel04-srv,IP:192.168.56.104"
sudo chmod 600 /etc/ssl/private/kdi-kel04.key

sudo nano /etc/nginx/sites-available/portal-rdh
```
Tambahkan blok kedua di bawah blok port 80:
```nginx
server {
    listen 443 ssl;
    server_name kdi-kel04-srv;
    root /var/www/portal;
    index index.html;

    ssl_certificate     /etc/ssl/certs/kdi-kel04.crt;
    ssl_certificate_key /etc/ssl/private/kdi-kel04.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /secure/ {
        auth_basic "Portal Internal RDH";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```
```bash
sudo nginx -t && sudo systemctl reload nginx
```

**4) Rekam ulang lewat HTTPS (klien):**

1. Mulai capture baru di Wireshark (enp0s8).
2. `curl -k -u userdummy:PassDummy-04 https://192.168.56.104/secure/` (`-k` karena sertifikat self-signed).
3. Filter `tls`: terlihat *Client Hello*, *Server Hello*, lalu **Application Data** yang tak terbaca. Filter `http.authbasic` kini **kosong**. *Follow TLS Stream* hanya menampilkan data acak.
4. Screenshot dan simpan `https_capture.pcapng`.

**Tabel perbandingan untuk BAB 4:**

| Aspek | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Header `Authorization` di Wireshark | Terbaca | Tidak terlihat |
| Isi percakapan | Teks biasa | *Application Data* terenkripsi |
| Risiko sniffing | Tinggi | Rendah |
| Risiko MITM | Tinggi (tak ada autentikasi server) | Lebih rendah (sertifikat memverifikasi server) |

**Kaitkan dengan serangan:** *sniffing* adalah penyadapan pasif di jalur jaringan; *MITM* adalah penyerang yang menyisip aktif di antara klien dan server untuk membaca/mengubah lalu lintas. TLS memberi enkripsi dan verifikasi identitas server. **Catatan jujur:** sertifikat self-signed tidak dipercaya publik sehingga pengguna harus memverifikasinya manual; di produksi dipakai CA tepercaya (misal Let's Encrypt).

### C4. Firewall (UFW)

**Logika:** hanya buka yang perlu. SSH dari **satu** IP (klien), HTTPS dari subnet, HTTP ditutup.

> Ambil snapshot `s2-sebelum-ufw`.

**1) Nmap SEBELUM aturan** (dari klien):

```bash
nmap -p 22,80,443,2204 192.168.56.104
```
Perkiraan: `22` closed (SSH sudah pindah), `80` open, `443` open, `2204` open. Screenshot.

**2) Terapkan aturan di server.** Izinkan SSH **sebelum** `enable` agar tidak terkunci:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.154 to any port 2204 proto tcp    # SSH hanya dari klien
sudo ufw allow from 192.168.56.0/24 to any port 443 proto tcp    # HTTPS dari subnet Host-only
sudo ufw deny 80/tcp                                             # HTTP diblokir
sudo ufw enable
sudo ufw status numbered
```

**3) Nmap SESUDAH aturan** (dari klien):

```bash
nmap -p 22,80,443,2204 192.168.56.104
```
Perkiraan: `22` dan `80` **filtered** (paket di-drop), `443` open, `2204` open (karena berasal dari IP klien yang diizinkan).

| Port | Sebelum UFW | Sesudah UFW | Penjelasan |
|---|---|---|---|
| 22 | closed | filtered | Tak ada layanan; kini juga di-drop firewall |
| 80 | open | filtered | HTTP diblokir sesuai kebijakan |
| 443 | open | open | Diizinkan dari subnet Host-only |
| 2204 | open | open | Diizinkan hanya dari `192.168.56.154` |

> Bedakan **closed** (port menjawab "tidak ada layanan") dan **filtered** (tidak ada jawaban karena di-drop firewall). Itu bagian dari analisis.

**Jenis firewall UFW:** *host-based firewall* (berjalan di server itu sendiri), **stateful packet filtering** (melacak status koneksi), dan merupakan *front-end* yang menyederhanakan iptables/nftables. Bukan *application-layer/next-generation firewall* karena tidak memeriksa isi aplikasi.

### C5. VPN WireGuard

**Cerita:** pelajaran Fortinet: akses jarak jauh harus lewat gerbang yang kuat dan sempit. Kita jadikan VPN **satu-satunya jalur** menuju SSH.

> Ambil snapshot `s3-sebelum-vpn`.

**1) Buat key pair (di server dan di klien, masing-masing):**

```bash
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key'
sudo cat /etc/wireguard/public.key        # public key boleh ditampilkan
# private.key JANGAN ditampilkan (sensor)
```

**2) Konfigurasi server** `/etc/wireguard/wg0.conf`:

```bash
sudo nano /etc/wireguard/wg0.conf
```
```ini
[Interface]
Address = 10.10.4.1/24
ListenPort = 51820
PrivateKey = <ISI_PRIVATE_KEY_SERVER>

[Peer]
PublicKey = <ISI_PUBLIC_KEY_KLIEN>
AllowedIPs = 10.10.4.2/32
```

**3) Konfigurasi klien** `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.4.2/24
PrivateKey = <ISI_PRIVATE_KEY_KLIEN>

[Peer]
PublicKey = <ISI_PUBLIC_KEY_SERVER>
Endpoint = 192.168.56.104:51820
AllowedIPs = 10.10.4.0/24
PersistentKeepalive = 25
```
```bash
sudo chmod 600 /etc/wireguard/wg0.conf     # di server dan klien
```

**4) Buka port WireGuard di UFW (server), lalu hidupkan tunnel di kedua sisi:**

```bash
# server
sudo ufw allow from 192.168.56.154 to any port 51820 proto udp
sudo systemctl enable --now wg-quick@wg0
sudo wg show

# klien
sudo systemctl enable --now wg-quick@wg0
sudo wg show
ping -c 3 10.10.4.1
```
Bukti: `wg show` memperlihatkan **latest handshake**, dan ping berhasil.

**5) Buka dulu sesi SSH lewat VPN, baru ubah aturan SSH:**

```bash
# klien: pastikan jalur VPN berfungsi (key + TOTP)
ssh -p 2204 admin04@10.10.4.1

# server (di sesi VPN tadi atau konsol VirtualBox):
sudo ufw allow from 10.10.4.0/24 to any port 2204 proto tcp
sudo ufw delete allow from 192.168.56.154 to any port 2204 proto tcp   # cabut jalur lama
sudo ufw status numbered
```

> Aturan dihapus **menurut isinya** (bukan nomor) agar tidak salah hapus karena nomor bergeser.
> Sesi SSH lama yang sudah terbentuk bisa tetap hidup karena koneksi *established* diizinkan; yang diuji adalah koneksi **baru**.

**6) Uji dari klien:**

```bash
ssh -p 2204 -o ConnectTimeout=5 admin04@192.168.56.104    # lewat Host-only → timeout/ditolak
ssh -p 2204 admin04@10.10.4.1                              # lewat VPN → berhasil (key + TOTP)
```

**7) Rekam Wireshark (klien, interface enp0s8):**

1. Mulai capture, lalu lakukan SSH lewat `10.10.4.1`.
2. Filter `udp.port == 51820` (atau `wireguard`): hanya paket UDP terenkripsi (*Transport Data*).
3. Filter `tcp.port == 2204` pada interface Host-only: **kosong**, karena SSH berjalan di dalam terowongan.
4. Screenshot kedua kondisi.

**Jenis VPN:** **remote-access VPN (client-to-server)** berbasis **WireGuard**; bekerja di **layer 3** (interface virtual `wg0`), memakai **UDP**, kriptografi modern (Curve25519, ChaCha20-Poly1305). Berbeda dengan *site-to-site VPN* yang menghubungkan dua jaringan kantor. Karena `AllowedIPs` hanya subnet VPN, ini *split tunnel*, bukan *full tunnel*.

**Kaitkan dengan A2 (Fortinet):** gerbang VPN yang terbuka ke seluruh internet dan tak ditambal adalah pintu masuk empuk. Pada RDH, port WireGuard hanya terbuka untuk IP klien tertentu, WireGuard tidak merespons paket tanpa kunci yang valid (permukaan serangan kecil), dan di balik VPN masih ada lapisan MFA di SSH.

---

## 7. Bagian D — Manajemen Insiden

> **Cerita fase ini:** *"Kalau tetap ada yang mencoba menjebol, sanggupkah kita mendeteksi, menahan, dan belajar?"*
> Ambil snapshot `s4-sebelum-insiden` dulu.

### D1. Simulasi insiden dengan fail2ban

**Skenario (sambungan cerita Fortinet):** kredensial VPN seorang staf RDH diduga bocor, sehingga seseorang memiliki akses ke terowongan `10.10.4.0/24`. Dari dalam VPN, ia mencoba menebak login SSH server. Pada simulasi, **`kdi-kel04-cli` (10.10.4.2) berperan sebagai penyerang.** Lapisan pertama (MFA) sudah menahan masuk; lapisan berikutnya (fail2ban) mendeteksi dan memblokir sumbernya.

**1) Pasang dan atur fail2ban (server):**

```bash
sudo nano /etc/fail2ban/jail.local
```
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = 2204
backend  = systemd
maxretry = 3
findtime = 10m
bantime  = 10m
```
```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd        # Currently banned: 0
```

> Ubuntu 24.04 memakai journald, jadi `backend = systemd`. Berkas `/var/log/auth.log` hanya ada jika `rsyslog` terpasang. Jangan masukkan subnet VPN ke `ignoreip`, kalau tidak, penyerang simulasi tak akan pernah diblokir.

**2) Percobaan login gagal (klien, lewat VPN), minimal 4 kali:**

```bash
ssh -p 2204 hacker@10.10.4.1      # user tidak ada; ulangi minimal 4 kali
ssh -p 2204 hacker@10.10.4.1
ssh -p 2204 hacker@10.10.4.1
ssh -p 2204 hacker@10.10.4.1
```

Pada percobaan awal akan muncul `Permission denied (publickey)`. Begitu IP diblokir, percobaan berikutnya **timeout**. Itu efek yang ditunjukkan di video.

> Satu percobaan bisa menghasilkan lebih dari satu baris log, jadi blokir bisa terjadi sebelum percobaan ke-4. Itu wajar. Catat jumlah percobaan sebenarnya dari log, dan tetap lakukan percobaan ≥ 4 kali agar efek blokir terlihat.
>
> *Varian opsional (menunjukkan "false positive"):* login sebagai `admin04` dengan kode TOTP yang sengaja salah. Perhatikan: karena fail2ban memblokir **IP**, admin asli dari IP yang sama ikut terkunci selama 10 menit. Ini bahan bagus untuk *lessons learned*.

**3) Buktikan IP terblokir dan kumpulkan jejak (server; pakai konsol VirtualBox bila perlu):**

```bash
sudo fail2ban-client status sshd                          # Banned IP list: 10.10.4.2
sudo journalctl -u ssh --since "15 minutes ago" --no-pager | tail -30     # "Invalid user hacker ..."
sudo journalctl -u fail2ban --since "15 minutes ago" --no-pager | tail
sudo tail -20 /var/log/fail2ban.log
sudo iptables -L -n | grep -i f2b          # bukti aturan blokir (atau: sudo nft list ruleset | grep -i f2b)
```

**Amankan bukti seperti SOC sungguhan** (menyambung ke B3: integritas):

```bash
mkdir -p ~/bukti_insiden
sudo journalctl -u ssh --since "today" --no-pager > ~/bukti_insiden/ssh_insiden.log
sudo journalctl -u fail2ban --since "today" --no-pager > ~/bukti_insiden/fail2ban_insiden.log
sha256sum ~/bukti_insiden/*.log | tee ~/bukti_insiden/hash_bukti.txt
```
Hash ini membuktikan log tidak diubah setelah diambil (*chain of custody* sederhana).

**4) Buka blokir (recovery):**

```bash
sudo fail2ban-client set sshd unbanip 10.10.4.2
sudo fail2ban-client status sshd             # daftar blokir kosong lagi
```
```bash
# klien: akses sah pulih
ssh -p 2204 admin04@10.10.4.1                # key + TOTP berhasil
```

**Catat waktu** tiap kejadian (percobaan pertama, ambang tercapai, blokir, unban) untuk timeline D2. Zona waktu sudah WIB.

### D2. Laporan insiden NIST

> **Wajib ditulis dengan kata-kata sendiri.** Di bawah ini hanya kerangka dan logikanya.

**Identitas insiden**

| Item | Isi |
|---|---|
| ID | INC-KEL04-001 |
| Judul | Percobaan brute force SSH dari akun VPN yang diduga terkompromi |
| Tingkat keparahan | Sedang (percobaan gagal dan terblokir; tidak ada akses tidak sah) |
| Sistem terdampak | `kdi-kel04-srv` (layanan SSH port 2204) |
| Pelapor / handler | (nama anggota 7) |

**Matriks keparahan RCSIRT** (bagian "Preparation"):

| Level | Kriteria | Contoh | Respons awal | Eskalasi |
|---|---|---|---|---|
| Rendah | Pemindaian/percobaan tunggal, tak ada akibat | 1–2 login gagal | Pantau | Analis |
| Sedang | Percobaan berulang, terdeteksi dan terblokir | **Insiden D1** | Blokir + analisis log | Incident Commander |
| Tinggi | Akses tidak sah berhasil atau layanan terganggu | Login sukses oleh pihak asing | Isolasi sistem, ganti kredensial | Pimpinan RDH + DPO |
| Kritis | Data pribadi bocor/rusak/terenkripsi pelaku | Ransomware, dump data | Aktifkan DRP, asesmen notifikasi | Pimpinan + DPO + regulator |

**Enam tahap NIST: isi dan logikanya**

| Tahap | Pertanyaan yang dijawab | Isi untuk insiden ini |
|---|---|---|
| 1. Preparation | Apa yang sudah disiapkan *sebelum* kejadian? | Kebijakan keamanan, struktur CSIRT, MFA, UFW, VPN, fail2ban, backup, snapshot, matriks keparahan |
| 2. Identification | Bagaimana kita tahu ini insiden? Seberapa luas? | Pola login gagal berulang di journald, IP sumber `10.10.4.2`, jumlah percobaan, user yang dicoba (`hacker`), tidak ada login sukses; hash bukti |
| 3. Containment | Bagaimana menghentikan penyebaran sementara? | fail2ban memblokir IP (jangka pendek); akses SSH hanya subnet VPN (jangka panjang); tidak ada isolasi total karena akses sah masih dibutuhkan |
| 4. Eradication | Bagaimana menghapus akar masalah? | Verifikasi tak ada akun/kunci/proses asing; tinjau `authorized_keys`; reset kredensial VPN yang diduga bocor (cabut peer WireGuard, buat key baru); tinjau `wg show` |
| 5. Recovery | Bagaimana layanan kembali normal dengan aman? | Unban IP sah, verifikasi login admin (key + TOTP), pantau log beberapa saat |
| 6. Lessons Learned | Apa yang berjalan baik, apa yang diperbaiki? | Baik: MFA dan fail2ban bekerja berlapis. Perbaikan: blokir per-IP bisa mengunci admin sah; pertimbangkan *allowlist* dan peringatan dini; kebijakan rotasi kredensial VPN |

**Timeline** (isi dengan waktu **nyata** dari log; kolom bukti harus menunjuk ke gambar/berkas):

| Waktu (WIB) | Kejadian | Sumber bukti |
|---|---|---|
| hh:mm:ss | Percobaan login gagal pertama dari 10.10.4.2 | `journalctl -u ssh` (Gambar 4.x) |
| hh:mm:ss | Ambang `maxretry` tercapai | log fail2ban |
| hh:mm:ss | IP 10.10.4.2 diblokir (ban 10 menit) | `fail2ban-client status sshd` |
| hh:mm:ss | Analis mengonfirmasi tidak ada login sukses | grep `Accepted` pada log |
| hh:mm:ss | Bukti diamankan dan di-hash | `hash_bukti.txt` |
| hh:mm:ss | IP dibuka blokirnya (recovery) | `fail2ban-client set sshd unbanip` |

Untuk memeriksa "tidak ada login sukses": `sudo journalctl -u ssh --since today | grep -i "accepted"` (hanya baris login admin sah di luar jendela insiden).

**Dampak terhadap CIA**

| Aspek | Dampak | Alasan |
|---|---|---|
| Confidentiality | Terancam, tidak terjadi | Percobaan gagal; tidak ada akses |
| Integrity | Tidak terdampak | Tidak ada perubahan sistem/data |
| Availability | Terdampak ringan | Blokir bisa menghalangi akses sah dari IP yang sama |

**Struktur CSIRT RDH (lampirkan)**

| Peran | Nama | Kontak (fiktif) | Tanggung jawab | Eskalasi ke |
|---|---|---|---|---|
| Incident Commander | (Ketua) | csirt@rdh.example.test | Memimpin penanganan, keputusan | Pimpinan RDH |
| Incident Handler / Analis | (Anggota 7) | | Analisis log, containment | Incident Commander |
| Engineer jaringan dan akses | (Anggota 4–6) | | Aturan firewall/VPN, reset kredensial | Incident Commander |
| Engineer data | (Anggota 2–3) | | Backup, pemulihan data | Incident Commander |
| GRC / Komunikasi / DPO liaison | (Anggota 8) | dpo@rdh.example.test | Dokumentasi, asesmen notifikasi PDP | Pimpinan RDH |

Alur eskalasi: **Deteksi → Analis → Incident Commander → Pimpinan RDH → (bila data pribadi terdampak) DPO → pemberitahuan ke subjek data dan lembaga berwenang.**

**Kewajiban pemberitahuan: UU PDP vs GDPR** (verifikasi dari teks resmi, lalu tulis ulang):

| Aspek | UU PDP (UU 27/2022) | GDPR |
|---|---|---|
| Pemicu | "Kegagalan Pelindungan Data Pribadi" (termasuk data bocor, rusak, hilang, berubah) | "Personal data breach" |
| Kepada siapa | Subjek data **dan** lembaga PDP (Pasal 46 ayat 1); dalam hal tertentu juga masyarakat (ayat 3) | Otoritas pengawas (Art. 33); subjek data bila berisiko tinggi (Art. 34) |
| Batas waktu | Tertulis, paling lambat **3×24 jam** | **72 jam** sejak mengetahui (ke otoritas); ke subjek "tanpa penundaan yang tidak semestinya" |
| Isi minimal | Data yang terungkap; kapan dan bagaimana terungkap; upaya penanganan dan pemulihan (Pasal 46 ayat 2) | Sifat pelanggaran, kategori/jumlah subjek, kontak DPO, konsekuensi, langkah mitigasi (Art. 33(3)) |
| Dokumentasi | Pengendali harus mampu membuktikan pemenuhan kewajiban (Pasal 47) | Semua pelanggaran wajib didokumentasikan, termasuk yang tidak dilaporkan (Art. 33(5)) |
| Sanksi | Sanksi administratif (antara lain peringatan tertulis, penghentian sementara pemrosesan, penghapusan data, denda administratif) dan ketentuan pidana | Denda administratif besar (hingga persentase omzet global atau nominal tertentu) |

> Catatan: UU PDP berlaku efektif sejak Oktober 2024 setelah masa transisi dua tahun. Cek **status terkini lembaga PDP dan peraturan pelaksana** (PP/peraturan turunan) di situs JDIH, karena ini bisa berubah. Sebutkan tanggal kamu mengaksesnya.

**Pertanyaan analisis (jawab di laporan):** *Apakah insiden D1 wajib diberitahukan?* Argumen logisnya: tidak ada login berhasil, jadi tidak ada data pribadi yang terungkap, maka kemungkinan **bukan** kegagalan pelindungan data yang memicu kewajiban notifikasi. Tetapi insiden **tetap didokumentasikan** (sejalan dengan semangat Pasal 47 dan Art. 33(5)). Sebagai latihan (*tabletop*), tulis satu paragraf: *seandainya* login berhasil dan CSV pelanggan terbaca, apa saja isi pemberitahuan 3×24 jam (data yang terungkap, kapan/bagaimana, penanganan, kontak DPO)?

---

## 8. Peta logika: ancaman → kontrol → bukti

Tabel ini adalah "tulang punggung logika" laporan. Taruh di BAB 4 atau ringkasan akhir, dan jadikan pegangan saat presentasi: *setiap kontrol ada karena ancaman tertentu, dan setiap klaim punya bukti.*

| Ancaman (risiko) | Kontrol RDH | Praktik | Aspek CIA | Bukti yang ditunjukkan |
|---|---|---|---|---|
| Data tercuri dari berkas/backup (R6) | Enkripsi AES-256 | B1, B4 | Confidentiality | File `.enc` tak terbaca |
| Pertukaran data diintip dan dipalsukan (R6) | GPG encrypt + sign | B2 | Confidentiality, Integrity | `Good signature`; `No secret key` untuk pihak lain |
| Data diubah diam-diam (R6) | SHA-256 + tanda tangan | B3 | Integrity | Hash berubah; `BAD signature` |
| Data hilang/ransomware (R3) | Backup terjadwal, terenkripsi, terpisah | B4 | Availability | Restore + hash sama |
| Akses berlebihan insider (R4) | RBAC + ACL | C1 | Confidentiality, Integrity | Tabel prediksi vs uji |
| Tebak/curi login SSH (R1) | Kunci + TOTP, port non-standar, AllowUsers | C2 | Confidentiality | Login butuh kunci dan kode |
| Penyadapan kredensial (R2) | HTTPS | C3 | Confidentiality | Wireshark HTTP vs HTTPS |
| Permukaan serangan terlalu luas (R1, R5) | UFW default deny | C4 | Confidentiality, Availability | nmap sebelum/sesudah |
| Kredensial/gateway VPN disalahgunakan (R5) | WireGuard + SSH hanya via VPN | C5 | Confidentiality | SSH Host-only ditolak, via VPN berhasil; UDP terenkripsi |
| Brute force lolos ke server (R1, R5) | fail2ban + respons NIST | D1, D2 | Confidentiality, Availability | IP terblokir + log + laporan |

---

## 9. Menyusun laporan PDF

**Format:** A4, Times New Roman atau Calibri 12 pt, spasi 1,5. **10–25 halaman di luar lampiran.**
**Nama file:** `Tugas Besar_KDI_Kelompok ke-4_(Unit).pdf` (isi bagian *Unit* dengan kelas/unit kalian).
Tulis di Google Docs/Word, lalu ekspor ke PDF.

| Bagian | Isi | Perkiraan halaman |
|---|---|---|
| Sampul | Judul, **Kelompok 4**, nama dan NIM semua anggota, kelas | 1 |
| Pembagian tugas | Tabel nama, NIM, peran, bagian (tabel 0.3). **Anggota yang tidak berkontribusi: nama diberi font merah** | 1 |
| **BAB 1** Pendahuluan | Profil RDH, spesifikasi host, topologi, tabel parameter, kebijakan keamanan, register aset/risiko | 2–3 |
| **BAB 2** Kajian konsep, etika, privasi | A1, A2, A3 | 4–5 |
| **BAB 3** Implementasi | Langkah dan bukti B1–B4, C1–C5, D1 | 8–10 |
| **BAB 4** Pengujian dan analisis | Tabel prediksi vs uji C1, HTTP vs HTTPS, nmap sebelum/sesudah, uji VPN, laporan NIST D2 | 4–5 |
| **BAB 5** Kesimpulan | Kesimpulan dan kendala | 1 |
| Daftar pustaka | Minimal 5 sumber (wajib UU 27/2022 dan panduan NIST) | 1 |
| Lampiran (tidak dihitung) | Change log, struktur CSIRT, daftar tool AI, skrip `backup.sh`, konfigurasi (tersensor) | bebas |

**Aturan gambar:**
- Setiap gambar punya **nomor dan keterangan**, misal *Gambar 3.12 Hasil nmap sebelum UFW dari kdi-kel04-cli*.
- **Hostname terlihat** (prompt `...@kdi-kel04-...`). Tanpa itu, butir tidak dinilai.
- **Sensor** dengan kotak hitam: password, passphrase, private key, QR TOTP, kode cadangan (blur tipis tidak aman).

**Nama berkas screenshot:** `B1-01-enkripsi.png`, `C3-02-wireshark-http.png`, dst. Lalu tinggal diurutkan.

| Kode | Screenshot minimal |
|---|---|
| Tahap 3 | `ip a` kedua VM, `ping`, `id` tiga user, struktur `/data` |
| B1 | enkripsi, isi `.enc` acak, dekripsi + `IDENTIK` |
| B2 | generate key, tukar public key + fingerprint, encrypt+sign, decrypt + `Good signature`, `No secret key` untuk staf04 |
| B3 | dua hash berbeda, `Good signature` vs `BAD signature` |
| B4 | isi `backup.sh`, hasil cron (≥2 file), salinan di klien, `rm -rf /data`, restore, hash sama |
| C1 | `ls -ld`, `getfacl`, tabel prediksi, minimal 10 pengujian `sudo -u` |
| C2 | `sshd -T` penting, prompt `Verification code`, login berhasil, root ditolak, tanpa kunci ditolak |
| C3 | Wireshark HTTP (kredensial terbaca), HTTPS (`tls`, Application Data), tabel perbandingan |
| C4 | nmap sebelum, `ufw status numbered`, nmap sesudah |
| C5 | `wg show` (handshake), SSH Host-only ditolak, SSH VPN berhasil, Wireshark UDP 51820 |
| D1 | `fail2ban-client status sshd` (banned), `journalctl`, hash bukti, unban |

**Disclosure AI (taruh di lampiran):** tuliskan tool AI yang dipakai dan untuk apa, contoh: *"Claude (Anthropic) digunakan untuk memahami konsep dan menelusuri pesan error; seluruh analisis A2 dan laporan D2 ditulis sendiri oleh anggota kelompok."* Pastikan kalimat ini benar sesuai kenyataan kelompok.

**Cek orisinalitas:** laporan yang identik dengan kelompok lain bernilai 0 untuk semua yang terlibat. Karena itu, pakai organisasi, nama user, dan skenario milik sendiri, dan jangan menyalin teks dari panduan ini mentah-mentah.

---

## 10. Merekam video presentasi

**Durasi 10–15 menit**, MP4, minimal 720p, suara jelas, teks terbaca. **Semua anggota wajib tampil (wajah atau suara) dan menjelaskan bagiannya.** Unggah video **dan file slide** ke folder Drive.

| Segmen | Durasi | Isi | Pembicara |
|---|---|---|---|
| 1. Pembukaan | ±1 mnt | Perkenalan anggota, peran, RDH, dan cerita besar (0.2) | Ketua (1) + semua menyebut nama dan peran |
| 2. Konsep dan etika | ±2 mnt | Klasifikasi data, ringkasan A2 Fortinet (pelanggaran, regulasi, 3 rekomendasi), kebijakan privasi | 1 dan 8 |
| 3. Kripto dan backup | ±3 mnt | AES, GPG (encrypt+sign, verifikasi), hash/BAD signature, backup → hapus → restore | 2 dan 3 |
| 4. Akses dan jaringan | ±6 mnt | Uji akses 3 user (C1), SSH + TOTP (C2), Wireshark HTTP vs HTTPS (C3), nmap sebelum/sesudah (C4), SSH via VPN (C5) | 4, 5, 6 (±2 mnt masing-masing) |
| 5. Insiden | ±2 mnt | Serangan simulasi → blokir fail2ban → ringkasan 6 tahap NIST | 7 (+ 8 untuk UU PDP vs GDPR) |
| 6. Penutup | ±1 mnt | Kesimpulan dan kendala | Ketua (1) |

**Rencana demo 6 menit (segmen 4) yang cepat dan kelihatan efeknya:**

| Menit | Demo | Apa yang harus terlihat |
|---|---|---|
| 0–1.5 | C1: `sudo -u staf04 ls /data/confidential` (ditolak) vs `auditor04` (berhasil), tulis ke internal | Prediksi = hasil, `Permission denied` |
| 1.5–3 | C2: login `admin04` dari klien, kode TOTP muncul, `root` ditolak | Prompt `Verification code:` |
| 3–4 | C3: Wireshark `http.authbasic` (terbaca) vs `tls` (terenkripsi) | Dua layar berdampingan |
| 4–5 | C4: nmap sebelum (80 open) vs sesudah (80 filtered) | Tabel port berubah |
| 5–6 | C5: SSH ke `192.168.56.104` ditolak, ke `10.10.4.1` berhasil, Wireshark hanya UDP | Efek VPN |

**Tips produksi:**
- Pakai OBS Studio atau perekam layar bawaan; **perbesar font terminal** (`Ctrl` + `+`).
- Susun **skrip demo** (daftar perintah di berkas teks) agar tidak mengetik panjang di depan kamera.
- Siapkan keadaan awal lewat snapshot (misal sebelum UFW untuk demo nmap "sebelum vs sesudah").
- Tunjukkan hostname di terminal tiap demo.
- Kode TOTP berganti tiap 30 detik: siapkan ponsel dan perhatikan sisa waktunya sebelum mengetik.
- Boleh memotong bagian menunggu (misal cron 2 menit) saat editing; tetap berupa demo langsung di lab sendiri.
- Cek total durasi 10–15 menit. Terlalu jauh dari ketentuan mengurangi nilai.

---

## 11. Pengumpulan dan checklist akhir

1. Buat folder Google Drive bernama **`Tugas Besar_KDI_Kelompok ke-4_(Unit)`**.
2. Unggah: **laporan PDF**, **video MP4**, dan **file slide**.
3. Atur akses **"Siapa saja yang memiliki link — Pelihat"**.
4. **Uji link** di jendela incognito/akun lain. Link yang tak bisa dibuka dianggap belum mengumpulkan video.
5. **Ketua** mengisi Google Form **sekali** per kelompok: https://s.id/KDI-2026 (perlu login akun Google verified).
6. **Jangan hapus/ubah isi folder** sampai nilai diumumkan.

**Checklist akhir:**

- [ ] Parameter Kelompok 4 benar di semua konfigurasi (kel04, .104/.154, port 2204, 10.10.4.x)
- [ ] Setiap screenshot menampilkan hostname `kdi-kel04-...`
- [ ] Password, passphrase, private key, QR TOTP, kode cadangan **tersensor**
- [ ] Laporan 10–25 halaman (di luar lampiran), format A4, 12 pt, spasi 1,5, gambar bernomor
- [ ] Daftar pustaka ≥ 5, termasuk UU 27/2022 dan panduan NIST
- [ ] Tool AI disebutkan; A2 dan D2 kata-kata sendiri
- [ ] Anggota tanpa kontribusi: nama berfont merah
- [ ] Video 10–15 menit, 720p, semua anggota tampil
- [ ] Slide ikut diunggah di folder yang sama
- [ ] Link Drive diuji publik
- [ ] Nama berkas dan folder sesuai ketentuan
- [ ] Hanya data dummy dan semua uji di VM sendiri

---

## 12. Jadwal 7 hari

| Hari | Kegiatan | Penanggung jawab |
|---|---|---|
| 1 | Bagian 3: VM, jaringan, snapshot, user, `/data`, data dummy. Paralel: Ketua dan No. 8 mulai A1, A3, kerangka laporan | Semua |
| 2 | B1–B3. Paralel: A2 (riset dan kerangka) | 2, 3; 1 dan 8 |
| 3 | B4 (backup, cron, restore). C1 (RBAC) | 3; 4 |
| 4 | C2 (SSH + TOTP). C3 (HTTP/HTTPS) | 4; 5 |
| 5 | C4 (UFW, nmap). C5 (WireGuard) | 5; 6 |
| 6 | D1 (fail2ban), D2 (laporan NIST). Rapikan laporan, sensor screenshot | 7 dan 8; semua |
| 7 | Latihan demo, rekam video, edit, unggah, uji link, kirim form | Semua; ketua mengirim |

> **Tiga aturan penyelamat waktu:** (1) ambil screenshot **sambil bekerja**, bukan di akhir; (2) kerjakan teori paralel dengan praktik; (3) selalu ambil snapshot sebelum perubahan berisiko.

---

## 13. Troubleshooting

| Masalah | Penyebab umum | Solusi |
|---|---|---|
| Terkunci dari SSH | Salah konfigurasi SSH/PAM/UFW | Masuk lewat konsol VirtualBox, kembalikan `sshd_config.bak`/`pam.d/sshd.bak`, atau restore snapshot |
| Port SSH tidak berubah | `ssh.socket` masih aktif (Ubuntu 24.04) | `sudo systemctl disable --now ssh.socket && sudo systemctl enable --now ssh`, cek `ss -tlnp` |
| Klien tidak bisa ping server | Adapter Host-only salah, IP salah, atau UFW | `ip a` di kedua VM, cek jaringan Host-only, `sudo ufw status` |
| Kode TOTP selalu salah | Jam VM tidak sinkron | `timedatectl`; `sudo timedatectl set-ntp true`; cek zona waktu ponsel |
| Masih ditanya password saat SSH | `common-auth` belum dikomentari atau berkas di `sshd_config.d` menimpa | Periksa `/etc/pam.d/sshd` dan `grep -r PasswordAuthentication /etc/ssh/sshd_config.d/` |
| Wireshark tak melihat interface | User belum di grup `wireshark` | `sudo usermod -aG wireshark $USER`, login ulang; atau jalankan dengan `sudo` |
| Tidak ada paket HTTP di capture | Salah interface | Pilih interface Host-only (`enp0s8`), bukan NAT |
| `setfacl: command not found` | Paket `acl` belum terpasang | `sudo apt install acl` |
| `Permission denied` saat `sudo -u staf04 ...` ke home admin | Home Ubuntu 24.04 bermode 750 | Pakai `/srv/pertukaran` atau salin berkas ke lokasi yang bisa dibaca |
| gpg: `Inappropriate ioctl for device` | Pinentry tak menemukan terminal | `export GPG_TTY=$(tty)` |
| WireGuard tidak handshake | Public key tertukar, `Endpoint` salah, port 51820/udp belum diizinkan | Cek `wg show`, `sudo ufw status`, ulangi pertukaran public key |
| fail2ban tidak memblokir | Port jail salah, backend salah, IP masuk `ignoreip` | `fail2ban-client get sshd ignoreip`; pastikan `port = 2204`, `backend = systemd` |
| Nginx gagal reload | Sintaks salah | `sudo nginx -t` dan baca baris error |
| Disk server penuh | Cron backup masih aktif | `sudo crontab -e` beri `#`, hapus backup lama |
| Pertanyaan teknis | | Tanyakan ke Pak Nizam di hari dan jam kerja |

---

## 14. Sumber rujukan

Gunakan minimal 5 untuk daftar pustaka (format sitasi mengikuti pedoman kampus; cantumkan tanggal akses untuk sumber web).

**Regulasi dan standar**
1. Undang-Undang Republik Indonesia Nomor 27 Tahun 2022 tentang Pelindungan Data Pribadi (cek teks resmi di JDIH).
2. NIST, *Computer Security Incident Handling Guide* (SP 800-61); cek revisi terbaru di csrc.nist.gov.
3. Regulation (EU) 2016/679, General Data Protection Regulation (GDPR).
4. California Consumer Privacy Act (CCPA).
5. RFC 6238, *TOTP: Time-Based One-Time Password Algorithm*.

**Studi kasus VPN Fortinet (A2)**
6. BleepingComputer, *Hacker posts exploits for over 49,000 vulnerable Fortinet VPNs*: https://www.bleepingcomputer.com/news/security/hacker-posts-exploits-for-over-49-000-vulnerable-fortinet-vpns/
7. SiliconANGLE, *Hacker publishes credentials stolen from Fortinet's FortiGate VPNs* (25 Nov 2020): https://siliconangle.com/2020/11/25/hacker-publishes-credentials-stolen-fortinet-fortigate-vpns/
8. Security Affairs, *The global impact of the Fortinet 50.000 VPN leak posted online*: https://securityaffairs.com/111548/hacking/fortinet-vpn-leak-impact.html
9. Fortinet PSIRT Advisory FG-IR-18-384: https://www.fortiguard.com/psirt/FG-IR-18-384

**Pemberitahuan kegagalan pelindungan data (D2)**
10. Hukumonline, *Kewajiban Penting dalam UU PDP dan Sanksinya Jika Melanggar*: https://www.hukumonline.com/klinik/a/kewajiban-penting-dalam-uu-pdp-dan-sanksinya-jika-melanggar-lt633abd26570aa
11. Teks Pasal 46 UU 27/2022: https://pasal.id/peraturan/uu/uu-no-27-tahun-2022/pasal-46

**Panduan teknis**
12. Ubuntu, *Configure SSH to use two-factor authentication* (tutorial resmi).
13. Ubuntu, *Set Up WireGuard VPN on Ubuntu 24.04*.
14. WireGuard, *Whitepaper*: https://www.wireguard.com/papers/wireguard.pdf

> Pastikan kamu benar-benar membaca sumber yang dikutip, dan kutip dengan kalimatmu sendiri.

---

Selamat bekerja, Rencong CSIRT. Jika ada langkah yang error atau hasil yang tidak sesuai prediksi, kirim pesan error/screenshot-nya. Kita telusuri bersama.
