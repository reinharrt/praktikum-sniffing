# Network Sniffing & Spoofing Lab

Panduan praktikum keamanan jaringan untuk memahami teknik sniffing dan spoofing dalam lingkungan laboratorium terkontrol.

> **DISCLAIMER**: Tutorial ini hanya untuk keperluan edukasi. Gunakan hanya di lab environment dengan izin yang sah.

---

## Daftar Isi

- [Topologi Jaringan](#topologi-jaringan)
- [Persiapan](#persiapan)
- [Lab 1: ARP Spoofing Attack](#lab-1-arp-spoofing-attack)
- [Lab 2: Network Sniffing](#lab-2-network-sniffing)
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
sudo apt install ettercap-graphical wireshark nmap -y
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

---

## Lab 1: ARP Spoofing Attack

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

Catat IP address semua device.

#### 3. Jalankan Ettercap

```bash
sudo ettercap -G
```

#### 4. Pilih Network Interface

1. Menu: **Sniff → Unified Sniffing**
2. Pilih interface aktif (misal: `enp0s3`)
3. Klik **OK**

#### 5. Scan Hosts

1. Menu: **Hosts → Scan for hosts**
2. Tunggu scan selesai
3. Menu
