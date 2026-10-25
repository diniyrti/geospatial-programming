---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
kernelspec:
  display_name: Python 3
  language: python
  name: python3
title: String Operation
authors:
  - name: Diniyarti
---

# String Operation

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

## Pendahuluan

Tidak semua data spasial berbentuk angka rapi. Sangat banyak yang datang sebagai **teks**: nama jalan yang penulisannya tidak seragam, koordinat yang tersimpan di kolom teks, alamat berkas, atau baris catatan dari alat GPS. Sebelum data seperti ini bisa dianalisis atau dipetakan, teksnya harus dibersihkan, dipisahkan, dan disusun ulang.

Python menyediakan banyak alat bawaan untuk mengolah teks (string). Halaman ini memperkenalkan yang paling sering dipakai dalam pekerjaan geospasial, dengan alur dari yang paling dasar sampai satu kasus praktis di akhir.

```{note}
Koordinat di halaman ini adalah perkiraan untuk keperluan belajar. Angka lain, seperti jumlah penduduk, adalah contoh. Untuk real analisis, gunakan data dari sumber resmi.
```

## Teks sebagai Deretan Karakter

String adalah deretan karakter yang urutannya tetap. Karena itu, string punya sifat yang mirip list: bisa dihitung panjangnya, diambil per posisi, dan slicing.

```{code-cell} ipython3
nama = "Makassar"

print(len(nama))      # jumlah karakter
print(nama[0])        # karakter pertama
print(nama[-1])       # karakter terakhir
print(nama[:4])       # empat karakter pertama
print(nama[4:])       # dari indeks 4 sampai akhir
```

Seperti pada list, indeks dimulai dari 0. Spasi dan tanda baca juga dihitung sebagai karakter.

Berbeda dengan list, string **tidak bisa diubah per karakter**. Perintah `nama[0] = "m"` menghasilkan `TypeError`. Jika ingin teks yang berbeda, buat string baru dari yang lama.

## Menggabung Teks

### Dengan Operator `+`

Operator `+` menyambung dua teks. Hanya teks yang bisa disambung dengan teks, jadi angka harus diubah dulu dengan `str()`.

```{code-cell} ipython3
kota = "Makassar"
provinsi = "Sulawesi Selatan"

lengkap = kota + ", " + provinsi
print(lengkap)

latitude = -5.1436
print("Latitude: " + str(latitude))
```

Menyambung teks dengan angka tanpa `str()` (misalnya `"Latitude: " + latitude`) menghasilkan `TypeError`.

Penyambungan juga dipakai untuk menyusun alamat berkas:

```{code-cell} ipython3
folder = "data_spasial"
berkas = "titik_survei.csv"

alamat = folder + "/" + berkas
print(alamat)
```

### Mengulang Teks dengan `*`

```{code-cell} ipython3
garis = "=" * 30
print(garis)
print("LAPORAN TITIK SURVEI")
print(garis)
```

### Menggabung Banyak Teks dengan `join()`

Untuk menyambung isi sebuah list menjadi satu teks, `join()` lebih praktis daripada rangkaian `+`. Teks pemisah ditulis di depan titik.

```{code-cell} ipython3
kecamatan = ["Tamalanrea", "Rappocini", "Panakkukang"]

print(", ".join(kecamatan))
print("/".join(["data", "raster", "dem.tif"]))
```

## Membersihkan Teks

Data hasil input manual atau unduhan jarang rapi. Empat jenis masalah yang paling sering muncul adalah spasi berlebih, huruf besar kecil tidak seragam, penulisan yang berbeda untuk hal yang sama, dan awalan yang tidak diperlukan.

### Spasi Berlebih

`strip()` membuang spasi di kedua ujung. `lstrip()` hanya di kiri dan `rstrip()` hanya di kanan.

```{code-cell} ipython3
mentah = "   Pantai Losari   "

print(f"[{mentah}]")
print(f"[{mentah.strip()}]")
print(f"[{mentah.lstrip()}]")
print(f"[{mentah.rstrip()}]")
```

Tanda kurung siku di dalam f-string di atas hanya membantu melihat batas teks, supaya spasi terlihat jelas.

### Huruf Besar dan Kecil

```{code-cell} ipython3
nama_jalan = "jalan PERINTIS kemerdekaan"

print(nama_jalan.upper())
print(nama_jalan.lower())
print(nama_jalan.title())        # tiap kata berawalan kapital
print(nama_jalan.capitalize())   # hanya huruf pertama kalimat
```

`lower()` sering dipakai sebelum membandingkan dua teks, supaya `"MAKASSAR"` dan `"Makassar"` dianggap sama.

### Mengganti Bagian Teks

`replace()` mengganti **semua** kemunculan teks lama dengan teks baru.

```{code-cell} ipython3
singkat = "Jl. Perintis Kemerdekaan"
print(singkat.replace("Jl.", "Jalan"))

alamat_ganda = "data/raw/raw/titik.csv"
print(alamat_ganda.replace("raw/", ""))
```

### Merangkai Beberapa Method

Method teks bisa dirangkai berurutan, dikerjakan dari kiri ke kanan.

```{code-cell} ipython3
mentah = "  kecamatan TAMALANREA "

rapi = mentah.strip().lower().replace("kecamatan ", "").title()
print(rapi)
```

Teks asli tidak berubah oleh method apa pun. Setiap method menghasilkan teks baru yang perlu disimpan ke variabel.

## Memecah Teks

`split()` memecah teks menjadi list, memakai pemisah yang ditentukan. Ini adalah alat utama untuk membaca data yang tersimpan sebagai satu baris teks.

```{code-cell} ipython3
baris = "Pantai Losari;Makassar;Sulawesi Selatan"

bagian = baris.split(";")
print(bagian)

nama, kota, provinsi = bagian
print(nama)
```

Jika `split()` dipanggil tanpa pemisah, teks dipecah berdasarkan spasi, dan spasi berlebih otomatis diabaikan.

```{code-cell} ipython3
print("Jalan   Perintis  Kemerdekaan".split())
```

### Membaca Koordinat dari Teks

Koordinat dari berkas teks umumnya berbentuk satu string. Caranya: pecah dulu, lalu ubah tiap bagian menjadi angka dengan `float()`.

```{code-cell} ipython3
teks = "-5.1436,119.4075"

lat_teks, lon_teks = teks.split(",")
lat = float(lat_teks)
lon = float(lon_teks)

print(lat, lon)
print(type(lat))
```

### Waspada: Koma Desimal

Di Indonesia, banyak data (terutama dari spreadsheet) memakai **koma** sebagai tanda desimal. Python hanya mengenal titik, sehingga `float("-5,1436")` menghasilkan `ValueError`. Solusinya mengganti koma menjadi titik sebelum diubah:

```{code-cell} ipython3
lat = float("-5,1436".replace(",", "."))
print(lat)
```

Kalau teks itu juga memakai koma sebagai pemisah antar kolom, pilih pemisah lain untuk `split()` (misalnya titik koma) agar tidak bentrok.

### Memecah Alamat Berkas

```{code-cell} ipython3
alamat = "data/raster/makassar/dem.tif"

komponen = alamat.split("/")
print(komponen)
print(komponen[-1])                 # nama berkas
print(komponen[-1].split(".")[-1])  # ekstensi berkas
```

## Memeriksa Isi Teks

Beberapa pemeriksaan sederhana membantu menyaring data sebelum diproses.

```{code-cell} ipython3
print("Makassar".isalpha())    # hanya huruf?
print("90245".isdigit())       # hanya angka?
print("-5.1436".isdigit())     # False: tanda minus dan titik bukan angka
```

Perhatikan hasil terakhir. `isdigit()` tidak cocok untuk memeriksa angka desimal atau negatif, jadi koordinat sebaiknya divalidasi dengan `float()`, bukan dengan `isdigit()`.

```{code-cell} ipython3
berkas = "dem_makassar.tif"

print(berkas.startswith("dem"))     # diawali "dem"?
print(berkas.endswith(".tif"))      # berakhiran ".tif"?
print("makassar" in berkas)         # mengandung kata "makassar"?
```

Untuk mencari **posisi** sebuah kata, gunakan `find()`. Hasilnya indeks kemunculan pertama, atau `-1` jika tidak ditemukan.

```{code-cell} ipython3
teks = "Pantai Losari"

print(teks.find("Losari"))
print(teks.find("Rotterdam"))
```

## Memformat Teks

### F-String

F-string adalah cara paling praktis dan paling mudah dibaca untuk menyisipkan nilai variabel ke dalam teks. Teks diawali huruf `f`, dan variabel ditulis di dalam `{}`.

```{code-cell} ipython3
nama_tempat = "Pantai Losari"
lat = -5.1436
lon = 119.4075
ketinggian = 3

print(f"{nama_tempat} berada di ({lat}, {lon}) dengan ketinggian {ketinggian} m")
```

### Mengatur Tampilan Angka

Setelah tanda titik dua di dalam `{}`, tampilan angka bisa diatur.

```{code-cell} ipython3
lat_presisi = -5.143627

print(f"{lat_presisi:.2f}")    # 2 angka di belakang koma
print(f"{lat_presisi:.4f}")    # 4 angka di belakang koma

penduduk = 1423877             # angka contoh
print(f"{penduduk:,}")         # pemisah ribuan

persen = 0.2345
print(f"{persen:.1%}")         # tampil sebagai persen
```

Pemisah ribuan bawaan Python adalah koma (gaya Inggris). Untuk gaya Indonesia yang memakai titik, ganti hasilnya:

```{code-cell} ipython3
print(f"{penduduk:,}".replace(",", "."))
```

### Cara Lama: `format()`

Pada kode lama, sering dijumpai metode `.format()`. Hasilnya sama, hanya penulisannya berbeda.

```{code-cell} ipython3
print("{} berada di ({}, {})".format(nama_tempat, lat, lon))
print("{nama} berada di ({a}, {b})".format(nama=nama_tempat, a=lat, b=lon))
```

Untuk kode baru, f-string lebih disarankan.

## Contoh Penerapan

### Membuat Nama Berkas Otomatis

```{code-cell} ipython3
kecamatan = "Tamalanrea"
lat = -5.1436

nama_berkas = f"survei_{kecamatan.lower()}_{abs(lat):.4f}S.csv"
print(nama_berkas)
```

### Membuat Teks WKT

WKT (Well-Known Text) adalah format teks standar untuk menuliskan objek geometri. Hal yang sering membuat bingung: urutan koordinatnya **bujur dulu, baru lintang** (x lalu y).

```{code-cell} ipython3
lat = -5.1436
lon = 119.4075

wkt = f"POINT({lon} {lat})"
print(wkt)
```

## Latihan Gabungan: Membersihkan Satu Baris Data

Kasus berikut menggabungkan hampir semua teknik di atas. Satu baris data berasal dari spreadsheet dengan spasi berlebih, huruf tidak seragam, dan koma desimal.

```{code-cell} ipython3
baris_mentah = "  PANTAI   losari ; -5,1436 ; 119,4075  "

# 1. Pecah menjadi tiga kolom
nama_mentah, lat_mentah, lon_mentah = baris_mentah.split(";")

# 2. Rapikan nama: buang spasi berlebih, lalu kapital tiap kata
nama = " ".join(nama_mentah.split()).title()

# 3. Ubah koordinat menjadi angka
lat = float(lat_mentah.strip().replace(",", "."))
lon = float(lon_mentah.strip().replace(",", "."))

print(f"{nama}: POINT({lon} {lat})")
```

Pada langkah 2, `split()` tanpa pemisah membuang semua spasi berlebih, lalu `" ".join(...)` menyambungnya kembali dengan satu spasi. Teknik ini praktis untuk menyeragamkan nama tempat.

## Panduan Memilih Operasi

| Tujuan | Gunakan |
|---|---|
| Menyambung teks atau list teks | `+` atau `join()` |
| Membersihkan dan menyeragamkan | `strip()`, `lower()`, `title()`, `replace()` |
| Memecah teks terstruktur | `split()` |
| Memeriksa isi teks | `in`, `startswith()`, `endswith()`, `find()` |
| Menyusun teks berisi variabel | f-string |

## Note Penting

- String adalah deretan karakter yang bisa diakses per posisi, tetapi tidak bisa diubah per karakter.
- Method teks selalu menghasilkan teks baru. Simpan hasilnya ke variabel.
- `split()` dan `join()` adalah pasangan: memecah teks menjadi list, dan sebaliknya.
- Data Indonesia sering memakai koma desimal. Ganti dengan titik sebelum memakai `float()`.
- `isdigit()` tidak cocok untuk angka desimal atau negatif. Validasi koordinat dengan `float()`.
- F-string menyederhanakan penyusunan teks, termasuk mengatur jumlah desimal dan pemisah ribuan.
- Dalam format WKT, urutan koordinat adalah bujur lalu lintang.