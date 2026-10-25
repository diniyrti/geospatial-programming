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
title: Dasar-Dasar Menggunakan Numpy
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

Data spasial sering datang dalam jumlah besar: ribuan titik koordinat dari GPS, deretan pengukuran suhu, atau jutaan piksel pada sebuah citra satelit. List dan perulangan `for` di Python bisa menanganinya, tetapi menjadi lambat dan merepotkan ketika datanya membesar.

**NumPy** (*Numerical Python*) adalah pustaka dasar untuk komputasi numerik di Python. Inti NumPy adalah **array**, yaitu kumpulan angka yang disimpan rapat dan bisa dihitung sekaligus tanpa perulangan. Banyak pustaka geospasial lain bergantung pada NumPy, sehingga memahaminya adalah bekal penting sebelum mengolah raster atau data vektor.

```{note}
Koordinat kota dan titik di halaman ini adalah perkiraan untuk keperluan belajar. Data ketinggian, suhu, dan pengukuran lainnya adalah contoh fiktif. Untuk real analisis, gunakan data dari sumber resmi.
```

## Mengapa Tidak Pakai List Saja?

Perhatikan perbedaan perilaku berikut. Mengalikan list dengan 2 **menggandakan isinya**, sedangkan mengalikan array dengan 2 **mengalikan tiap angkanya**.

```{code-cell} ipython3
import numpy as np

list_tinggi = [3.0, 5.5, 12.0]
print(list_tinggi * 2)

array_tinggi = np.array(list_tinggi)
print(array_tinggi * 2)
```

Kebiasaan menulis `import numpy as np` dipakai di hampir semua kode NumPy, sehingga `np` menjadi singkatan yang umum dikenal.

Empat keunggulan array dibanding list untuk data numerik:

- **Cepat.** Perhitungan dijalankan dalam kode yang sudah dioptimalkan, bukan perulangan Python biasa.
- **Hemat memori.** Semua elemen bertipe sama dan disimpan rapat.
- **Vektorisasi.** Satu perintah bisa berlaku untuk seluruh elemen.
- **Multidimensi.** Cocok untuk tabel angka, grid koordinat, dan citra raster.

## Membuat Array

### Dari List

```{code-cell} ipython3
ketinggian_m = np.array([3.0, 5.5, 12.0, 18.5, 30.0])

print(ketinggian_m)
print("Tipe data :", ketinggian_m.dtype)
print("Bentuk    :", ketinggian_m.shape)
print("Dimensi   :", ketinggian_m.ndim)
print("Jumlah    :", ketinggian_m.size)
```

Empat atribut di atas sering dicek untuk memahami sebuah array: `dtype` (tipe elemen), `shape` (ukuran tiap dimensi), `ndim` (jumlah dimensi), dan `size` (jumlah seluruh elemen).

Satu array hanya punya satu tipe data. Jika dicampur, NumPy menyeragamkannya ke tipe yang paling "luas".

“Paling luas” artinya tipe yang masih bisa menampung semua nilai tanpa ada yang hilang atau rusak. Dalam NumPy urutannya kira-kira:

bool → int → float → teks (string)

Tipe di kanan bisa menyimpan isi tipe di kiri, tapi tidak sebaliknya.

- Angka bulat bisa ditulis sebagai desimal: 3 jadi 3.0, tidak ada informasi yang hilang. Jadi float lebih luas dari int.
- Desimal tidak bisa jadi bulat tanpa kehilangan: 5.5 jadi 5 kehilangan .5. Jadi int lebih sempit.
- Angka apa pun bisa ditulis sebagai teks: 3 jadi "3". Jadi teks yang paling luas, karena bisa menampung semuanya.

```{code-cell} ipython3
campur = np.array([1, 2, 3.5])
print(campur, campur.dtype)
```

### Array 2 Dimensi

List di dalam list menghasilkan array 2 dimensi, yaitu seperti tabel dengan baris dan kolom. Contohnya, tiap baris adalah satu lokasi dan tiap kolom adalah lintang atau bujur.

```{code-cell} ipython3
koordinat = np.array([
    [-5.1436, 119.4075],   # Pantai Losari
    [-5.1344, 119.4059],   # Benteng Rotterdam
    [-5.1330, 119.4880],   # Kampus Unhas
])

print(koordinat)
print("Bentuk:", koordinat.shape)   # (baris, kolom)
print("Jumlah lokasi:", koordinat.shape[0])
```

Bentuk `(3, 2)` dibaca "3 baris, 2 kolom".

### Array dengan Pola Tertentu

Sering kali array perlu dibuat dulu dengan isi awal tertentu, lalu diisi hasil perhitungan.

```{code-cell} ipython3
nol = np.zeros((2, 3))               # semua 0
satu = np.ones((2, 3))               # semua 1
nodata = np.full((2, 3), -9999)      # semua bernilai -9999

print(nol)
print(satu)
print(nodata)
```

Untuk deretan angka teratur ada `arange` dan `linspace`:

```{code-cell} ipython3
tahun = np.arange(2015, 2026)        # 2015 sampai 2025 (batas akhir tidak ikut)
lintang = np.arange(-10, 11, 5)      # dari -10 sampai 10, loncat 5
bujur = np.linspace(95, 141, 5)      # 5 angka berjarak sama, batas akhir ikut

print(tahun)
print(lintang)
print(bujur)
```

Perbedaannya: `arange(awal, akhir, langkah)` menentukan **besar langkah** dan batas akhirnya tidak ikut, sedangkan `linspace(awal, akhir, jumlah)` menentukan **jumlah titik** dan batas akhirnya ikut.

Tipe elemen juga bisa ditentukan sendiri, misalnya untuk penanda benar-salah:

```{code-cell} ipython3
valid = np.ones(5, dtype=bool)
print(valid)
```

## Menghitung Seluruh Array Sekaligus

Inilah kekuatan utama NumPy. Operasi aritmetika berlaku untuk **tiap elemen** tanpa perlu perulangan. Cara ini disebut **vektorisasi**.

```{code-cell} ipython3
ketinggian_m = np.array([3.0, 5.5, 12.0, 18.5, 30.0])

print(ketinggian_m + 10)

ketinggian_ft = ketinggian_m * 3.28084    # meter ke kaki
print(ketinggian_ft.round(1))

suhu_c = np.array([26.5, 28.0, 31.2, 33.4])
suhu_f = suhu_c * 9 / 5 + 32
print(suhu_f.round(1))
```

### Operasi Antar Array

Dua array berukuran sama dihitung elemen demi elemen. Misalnya membandingkan ketinggian hasil pengukuran dua tahun:

```{code-cell} ipython3
tinggi_2015 = np.array([3.0, 5.5, 12.0])
tinggi_2025 = np.array([2.8, 5.4, 12.3])

perubahan = tinggi_2025 - tinggi_2015
print(perubahan.round(2))

print(tinggi_2025 > tinggi_2015)    # hasilnya array benar-salah
```

### Broadcasting: Array Kecil Dipakai Berulang pada Array Besar

Bagaimana jika ukurannya berbeda? NumPy otomatis **mengulang** array yang
lebih kecil untuk setiap baris array yang lebih besar, selama ukuran
dimensi paling belakang cocok. Proses ini disebut *broadcasting*.

Syaratnya: ukuran dimensi paling belakang harus cocok misal:
- Koordinat berbentuk (3, 2) → dimensi paling belakang = 2
- Titik_acuan berbentuk (2,) → dimensi paling belakang = 2

```{code-cell} ipython3

koordinat = np.array([
    [-5.1436, 119.4075],   # Pantai Losari
    [-5.1344, 119.4059],   # Benteng Rotterdam
    [-5.1330, 119.4880],   # Kampus Unhas
])
titik_acuan = np.array([-5.1436, 119.4075])   # Pantai Losari
selisih = koordinat - titik_acuan

print(selisih.round(4))
```

Array `koordinat` berbentuk `(3, 2)` dan `titik_acuan` berbentuk `(2,)`. NumPy mengulang `titik_acuan` untuk setiap baris, sehingga hasilnya berbentuk `(3, 2)`. 

Jika ukuran nya tidak cocok, misalnya menjumlahkan array berisi 3 angka dengan array berisi 2 angka, NumPy akan menghasilkan `ValueError`.

## Mengakses Isi Array

### Indeks

Seperti list, indeks dimulai dari 0 dan indeks negatif menghitung dari belakang. Pada array 2 dimensi, tulis `[baris, kolom]`.

```{code-cell} ipython3
suhu = np.array([26.5, 28.0, 31.2, 33.4, 29.8])

print(suhu[0])
print(suhu[-1])

grid = np.array([
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
])

print(grid[0, 1])      # baris pertama, kolom kedua
print(grid[-1, -1])    # baris terakhir, kolom terakhir
```

### Slicing

Pola `[awal:akhir]` berlaku untuk tiap dimensi. Pada array 2 dimensi, bagian sebelum koma mengatur baris dan bagian sesudahnya mengatur kolom.

```{code-cell} ipython3
print(suhu[1:4])
print(suhu[2:])

print(grid[:2, :])     # dua baris pertama, semua kolom
print(grid[1:, :2])    # dari baris kedua, dua kolom pertama
print(grid[:, 0])      # seluruh baris, hanya kolom pertama
print(grid[1])         # baris kedua utuh
```

### Menyaring dengan Syarat (Boolean Indexing)

Perbandingan pada array menghasilkan array benar-salah. Array itu bisa dipakai untuk memilih elemen yang memenuhi syarat.

```{code-cell} ipython3
kondisi = suhu > 30
print(kondisi)
print(suhu[kondisi])

print(suhu[(suhu > 27) & (suhu < 33)])
```

```{warning}
Untuk menggabungkan syarat pada array, gunakan `&` (dan), `|` (atau), dan `~` (bukan), dengan tiap syarat dibungkus kurung. Kata `and` dan `or` tidak bisa dipakai pada array dan akan menghasilkan error.
```

Fungsi `np.where` membuat nilai baru berdasarkan syarat, dengan pola `np.where(syarat, nilai_jika_benar, nilai_jika_salah)`.

```{code-cell} ipython3
kelas = np.where(suhu > 30, "panas", "normal")
print(kelas)

print(np.where(suhu > 30)[0])    # tanpa pilihan nilai: hanya posisi yang memenuhi syarat
```

### Mengubah Isi Array

```{code-cell} ipython3
arr = np.array([10, 20, 30, 40, 50])

arr[1] = 25               # ubah satu elemen
arr[2:4] = [35, 45]       # ubah beberapa elemen
arr[arr > 40] = 40        # ubah semua yang memenuhi syarat

print(arr)
```

### Hati-hati: Irisan Adalah Tampilan, Bukan Salinan

Sama seperti `b = a` pada list, irisan array tidak membuat salinan baru, melainkan **tampilan** (*view*) ke data yang sama. Mengubah irisan berarti mengubah array aslinya.

```{code-cell} ipython3
a = np.array([1, 2, 3, 4])

b = a[:2]
b[0] = 99
print(a)               # ikut berubah

c = a[:2].copy()       # salinan sungguhan
c[0] = 0
print(a)               # tidak berubah
```

## Fungsi Matematika

NumPy menyediakan banyak fungsi matematika yang langsung bekerja pada seluruh array.

```{code-cell} ipython3
luas_m2 = np.array([100, 400, 900, 1600, 2500])
print(np.sqrt(luas_m2))          # panjang sisi persegi dari luasnya

sudut_derajat = np.array([0, 30, 45, 60, 90])
sudut_radian = np.radians(sudut_derajat)    # sin dan cos bekerja dalam radian
print(np.sin(sudut_radian).round(3))
print(np.cos(sudut_radian).round(3))

penduduk = np.array([100_000, 500_000, 1_000_000, 5_000_000, 10_000_000])   # angka contoh
print(np.log10(penduduk).round(2))
```

Transformasi logaritma sering dipakai untuk data yang sebarannya sangat timpang, seperti jumlah penduduk antar kota.

## Statistik

```{code-cell} ipython3
profil = np.array([12, 18, 25, 31, 28, 22, 15, 9, 6, 4])   # ketinggian sepanjang jalur (contoh)

print("Rata-rata :", np.mean(profil))
print("Median    :", np.median(profil))
print("Simpangan baku:", np.std(profil).round(2))
print("Terendah  :", np.min(profil))
print("Tertinggi :", np.max(profil))
print("Kuartil   :", np.percentile(profil, [25, 50, 75]))

print("Posisi tertinggi:", np.argmax(profil))
print("Posisi terendah :", np.argmin(profil))
```

Fungsi `argmax()` dan `argmin()` mengembalikan **posisi** atau index nilai terbesar dan terkecil, bukan nilainya. Posisi ini berguna misalnya untuk mencari nama lokasi dengan nilai tertinggi.

### Statistik per Baris atau per Kolom

Pada array 2 dimensi, parameter `axis` menentukan arah perhitungan. `axis=0` menghitung menurun (satu hasil per kolom), sedangkan `axis=1` menghitung mendatar (satu hasil per baris).

```{code-cell} ipython3
suhu_stasiun = np.array([
    [26.5, 27.1, 28.0, 27.4],   # stasiun A, empat hari
    [28.2, 29.0, 30.1, 29.5],   # stasiun B
    [25.1, 25.8, 26.3, 26.0],   # stasiun C
])

print("Rata-rata tiap stasiun:", suhu_stasiun.mean(axis=1).round(2))
print("Rata-rata tiap hari   :", suhu_stasiun.mean(axis=0).round(2))
print("Tertinggi keseluruhan :", suhu_stasiun.max())
```

### Nilai Kosong (NaN) dan Nodata

Data nyata sering memiliki nilai kosong. Dalam NumPy, nilai kosong ditulis `np.nan`. Jika ada satu `nan`, fungsi biasa seperti `mean()` akan menghasilkan `nan`. Gunakan versi `nan` dari fungsi itu untuk mengabaikannya.

```{code-cell} ipython3
data = np.array([3.0, 5.5, np.nan, 18.5, 30.0])

print(np.mean(data))
print(np.nanmean(data))
```

Pada citra raster, nilai kosong biasanya ditandai angka khusus seperti `-9999` (*nodata*). Angka itu harus disingkirkan agar tidak merusak perhitungan.

```{code-cell} ipython3
dem = np.array([
    [12, 15, -9999],
    [18, 22, 25],
    [-9999, 30, 33],
], dtype=float)

valid = dem != -9999
print("Rata-rata ketinggian valid:", dem[valid].mean().round(2))

dem_bersih = np.where(valid, dem, np.nan)
print("Cara lain, dengan nan     :", np.nanmean(dem_bersih).round(2))
print("Piksel di atas 20 m       :", np.sum(dem_bersih > 20))
```

Contoh di atas adalah gambaran kecil bagaimana citra diolah: sebuah raster pada dasarnya adalah array 2 dimensi, dan satu piksel adalah satu elemennya.

## Mengubah Bentuk Array

Data kadang datang dalam bentuk yang tidak sesuai kebutuhan. Misalnya koordinat datang sebagai satu deret panjang `[lintang1, bujur1, lintang2, bujur2, ...]`. Dengan `reshape`, deret itu disusun ulang menjadi pasangan tanpa mengubah isinya.

```{code-cell} ipython3
datar = np.array([-5.1436, 119.4075, -5.1344, 119.4059, -5.1330, 119.4880])

pasangan = datar.reshape(3, 2)
otomatis = datar.reshape(-1, 2)      # -1: biarkan NumPy menghitung sendiri

print(pasangan)
print(pasangan.shape, otomatis.shape)
```

Jumlah elemen sebelum dan sesudah harus sama. Meminta `reshape(4, 2)` pada array berisi 6 elemen akan menghasilkan `ValueError`.

Dua hal terkait yang sering berguna:

```{code-cell} ipython3
print(pasangan.flatten())     # kembali menjadi satu deret

lintang, bujur = pasangan.T   # .T menukar baris dan kolom
print(lintang)
print(bujur)
```

Atribut `.T` (transpos) menukar baris dan kolom, sehingga kolom lintang dan kolom bujur bisa dipisahkan menjadi dua array sekaligus lewat pembongkaran.

## Data Acak untuk Simulasi

Data acak berguna untuk menguji kode atau membuat data contoh. Pendekatan modern memakai *generator* yang dibuat dengan `default_rng`. Angka `seed` membuat hasil acaknya sama setiap kali dijalankan, jadi hasilnya bisa diulang.

```{code-cell} ipython3
rng = np.random.default_rng(seed=42)

lintang_acak = rng.uniform(-11, 6, size=5)             # sebaran merata
bujur_acak = rng.uniform(95, 141, size=5)
suhu_acak = rng.normal(loc=28, scale=1.5, size=8)      # sebaran normal
indeks_acak = rng.integers(0, 100, size=5)             # bilangan bulat acak

print(lintang_acak.round(4))
print(bujur_acak.round(4))
print(suhu_acak.round(1))
print(indeks_acak)
```

Dua hal yang perlu diingat:

`round` hanya mengubah tampilan hasilnya, bukan data aslinya. matriks tetap menyimpan angka lengkap kecuali kamu menyimpan hasil round ke variabel baru.
Hasil round(0) tetap bertipe `float`, jadi muncul 2379. dan bukan 2379. Kalau ingin benar-benar bilangan bulat, tambahkan `.astype(int)`, misalnya `matriks.round(0).astype(int)`.

## Penerapan: Menghitung Jarak Banyak Titik Sekaligus

Pada halaman fungsi, rumus Haversine dipakai untuk dua titik. Dengan NumPy, rumus yang sama bisa menghitung **banyak titik sekaligus** tanpa perulangan, karena `np.sin`, `np.cos`, dan kawan-kawannya bekerja pada seluruh array.

```{code-cell} ipython3
def haversine_np(lat1, lon1, lat2, lon2, radius=6371.0):
    """Jarak lingkaran besar (km). Menerima angka tunggal maupun array."""
    lat1, lon1, lat2, lon2 = map(np.radians, (lat1, lon1, lat2, lon2))
    a = np.sin((lat2 - lat1) / 2) ** 2 + np.cos(lat1) * np.cos(lat2) * np.sin((lon2 - lon1) / 2) ** 2
    return radius * 2 * np.arcsin(np.sqrt(a))
```

Data lima kota (koordinat perkiraan):

```{code-cell} ipython3
nama_kota = np.array(["Makassar", "Medan", "Pontianak", "Jayapura", "Banda Aceh"])
lintang = np.array([-5.1477, 3.5952, -0.0263, -2.5337, 5.5483])
bujur = np.array([119.4327, 98.6722, 109.3425, 140.7181, 95.3238])

# Jarak dari Makassar (kota pertama) ke semua kota
jarak = haversine_np(lintang[0], bujur[0], lintang, bujur)

for nama, j in zip(nama_kota, jarak):
    print(f"{nama:<12} {j:8.1f} km")
```

Fungsi `zip()` memasangkan dua deretan agar bisa diulang bersamaan. Baris pertama bernilai 0 karena jaraknya ke diri sendiri.

Nama kota juga bisa diurutkan dari yang terdekat, atau disaring berdasarkan jarak:

```{code-cell} ipython3
urutan = np.argsort(jarak)       # posisi dari yang terkecil ke terbesar
print(nama_kota[urutan])

print(nama_kota[jarak < 2000])   # kota dalam radius 2000 km
```

### Matriks Jarak Semua Pasangan Kota

Dengan broadcasting, jarak **setiap pasangan** kota bisa dihitung sekali jalan. Triknya: ubah satu array menjadi kolom lewat `[:, np.newaxis]`, sehingga hasilnya berbentuk tabel `5 × 5`.

```{code-cell} ipython3
matriks = haversine_np(lintang[:, np.newaxis], bujur[:, np.newaxis], lintang, bujur)

print(matriks.shape)
print(matriks.round(0))
print("Rata-rata jarak tiap kota:", matriks.mean(axis=1).round(0))
```

Baris ke-*i* dan kolom ke-*j* berisi jarak antara kota ke-*i* dan kota ke-*j*. Diagonalnya nol, dan tabelnya simetris.

## Panduan Singkat

| Kebutuhan | Alat NumPy |
|---|---|
| Membuat array dari data | `np.array()` |
| Membuat array berpola | `zeros`, `ones`, `full`, `arange`, `linspace` |
| Menghitung semua elemen sekaligus | operator `+ - * /` dan fungsi seperti `np.sqrt` |
| Memilih sebagian data | indeks, slicing, boolean indexing |
| Membuat nilai baru berdasarkan syarat | `np.where` |
| Ringkasan statistik | `mean`, `median`, `std`, `percentile`, `argmax` |
| Mengabaikan nilai kosong | `nanmean`, `nansum`, dan sejenisnya |
| Menyusun ulang bentuk | `reshape`, `flatten`, `.T` |

## Note Penting

- Array menyimpan angka dengan rapat dan memungkinkan perhitungan seluruh elemen tanpa perulangan (vektorisasi).
- Periksa `shape`, `dtype`, `ndim`, dan `size` untuk memahami sebuah array.
- Broadcasting membuat array kecil otomatis menyesuaikan diri dengan array besar, asalkan ukurannya cocok.
- Gunakan `&`, `|`, dan `~` (bukan `and`, `or`, `not`) untuk menggabungkan syarat pada array.
- Irisan array adalah tampilan, bukan salinan. Gunakan `.copy()` bila diperlukan.
- Parameter `axis` menentukan arah perhitungan pada array 2 dimensi.
- Nilai kosong (`nan` atau nodata seperti `-9999`) harus ditangani sebelum menghitung statistik.
- Raster pada dasarnya adalah array 2 dimensi, sehingga konsep di halaman ini langsung terpakai untuk pengolahan citra.
