# Backup Repository Calculator

Kalkulator berbasis web (satu file HTML, tanpa dependensi) untuk menghitung kebutuhan kapasitas repository backup (skenario **Non-Immutable** dan **Immutable** berdampingan), resource server backup, dan lisensi, mengikuti best practice tiap vendor dan tiap workload.

Demo: https://fadliramm.github.io/backup-repository-calculator/

## Cara pakai

Buka `index.html` di browser. Tidak perlu server atau instalasi.

1. Pilih **vendor backup**. Target penyimpanan, dedup global, buffer, dan asumsi sizing otomatis mengikuti best practice vendor tersebut.
2. Atur **Pengaturan Global**: pertumbuhan data per tahun, lama proyeksi, free space buffer, pola perubahan data, dan backup window.
3. Pilih **Target Penyimpanan** untuk skenario Non-Immutable dan Immutable, plus periode immutability.
4. Tambahkan **Workload** (ukuran TB + jumlah VM/server/user). Saat tipe workload dipilih, parameter terisi otomatis dari best practice dan masih bisa diubah.
5. Lihat hasil: Executive Summary, kapasitas kedua skenario, proyeksi per tahun, rincian per workload, resource server backup, kebutuhan lisensi, dan rekomendasi. Hasil bisa diekspor ke CSV atau dicetak ke PDF.

## Vendor yang didukung

Pilih vendor dari dropdown di kanan atas. Setiap vendor punya target penyimpanan, aturan retensi/immutability, rasio dedup global, buffer, sizing server, dan metrik lisensi sendiri.

| Vendor | Target Non-Immutable | Target Immutable | Dedup global default | Buffer | Komponen yang di-size | Metrik lisensi |
|---|---|---|---|---|---|---|
| Veeam Data Platform | ReFS / XFS (Fast Clone), NTFS | Hardened Repository, S3 Object Lock | 0% | 20% | VBR server, proxy, repository/gateway, VB365 | Instance (VUL), NAS per 500 GB, M365 per user, Kasten per node |
| Commvault Cloud | Disk library + DDB | HyperScale X / Hardened MA, Cloud WORM (2× retensi) | 25% | 20% | CommServe, MediaAgent (+DDB), VSA access node | FETB, M365 per user |
| Rubrik Security Cloud | Cluster tanpa Retention Lock | Retention Lock, Cloud Vault | 20% | 25% | Node appliance (kapasitas/throughput/minimum) | FETB, M365 per user, hardware node |
| Cohesity DataProtect | Cluster tanpa DataLock | DataLock (WORM), FortKnox | 25% | 25% | Node + N+1 | FETB, M365 per user, hardware node |
| Dell PowerProtect DM + DD | DD tanpa Retention Lock | DD Retention Lock Compliance, Cyber Recovery | 35% | 20% | PPDM, protection engine, model DD | FETB (PPDM), TB usable (DD), M365 per user |
| Druva (SaaS) | Cloud standar | Cloud immutable (default) | 20% | 0% | Backup proxy, CloudCache (opsional) | TB terpakai (credits), M365 per user |
| HYCU R-Cloud | NFS/SMB / Nutanix Objects | S3 Object Lock | 0% | 20% | HYCU instance, backup controller | Per instance, NAS per TB, M365 per user |
| IBM Storage Protect | Directory container pool | Container pool + cloud Object Lock | 25% | 20% | Server (Blueprint S/M/L), data mover | FETB, M365 per user |

Untuk vendor yang memproteksi Microsoft 365 via SaaS (Commvault, Rubrik, Cohesity, Dell, Druva), data M365 tidak dihitung di repository on-prem.

Asumsi hardware tiap vendor (throughput per task/node, kapasitas per node, minimum node, dan lainnya) bisa diubah di panel "Asumsi sizing". Perbandingan antar vendor di Executive Summary memakai target dan asumsi default masing-masing vendor.

> Daftar vendor diambil dari pemain yang umum muncul di Gartner Magic Quadrant for Backup and Data Protection Platforms. Posisi kuadran berubah setiap tahun; cek laporan terbaru.

## Executive Summary

Bagian paling atas berisi KPI (data terproteksi, kapasitas immutable/non-immutable, infrastruktur, lisensi utama, throughput), ringkasan naratif, grafik proyeksi per tahun, dan tabel perbandingan semua vendor dengan input yang sama.

## Default best practice per workload

| Workload | Change/hari | Data reduction | Log/hari | Retensi harian | Weekly | Monthly | Yearly |
|---|---|---|---|---|---|---|---|
| VMware vSphere VM | 5% | 50% | – | 14 | 4 | 12 | 1 |
| Microsoft Hyper-V VM | 5% | 50% | – | 14 | 4 | 12 | 1 |
| Physical Server (Agent) | 3% | 50% | – | 14 | 4 | 12 | 0 |
| Microsoft SQL Server | 10% | 60% | 5% | 14 | 4 | 12 | 3 |
| Oracle Database (RMAN) | 10% | 60% | 8% | 14 | 4 | 12 | 1 |
| SAP HANA (Backint) | 15% | 40% | 10% | 14 | 4 | 12 | 1 |
| Microsoft Exchange (on-prem) | 4% | 40% | 3% | 14 | 4 | 12 | 1 |
| File Server / NAS | 2% | 30% | – | 30 | 4 | 12 | 7 |
| Microsoft 365 | 1% | 30% | – | 30 | 0 | 12 | 3 |
| Kubernetes (PV) | 5% | 50% | – | 7 | 4 | 6 | 0 |
| VDI (Persistent) | 8% | 60% | – | 7 | 4 | 3 | 0 |

Angka ini adalah titik awal yang umum dipakai di industri. Gunakan data change rate riil dari hasil monitoring bila tersedia.

## Metodologi

- `S` = ukuran sumber × (1 + growth)^tahun
- Full `F` = S × (1 − data reduction); incremental harian `I` = F × change rate
- Jumlah restore point harian yang tersimpan `P`:
  - Forever forward incremental: retensi + 1
  - Forward incremental dengan weekly full: retensi + 6
  - Hardened Repository: max(retensi, immutability) + 7 (wajib periodic full; seluruh chain ikut terkunci)
  - Object Storage: max(retensi, immutability) + 10 (block generation)
- Short-term:
  - Fast Clone + synthetic full: F + (P − 1) × I + (K − 1) × 1% F, dengan K = ⌈P / 7⌉
  - Tanpa Fast Clone atau active full: K × F + (P − K) × I
  - Forever incremental: F + (P − 1) × I
- GFS di luar jendela harian disusun sebagai timeline restore point. Dengan Fast Clone/object storage, tiap titik menyimpan blok unik terhadap titik sebelumnya:
  - Realistis (hotspot): `F × (1 − (1 − d)^√N)`
  - Konservatif (acak): `F × (1 − (1 − d)^N)`
  - N = jarak hari antar titik. Tanpa Fast Clone atau dengan active full, tiap titik GFS = F.
- Log database = S × log% × (1 − reduction) × P
- Kebutuhan akhir = total × (1 + buffer)

## Batasan

- Sizing server dan model appliance adalah perkiraan dari guideline publik vendor; gunakan sizing tool resmi vendor untuk angka final.
- Menghitung satu repository per skenario. Backup copy/offsite (aturan 3-2-1-1-0) perlu dihitung terpisah.
- Overhead RAID/filesystem tidak dihitung. Kapasitas di atas adalah kapasitas usable.
- Hasil adalah estimasi perencanaan, bukan pengganti sizing resmi vendor.
