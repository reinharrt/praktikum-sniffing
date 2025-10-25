# Network Sniffing & Spoofing Lab

Panduan praktikum keamanan jaringan untuk memahami teknik sniffing dan spoofing dalam lingkungan laboratorium terkontrol.

> **DISCLAIMER**: Tutorial ini hanya untuk keperluan edukasi. Gunakan hanya di lab environment dengan izin yang sah.

---

## Daftar Isi

- [Topologi Jaringan](#topologi-jaringan)
- [Persiapan](#persiapan)
- [Lab 1: Network Sniffing](#lab-1-network-sniffing)
- [Lab 2: ARP Spoofing Attack](#lab-2-arp-spoofing-attack)
- [Pencegahan](#pencegahan)

---

## Topologi Jaringan

### Konfigurasi IP

| Device | IP Address | Peran | Platform |
|--------|------------|-------|----------|
| Host Machine | 192.168.1.20 | Testing client | Windows/Linux/Mac |
| Attacker | 192.168.1.50 | Melakukan attack | Kali Linux VM |
| Victim | 192.168.1.30 | Target attack | Ubuntu/Debian VM |
| Server | 192.168.1.100 | Web server (HTTP) | Ubuntu/Debian VM |

### Diagram

```
    Host Machine (192.168.1.20)
              |
    +---------+---------+
    |         |         |
Attacker   Victim   Server
  (.50)     (.30)    (.100)
```

**Catatan:**
- Menggunakan **Host-Only Network** (tidak ada gateway/router)
- Semua VM dalam satu subnet: 192.168.1.0/24
- Host machine untuk testing koneksi ke server dari luar VM

---

## Persiapan

### Install Tools di Attacker VM

```bash
sudo apt update
sudo apt install wireshark ettercap-graphical nmap fping -y
sudo usermod -aG wireshark $USER
```

Logout dan login kembali.

### Setup Web Server di Server VM

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Deploy vulnerable login page:
```bash
cd web-vulnerable
sudo cp -r * /var/www/html/
```

Test akses dari Host Machine: `http://192.168.1.100`

**Test Accounts:**
- Username: `admin` | Password: `admin123`
- Username: `user1` | Password: `password`

### Verifikasi Koneksi

```bash
# Ping ke 3 IP sekaligus
fping -c 4 192.168.1.20 192.168.1.30 192.168.1.100
```

---

## Lab 1: Network Sniffing

### Konsep

Network sniffing menangkap paket data di jaringan menggunakan Wireshark untuk melihat data yang dikirim tanpa enkripsi.

### Step-by-Step

#### 1. Jalankan Wireshark

```bash
sudo wireshark
```

#### 2. Pilih Network Interface

- Double-click interface aktif (`enp0s3` atau `eth0`)
- Jangan pilih `lo` (localhost)

#### 3. Victim Login ke Website

Di Host Machine atau Victim VM:
1. Buka browser
2. Akses: `http://192.168.1.100`
3. Login: `admin` / `admin123`
4. Klik **Sign In**

#### 4. Filter Paket HTTP POST

Di Wireshark, masukkan filter:
```
http.request.method == "POST"
```
Tekan **Enter**

#### 5. Follow HTTP Stream

1. **Klik kanan** pada paket POST
2. Pilih: **Follow → HTTP Stream**
3. Lihat credentials:
```
username=admin&password=admin123
```

#### 6. Analisis Packet Details

Alternatif: Klik paket POST, expand di **Packet Details**:
- `▶ Hypertext Transfer Protocol`
- `▶ HTML Form URL Encoded`
- Lihat: `Form item: "username" = "admin"`

#### 7. Save Capture

```
File → Save As → sniffing_capture.pcapng
```

#### 8. Stop Capture

Klik tombol **Stop** atau tekan **Ctrl+E**

---

## Lab 2: ARP Spoofing Attack

### Konsep

ARP Spoofing memalsukan MAC address untuk melakukan Man-in-the-Middle attack. Traffic victim akan melewati attacker.

```
NORMAL:
Victim <-> Server

SETELAH ARP POISONING:
Victim <-> Attacker <-> Server
            (MITM)
```

### Step-by-Step

#### 1. Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Verifikasi:
```bash
cat /proc/sys/net/ipv4/ip_forward
```
Output harus: `1`

#### 2. Scan Network

```bash
sudo nmap -sn 192.168.1.0/24
```

Catat IP: Victim (192.168.1.30) dan Server (192.168.1.100)

#### 3. Jalankan Ettercap

```bash
sudo ettercap -G
```

#### 4. Pilih Network Interface

1. Menu: **Sniff → Unified Sniffing**
2. Pilih interface (`enp0s3`)
3. Klik **OK**

#### 5. Scan Hosts

1. Menu: **Hosts → Scan for hosts**
2. Tunggu scan selesai

#### 6. Tampilkan Hosts List

1. Menu: **Hosts → Hosts list**
2. Lihat semua device

#### 7. Pilih Target

**Target 1 (Victim):**
1. Klik IP: `192.168.1.30`
2. Klik: **Add to Target 1**

**Target 2 (Server):**
1. Klik IP: `192.168.1.100`
2. Klik: **Add to Target 2**

Verifikasi: **Targets → Current targets**

#### 8. Mulai ARP Poisoning

1. Menu: **Mitm → ARP poisoning**
2. Centang: **Sniff remote connections**
3. Klik **OK**

#### 9. Start Sniffing

1. Menu: **Start → Start sniffing**
2. Status: "Sniffing started"

#### 10. Victim Login

Di Host Machine atau Victim VM:
1. Buka browser
2. Akses: `http://192.168.1.100`
3. Login: `admin` / `admin123`

#### 11. Monitor Credentials

1. Menu: **View → Connections**
2. Menu: **View → Messages**
3. Cari HTTP POST dengan credentials

#### 12. Verifikasi ARP Poisoning

Di Victim VM:
```bash
arp -a
```

MAC address server akan berubah menjadi MAC attacker.

#### 13. Stop Attack

1. **Mitm → Stop mitm attack(s)**
2. **Start → Stop sniffing**
3. **File → Exit**

---

## Pencegahan

### Untuk User

**Gunakan HTTPS**
- Selalu akses dengan `https://`
- Pastikan ada icon gembok
- HTTPS mengenkripsi semua data

**Gunakan VPN**
- Enkripsi semua traffic
- Recommended: OpenVPN, WireGuard

**Monitor ARP Table**
```bash
arp -a
sudo apt install arpwatch
```

### Untuk Network Administrator

**Implementasi HTTPS**
- Deploy SSL/TLS certificate
- Redirect HTTP ke HTTPS

**Port Security**
- Enable di managed switch
- Batasi MAC address per port

**Dynamic ARP Inspection (DAI)**
- Validasi ARP packets otomatis
- Drop ARP mencurigakan

**Static ARP Entries**
```bash
sudo arp -s 192.168.1.100 aa:bb:cc:dd:ee:ff
```

**Network Segmentation**
- Gunakan VLAN
- Pisahkan network berdasarkan trust level

**Deploy IDS/IPS**
- Snort, Suricata
- Monitor ARP spoofing patterns

---

## Legal Notice

**DILARANG untuk:**
- Mengakses jaringan tanpa izin
- Mencuri data orang lain
- Merugikan pihak lain
- Aktivitas ilegal

**Konsekuensi:** Pelanggaran UU ITE dan hukum cyber security.

---

## Referensi

- [Wireshark Documentation](https://www.wireshark.org/docs/)
- [Ettercap Project](https://www.ettercap-project.org/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)

---

*Last Updated: October 2025*
