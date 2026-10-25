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
title: Functions dan Classes
authors:
  - name: Diniyarti
---

# Functions dan Classes

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

## Pendahuluan

Ketika menulis kode, ada pekerjaan yang muncul berulang kali: mengubah meter menjadi kilometer, memeriksa apakah koordinat valid, atau menghitung jarak antara dua kota. Menyalin kode yang sama ke banyak tempat membuat program panjang dan rawan salah. Kalau rumusnya ternyata perlu diperbaiki, semua salinan harus dicari dan diubah satu per satu.

Dua alat berikut menyelesaikan masalah itu:

- **Functions** membungkus satu pekerjaan menjadi unit bernama yang bisa dipanggil kapan saja.
- **Classes** membungkus data beserta pekerjaan yang berkaitan dengan data itu, misalnya sebuah titik lokasi lengkap dengan kemampuan menghitung jaraknya ke titik lain.

Prinsip yang mendasarinya sering disebut DRY (*Don't Repeat Yourself*): satu logika cukup ditulis di satu tempat.

```{note}
Koordinat kota di halaman ini adalah perkiraan untuk keperluan belajar. Untuk real analisis, gunakan data dari sumber resmi.
```

## Functions: Pekerjaan yang Bisa Dipanggil Ulang

### Membuat Functions Pertama

Functions dibuat dengan kata kunci `def`, diikuti nama fungsi, parameter di dalam kurung, dan tanda titik dua. Isi fungsi menjorok ke dalam. Perintah `return` mengirim hasil kembali ke tempat fungsi dipanggil.

```{code-cell} ipython3
def meter_ke_km(meter):
    return meter / 1000

jarak_km = meter_ke_km(3250)
print(jarak_km)
```

Beberapa istilah yang perlu dikenal:

- **Parameter** adalah nama di dalam kurung saat fungsi didefinisikan (`meter`).
- **Argumen** adalah nilai nyata yang diberikan saat fungsi dipanggil (`3250`).
- Memanggil fungsi berarti menulis namanya diikuti kurung, dan hasilnya bisa disimpan ke variabel.

### `print()` Tidak Sama dengan `return`

Ini salah satu hal yang paling sering membingungkan pemula. `print()` hanya **menampilkan** nilai di layar. `return` **mengirim** nilai kembali agar bisa dipakai lagi oleh kode lain.

```{code-cell} ipython3
def kuadrat_print(x):
    print(x ** 2)

def kuadrat_return(x):
    return x ** 2

a = kuadrat_print(4)    # layar menampilkan 16
b = kuadrat_return(4)   # hasil disimpan ke b

print(a, b)
```

Variabel `a` berisi `None` (artinya "tidak ada nilai") karena fungsi pertama tidak mengembalikan apa pun sehingga kita tidak bisa menggunakan variabel `a` untuk perhitungan lain misal `c = a + 2`, maka hasilnya akan `error`. Jika hasil fungsi akan dipakai untuk perhitungan lanjutan, gunakan `return`.

### Nilai Bawaan pada Parameter

Parameter bisa diberi nilai bawaan, yang dipakai kalau argumennya tidak diisi. Cara ini membuat fungsi mudah dipakai untuk kasus umum, tetapi tetap bisa disesuaikan.

```{code-cell} ipython3
def klasifikasi_ketinggian(tinggi_m, batas_rendah=50, batas_sedang=500):
    if tinggi_m < batas_rendah:
        return "rendah"
    elif tinggi_m < batas_sedang:
        return "sedang"
    else:
        return "tinggi"

print(klasifikasi_ketinggian(12))
print(klasifikasi_ketinggian(12, batas_rendah=10))
```

Pada panggilan kedua, argumen ditulis dengan **nama** (`batas_rendah=10`). Penulisan seperti ini membuat panggilan lebih jelas dibaca, dan urutannya boleh berbeda dari urutan parameter. Parameter bernilai bawaan harus ditulis setelah parameter yang wajib diisi.

### Mengembalikan Lebih dari Satu Nilai

Fungsi bisa mengembalikan beberapa nilai sekaligus (sebenarnya sebuah tuple), yang kemudian bisa dibongkar ke beberapa variabel.

```{code-cell} ipython3
def ringkas(daftar):
    return min(daftar), max(daftar), sum(daftar) / len(daftar)

terendah, tertinggi, rata_rata = ringkas([3.0, 5.5, 12.0, 18.5, 30.0])
print(terendah, tertinggi, rata_rata)
```

### Dokumentasi Fungsi (Docstring)

Teks di antara tiga tanda kutip pada baris pertama fungsi disebut *docstring*. Isinya menjelaskan fungsi itu, dan bisa ditampilkan dengan `help()`.

```{code-cell} ipython3
def koordinat_valid(lat, lon):
    """Mengembalikan True jika lintang dan bujur berada dalam rentang yang sah."""
    return -90 <= lat <= 90 and -180 <= lon <= 180

print(koordinat_valid(-5.1477, 119.4327))
print(koordinat_valid(119.43, -5.15))
help(koordinat_valid)
```

### Variabel di dalam Fungsi Bersifat Lokal

Variabel yang dibuat di dalam fungsi hanya ada selama fungsi berjalan. Di luar fungsi, variabel itu tidak dikenal.

```python
def hitung():
    hasil = 10
    return hasil

hitung()
print(hasil)    # NameError: name 'hasil' is not defined
```

Kalau nilainya dibutuhkan di luar, kembalikan dengan `return` lalu simpan ke variabel baru.

## Menghitung Jarak: Fungsi Haversine

### Mengapa Tidak Pakai Rumus Jarak Biasa?

Rumus jarak lurus (Pythagoras) pada koordinat derajat akan keliru, karena Bumi berbentuk bulat dan panjang satu derajat bujur mengecil ke arah kutub. Jarak terpendek di permukaan bola mengikuti busur **lingkaran besar** (*great circle*), bukan garis lurus pada peta datar.

Rumus **Haversine** menghitung jarak busur itu dari dua pasang koordinat. Hasilnya cukup akurat untuk banyak keperluan, dengan catatan Bumi dianggap bola sempurna sehingga ada selisih kecil dibandingkan model yang lebih teliti.


### Menulis Fungsinya

Langkahnya: ubah selisih derajat menjadi radian, hitung nilai antara `a`, lalu ubah menjadi sudut pusat `c`, dan kalikan dengan jari-jari Bumi.

```{code-cell} ipython3
from math import radians, sin, cos, sqrt, asin

def haversine(lat1, lon1, lat2, lon2, radius=6371.0):
    """Jarak lingkaran besar antara dua titik (derajat). Satuan hasil mengikuti radius."""
    d_lat = radians(lat2 - lat1)
    d_lon = radians(lon2 - lon1)

    a = sin(d_lat / 2) ** 2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(d_lon / 2) ** 2
    c = 2 * asin(sqrt(a))

    return radius * c
```

Parameter `radius` bernilai bawaan 6371,0 (jari-jari rata-rata Bumi dalam kilometer), sehingga hasilnya otomatis dalam kilometer. Untuk satuan lain, ganti radiusnya.

```{code-cell} ipython3
# Makassar -> Jayapura (koordinat perkiraan)
jarak_km = haversine(-5.1477, 119.4327, -2.5337, 140.7181)
print(f"Jarak: {jarak_km:.1f} km")

# Dalam mil (jari-jari Bumi sekitar 3958,8 mil)
jarak_mil = haversine(-5.1477, 119.4327, -2.5337, 140.7181, radius=3958.8)
print(f"Jarak: {jarak_mil:.1f} mil")
```

Karena sudah dibungkus sebagai fungsi, jarak antar dua kota lain cukup dihitung dengan memanggilnya lagi, tanpa menyalin rumusnya.

### Menghitung Panjang Sebuah Rute

Fungsi bisa memanggil fungsi lain. Berikut fungsi yang menjumlahkan jarak antar titik berurutan pada sebuah rute.

```{code-cell} ipython3
def panjang_rute(daftar_titik, radius=6371.0):
    total = 0
    for i in range(len(daftar_titik) - 1):
        lat1, lon1 = daftar_titik[i]
        lat2, lon2 = daftar_titik[i + 1]
        total += haversine(lat1, lon1, lat2, lon2, radius)
    return total

rute = [
    (-5.1477, 119.4327),   # Makassar
    (-0.0263, 109.3425),   # Pontianak
    (3.5952, 98.6722),     # Medan
]

print(f"Panjang rute: {panjang_rute(rute):.1f} km")
```

## Jumlah Argumen yang Fleksibel

Dua bentuk parameter berikut berguna ketika jumlah masukan tidak pasti. Keduanya termasuk topik lanjutan, jadi boleh dilewati dulu oleh pemula.

### `*args`: Sebanyak Apa Pun Nilai Tanpa Nama

Tanda `*` di depan parameter menampung semua argumen ke dalam sebuah tuple.

```{code-cell} ipython3
def rata_rata_angka(*angka):
    return sum(angka) / len(angka)

print(rata_rata_angka(3.0, 5.5, 12.0))
print(rata_rata_angka(3.0, 5.5, 12.0, 18.5, 30.0))
```

### `**kwargs`: Sebanyak Apa Pun Nilai Bernama

Tanda `**` menampung semua argumen bernama ke dalam sebuah dictionary. Cocok untuk atribut tambahan yang tidak selalu ada.

```{code-cell} ipython3
def deskripsi_titik(lat, lon, **atribut):
    teks = f"Titik ({lat}, {lon})"
    for kunci, nilai in atribut.items():
        teks += f", {kunci}: {nilai}"
    return teks

print(deskripsi_titik(-5.1436, 119.4075, nama="Pantai Losari", jenis="wisata"))
print(deskripsi_titik(-5.1330, 119.4880, nama="Kampus Unhas"))
```

## Classes: Data dan Perilakunya Dijadikan Satu

### Membuat Classes Pertama

Pada sebuah titik lokasi, ada beberapa data yang selalu menyertai: lintang, bujur, dan nama. Mengurusnya sebagai variabel terpisah akan merepotkan jadi kita bisa menggunakan classes untuk menyatukannya.

Classes ibarat **blue print**, dan setiap titik yang dibuat dari blue print itu disebut **objek** (atau *instance*).

```{code-cell} ipython3
class Titik:
    def __init__(self, latitude, longitude, nama="Tanpa Nama"):
        self.latitude = latitude
        self.longitude = longitude
        self.nama = nama

    def __str__(self):
        return f"{self.nama} ({self.latitude}, {self.longitude})"

losari = Titik(-5.1436, 119.4075, "Pantai Losari")
print(losari)
print(losari.latitude)
```

Beberapa istilah penting:

- `__init__` adalah method khusus yang otomatis berjalan saat objek dibuat. Fungsinya menyiapkan data awal.
- `self` mewakili objek yang sedang dikerjakan. Tiap objek punya data sendiri.
- `self.latitude`, `self.longitude`, dan `self.nama` adalah **atribut**, yaitu data yang melekat pada objek.
- `__str__` menentukan tampilan objek saat dicetak dengan `print()`.

Karena `nama` punya nilai bawaan, titik bisa dibuat tanpa nama:

```{code-cell} ipython3
titik_x = Titik(-5.20, 119.45)
print(titik_x)
```

### Menambah Method

**Method** adalah fungsi yang berada di dalam Classes dan bekerja menggunakan data objeknya. Berikut method untuk menghitung jarak ke titik lain, memakai fungsi `haversine` yang sudah dibuat.

```{code-cell} ipython3
class Titik:
    def __init__(self, latitude, longitude, nama="Tanpa Nama"):
        self.latitude = latitude
        self.longitude = longitude
        self.nama = nama

    def __str__(self):
        return f"{self.nama} ({self.latitude}, {self.longitude})"

    def jarak_ke(self, titik_lain):
        return haversine(self.latitude, self.longitude,
                         titik_lain.latitude, titik_lain.longitude)

    def belahan(self):
        if self.latitude > 0:
            return "Utara"
        elif self.latitude < 0:
            return "Selatan"
        return "Khatulistiwa"

losari = Titik(-5.1436, 119.4075, "Pantai Losari")
unhas = Titik(-5.1330, 119.4880, "Kampus Unhas")

print(f"Jarak {losari.nama} ke {unhas.nama}: {losari.jarak_ke(unhas):.2f} km")
print(losari.belahan())
```

Perhatikan bahwa saat dipanggil (`losari.jarak_ke(unhas)`), argumen `self` tidak ditulis. Python mengisinya otomatis dengan objek di depan titik.

### Menggabungkan Classes: Rute

Objek dari satu Classes boleh dipakai di dalam Classes lain. Berikut Classes `Rute` yang menyimpan kumpulan titik dan bisa menghitung panjang totalnya.

```{code-cell} ipython3
class Rute:
    def __init__(self, titik=None):
        self.titik = titik if titik is not None else []

    def tambah(self, titik_baru):
        self.titik.append(titik_baru)

    def total_jarak(self):
        total = 0
        for i in range(len(self.titik) - 1):
            total += self.titik[i].jarak_ke(self.titik[i + 1])
        return total

jalur = Rute([losari, unhas])
jalur.tambah(Titik(-5.0617, 119.5540, "Bandara"))

print(f"Jumlah titik: {len(jalur.titik)}")
print(f"Total jarak: {jalur.total_jarak():.2f} km")
```

```{warning}
Pada `__init__` di atas, nilai bawaan ditulis `None`, bukan `[]`. Jika memakai `titik=[]`, semua objek `Rute` akan **berbagi satu list yang sama** (masalah mirip `b = a` pada list). Pola `None` lalu membuat list baru di dalam fungsi adalah cara aman yang umum dipakai.
```

### Memperluas Classes dengan Pewarisan

Kadang diperlukan versi khusus dari sebuah Classes. Daripada menulis ulang semuanya, Classes baru bisa **mewarisi** Classes yang sudah ada, lalu menambah atau mengubah bagian tertentu.

```{code-cell} ipython3
class Kota(Titik):
    def __init__(self, latitude, longitude, nama, provinsi):
        super().__init__(latitude, longitude, nama)
        self.provinsi = provinsi

    def __str__(self):
        return f"{self.nama}, {self.provinsi} ({self.latitude}, {self.longitude})"

makassar = Kota(-5.1477, 119.4327, "Makassar", "Sulawesi Selatan")
print(makassar)
print(makassar.belahan())
print(f"{makassar.jarak_ke(losari):.2f} km ke {losari.nama}")
```

Classes `Kota` otomatis memiliki method `belahan()` dan `jarak_ke()` dari `Titik`, tanpa perlu menulisnya lagi. Fungsi `super()` dipakai untuk memanggil `__init__` milik Classes induk.

## Kapan Memakai Fungsi, Kapan Classes?

| Kebutuhan | Pilihan |
|---|---|
| Melakukan satu pekerjaan berulang dengan masukan berbeda | Fungsi |
| Memecah perhitungan rumit menjadi bagian kecil | Fungsi |
| Satu objek punya banyak data yang selalu menyertai | Classes |
| Data dan perilakunya perlu dijaga bersama | Classes |

Beberapa kebiasaan baik:

- Beri nama fungsi dan Classes yang jelas menggambarkan tugasnya.
- Satu fungsi sebaiknya mengerjakan satu hal saja.
- Gunakan nilai bawaan agar fungsi mudah dipakai untuk kasus umum.
- Uji fungsi dengan beberapa masukan, termasuk yang tidak biasa.
- Mulai dari yang sederhana. Tidak semua program membutuhkan Classes.

## Note Penting

- Fungsi membungkus satu pekerjaan agar bisa dipanggil berulang tanpa menyalin kode.
- `return` mengirim hasil kembali, sedangkan `print()` hanya menampilkan di layar.
- Parameter bernilai bawaan dan argumen bernama membuat fungsi fleksibel dan mudah dibaca.
- Rumus Haversine menghitung jarak di permukaan Bumi dari dua koordinat, dan hasilnya lebih tepat daripada jarak lurus pada derajat.
- Classes menyatukan data (atribut) dan perilaku (method) dalam satu objek, dengan `__init__` sebagai penyiap data awal.
- Pewarisan memungkinkan Classes baru memakai ulang Classes yang sudah ada.
- Gunakan `None` sebagai nilai bawaan untuk list atau dictionary yang bisa berubah.





