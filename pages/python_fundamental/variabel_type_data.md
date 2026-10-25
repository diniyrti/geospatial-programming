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
title: Variabel dan Tipe Data
authors:
  - name: Diniyarti
---

# Variabel dan Tipe Data

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

## Pendahuluan
Sebelum bisa mengolah data spasial dengan Python, ada dua konsep dasar yang harus dipahami yaitu **variabel** dan **tipe data**.

**Variabel** itu seperti kotak berlabel untuk menyimpan data. Kamu beri nama, isi dengan nilai, lalu bisa dipakai atau diubah kapan saja selama program berjalan. Dalam kerja geospasial, variabel bisa menyimpan data/informasi seperti koordinat sebuah titik, nilai ketinggian, atau nama tempat.

## Variabel di Python

**Variabel** adalah nama yang merujuk ke sebuah data yang tersimpan di memori komputer. Cara mudah membayangkannya adalah kotak penyimpanan yang diberi label: isinya adalah data, dan labelnya adalah nama variabel. Setiap kali data itu dibutuhkan, kita cukup memanggil namanya.

### Membuat dan Menampilkan Variabel

Variabel dibuat dengan tanda sama dengan (`=`): nama di sebelah kiri, nilai di sebelah kanan. Tanda ini berarti "simpan nilai di kanan ke dalam nama di kiri", bukan "sama dengan" seperti di matematika.

```{code-cell} ipython3
jumlah_titik = 120
```

Variabel `jumlah_titik` sekarang menyimpan bilangan bulat 120, yang bisa dipakai untuk perhitungan atau logika program. Untuk melihat isinya ada dua cara. Cara pertama memakai fungsi `print()`:

```{code-cell} ipython3
print(jumlah_titik)
```

Cara kedua cukup menulis nama variabelnya saja di akhir sel, lalu menjalankan sel tersebut. Notebook akan menampilkan nilainya otomatis:

```{code-cell} ipython3
jumlah_titik
```

### Mengubah Isi Variabel

Isi variabel boleh diganti kapan saja, bahkan dengan jenis data yang berbeda. Python tidak mewajibkan jenis data ditentukan di awal.

```{code-cell} ipython3
# Mulai dengan angka
lokasi = -5.1436

# Ganti dengan teks
lokasi = "Pantai Losari"

# Ganti dengan daftar koordinat (lintang, bujur)
lokasi = [-5.1436, 119.4075]

print(lokasi)
```

Yang tercetak hanya nilai terakhir, karena setiap penugasan baru menimpa isi sebelumnya.

### Aturan Penamaan Variabel

Nama variabel di Python harus memenuhi aturan berikut:

- Diawali huruf atau garis bawah (`_`), tidak boleh diawali angka.
- Bagian selanjutnya boleh berupa huruf, angka, dan garis bawah. Spasi dan tanda seperti `-` atau `@` tidak boleh dipakai.
- Huruf besar dan kecil dianggap berbeda. `jumlah_titik` dan `Jumlah_Titik` adalah dua variabel yang berbeda.
- Nama sebaiknya jelas dan bermakna, misalnya `jumlah_titik` lebih baik daripada `n`.
- Hindari kata yang sudah dipakai Python, seperti `if`, `for`, `class`, serta nama fungsi bawaan seperti `print`, `sum`, `str`, dan `int`. Memakainya sebagai nama variabel bisa membuat fungsi aslinya tidak bisa dipanggil lagi.

### Contoh Nama yang Baik untuk Data Geospasial

Nama yang baik langsung memberi tahu apa isi variabelnya, bahkan bagi orang yang baru membaca kode.

```{code-cell} ipython3
# Nama variabel yang baik
latitude = -5.1436
longitude = 119.4075
elevasi = 3.0
nama_kota = "Makassar"
jumlah_penduduk = 1423877
sistem_koordinat = "WGS 84"
```

### Contoh Nama yang Sebaiknya Dihindari

Nama yang terlalu singkat atau terlalu umum membuat kode sulit dipahami, terutama saat program makin panjang.

```{code-cell} ipython3
# Nama variabel yang kurang baik
x = -5.1436            # Terlalu umum, x itu apa?
data = "Makassar"      # Terlalu samar
temp = 28.5            # Ambigu: suhu atau sementara (temporary)?
l = [-5.14, 119.41]    # Satu huruf sulit dipahami
```

## Tipe Data

Setiap nilai di Python punya tipe, dan tipe menentukan apa yang boleh dilakukan pada nilai itu. Angka bisa dijumlahkan, teks bisa diubah hurufnya, tapi nama kota tidak bisa dikalikan dengan koordinat.

### Bilangan bulat (`int`)
`int` yaitu angka tanpa desimal, seperti 1, 120, atau -5. Cocok untuk jumlah atau hitungan.

```{code-cell} ipython3
jumlah_fitur = 500  # jumlah fitur dalam sebuah dataset
```

### Bilangan desimal (`float`)
`float` yaitu angka dengan koma desimal (ditulis dengan titik di Python). Koordinat hampir selalu bertipe ini.

```{code-cell} ipython3
latitude = -5.1436    # lintang sebuah titik
longitude = 119.4075  # bujur sebuah titik
```

### Teks (`str`)
`str` yaitu rangkaian karakter yang diapit tanda kutip. Dipakai untuk nama tempat, alamat, atau label. Kutip tunggal (`'...'`) dan kutip ganda (`"..."`) sama saja hasilnya. Untuk teks yang terdiri dari beberapa baris, gunakan kutip tiga (`"""..."""`).

```{code-cell} ipython3
sistem_koordinat = "WGS 84"  # sistem koordinat yang umum dipakai
```

### Boolean (`bool`)
`bool` yaitu hanya punya dua nilai, `True` atau `False`. Berguna sebagai penanda ya atau tidak, misalnya apakah data sudah memiliki referensi koordinat.

```{code-cell} ipython3
sudah_georeferensi = True  # apakah dataset sudah memiliki referensi lokasi
```

### List
List yaitu kumpulan data berurutan yang diapit kurung siku. Isinya boleh berbagai tipe. Contoh paling umum adalah pasangan koordinat.

```{code-cell} ipython3
koordinat = [-5.1436, 119.4075]  # [lintang, bujur] sebuah titik
```

### Dictionary (`dict`)
`dict` yaitu kumpulan pasangan kunci dan nilai, mirip kartu data dengan kolom bernama. Cocok untuk menyimpan atribut satu objek geografis.

```{code-cell} ipython3
atribut_fitur = {
    "nama": "Benteng Rotterdam",
    "jenis": "Benteng bersejarah",
    "jumlah_bangunan": 5,
    "lokasi": [-5.1344, 119.4059],
}
```

Contoh di atas menunjukkan bahwa isi dictionary bisa campuran: teks, angka, bahkan list.

## Karakter Escape

Karakter escape adalah kombinasi garis miring terbalik (`\`) dengan satu karakter, yang dipakai untuk memasukkan karakter khusus ke dalam teks. Dua yang paling sering dipakai: `\n` untuk enter/ke baris selanjutnya dan `\t` untuk tab (menjorok ke kanan).

```{code-cell} ipython3
print("Halo, Makassar!\nIni baris kedua.")
```

```{code-cell} ipython3
print("Baris pertama.\n\tBaris kedua, menjorok ke kanan.")
```

Ada satu masalah yang sering muncul: teks yang mengandung tanda kutip. Jika teks diapit kutip ganda, kutip tunggal di dalamnya aman dipakai.

```{code-cell} ipython3
print("Nama variabel itu 'case-sensitive'")
```

Cara lain adalah menambahkan garis miring terbalik sebelum tanda kutipnya, sehingga Python tahu bahwa kutip itu bagian dari teks, bukan penutup.

```{code-cell} ipython3
print('Nama variabel itu \'case-sensitive\'')
```

## Komentar

Komentar adalah catatan di dalam kode untuk menjelaskan maksudnya. Python mengabaikan semua teks setelah tanda `#` pada baris itu, jadi komentar tidak memengaruhi hasil program. Komentar bisa berdiri sendiri atau ditulis di samping kode.

```{code-cell} ipython3
# Ini komentar satu baris
jumlah_titik = 120  # Ini komentar di samping kode
```

Kebiasaan menulis komentar singkat sangat membantu, terutama saat kode dibuka lagi beberapa minggu kemudian atau akan digunakan oleh orang lain.

## Bekerja dengan Variabel dan Tipe Data

Setelah mengenal tipenya, variabel bisa dipakai untuk operasi sederhana.

**Menambah nilai angka.** Operator `+=` menambahkan angka ke nilai yang sudah ada.

```{code-cell} ipython3
suhu_udara = 20
print("Jumlah fitur sekarang:", jumlah_fitur)
```

```{code-cell} ipython3
suhu_udara += 5
print("Suhu udara sekarang:", suhu_udara)
```

**Mengubah derajat menjadi radian.** Banyak rumus geospasial memakai satuan radian, bukan derajat. Python menyediakan modul `math` untuk keperluan ini. Kata `import` berfungsi memanggil modul tersebut sebelum fungsinya dipakai.

```{code-cell} ipython3
import math

latitude = -5.1436
latitude_radian = math.radians(latitude)
print("Latitude dalam radian:", latitude_radian)
```

**Menambah isi list.** Method `append()` menambahkan satu item di akhir list.

```{code-cell} ipython3
koordinat = [-5.1436, 119.4075]
koordinat.append(-5.0617)    
koordinat.append(119.5540)
print("Koordinat sekarang:", koordinat)
```

Hasilnya list berisi empat angka, yaitu dua pasang koordinat yang tercampur dalam satu list. Di materi selanjutnya akan dibahas cara menyusunnya dengan lebih rapi.

**Mengambil nilai dari dictionary.** Gunakan kurung siku dengan nama kuncinya.

```{code-cell} ipython3
atribut_fitur = {
    "nama": "Benteng Rotterdam",
    "jenis": "Benteng bersejarah",
    "jumlah_bangunan": 5,
    "lokasi": [-5.1344, 119.4059],
}
```

```{code-cell} ipython3
nama = atribut_fitur["nama"]
jumlah_bangunan = atribut_fitur["jumlah_bangunan"]
print(f"{nama} memiliki {jumlah_bangunan} bangunan.")
```

Huruf `f` di depan teks membuat Python mengisi bagian `{...}` dengan nilai variabelnya (disebut f-string).

## Operasi Dasar pada Teks

Tipe teks punya banyak method bawaan. Ini berguna di dunia geospasial karena data sering berisi nama tempat, alamat, dan label yang tidak seragam penulisannya.

### Mengubah Huruf Besar dan Kecil

```{code-cell} ipython3
nama_kota = "Kota Makassar"

print("Huruf kecil:", nama_kota.lower())
print("Huruf besar:", nama_kota.upper())
print("Huruf kapital di awal tiap kata:", nama_kota.title())
```

Method `lower()` dan `upper()` sering dipakai untuk menyeragamkan teks sebelum dibandingkan. Misalnya "makassar" dan "MAKASSAR" perlu dianggap sama.

### Mengganti Bagian Teks

```{code-cell} ipython3
nama_asli = "Jalan Perintis"
nama_singkat = nama_asli.replace("Jalan", "Jl.")
print("Asli:", nama_asli)
print("Singkat:", nama_singkat)
```

Perhatikan bahwa teks asli tidak berubah. `replace()` menghasilkan teks baru, yang perlu disimpan ke variabel lain.

### Menghapus Spasi Berlebih

```{code-cell} ipython3
lokasi = "  Pantai Losari  "
lokasi_bersih = lokasi.strip()
print("Setelah dibersihkan:", lokasi_bersih)
```

Method `strip()` menghapus spasi di awal dan akhir teks. Masalah spasi tersembunyi seperti ini sering muncul pada data hasil unduhan atau input manual.

Operasi teks lainnya akan dibahas di materi khusus tentang string.

## Note Penting

- Variabel adalah nama yang menunjuk ke data, dan datanya selalu punya tipe.
- Enam tipe dasar: `int`, `float`, `str`, `bool`, list, dan dictionary.
- Nama variabel yang jelas (misalnya `latitude`, bukan `x`) membuat kode lebih mudah dibaca.
- Karakter escape seperti `\n` dan `\t` mengatur tampilan teks, dan komentar (`#`) menjelaskan kode tanpa memengaruhi hasilnya.
- Method pada teks seperti `lower()`, `replace()`, dan `strip()` sangat berguna untuk merapikan nama tempat dan label data.

Konsep-konsep ini menjadi fondasi untuk semua materi berikutnya. Cara terbaik menguasainya adalah mencoba langsung dengan data lokasi yang dikenal, misalnya sekitar kampus atau kota sendiri.