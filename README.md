# 🔐 Network Sniffing & Spoofing Tutorial

Panduan praktikum lengkap untuk memahami teknik **Sniffing** dan **Spoofing** dalam keamanan jaringan untuk tujuan edukasi.

> ⚠️ **DISCLAIMER**: Tutorial ini hanya untuk keperluan edukasi dan pembelajaran keamanan jaringan. Jangan gunakan untuk tujuan ilegal atau merugikan orang lain. Lakukan hanya di lab environment atau isolated network dengan izin yang sah.

---

## 📋 Daftar Isi

- [Topologi Jaringan](#topologi-jaringan)
- [Persiapan](#persiapan)
- [Praktikum 1: Sniffing dengan Wireshark](#praktikum-1-sniffing-dengan-wireshark)
- [Praktikum 2: Spoofing dengan Ettercap](#praktikum-2-spoofing-dengan-ettercap-arp-poisoning)
- [Troubleshooting](#troubleshooting)
- [Pencegahan](#pencegahan)
- [Referensi](#referensi)

---

## 🌐 Topologi Jaringan

### Konfigurasi IP Address

| Device | IP Address | MAC Address | Peran | OS |
|--------|------------|-------------|-------|-----|
| **Server** | `192.168.1.100` | `22:33:44:55:66:77` | Web Server (Vulnerable Login Page) | Debian/Ubuntu |
| **Attacker** | `192.168.1.50` | `77:88:99:aa:bb:cc` | Melakukan Sniffing & Spoofing | Kali Linux / Ubuntu |
| **Victim** | `192.168.1.30` | `11:22:33:44:55:66` | Target Attack | Windows / Linux |
| **Gateway** | `192.168.1.1` | `aa:bb:cc:dd:ee:ff` | Router/Gateway | Router |

### Diagram Topologi

```
                    ┌─────────────┐
                    │   Gateway   │
                    │ 192.168.1.1 │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴─────┐ ┌────┴────┐ ┌────┴─────┐
        │  Attacker │ │ Victim  │ │  Server  │
        │   .50     │ │   .30   │ │   .100   │
        └───────────┘ └─────────┘ └──────────┘
```

**Catatan:** 
- Semua device harus dalam **satu jaringan** yang sama (192.168.1.0/24)
- Sesuaikan IP address dengan konfigurasi jaringan lab kamu

---

## 🛠️ Persiapan

### Requirements

**Software yang dibutuhkan:**
- Wireshark (untuk Sniffing)
- Ettercap (untuk ARP Poisoning)
- Nmap (untuk network scanning)
- Python 3 (untuk web server)

**Sistem Operasi:**
- Debian/Ubuntu/Kali Linux (recommended untuk Attacker)
- Windows/Linux (untuk Victim)

### Install Tools di Attacker PC

```bash
# Update repository
sudo apt update && sudo apt upgrade -y

# Install Wireshark
sudo apt install wireshark -y
sudo usermod -aG wireshark $USER

# Install Ettercap
sudo apt install ettercap-graphical -y

# Install Nmap
sudo apt install nmap -y

# Logout dan login lagi untuk apply group changes
newgrp wireshark
```

### Setup Web Server di Debian

Web vulnerable login page sudah tersedia di repository ini. Deploy dengan cara:

```bash
# Clone repository
git clone https://github.com/username/network-sniffing-spoofing.git
cd network-sniffing-spoofing

# Jalankan web server
cd web-vulnerable
python3 -m http.server 80
```

**Akses dari browser:**
```
http://192.168.1.100/index.html
```

**Test Accounts:**
- Username: `admin` | Password: `admin123`
- Username: `user1` | Password: `password`
- Username: `siswa` | Password: `12345678`

---

## 📡 Praktikum 1: Sniffing dengan Wireshark

### Penjelasan

**Sniffing** adalah teknik menangkap dan menganalisis paket data yang lewat di jaringan untuk mencuri informasi sensitif seperti password yang dikirim tanpa enkripsi (HTTP).

**Karakteristik:**
- ✅ Teknik **pasif** (hanya mendengarkan)
- ✅ Tidak memanipulasi traffic
- ✅ Sulit dideteksi
- ❌ Hanya efektif di hub atau shared network

---

### Langkah-Langkah Sniffing

#### 1. Jalankan Wireshark

```bash
sudo wireshark
```

#### 2. Pilih Network Interface

- Pilih interface yang aktif (`eth0`, `enp0s3`, atau `wlan0`)
- Lihat grafik activity - pilih yang ada traffic
- **Jangan pilih** `lo` (localhost)
- **Double-click** interface untuk start capture

#### 3. Victim Login ke Website

**Di Victim PC (192.168.1.30):**
1. Buka browser
2. Akses: `http://192.168.1.100/index.html`
3. Login dengan:
   - Username: `admin`
   - Password: `admin123`
4. Klik **Sign In**

#### 4. Filter Paket HTTP POST

Di Wireshark, masukkan filter:

```
http.request.method == "POST"
```

Tekan **Enter**

**Filter alternatif:**
```
http
http.request
ip.addr == 192.168.1.100
http.request.method == "POST" && ip.dst == 192.168.1.100
```

#### 5. Follow HTTP Stream

1. **Klik kanan** pada paket POST
2. Pilih: **Follow → HTTP Stream**
3. Window baru muncul menampilkan:

```http
POST /index.html HTTP/1.1
Host: 192.168.1.100
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin123
```

**🎯 Password terlihat jelas dalam plaintext!**

#### 6. Alternatif - Packet Details

1. Klik paket POST
2. Expand di **Packet Details**:
   - `▶ Hypertext Transfer Protocol`
   - `▶ HTML Form URL Encoded`
3. Lihat:
   - `Form item: "username" = "admin"`
   - `Form item: "password" = "admin123"`

#### 7. Save Capture

```
File → Save As
Simpan: sniffing_capture.pcapng
```

---

### Screenshot untuk Laporan - Sniffing

✅ **Wajib di-screenshot:**

1. Wireshark main window dengan packets captured
2. Filter bar: `http.request.method == "POST"`
3. List paket hasil filter
4. Follow HTTP Stream dengan credentials
5. Packet Details dengan form data
6. **ZOOM** ke `username=admin&password=admin123`

---

## 🎭 Praktikum 2: Spoofing dengan Ettercap (ARP Poisoning)

### Penjelasan

**ARP Poisoning** adalah teknik spoofing yang memalsukan ARP reply untuk melakukan Man-in-the-Middle (MITM) attack.

**Konsep:**
- **ARP** = Address Resolution Protocol
- Mapping: IP Address ↔ MAC Address
- Attacker memalsukan ARP reply
- Victim mengira Attacker adalah Gateway
- Semua traffic melewati Attacker

**Diagram Attack:**

```
NORMAL:
[Victim] ←----------→ [Gateway] ←--→ [Internet]

SETELAH ARP POISONING:
[Victim] ←--→ [Attacker] ←--→ [Gateway] ←--→ [Internet]
                  ↑
           (MITM - bisa baca semua!)
```

---

### Langkah-Langkah ARP Poisoning

#### 1. Enable IP Forwarding

**PENTING!** Tanpa ini, koneksi victim akan putus.

```bash
# Cek status
cat /proc/sys/net/ipv4/ip_forward

# Enable IP Forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Verifikasi (harus output: 1)
cat /proc/sys/net/ipv4/ip_forward
```

**Enable Permanent:**

```bash
sudo nano /etc/sysctl.conf
# Tambahkan: net.ipv4.ip_forward=1
sudo sysctl -p
```

#### 2. Scan Network

```bash
# Scan jaringan untuk cari device
sudo nmap -sn 192.168.1.0/24
```

**Catat IP:**
- Attacker: `192.168.1.50`
- Victim: `192.168.1.30`
- Gateway: `192.168.1.1`
- Server: `192.168.1.100`

#### 3. Jalankan Ettercap GUI

```bash
sudo ettercap -G
```

#### 4. Pilih Network Interface

1. Menu: **Sniff → Unified Sniffing...**
2. Pilih interface aktif (`enp0s3` atau `wlan0`)
3. Klik **OK**

Status bar:
```
Listening on enp0s3... (192.168.1.50/255.255.255.0)
```

#### 5. Scan Hosts

1. Menu: **Hosts → Scan for hosts**
2. Tunggu scan selesai
3. Status: `X hosts added to the hosts list...`

#### 6. Lihat Hosts List

1. Menu: **Hosts → Hosts list**
2. Akan muncul daftar semua device di network

**Contoh:**

| IP | MAC | Hostname |
|----|-----|----------|
| 192.168.1.1 | aa:bb:cc:dd:ee:ff | gateway |
| 192.168.1.30 | 11:22:33:44:55:66 | victim-pc |
| 192.168.1.50 | 77:88:99:aa:bb:cc | attacker |
| 192.168.1.100 | 22:33:44:55:66:77 | server |

#### 7. Pilih Target

**Add Target 1 (Victim):**
1. Klik IP: `192.168.1.30`
2. Klik: **Add to Target 1**

**Add Target 2 (Gateway):**
1. Klik IP: `192.168.1.1`
2. Klik: **Add to Target 2**

**Verifikasi:**

Menu: **Targets → Current targets**

```
TARGET 1: 192.168.1.30
TARGET 2: 192.168.1.1
```

#### 8. Mulai ARP Poisoning

1. Menu: **Mitm → ARP poisoning...**
2. **CENTANG**: ☑️ `Sniff remote connections`
3. Klik **OK**

**Status bar:**
```
Starting MITM attack...
ARP poisoning victims:
GROUP 1: 192.168.1.30
GROUP 2: 192.168.1.1
```

**🎯 Attack AKTIF!**

#### 9. Start Sniffing

1. Menu: **Start → Start sniffing**
2. Status: `Sniffing started...`

#### 10. Monitor Connections

1. Menu: **View → Connections**
2. Lihat semua koneksi victim

**Contoh:**
```
192.168.1.30:54321 <-> 192.168.1.100:80 [HTTP]
192.168.1.30:54322 <-> 8.8.8.8:53 [DNS]
```

#### 11. Victim Login - Capture Credentials

**Di Victim PC:**
1. Buka browser
2. Akses: `http://192.168.1.100/index.html`
3. Login: `admin` / `admin123`
4. Klik Sign In

**Di Attacker (Ettercap):**

Menu: **View → Messages**

Cari HTTP POST:
```
HTTP : 192.168.1.30:54321 -> 192.168.1.100:80
POST /index.html HTTP/1.1

username=admin&password=admin123
```

**🎯 CREDENTIALS CAPTURED!**

#### 12. Verifikasi ARP Poisoning

**Di Victim PC, cek ARP table:**

**Sebelum attack:**
```bash
arp -a
# Output: 192.168.1.1 at aa:bb:cc:dd:ee:ff
```

**Saat attack:**
```bash
arp -a
# Output: 192.168.1.1 at 77:88:99:aa:bb:cc (MAC Attacker!)
```

**MAC address gateway BERUBAH = Bukti ARP Poisoning!**

#### 13. Stop Attack

**Stop ARP Poisoning:**
- Menu: **Mitm → Stop mitm attack(s)**

**Stop Sniffing:**
- Menu: **Start → Stop sniffing**

**Exit:**
- **File → Exit**

---

### Screenshot untuk Laporan - Spoofing

✅ **Wajib di-screenshot:**

1. Terminal: `ip_forward = 1`
2. Output `nmap -sn` scan network
3. Ettercap hosts list
4. Current targets (Target 1 & 2)
5. Status bar "Starting MITM attack"
6. Message log: ARP poison packets
7. Message log: Credentials captured
8. Connections window
9. ARP table victim **SEBELUM** attack
10. ARP table victim **SAAT** attack
11. Diagram topologi jaringan

---

## 📊 Perbandingan Sniffing vs Spoofing

| Aspek | Sniffing (Wireshark) | Spoofing (ARP Poisoning) |
|-------|----------------------|--------------------------|
| **Teknik** | Passive listening | Active attack (MITM) |
| **Target** | Semua traffic di segment | Traffic victim ↔ gateway |
| **Jaringan** | Hub / Shared network | Bisa di switched network |
| **Kemampuan** | Hanya membaca paket | Bisa intercept & modify |
| **Deteksi** | Sangat sulit | Bisa dideteksi (ARP anomaly) |
| **Kompleksitas** | Mudah | Sedang |
| **Risiko Tertangkap** | Rendah | Tinggi |
| **Tools** | Wireshark, tcpdump | Ettercap, arpspoof, bettercap |

---

## 🔧 Troubleshooting

### Sniffing Issues

**Problem:** Tidak bisa capture packets
```bash
# Solusi: Tambahkan user ke group wireshark
sudo usermod -aG wireshark $USER
newgrp wireshark
```

**Problem:** Paket POST tidak muncul
- Pastikan victim login via **HTTP** (bukan HTTPS)
- Cek filter: `http.request.method == "POST"`
- Pastikan di network yang sama

### Spoofing Issues

**Problem:** Victim koneksi putus saat ARP poisoning
```bash
# Solusi: Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
```

**Problem:** ARP poisoning tidak bekerja
- Pastikan Target 1 = Victim, Target 2 = Gateway
- Centang "Sniff remote connections"
- Restart Ettercap

**Problem:** Credentials tidak muncul
- Pastikan sniffing sudah started
- Lihat di: View → Messages
- Pastikan victim login via HTTP

### Network Issues

**Problem:** Tidak bisa ping antar device
```bash
# Cek firewall
sudo ufw status
sudo ufw disable  # Untuk testing

# Cek IP address
ip addr show
```

---

## 🛡️ Pencegahan

### Untuk User

✅ **Gunakan HTTPS**
- Selalu akses website dengan `https://`
- Install HTTPS Everywhere extension
- Credentials terenkripsi end-to-end

✅ **Gunakan VPN**
- Enkripsi semua traffic
- Lindungi dari MITM attack
- Recommended: OpenVPN, WireGuard

✅ **Monitor ARP Table**
```bash
# Cek ARP table secara berkala
watch -n 5 arp -a

# Install ARP monitoring tools
sudo apt install arpwatch
```

### Untuk Network Administrator

✅ **Port Security**
- Enable port security di managed switch
- Limit MAC address per port

✅ **Dynamic ARP Inspection (DAI)**
- Available di enterprise switch
- Validasi ARP packets

✅ **Static ARP Entries**
```bash
# Set static ARP untuk gateway
sudo arp -s 192.168.1.1 aa:bb:cc:dd:ee:ff
```

✅ **Network Segmentation**
- Gunakan VLAN untuk isolasi
- Pisahkan network berdasarkan trust level

✅ **IDS/IPS**
- Deploy Intrusion Detection System
- Detect ARP spoofing patterns
- Recommended: Snort, Suricata

---

## 📚 Referensi

### Tools Documentation

- [Wireshark Official Docs](https://www.wireshark.org/docs/)
- [Ettercap Documentation](https://www.ettercap-project.org/documentation.html)
- [Nmap Reference Guide](https://nmap.org/book/man.html)

### Learning Resources

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Kali Linux Tools](https://www.kali.org/tools/)
- [Wireshark Tutorial](https://www.wireshark.org/docs/wsug_html_chunked/)

### Security Standards

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [ISO 27001](https://www.iso.org/isoiec-27001-information-security.html)

---

## ⚖️ Legal & Ethical Considerations

### ⚠️ PERINGATAN PENTING

**Praktikum ini hanya untuk:**
- ✅ Keperluan edukasi dan pembelajaran
- ✅ Lab environment atau isolated network
- ✅ Dengan izin tertulis dari pemilik jaringan
- ✅ Memahami cara kerja attack untuk pertahanan

**DILARANG untuk:**
- ❌ Mengakses jaringan tanpa izin
- ❌ Mencuri data orang lain
- ❌ Merugikan pihak lain
- ❌ Aktivitas ilegal apapun

**Konsekuensi Hukum:**
- Pelanggaran UU ITE (Indonesia)
- Computer Fraud and Abuse Act (USA)
- Hukuman pidana dan denda

**Etika Hacker:**
- Selalu minta izin sebelum testing
- Laporkan vulnerability yang ditemukan
- Jangan exploit untuk keuntungan pribadi
- Respect privacy orang lain

---

## 📝 Laporan Praktikum

### Struktur Laporan

```
Laporan_Praktikum_Network_Security/
├── 1_Pendahuluan/
│   ├── Tujuan_Praktikum.md
│   └── Teori_Dasar.md
├── 2_Praktikum_Sniffing/
│   ├── Langkah_Kerja.md
│   ├── Screenshots/
│   │   ├── 01_wireshark_interface.png
│   │   ├── 02_http_filter.png
│   │   ├── 03_http_stream.png
│   │   └── 04_credentials.png
│   └── Analisis.md
├── 3_Praktikum_Spoofing/
│   ├── Langkah_Kerja.md
│   ├── Screenshots/
│   │   ├── 01_nmap_scan.png
│   │   ├── 02_ettercap_hosts.png
│   │   ├── 03_arp_poisoning.png
│   │   ├── 04_arp_table_before.png
│   │   ├── 05_arp_table_during.png
│   │   └── 06_credentials_captured.png
│   └── Analisis.md
└── 4_Kesimpulan/
    ├── Perbandingan.md
    ├── Pencegahan.md
    └── Pembelajaran.md
```

### Template Laporan

**Isi minimum laporan:**

1. **Pendahuluan**
   - Definisi Sniffing & Spoofing
   - Tujuan praktikum
   - Topologi jaringan

2. **Landasan Teori**
   - Cara kerja HTTP
   - Protokol ARP
   - Man-in-the-Middle Attack

3. **Praktikum Sniffing**
   - Langkah-langkah detail
   - Screenshots (minimum 6)
   - Analisis paket yang tertangkap

4. **Praktikum Spoofing**
   - Langkah-langkah detail
   - Screenshots (minimum 10)
   - Analisis ARP table

5. **Kesimpulan**
   - Perbandingan kedua metode
   - Cara deteksi & pencegahan
   - Pembelajaran yang didapat

---

## 👨‍💻 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### How to Contribute

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Your Name**
- GitHub: [@yourusername](https://github.com/yourusername)
- Email: your.email@example.com

---

## 🙏 Acknowledgments

- Terima kasih kepada dosen pembimbing
- Wireshark & Ettercap development team
- Kali Linux community
- OWASP Foundation

---

## 📮 Contact & Support

Jika ada pertanyaan atau butuh bantuan:

- 📧 Email: support@example.com
- 💬 Discord: [Join Server](https://discord.gg/example)
- 📖 Wiki: [Project Wiki](https://github.com/username/repo/wiki)

---

**⭐ Jika tutorial ini membantu, jangan lupa beri star di repository!**

---

*Last Updated: January 2025*
