# AbuseIPDB Blacklist + IPinfo (JSON)

Kumpulan snapshot **IP blacklist dari AbuseIPDB** yang diambil **5x sehari**. Setiap IP dilengkapi data jaringan dari **IPinfo** (ASN, nama provider/ISP, domain, negara) dan disimpan sebagai file **JSON** di folder [`data/`](data/).

Tujuannya ada dua:

1. Menyimpan riwayat IP yang dilaporkan melakukan serangan (scanning, brute force, spam, exploit, dan sebagainya) dalam format yang mudah dipakai ulang.
2. Membantu memantau **nama baik (reputasi) IP address Indonesia**: IP Indonesia mana yang masuk blacklist, milik provider apa, dan berapa lama bertahan di sana.

| | |
|---|---|
| Jadwal pengambilan (WIB) | 01:00 · 06:00 · 11:00 · 16:00 · 21:00 |
| Sumber blacklist | [AbuseIPDB](https://www.abuseipdb.com) — endpoint `/api/v2/blacklist` |
| Sumber data jaringan | [IPinfo Lite](https://ipinfo.io) |
| Format | JSON (1 record per baris) + TXT daftar IP |
| Kumpulan 30 hari | [`data/last-30-days.txt`](data/last-30-days.txt) — **semua** IP unik 30 hari terakhir, bisa jauh lebih dari 10.000 |
| Dipisah per wilayah | [`last-30-days-indonesia.txt`](data/last-30-days-indonesia.txt) (hanya IP Indonesia) · [`last-30-days-international.txt`](data/last-30-days-international.txt) (tanpa IP Indonesia) |

---

## Apa itu AbuseIPDB?

[AbuseIPDB](https://www.abuseipdb.com) adalah basis data terpusat tempat admin server, penyedia hosting, dan sistem keamanan di seluruh dunia **melaporkan IP address yang melakukan aktivitas berbahaya**, misalnya:

- port scan dan *web scanning* (mencari celah, file `.env`, `wp-login.php`, dan sejenisnya),
- *brute force* SSH, RDP, FTP, email, atau login web,
- spam, *phishing*, DDoS, *exploit*, dan *bad bot*.

Setiap IP mendapat **Abuse Confidence Score** (0–100%), yaitu tingkat keyakinan AbuseIPDB bahwa IP tersebut berbahaya. Skor dihitung dari jumlah laporan, jumlah pelapor yang berbeda, umur laporan, dan reputasi pelapor. Makin tinggi skornya, makin besar kemungkinan IP itu memang sumber serangan.

**Blacklist** AbuseIPDB adalah daftar IP dengan skor tertinggi yang paling banyak dan paling baru dilaporkan. Daftar ini biasa dipakai untuk memblokir di firewall, WAF, atau router.

## Apa itu IPinfo?

[IPinfo](https://ipinfo.io) adalah penyedia data IP address, yaitu informasi **siapa pemilik dan di mana lokasi** sebuah IP. Repo ini memakai **IPinfo Lite** (gratis) untuk melengkapi setiap IP dengan:

| Field | Contoh | Arti |
|---|---|---|
| `asn` | `AS7713` | *Autonomous System Number*: nomor jaringan pemilik blok IP |
| `as_name` | `PT Telekomunikasi Indonesia` | nama pemilik jaringan (ISP, hosting, cloud, kampus, dan sebagainya) |
| `as_domain` | `telkom.co.id` | domain milik pemilik jaringan |
| `country` | `Indonesia` | negara versi IPinfo |

Data AbuseIPDB hanya menyebut IP-nya. Dengan data IPinfo, kita bisa tahu **provider mana** yang paling banyak IP-nya masuk blacklist.

---

## Batasan pengambilan data

Repo ini memakai akun AbuseIPDB **gratis**, sehingga ada batasan berikut:

| Batasan | Nilai |
|---|---|
| Pemanggilan endpoint blacklist | **5x per hari** (kuota reset 00:00 UTC = 07:00 WIB) |
| Jumlah IP per pengambilan | **maksimal 10.000 IP** |
| Skor minimum | 100% (akun gratis tidak bisa mengatur `confidenceMinimum`) |
| Pembaruan daftar di sisi AbuseIPDB | daftar dihitung ulang secara berkala, bukan *real-time* |

Karena kuota cuma 5x sehari, jadwal dibagi rata kira-kira tiap 5 jam: **01:00, 06:00, 11:00, 16:00, 21:00 WIB**.

### Setiap pengambilan ≠ 10.000 IP baru

Ini penting dipahami. **Setiap kali diambil, AbuseIPDB mengembalikan "10.000 IP teratas saat itu", bukan 10.000 IP yang belum pernah kita lihat.** Isinya campuran:

- **IP lama** yang masih aktif menyerang dan sudah ada di pengambilan sebelumnya (biasanya sebagian besar),
- **IP baru** yang baru naik ke daftar teratas,
- sementara sebagian IP dari pengambilan sebelumnya **keluar** dari daftar (laporannya mulai lama, atau kalah dari IP lain).

```
Pengambilan ke-1:  [A B C D E ... 10.000 IP]
Pengambilan ke-2:  [A B C   E ... F G H ...]   -> A,B,C,E lama | F,G,H baru | D keluar
                         ▲ masih ada  ▲ baru masuk
```

Karena itu repo ini membedakan tiga hal:

| Data | File | Isi |
|---|---|---|
| Snapshot terbaru | `data/latest.json`, `data/latest.txt` | persis 10.000 IP (atau kurang) dari pengambilan terakhir |
| Perubahan | `data/changes/YYYY/MM/YYYY-MM-DD_HHMM.json` | IP yang **baru masuk** dan IP yang **keluar** dibanding pengambilan sebelumnya |
| List kumulatif | `data/list.json` | **semua IP unik** yang pernah masuk blacklist, dengan `first_seen` (pertama terlihat), `last_seen` (terakhir terlihat), `times_seen` (berapa kali muncul), dan `in_latest` (masih ada di snapshot terbaru atau tidak) |

IP dengan `times_seen` besar berarti **terus-menerus** berada di blacklist. IP seperti itu adalah penyerang yang konsisten, bukan kebetulan.

### Kumpulan IP blacklist 30 hari terakhir (lebih dari 10.000 IP)

Satu pengambilan dibatasi 10.000 IP. Tapi karena isinya berganti setiap pengambilan, gabungan beberapa pengambilan menghasilkan **jauh lebih banyak IP unik**. File ini menggabungkan **semua IP yang pernah muncul di blacklist dalam 30 hari terakhir**, dari pengambilan mana pun, tanpa duplikat:

```
Pengambilan 1 (10.000) ─┐
Pengambilan 2 (10.000) ─┤
...                     ├─▶ gabung + buang duplikat ─▶ data/last-30-days.txt  (mis. 25.000+ IP unik)
Pengambilan 150 (10.000)┘     (5x sehari × 30 hari)
```

| File | Isi |
|---|---|
| `data/last-30-days.txt` | daftar IP unik 30 hari terakhir, 1 IP per baris. Paling praktis untuk ditarik semua ke firewall / ipset / address-list |
| `data/last-30-days.json` | sama, lengkap dengan data AbuseIPDB + IPinfo + `first_seen`, `last_seen`, `times_seen`, `in_latest` |
| `data/last-30-days-indonesia.txt` / `.json` | **hanya IP Indonesia** dari kumpulan 30 hari |
| `data/last-30-days-international.txt` / `.json` | **tanpa IP Indonesia** (semua negara lain) dari kumpulan 30 hari |
| `data/summary/by-country-30d.json` | jumlah IP per negara dalam 30 hari terakhir |

**IP Indonesia** = IP dengan `country_code` (AbuseIPDB) **atau** `ipinfo_country_code` (IPinfo) bernilai `ID`. Kedua sumber dipakai karena data negaranya kadang berbeda. Dengan aturan ini, `indonesia` + `international` selalu sama dengan `last-30-days` (tidak ada IP yang dobel atau hilang). Jumlah masing-masing tercatat di `meta.indonesia` dan `meta.international` pada `last-30-days.json`.

Kenapa dipisah?

- **`international`** cocok untuk **diblokir** di firewall/WAF: tidak ada risiko memblokir pelanggan, mitra, atau pengguna dari Indonesia yang IP-nya dipakai bergantian (CGNAT / IP dinamis).
- **`indonesia`** cocok untuk **dipantau**: memeriksa reputasi IP dalam negeri, menemukan server atau perangkat Indonesia yang disusupi, dan melapor ke provider-nya. Sebaiknya tidak diblokir mentah-mentah.

Aturannya: sebuah IP masuk file ini kalau `last_seen`-nya (terakhir terlihat di blacklist) masih dalam 30 hari dari pengambilan terbaru. IP yang sudah lebih dari 30 hari tidak muncul lagi akan **otomatis keluar** dari file ini, tapi tetap tercatat di `list.json`. `meta` di file JSON mencatat `window_start` (awal jendela 30 hari), total IP, jumlah IPv4/IPv6, berapa yang masih ada di snapshot terbaru (`in_latest`), jumlah negara, dan jumlah IP Indonesia.

> Jumlah IP di file ini terus bertambah selama 30 hari pertama pengumpulan data, lalu stabil karena IP lama mulai keluar dari jendela.

Perbandingan singkat:

| Kebutuhan | Pakai file |
|---|---|
| IP paling "panas" saat ini (maks 10.000) | `latest.txt` / `latest.json` |
| Semua IP yang aktif menyerang sebulan terakhir | `last-30-days.txt` / `last-30-days.json` |
| Blokir IP luar negeri saja, sebulan terakhir | `last-30-days-international.txt` |
| Pantau IP Indonesia yang masuk blacklist sebulan terakhir | `last-30-days-indonesia.txt` / `.json` |
| Semua IP yang pernah tercatat sejak awal | `list.json` |
| Apa yang berubah di tiap pengambilan | `changes/…` |

---

## Nama baik IP address Indonesia

IP address punya **reputasi**. Kalau sebuah IP sering dilaporkan menyerang, IP itu masuk blacklist dan **dicurigai atau diblokir** oleh banyak layanan di seluruh dunia: firewall, WAF (misalnya Cloudflare), server email, layanan cloud, sampai situs e-commerce.

Masalahnya, dampaknya **tidak hanya kena ke pelaku**:

- **IP dipakai bergantian.** Di Indonesia, banyak pelanggan internet rumah dan seluler berbagi IP publik lewat NAT/CGNAT, dan IP dinamis berpindah dari satu pelanggan ke pelanggan lain. Pelanggan yang tidak bersalah bisa ikut terkena blokir atau CAPTCHA terus-menerus.
- **Server yang disusupi.** Banyak IP Indonesia di blacklist berasal dari server, VPS, router, atau CCTV yang **diretas** lalu dipakai menyerang pihak lain, sering kali tanpa disadari pemiliknya.
- **Email tidak terkirim.** IP server mail yang masuk blacklist membuat email perusahaan masuk spam atau ditolak.
- **Akses ke layanan luar terhambat.** Integrasi API, payment gateway, atau layanan cloud bisa menolak koneksi dari IP berreputasi buruk.
- **Citra jaringan Indonesia.** Makin banyak IP dari sebuah negara atau ASN yang masuk blacklist, makin besar kemungkinan administrator di luar negeri memblokir **seluruh** blok IP atau negara tersebut.

### Kenapa perlu mengecek kualitas IP

Sebelum memakai, menyewa, atau mempercayai sebuah IP, cek dulu apakah IP itu **bersih atau sedang di blacklist**:

| Situasi | Kenapa perlu cek |
|---|---|
| Menyewa VPS / IP publik baru | IP bekas pakai bisa membawa reputasi buruk dari penyewa sebelumnya |
| Memasang server mail | IP yang di-blacklist membuat email ditolak atau masuk spam |
| Ada lonjakan traffic atau login gagal | Cocokkan IP sumber dengan blacklist untuk memutuskan perlu blokir atau tidak |
| Memeriksa IP milik sendiri | Kalau IP kantor/server sendiri masuk blacklist, kemungkinan ada perangkat yang **terinfeksi** dan sedang menyerang pihak lain. Ini perlu segera ditangani |
| Provider / ISP | Pantau ASN sendiri untuk menemukan pelanggan atau server yang disusupi |

Cara cek cepat dari data repo ini:

```bash
# apakah sebuah IP ada di snapshot terbaru?
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/latest.txt | grep -x '203.0.113.10'

# apakah sebuah IP pernah masuk blacklist dalam 30 hari terakhir?
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days.txt | grep -x '203.0.113.10'

# tarik semua IP 30 hari terakhir ke file lokal (mis. untuk ipset / firewall)
curl -s -o abuseipdb-30d.txt https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days.txt
wc -l abuseipdb-30d.txt

# jumlah IP Indonesia dalam 30 hari terakhir
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days.json | jq '.meta.indonesia'

# daftar IP Indonesia 30 hari terakhir + provider-nya
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days-indonesia.json \
  | jq -r '.data[] | "\(.ip)\t\(.asn)\t\(.as_name)\t\(.times_seen)x"'

# provider Indonesia dengan IP blacklist terbanyak (30 hari)
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days-indonesia.json \
  | jq -r '.data[] | "\(.asn) \(.as_name)"' | sort | uniq -c | sort -rn | head -20

# IP luar negeri saja untuk diblokir (tanpa IP Indonesia)
curl -s -o block-international.txt https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/last-30-days-international.txt

# semua IP Indonesia di snapshot terbaru (IP, ASN, provider)
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/latest.json \
  | jq -r '.data[] | select(.country_code=="ID") | "\(.ip)\t\(.asn)\t\(.as_name)"'

# provider Indonesia dengan IP blacklist terbanyak
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/latest.json \
  | jq -r '.data[] | select(.country_code=="ID") | "\(.asn) \(.as_name)"' | sort | uniq -c | sort -rn | head -20

# IP Indonesia yang paling lama bertahan di blacklist (list kumulatif)
curl -s https://raw.githubusercontent.com/kakanda007/abuseipdb/main/data/list.json \
  | jq -r '.data[] | select(.country_code=="ID") | "\(.times_seen)\t\(.ip)\t\(.as_name)"' | sort -rn | head -20
```

Untuk satu IP yang **tidak** ada di repo ini, cek langsung di `https://www.abuseipdb.com/check/<IP>`. Ingat, blacklist ini hanya berisi 10.000 IP teratas; IP yang tidak ada di sini **belum tentu bersih**.

Kalau IP milik Anda ternyata masuk daftar:

1. Cari dan bersihkan sumbernya (malware, akun yang dibobol, port yang terbuka, proxy/router yang disalahgunakan).
2. Ganti password dan tutup layanan yang tidak perlu.
3. Setelah bersih, laporan lama akan berkurang bobotnya seiring waktu. Pemilik IP juga bisa mengklaim IP/jaringannya di AbuseIPDB.

---

## Isi folder `data/`

| File | Isi |
|---|---|
| `latest.json` | snapshot terbaru: semua IP + data IPinfo |
| `latest.txt` | daftar IP saja (1 IP per baris), untuk firewall / ipset / address-list MikroTik |
| `last-30-days.json` | **semua IP unik 30 hari terakhir** (gabungan semua pengambilan, lebih dari 10.000) + data IPinfo |
| `last-30-days.txt` | daftar IP unik 30 hari terakhir, 1 IP per baris |
| `last-30-days-indonesia.txt` / `.json` | kumpulan 30 hari, **hanya IP Indonesia** |
| `last-30-days-international.txt` / `.json` | kumpulan 30 hari, **tanpa IP Indonesia** |
| `changes/YYYY/MM/YYYY-MM-DD_HHMM.json` | IP **baru masuk** (lengkap dengan data) + `meta.removed_ips` (IP yang keluar); waktu di nama file dalam WIB |
| `summary/by-country.json` | jumlah dan persentase IP per negara (snapshot terbaru) |
| `summary/by-country-30d.json` | jumlah dan persentase IP per negara (30 hari terakhir) |
| `summary/by-asn.json` | top 100 ASN / provider |
| `history.json` | ringkasan tiap pengambilan: total, IPv4/IPv6, baru, keluar, dan `unique_last_window` (jumlah IP unik 30 hari saat itu) |
| `list.json` | list kumulatif semua IP unik (jadi `list.json.gz` kalau ukurannya lebih dari 40 MB) |

Riwayat setiap snapshot juga tersimpan di **git history** (1 commit per pengambilan). Contoh: `git log -p data/latest.txt`.

### Format record

```json
{"meta":{"source":"AbuseIPDB /api/v2/blacklist (https://www.abuseipdb.com)","enrichment":"IPinfo Lite (https://ipinfo.io)",
 "snapshot_at":"2026-11-23T04:00:12Z","snapshot_at_wib":"2026-11-23T11:00:12+07:00","previous_snapshot_at":"2026-11-22T23:00:09Z",
 "total":10000,"ipv4":9871,"ipv6":129,"added":412,"removed":412,"without_ipinfo":0,"generator":"abuseipdb-github-publish 1.2.0"},
"data":[
{"ip":"203.0.113.10","ip_version":4,"abuse_confidence_score":100,"last_reported_at":"2026-11-23T03:58:01Z",
 "country_code":"ID","ipinfo_country_code":"ID","country":"Indonesia","asn":"AS64500","as_name":"Contoh Provider","as_domain":"contoh.id",
 "first_seen":"2026-11-21T00:14:06Z","last_seen":"2026-11-23T04:00:12Z","times_seen":9}
]}
```

| Field | Sumber | Keterangan |
|---|---|---|
| `ip`, `ip_version` | AbuseIPDB | IPv4 / IPv6 |
| `abuse_confidence_score` | AbuseIPDB | 0–100 |
| `last_reported_at` | AbuseIPDB | waktu laporan terakhir (UTC) |
| `country_code` | AbuseIPDB | kode negara ISO 3166-1 alpha-2 |
| `ipinfo_country_code`, `country`, `asn`, `as_name`, `as_domain` | IPinfo Lite | negara versi IPinfo bisa berbeda dengan versi AbuseIPDB |
| `first_seen`, `last_seen`, `times_seen` | dihitung sendiri | dari semua snapshot sejak data dikumpulkan |
| `in_latest` | dihitung sendiri | hanya di `list.json` dan `last-30-days*.json` |

Semua waktu memakai format ISO 8601 UTC (`Z`), kecuali `snapshot_at_wib`.

---

## Cara kerja

```
AbuseIPDB /blacklist ──5x/hari, maks 10.000 IP──▶ server monitoring
                                                     │  bandingkan dengan snapshot & list sebelumnya
                                                     │  IP baru ──▶ IPinfo Lite (cache, tiap IP cukup dicek sekali)
                                                     ▼
                                          abuseipdb-github-publish ──git push──▶ repo ini
                                            ├─ latest.*          (snapshot terbaru, ≤ 10.000)
                                            ├─ last-30-days.*    (gabungan 30 hari, > 10.000)
                                            │   ├─ last-30-days-indonesia.*      (hanya ID)
                                            │   └─ last-30-days-international.*  (tanpa ID)
                                            ├─ changes/, summary/, history.json
                                            └─ list.json         (kumulatif sejak awal)
```

- Pengambilan dilakukan oleh server monitoring internal, **bukan** GitHub Actions, supaya kuota 5x sehari tidak terpakai dua kali.
- IPinfo hanya dipanggil untuk IP yang **belum pernah** dicek, jadi data IP lama diambil dari cache.
- Hanya data AbuseIPDB dan IPinfo yang dipublikasikan. Data internal tidak ikut, misalnya log serangan ke server sendiri, keputusan blokir, atau IP milik sendiri.

## Catatan pemakaian

- Data ini bersifat **informatif**. Blacklist bisa berisi IP dinamis atau IP bersama (CGNAT, VPN, proxy, cloud). Jangan memblokir tanpa pertimbangan di layanan yang melayani publik, terutama untuk IP Indonesia yang mungkin dipakai pelanggan sendiri.
- Data bisa terlambat beberapa jam dari kondisi sebenarnya.
- IP yang **tidak** ada di sini belum tentu bersih (lihat batas 10.000 IP). Untuk cakupan yang lebih luas, pakai `last-30-days.txt`.
- `last-30-days.txt` lebih lengkap tapi juga lebih "longgar": sebagian IP di dalamnya mungkin sudah tidak aktif. Untuk blokir yang ketat, `latest.txt` lebih aman.
- Untuk layanan yang melayani pengguna Indonesia, pakai `last-30-days-international.txt` untuk blokir dan perlakukan `last-30-days-indonesia.txt` sebagai daftar pantau.

## Lisensi & atribusi

Kode dirilis dengan lisensi MIT (lihat `LICENSE`). **Data** tetap mengikuti ketentuan sumbernya: [AbuseIPDB](https://www.abuseipdb.com/legal) dan [IPinfo](https://ipinfo.io/terms-of-service).

Blacklist data by [AbuseIPDB](https://www.abuseipdb.com) · IP address data powered by [IPinfo](https://ipinfo.io).
