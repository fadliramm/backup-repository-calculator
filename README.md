# Backup Repository Calculator

Kalkulator berbasis web (satu file HTML, tanpa dependensi) untuk menghitung kebutuhan kapasitas repository backup, membandingkan skenario **Non-Immutable** dan **Immutable** secara berdampingan.

## Cara pakai

Buka `index.html` di browser. Tidak perlu server atau instalasi.

1. Atur **Pengaturan Global**: pertumbuhan data per tahun, lama proyeksi, free space buffer, dan pola perubahan data.
2. Pilih **Repository**:
   - Non-Immutable: Windows ReFS (Fast Clone), Linux XFS reflink (Fast Clone), atau NTFS/ext4 (tanpa Fast Clone), plus metode backup.
   - Immutable: Linux Hardened Repository (XFS) atau Object Storage (S3 Object Lock), plus periode immutability.
3. Tambahkan **Workload**. Saat tipe workload dipilih, parameter terisi otomatis dari best practice dan masih bisa diubah.
4. Lihat hasil: kebutuhan kapasitas kedua skenario, proyeksi per tahun, rincian per workload, dan rekomendasi. Hasil bisa diekspor ke CSV atau dicetak ke PDF.

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

- Menghitung satu repository per skenario. Backup copy/offsite (aturan 3-2-1-1-0) perlu dihitung terpisah.
- Overhead RAID/filesystem tidak dihitung. Kapasitas di atas adalah kapasitas usable.
- Hasil adalah estimasi perencanaan, bukan pengganti sizing resmi vendor.
