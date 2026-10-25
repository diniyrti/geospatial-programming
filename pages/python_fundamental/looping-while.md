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
title: Looping dan Conditional Statement
authors:
  - name: Diniyarti
---

# Looping dan Conditional Statement

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

## Pendahuluan

Bayangkan memegang daftar 500 titik koordinat hasil survei. Memeriksa titik satu per satu secara manual tentu tidak masuk akal. Di sinilah dua konsep ini berperan:

- **Kondisional** (`if`) membuat program bisa **memutuskan**: titik ini di utara atau selatan khatulistiwa? Ketinggiannya masuk kelas apa? Koordinatnya valid atau tidak?
- **Perulangan** (`for` dan `while`) membuat program bisa **mengulang** pekerjaan yang sama untuk banyak data tanpa menulis kode berulang kali.

Keduanya hampir selalu dipakai bersama, dan menjadi dasar otomatisasi pengolahan data spasial.

```{note}
Koordinat kota di halaman ini adalah perkiraan untuk keperluan belajar. Untuk real analisis, gunakan data dari sumber resmi.
```

## Kondisional: Membuat Keputusan

### Bentuk Dasar `if`

Blok `if` hanya dijalankan kalau kondisinya benar (`True`). Perhatikan dua hal penting: ada tanda titik dua (`:`) di akhir baris, dan baris di bawahnya **menjorok ke dalam** (indentasi). Indentasi inilah yang memberi tahu Python mana kode yang menjadi bagian dari `if`.

```{code-cell} ipython3
ketinggian_m = 12

if ketinggian_m < 50:
    print("Dataran rendah")
```

Jika ada beberapa kemungkinan, tambahkan `elif` (singkatan dari "else if") dan `else` sebagai pilihan terakhir. Python memeriksa dari atas ke bawah dan berhenti di kondisi pertama yang benar.

```{code-cell} ipython3
ketinggian_m = 780

if ketinggian_m < 50:
    print("Dataran rendah")
elif ketinggian_m < 500:
    print("Dataran sedang")
else:
    print("Dataran tinggi")
```

Batas 50 dan 500 di atas hanya contoh sederhana, bukan klasifikasi baku.

### Operator Perbandingan

| Operator | Arti |
|---|---|
| `==` | sama dengan |
| `!=` | tidak sama dengan |
| `<`, `>` | kurang dari, lebih dari |
| `<=`, `>=` | kurang dari atau sama dengan, lebih dari atau sama dengan |

```{warning}
Tanda `=` dipakai untuk **menyimpan** nilai ke variabel, sedangkan `==` dipakai untuk **membandingkan**. Menukar keduanya adalah kesalahan yang sangat umum.
```

### Operator Logika

Beberapa kondisi bisa digabung dengan `and` (keduanya harus benar), `or` (salah satu cukup), dan `not` (membalik hasil).

```{code-cell} ipython3
lat = -5.1477
lon = 119.4327

# Perkiraan kasar kotak wilayah Indonesia
if lat >= -11 and lat <= 6 and lon >= 95 and lon <= 141:
    print("Koordinat berada di sekitar wilayah Indonesia")
else:
    print("Koordinat di luar wilayah Indonesia")
```

Python mengizinkan penulisan perbandingan berantai yang lebih ringkas dan mudah dibaca:

```{code-cell} ipython3
if -11 <= lat <= 6 and 95 <= lon <= 141:
    print("Koordinat berada di sekitar wilayah Indonesia")
```

## Perulangan `for`: Mengulang pada Kumpulan Data

Perulangan `for` mengambil isi sebuah kumpulan data satu per satu, lalu menjalankan blok kode yang menjorok untuk tiap isi itu.

```{code-cell} ipython3
lokasi = ["Pantai Losari", "Benteng Rotterdam", "Universitas Hasanuddin"]

for nama in lokasi:
    print("Titik survei:", nama) #print(f"Titik survei: {nama}")
```

Variabel `nama` bebas diberi nama apa saja. Pada setiap putaran, ia berisi satu elemen list.

### Menghitung dengan `range()`

Untuk mengulang sejumlah kali tertentu, atau menghitung angka berurutan, gunakan `range()`.

```{code-cell} ipython3
for i in range(3):
    print("Putaran ke", i)
```

`range(3)` menghasilkan 0, 1, 2 (berhenti **sebelum** 3). Awal dan akhir juga bisa ditentukan: `range(1, 4)` menghasilkan 1, 2, 3.

### Mengakumulasi Hasil

Pola yang sangat sering muncul: siapkan variabel awal, lalu perbarui di setiap putaran.

```{code-cell} ipython3
ketinggian_m = [3.0, 5.5, 12.0, 18.5, 30.0]   # data contoh

total = 0
for tinggi in ketinggian_m:
    total += tinggi

print("Total:", total)
print("Rata-rata:", total / len(ketinggian_m))
```

### Memberi Nomor dengan `enumerate()`

Jika nomor urut dibutuhkan bersama isinya, `enumerate()` lebih praktis daripada mengurutkan penghitung secara manual. Argumen `start=1` membuat penomoran dimulai dari 1.

```{code-cell} ipython3
for nomor, nama in enumerate(lokasi, start=1):
    print(f"{nomor}. {nama}")
```

### Membongkar Tuple di dalam Perulangan

Jika isi list berupa tuple koordinat, pembongkaran (*unpacking*) bisa dilakukan langsung di baris `for`.

```{code-cell} ipython3
titik = [
    (-5.1436, 119.4075),   # Pantai Losari
    (-5.1344, 119.4059),   # Benteng Rotterdam
    (-5.1330, 119.4880),   # Universitas Hasanuddin
]

for lat, lon in titik:
    print(f"Lintang {lat}, Bujur {lon}")
```

### Perulangan pada Dictionary

Dictionary bisa diulang lewat method `items()`, yang memberi pasangan kunci dan nilai sekaligus.

```{code-cell} ipython3
data_titik = {"nama": "Pantai Losari", "kota": "Makassar", "ketinggian_m": 3.0}

for kunci, nilai in data_titik.items():
    print(f"{kunci}: {nilai}")
```

## Perulangan `while`: Mengulang Selama Kondisi Benar

`while` dipakai ketika jumlah data tidak diketahui di awal, dan pengulangan berhenti saat suatu kondisi tidak lagi terpenuhi.

Contoh: sebuah citra beresolusi 100 meter diperhalus berulang kali (resolusi dibagi dua) sampai mencapai 10 meter atau lebih halus. Berapa kali pengulangannya?

```{code-cell} ipython3
resolusi_m = 100
putaran = 0

while resolusi_m > 10:
    resolusi_m = resolusi_m / 2
    putaran += 1
    print(f"Putaran {putaran}: resolusi {resolusi_m} m")

print("Selesai setelah", putaran, "putaran")
```

```{warning}
Pada `while`, pastikan isi perulangan mengubah sesuatu yang akhirnya membuat kondisi menjadi salah. Pada contoh di atas, `resolusi_m` terus mengecil. Jika tidak ada yang berubah, perulangan tidak akan pernah berhenti (*infinite loop*). Jika terjadi di notebook, hentikan dengan tombol Interrupt.
```

## Mengendalikan Perulangan: `break` dan `continue`

- `break` menghentikan perulangan seketika.
- `continue` melewati sisa putaran saat ini, lalu lanjut ke putaran berikutnya.

Contoh berikut memakai data beberapa kota di Indonesia (koordinat perkiraan).

```{code-cell} ipython3
kota = [
    {"nama": "Makassar",   "lat": -5.1477, "lon": 119.4327},
    {"nama": "Medan",      "lat": 3.5952,  "lon": 98.6722},
    {"nama": "Pontianak",  "lat": -0.0263, "lon": 109.3425},
    {"nama": "Jayapura",   "lat": -2.5337, "lon": 140.7181},
    {"nama": "Banda Aceh", "lat": 5.5483,  "lon": 95.3238},
]
```

Mencari kota pertama yang bujurnya lebih dari 130, lalu berhenti:

```{code-cell} ipython3
for k in kota:
    if k["lon"] > 130:
        print("Ditemukan:", k["nama"])
        break
```

Menampilkan hanya kota di selatan khatulistiwa, dengan melewati yang di utara:

```{code-cell} ipython3
for k in kota:
    if k["lat"] > 0:
        continue
    print(k["nama"], "ada di selatan khatulistiwa")
```

Khusus Pontianak, lintangnya sangat dekat dengan 0, sehingga menarik untuk dicermati hasilnya.

## Menggabungkan Perulangan dan Kondisional

Gabungan keduanya adalah inti otomatisasi data. Pola yang paling sering dipakai ada tiga: menyaring, menghitung, dan memvalidasi.

### Menyaring: Membuat List Baru dari yang Memenuhi Syarat

```{code-cell} ipython3
selatan = []

for k in kota:
    if k["lat"] < 0:
        selatan.append(k["nama"])

print("Kota di belahan selatan:", selatan)
```

### Menghitung: Berapa yang Memenuhi Syarat

```{code-cell} ipython3
jumlah_utara = 0
jumlah_selatan = 0

for k in kota:
    if k["lat"] >= 0:
        jumlah_utara += 1
    else:
        jumlah_selatan += 1

print("Utara  :", jumlah_utara)
print("Selatan:", jumlah_selatan)
```

### Memvalidasi: Mendeteksi Koordinat yang Salah

Lintang harus berada di antara -90 dan 90, bujur di antara -180 dan 180. Kesalahan yang sering terjadi adalah lintang dan bujur **tertukar**. Contoh berikut mendeteksi keduanya.

```{code-cell} ipython3
data_uji = [
    (-5.1477, 119.4327),   # benar
    (3.5952, 98.6722),     # benar
    (119.43, -5.15),       # lintang dan bujur tertukar
    (0.0, 200.0),          # bujur di luar rentang
]

valid = []

for lat, lon in data_uji:
    if -90 <= lat <= 90 and -180 <= lon <= 180:
        valid.append((lat, lon))
    elif -90 <= lon <= 90 and -180 <= lat <= 180:
        print(f"Tidak valid: ({lat}, {lon}), kemungkinan lintang dan bujur tertukar")
    else:
        print(f"Tidak valid: ({lat}, {lon}), di luar rentang koordinat")

print("Jumlah titik valid:", len(valid))
```

Untuk setiap titik, Python memeriksa kondisi dari atas ke bawah. Titik ketiga gagal pada `if` pertama (lintang 119,43 melebihi 90), tetapi lolos di `elif` karena jika ditukar, nilainya masuk akal.

### Rangkuman Statistik Sederhana

```{code-cell} ipython3
print("Ringkasan kota")
print("-" * 40)

for k in kota:
    belahan = "Utara" if k["lat"] >= 0 else "Selatan"
    print(f"{k['nama']:<12} {k['lat']:>8.4f} {k['lon']:>9.4f}  {belahan}")
```

Bentuk `A if kondisi else B` pada baris `belahan` adalah cara ringkas menulis `if` sederhana dalam satu baris. Format `:<12` dan `:>8.4f` mengatur lebar kolom supaya tampilan rapi.

## Panduan Memilih

| Situasi | Gunakan |
|---|---|
| Memproses setiap isi sebuah kumpulan data | `for` |
| Mengulang sejumlah kali tertentu | `for` dengan `range()` |
| Mengulang sampai syarat tercapai, jumlah looping tidak pasti | `while` |
| Memutuskan tindakan berdasarkan nilai data | `if`, `elif`, `else` |
| Berhenti lebih awal atau melewati satu putaran | `break` atau `continue` |

## Note Penting

- Kondisional membuat program bisa memutuskan, perulangan membuat program bisa mengulang. Keduanya paling kuat bila digabung.
- Indentasi menentukan isi blok. Titik dua (`:`) wajib ada di akhir baris `if`, `elif`, `else`, `for`, dan `while`.
- `=` untuk menyimpan nilai, `==` untuk membandingkan.
- `for` cocok untuk kumpulan data yang sudah ada, `while` cocok ketika berhenti bergantung pada kondisi.
- Pada `while`, pastikan ada sesuatu yang berubah agar kondisi akhirnya salah, supaya tidak terjadi perulangan tanpa akhir.
- Tiga pola gabungan yang paling sering dipakai: menyaring, menghitung, dan memvalidasi.
- Validasi koordinat sebaiknya mencakup pemeriksaan rentang dan kemungkinan lintang-bujur tertukar.