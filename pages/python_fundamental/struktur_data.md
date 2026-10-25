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
title: Struktur Data Python
authors:
  - name: Diniyarti
---

# Struktur Data Python

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

## Pendahuluan

Satu variabel hanya menyimpan satu nilai. Padahal data spasial hampir selalu berupa **kumpulan**: puluhan titik koordinat sepanjang jalur survei, daftar nama kecamatan, atau seperangkat atribut untuk satu lokasi. Menyimpan semuanya di variabel terpisah (`titik1`, `titik2`, `titik3`, ...) menjadi tidak praktis.

Python menyediakan empat struktur data bawaan untuk menyimpan kumpulan data: **list**, **tuple**, **set**, dan **dictionary**. Masing-masing punya sifat yang berbeda, dan memilih yang tepat membuat kode lebih singkat, lebih aman, dan lebih mudah dibaca.

```{note}
Koordinat di halaman ini adalah perkiraan untuk keperluan belajar. Nilai ketinggian dan data survei lainnya adalah contoh fiktif. Untuk real analisis, gunakan data dari sumber resmi.
```

## Empat Pertanyaan untuk Memilih Struktur

Sebelum mengenal satu per satu, ada empat pertanyaan yang membantu menentukan struktur yang tepat:

1. Apakah isinya boleh berubah setelah dibuat?
2. Apakah urutan penting?
3. Apakah isinya harus unik (tanpa duplikat)?
4. Apakah tiap nilai perlu diberi nama, bukan sekadar nomor urut?

Jawaban atas pertanyaan-pertanyaan ini akan menunjuk ke struktur yang cocok. Tabel ringkasnya ada di bagian akhir halaman.

## List: Kumpulan Berurutan yang Bisa Diubah

List ditulis dengan kurung siku. Urutan isinya terjaga, isinya boleh diubah, dan boleh berisi tipe data campuran. Bayangkan daftar tempat yang akan dikunjungi dalam satu hari survei.

```{code-cell} ipython3
rute_survei = ["Pantai Losari", "Benteng Rotterdam", "Universitas Hasanuddin"]
print(rute_survei)
```

### Mengakses Isi List

Isi list diakses dengan nomor urut (indeks) yang **dimulai dari 0**. Indeks negatif menghitung dari belakang.

```{code-cell} ipython3
print(rute_survei[0])    # elemen pertama
print(rute_survei[1])    # elemen kedua
print(rute_survei[-1])   # elemen terakhir
```

Untuk mengambil sebagian isi, gunakan *slicing* dengan pola `[awal:akhir]`. Elemen di posisi `akhir` tidak ikut diambil.

```{code-cell} ipython3
print(rute_survei[:2])    # dari awal sampai sebelum indeks 2
print(rute_survei[1:3])   # indeks 1 dan 2
```

### Mengubah Isi List

Karena bersifat mutable, list bisa ditambah, disisipkan, atau dikurangi setelah dibuat.

```{code-cell} ipython3
rute_survei.append("Bandara Sultan Hasanuddin")   # tambah di akhir
rute_survei.insert(1, "Fort Rotterdam Timur")      # sisip di indeks 1
print(rute_survei)

rute_survei.remove("Fort Rotterdam Timur")         # hapus berdasarkan nilai
terakhir = rute_survei.pop()                       # ambil dan hapus elemen terakhir
print(terakhir)
print(rute_survei)
```

Elemen list juga boleh diganti langsung lewat indeksnya:

```{code-cell} ipython3
rute_survei[0] = "Anjungan Pantai Losari"
print(rute_survei)
```

### Ringkasan Statistik Sederhana

List angka sering dipakai untuk deret pengukuran. Fungsi bawaan `len()`, `max()`, `min()`, dan `sum()` langsung bisa dipakai.

```{code-cell} ipython3
ketinggian_m = [3.0, 5.5, 12.0, 18.5, 30.0]   # data contoh

print("Jumlah data :", len(ketinggian_m))
print("Tertinggi   :", max(ketinggian_m))
print("Terendah    :", min(ketinggian_m))

rata_rata = sum(ketinggian_m) / len(ketinggian_m)
print(f"Rata-rata   : {rata_rata:.1f} m")
```

Format `:.1f` di dalam f-string artinya "tampilkan satu angka di belakang koma".

### Hati-hati: Dua Nama, Satu List

Karena list bisa diubah, ada jebakan yang sering membingungkan pemula. Menulis `b = a` tidak membuat salinan, melainkan membuat nama kedua untuk list yang sama.

```{code-cell} ipython3
rute_a = ["Losari", "Rotterdam"]
rute_b = rute_a            # hanya nama kedua, bukan salinan
rute_b.append("Unhas")

print(rute_a)              # ikut berubah!
```

Jika yang dibutuhkan adalah salinan yang terpisah, gunakan `.copy()`:

```{code-cell} ipython3
rute_c = rute_a.copy()
rute_c.append("Bandara")

print(rute_a)              # tidak berubah
print(rute_c)
```

## Tuple: Kumpulan Berurutan yang Tidak Bisa Diubah

Tuple mirip list, tetapi ditulis dengan kurung biasa dan **tidak bisa diubah** setelah dibuat. Sifat ini cocok untuk data yang memang tidak boleh berubah, misalnya sepasang koordinat. Lintang dan bujur satu titik tidak seharusnya tergeser tanpa sengaja di tengah program.

```{code-cell} ipython3
losari = (-5.1436, 119.4075)   # (lintang, bujur)
print(losari)
print(losari[0])
```

Mencoba mengubah isi tuple akan menghasilkan error:

```python
losari[0] = -5.2
```

Hasilnya `TypeError: 'tuple' object does not support item assignment`. Error ini sebenarnya berguna, karena melindungi data penting dari perubahan yang tidak disengaja.

### Membongkar Tuple (Unpacking)

Isi tuple bisa langsung dipisah ke beberapa variabel dalam satu baris. Cara ini sangat sering dipakai untuk koordinat.

```{code-cell} ipython3
lat, lon = losari
print(f"Lintang: {lat}, Bujur: {lon}")
```

### Tuple dalam List

Kombinasi yang paling umum dalam data spasial adalah list yang berisi tuple koordinat, yaitu sebuah jalur.

```{code-cell} ipython3
jalur = [
    (-5.1436, 119.4075),   # Pantai Losari
    (-5.1344, 119.4059),   # Benteng Rotterdam
    (-5.1330, 119.4880),   # Universitas Hasanuddin
]

print(jalur[1])
lat_awal, lon_awal = jalur[0]
print(lat_awal, lon_awal)
```

List-nya fleksibel untuk ditambah titik, sementara tiap koordinatnya terlindungi.

## Set: Kumpulan Unik Tanpa Urutan

Set ditulis dengan kurung kurawal dan hanya berisi nilai yang **unik**. Nilai ganda otomatis dibuang, dan urutannya tidak dijamin. Set cocok untuk pertanyaan seperti "ada berapa jenis yang berbeda?".

```{code-cell} ipython3
tutupan_2015 = {"hutan", "sawah", "tambak", "permukiman"}
print(tutupan_2015)
```

### Membuang Duplikat

Set juga sering dipakai untuk membersihkan duplikat dari list lewat `set()`.

```{code-cell} ipython3
kecamatan_survei = ["Tamalanrea", "Rappocini", "Tamalanrea", "Biringkanaya", "Rappocini"]

kecamatan_unik = set(kecamatan_survei)
print(kecamatan_unik)
print("Jumlah kecamatan berbeda:", len(kecamatan_unik))
```

Urutan tampilan set bisa berbeda tiap kali dijalankan, dan itu normal.

### Menambah dan Mengecek Anggota

```{code-cell} ipython3
tutupan_2015.add("industri")
tutupan_2015.add("sawah")      # sudah ada, tidak ada perubahan
print(tutupan_2015)

print("hutan" in tutupan_2015)
print("gurun" in tutupan_2015)
```

Pengecekan keanggotaan dengan `in` sangat cepat pada set, bahkan untuk data yang besar.

:::{tip}
**`append()` pada list dan `add()` pada set sama-sama menambah satu elemen, tetapi hasilnya berbeda.**

| | `list.append(x)` | `set.add(x)` |
|---|---|---|
| Posisi elemen baru | Selalu di akhir | Tidak ada posisi |
| Jika elemen sudah ada | Tetap ditambahkan (duplikat) | Diabaikan, set tidak berubah |
| Tambah banyak sekaligus | `extend([...])` | `update([...])` |

```python
rute = ["Losari", "Rotterdam"]
rute.append("Losari")        # ['Losari', 'Rotterdam', 'Losari']

kecamatan = {"Tamalanrea", "Rappocini"}
kecamatan.add("Rappocini")   # tidak ada perubahan
```

Perhatikan juga bahwa `rute.append(["Unhas", "Bandara"])` menambahkan satu list di dalam list, sedangkan `rute.extend(["Unhas", "Bandara"])` menambahkan kedua isinya satu per satu.
:::

### Membandingkan Dua Himpunan

Kekuatan utama set adalah operasi himpunan. Misalnya membandingkan jenis tutupan lahan pada dua tahun berbeda:

```{code-cell} ipython3
tutupan_2015 = {"hutan", "sawah", "tambak", "permukiman"}
tutupan_2025 = {"sawah", "tambak", "permukiman", "industri"}

print("Ada di keduanya :", tutupan_2015 & tutupan_2025)    # irisan
print("Gabungan        :", tutupan_2015 | tutupan_2025)    # gabungan
print("Hilang di 2025  :", tutupan_2015 - tutupan_2025)    # hanya di 2015
print("Baru di 2025    :", tutupan_2025 - tutupan_2015)    # hanya di 2025
```

Pola seperti ini muncul dalam analisis perubahan tutupan lahan, misalnya untuk mengetahui kelas apa yang hilang atau muncul antar periode.

```{warning}
Set tidak punya indeks, jadi `tutupan_2015[0]` akan error. Selain itu, `{}` yang kosong adalah dictionary, bukan set. Untuk membuat set kosong, tulis `set()`.
```

## Dictionary: Data dengan Label

Dictionary menyimpan pasangan **kunci dan nilai**. Nilai diambil lewat nama kunci, bukan nomor urut. Cara ini mirip satu baris pada tabel atribut: tiap kolom punya nama.

```{code-cell} ipython3
titik = {
    "id": "T01",
    "nama": "Pantai Losari",
    "koordinat": (-5.1436, 119.4075),
    "ketinggian_m": 3.0,
    "tutupan_lahan": "ruang terbuka",
}

print(titik["nama"])
print(titik["koordinat"])
```

Kode `titik["nama"]` jauh lebih mudah dipahami daripada `data[1]`. Kode jadi lebih jelas dan kecil kemungkinan salah ambil.

### Mengambil Nilai dengan Aman

Mengambil kunci yang tidak ada menghasilkan `KeyError`. Method `get()` menghindarinya dengan memberi nilai cadangan.

```{code-cell} ipython3
kecamatan = titik.get("kecamatan", "belum diisi")
print(kecamatan)
```

### Menambah, Mengubah, dan Menghapus

```{code-cell} ipython3
titik["kecamatan"] = "Ujung Pandang"   # tambah kunci baru
titik["ketinggian_m"] = 3.5            # ubah nilai yang ada
titik.pop("tutupan_lahan")             # hapus satu kunci

print(titik)
```

:::{tip}
**`pop()` pada list dan dictionary bekerja berbeda.**

- Pada **list**, `pop()` menghapus berdasarkan **posisi**. Tanpa argumen, yang dihapus adalah elemen terakhir.
- Pada **dictionary**, `pop()` menghapus berdasarkan **kunci**, sehingga kuncinya wajib ditulis. Dictionary tidak punya nomor urut yang bisa dijadikan patokan.

```python
rute = ["Losari", "Rotterdam", "Unhas"]
rute.pop()                 # "Unhas" (elemen terakhir)

titik = {"nama": "Pantai Losari", "ketinggian_m": 3.0}
titik.pop("nama")          # "Pantai Losari"
# titik.pop()              # TypeError, karena kunci tidak diberikan
```

Jika kunci yang dimaksud tidak ada, hasilnya `KeyError`. Untuk menghindarinya, beri nilai cadangan: `titik.pop("kecamatan", None)`. Untuk menghapus pasangan terakhir yang ditambahkan ke dictionary, gunakan `popitem()`.
:::

### Menjelajahi Isi Dictionary

```{code-cell} ipython3
print(list(titik.keys()))     # semua kunci
print(list(titik.values()))   # semua nilai
print("koordinat" in titik)   # cek apakah kunci ada
```

Method `items()` akan sangat berguna setelah mempelajari perulangan di bab berikutnya.

### Dictionary di dalam Dictionary

Nilai dictionary boleh berupa dictionary lain, sehingga data bertingkat bisa disusun.

```{code-cell} ipython3
kampus = {
    "unhas": {
        "nama": "Universitas Hasanuddin",
        "kecamatan": "Tamalanrea",
        "koordinat": (-5.1330, 119.4880),
    },
    "unm": {
        "nama": "Universitas Negeri Makassar",
        "kecamatan": "Rappocini",
        "koordinat": (-5.1655, 119.4325),
    },
}

print(kampus["unhas"]["kecamatan"])
```

Dua kurung siku berturut-turut dibaca dari kiri ke kanan: ambil `"unhas"` dulu, lalu ambil `"kecamatan"` dari hasilnya.

## Menggabungkan Struktur: List yang Berisi Dictionary

Dalam praktik, struktur-struktur ini sering digabung. Kombinasi paling berguna adalah **list yang berisi dictionary**: tiap dictionary mewakili satu lokasi, dan list-nya adalah kumpulan lokasi. Susunan ini sangat mirip dengan tabel atribut, dan menjadi dasar pemahaman terhadap tabel data spasial yang akan ditemui nanti, seperti GeoDataFrame.

```{code-cell} ipython3
titik_survei = [
    {"id": "T01", "nama": "Pantai Losari",     "ketinggian_m": 3.0,  "koordinat": (-5.1436, 119.4075)},
    {"id": "T02", "nama": "Benteng Rotterdam", "ketinggian_m": 5.5,  "koordinat": (-5.1344, 119.4059)},
    {"id": "T03", "nama": "Kampus Unhas",      "ketinggian_m": 12.0, "koordinat": (-5.1330, 119.4880)},
]

print(titik_survei[1]["nama"])
print(titik_survei[2]["koordinat"])

titik_survei.append(
    {"id": "T04", "nama": "Bandara", "ketinggian_m": 14.0, "koordinat": (-5.0617, 119.5540)}
)
print(len(titik_survei))
```

Bagian pertama kurung siku memilih lokasi (nomor urut di list), bagian kedua memilih atribut (nama kunci di dictionary). Setelah mempelajari perulangan, pola ini akan sangat memudahkan pengolahan banyak titik sekaligus.

## Panduan Memilih Struktur

| Struktur | Urutan terjaga | Bisa diubah | Isi harus unik | Diakses lewat | Cocok untuk |
|---|---|---|---|---|---|
| `list` | Ya | Ya | Tidak | Nomor urut | Jalur, deret pengukuran, daftar yang bertambah |
| `tuple` | Ya | Tidak | Tidak | Nomor urut | Sepasang koordinat, data tetap |
| `set` | Tidak | Ya | Ya | Cek `in` | Kode unik, jenis kelas, perbandingan himpunan |
| `dict` | Ya | Ya | Kunci unik | Nama kunci | Atribut lokasi, metadata, pengaturan |

Kembali ke empat pertanyaan di awal: kalau isinya tidak boleh berubah, pilih tuple. Kalau urutan penting dan isinya berubah, pilih list. Kalau yang penting hanya keunikan, pilih set. Kalau tiap nilai butuh nama, pilih dictionary.

## Note Penting

- List: berurutan dan bisa diubah. Hati-hati, `b = a` tidak membuat salinan, gunakan `.copy()`.
- Tuple: berurutan tetapi tidak bisa diubah, pas untuk pasangan koordinat dan boleh jadi kunci dictionary.
- Set: unik dan tanpa urutan, membuang duplikat dan membandingkan dua himpunan.
- Dictionary: pasangan kunci dan nilai, cocok untuk atribut yang butuh nama.
- Dalam praktik struktur ini digabung, terutama list yang berisi dictionary, yang bentuknya mirip tabel atribut.

