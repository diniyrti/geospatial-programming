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
title: Dasar-Dasar Menggunakan Pandas
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

NumPy sangat cocok untuk tumpukan angka yang seragam. Namun data spasial di lapangan biasanya berbentuk **tabel campuran**: kolom nama kota (teks), koordinat (desimal), jumlah penduduk, dan tanggal pengamatan. Tabel semacam ini lebih nyaman diolah dengan **Pandas**.

Pandas dibangun di atas NumPy, tetapi menambahkan **label** pada baris dan kolom. Bayangkan sebuah lembar kerja spreadsheet yang bisa diprogram: bisa menyaring baris, menghitung kolom baru, merangkum per kelompok, menggabungkan dua tabel, dan membaca langsung dari berkas CSV. Karena itulah Pandas menjadi dasar bagi pustaka data spasial seperti GeoPandas, yang akan dibahas nanti.

Dua struktur utama Pandas:

- **Series**: satu kolom data berlabel.
- **DataFrame**: tabel dua dimensi, yaitu kumpulan Series yang berbagi label baris.

```{note}
Koordinat kota di halaman ini adalah perkiraan untuk keperluan belajar. Jumlah penduduk, curah hujan, dan data lainnya adalah contoh fiktif, bukan data resmi. Untuk real analisis, gunakan data dari sumber resmi seperti BPS atau BIG.
```

## Series dan DataFrame

### Series: Satu Kolom Berlabel

Series mirip array NumPy, tetapi setiap elemennya punya label (indeks), sehingga bisa diambil dengan nama.

```{code-cell} ipython3
import numpy as np
import pandas as pd

tinggi = pd.Series(
    [3.0, 5.5, 12.0],
    index=["Losari", "Rotterdam", "Unhas"]
)

print(tinggi)
print(tinggi["Unhas"])
print(tinggi.mean())
```

Perhitungan pada Series berlaku untuk seluruh elemen, sama seperti pada array NumPy:

```{code-cell} ipython3
print((tinggi * 3.28084).round(1))
```

### DataFrame: Tabel Lengkap

Cara paling mudah membuat DataFrame adalah dari dictionary, dengan nama kolom sebagai kunci dan isi kolom sebagai list.

```{code-cell} ipython3
data_kota = {
    "kota": ["Makassar", "Medan", "Pontianak", "Jayapura", "Banda Aceh"],
    "pulau": ["Sulawesi", "Sumatera", "Kalimantan", "Papua", "Sumatera"],
    "latitude": [-5.1477, 3.5952, -0.0263, -2.5337, 5.5483],
    "longitude": [119.4327, 98.6722, 109.3425, 140.7181, 95.3238],
}

df = pd.DataFrame(data_kota)
df
```

Di notebook, menulis nama DataFrame di baris terakhir sel menampilkan tabel yang rapi. Hasilnya lebih baik daripada `print(df)`.

### Memeriksa Isi DataFrame

Beberapa perintah yang hampir selalu dijalankan pertama kali setelah data dimuat:

```{code-cell} ipython3
print(df.shape)                 # (jumlah baris, jumlah kolom)
print(df.columns.tolist())      # nama kolom
print(df.dtypes)                # tipe tiap kolom
```

```{code-cell} ipython3
df.head(3)                      # tiga baris pertama
```

```{code-cell} ipython3
df.info()                       # ringkasan struktur dan jumlah data terisi
```

```{code-cell} ipython3
df[["latitude", "longitude"]].describe().round(2)    # ringkasan statistik kolom angka
```

Kolom teks akan tampil dengan tipe `object` atau `str`, tergantung versi Pandas yang dipakai. Keduanya menandakan data berupa teks.

## Memilih Data

### Memilih Kolom

Satu nama kolom di dalam kurung siku menghasilkan sebuah Series. Daftar nama kolom (kurung siku ganda) menghasilkan DataFrame baru.

```{code-cell} ipython3
lintang = df["latitude"]
print(type(lintang))
print(lintang.tolist())
```

```{code-cell} ipython3
koordinat = df[["kota", "latitude", "longitude"]]
print(type(koordinat))
koordinat
```

### Memilih Baris dengan `loc` dan `iloc`

Ada dua cara memilih baris dan kolom sekaligus:

- `loc` memakai **label** (nama baris dan nama kolom).
- `iloc` memakai **posisi** (nomor urut), seperti indeks pada list.

```{code-cell} ipython3
print(df.loc[0, "kota"])                      # baris berlabel 0, kolom "kota"
print(df.iloc[0, 0])                          # baris posisi 0, kolom posisi 0
```

```{code-cell} ipython3
df.loc[1:3, ["kota", "pulau"]]
```

```{code-cell} ipython3
df.iloc[0:2, 0:2]
```

Perhatikan perbedaan penting pada irisan: `loc[1:3]` **menyertakan** baris berlabel 3, sedangkan `iloc[0:2]` **tidak menyertakan** posisi 2, sama seperti slicing biasa di Python.

### Menyaring Baris dengan Syarat

Syarat pada kolom menghasilkan deretan benar-salah yang dipakai untuk menyaring baris.

```{code-cell} ipython3
df[df["latitude"] < 0]                        # kota di selatan khatulistiwa
```

```{code-cell} ipython3
df[df["pulau"] == "Sumatera"]
```

Beberapa syarat digabung dengan `&` (dan), `|` (atau), dan `~` (bukan). Tiap syarat wajib dibungkus kurung.

```{code-cell} ipython3
df[(df["latitude"] < 0) & (df["longitude"] > 115)]
```

```{code-cell} ipython3
df[df["pulau"].isin(["Sulawesi", "Papua"])]   # nilai ada di dalam daftar
```

```{warning}
Seperti pada NumPy, kata `and`, `or`, dan `not` tidak bisa dipakai untuk menggabungkan syarat pada kolom. Gunakan `&`, `|`, dan `~`, dengan kurung di sekeliling tiap syarat.
```

## Membuat Kolom Baru

Kolom baru dibuat dengan menugaskan hasil perhitungan ke nama kolom yang belum ada.

```{code-cell} ipython3
df["lat_radian"] = np.radians(df["latitude"])
df["belahan"] = np.where(df["latitude"] >= 0, "Utara", "Selatan")

df[["kota", "latitude", "lat_radian", "belahan"]]
```

Kolom teks bisa digabung dengan `+`, dan metode `.str` menyediakan operasi teks yang sama dengan yang ada pada halaman operasi string, tetapi berlaku untuk seluruh kolom sekaligus.

```{code-cell} ipython3
df["label"] = df["kota"] + " (" + df["pulau"] + ")"
df["kota_besar"] = df["kota"].str.upper()

df[["label", "kota_besar"]]
```

Pembersihan teks pada seluruh kolom juga mudah:

```{code-cell} ipython3
kolom_kotor = pd.Series(["  makassar ", "MEDAN", " Pontianak"])
print(kolom_kotor.str.strip().str.title().tolist())
```

### Mengelompokkan Angka ke Kategori

Fungsi `pd.cut` mengubah angka menjadi kategori berdasarkan batas-batas yang ditentukan. Contoh berikut membuat pembagian bujur yang sangat sederhana (bukan pembagian zona waktu resmi).

```{code-cell} ipython3
df["zona_bujur"] = pd.cut(
    df["longitude"],
    bins=[90, 105, 125, 145],
    labels=["barat", "tengah", "timur"],
)

df[["kota", "longitude", "zona_bujur"]]
```

## Mengurutkan dan Menghitung Frekuensi

```{code-cell} ipython3
df.sort_values("longitude")[["kota", "longitude"]]
```

```{code-cell} ipython3
df.sort_values("latitude", ascending=False)[["kota", "latitude"]]
```

Untuk mengetahui berapa kali tiap nilai muncul pada sebuah kolom:

```{code-cell} ipython3
df["pulau"].value_counts()
```

## Merangkum per Kelompok: `groupby`

`groupby` membagi tabel menjadi kelompok berdasarkan satu kolom, lalu menghitung ringkasan untuk tiap kelompok. Bayangkan seperti fitur *pivot table* di spreadsheet.

```{code-cell} ipython3
penduduk = pd.DataFrame({
    "kota": ["Makassar", "Manado", "Medan", "Banda Aceh", "Pontianak", "Jayapura"],
    "pulau": ["Sulawesi", "Sulawesi", "Sumatera", "Sumatera", "Kalimantan", "Papua"],
    "penduduk_ribu": [1400, 450, 2400, 250, 650, 400],    # angka contoh, bukan data resmi
})

penduduk
```

```{code-cell} ipython3
penduduk.groupby("pulau")["penduduk_ribu"].sum()
```

Beberapa ringkasan bisa dihitung sekaligus dengan `agg`:

```{code-cell} ipython3
ringkasan = penduduk.groupby("pulau")["penduduk_ribu"].agg(["count", "sum", "mean"])
ringkasan
```

Hasil `groupby` memakai nama kelompok sebagai indeks. Jika ingin kembali menjadi kolom biasa, tambahkan `reset_index()`:

```{code-cell} ipython3
ringkasan.reset_index()
```

## Menggabungkan Dua Tabel: `merge`

Data spasial sering tersebar di beberapa tabel, misalnya satu tabel berisi koordinat dan tabel lain berisi data statistik. `merge` menyambungkannya berdasarkan kolom kunci yang sama, mirip fitur *join* pada SIG.

```{code-cell} ipython3
tabel_koordinat = df[["kota", "latitude", "longitude"]]
tabel_penduduk = penduduk[["kota", "penduduk_ribu"]]

gabungan = pd.merge(tabel_koordinat, tabel_penduduk, on="kota")
gabungan
```

Secara bawaan, `merge` hanya menyimpan baris yang kuncinya ada di **kedua** tabel (*inner join*). Parameter `how` mengubah perilaku ini:

```{code-cell} ipython3
pd.merge(tabel_koordinat, tabel_penduduk, on="kota", how="outer")
```

Pada hasil di atas, Manado hanya ada di tabel penduduk sehingga kolom koordinatnya kosong (`NaN`). Dengan `how="left"`, semua baris tabel kiri dipertahankan, dan dengan `how="outer"`, semua baris dari kedua tabel dipertahankan.

```{tip}
Penggabungan sering gagal karena penulisan kunci yang tidak seragam, misalnya `"Makassar"` dengan `"makassar "`. Seragamkan dulu dengan `.str.strip().str.title()` sebelum `merge`.
```

## Menangani Data Kosong

Data dari lapangan hampir selalu punya nilai yang hilang. Pandas menandainya sebagai `NaN`.

```{code-cell} ipython3
hujan_stasiun = pd.DataFrame({
    "stasiun": ["A", "B", "C", "D"],
    "curah_hujan_mm": [210.0, np.nan, 180.0, np.nan],
})

hujan_stasiun
```

Mendeteksi dan menghitung nilai kosong:

```{code-cell} ipython3
print(hujan_stasiun["curah_hujan_mm"].isna().sum())
hujan_stasiun.isna()
```

Dua pilihan penanganan yang paling umum:

```{code-cell} ipython3
hujan_stasiun.dropna()                     # buang baris yang ada nilai kosongnya
```

```{code-cell} ipython3
rata_rata = hujan_stasiun["curah_hujan_mm"].mean()    # nilai kosong otomatis diabaikan
hujan_stasiun.fillna({"curah_hujan_mm": rata_rata})   # isi dengan rata-rata
```

Pada perintah `fillna`, kolom yang diisi disebutkan lewat dictionary supaya kolom teks tidak ikut terpengaruh.

```{warning}
Memilih cara penanganan nilai kosong adalah keputusan analisis, bukan sekadar teknis. Mengisi dengan 0 akan mengubah arti data (hujan 0 mm berbeda dengan hujan yang tidak tercatat), sedangkan mengisi dengan rata-rata menyembunyikan variasi. Catat selalu cara yang dipilih.
```

## Membaca dan Menyimpan CSV

Pandas bisa membaca berbagai format, di antaranya CSV, Excel, dan basis data. Untuk berkas CSV dipakai `pd.read_csv()`, yang menerima alamat berkas di komputer maupun tautan web.

Di Indonesia, berkas CSV dari spreadsheet sering memakai **titik koma** sebagai pemisah kolom dan **koma** sebagai tanda desimal. Kedua hal itu diatur lewat parameter `sep` dan `decimal`.

Contoh berikut memakai teks yang meniru isi sebuah berkas, melalui `io.StringIO`, supaya bisa dicoba tanpa mengunduh apa pun:

```{code-cell} ipython3
import io

teks_csv = """stasiun;lintang;bujur;curah_hujan_mm
A;-5,1436;119,4075;210,5
B;-5,1344;119,4059;180,0
C;-5,1330;119,4880;195,3
"""

df_csv = pd.read_csv(io.StringIO(teks_csv), sep=";", decimal=",")

print(df_csv.dtypes)
df_csv
```

Tanpa `decimal=","`, kolom angka akan terbaca sebagai teks. Untuk berkas sungguhan, cukup ganti `io.StringIO(teks_csv)` dengan alamat berkasnya, misalnya `"data/stasiun.csv"`.

Menyimpan DataFrame ke CSV memakai `to_csv`. Parameter `index=False` mencegah nomor baris ikut tersimpan sebagai kolom:

```python
df_csv.to_csv("stasiun_bersih.csv", index=False)
```

Untuk berkas Excel, gunakan `pd.read_excel()` dan `to_excel()`, yang membutuhkan pustaka tambahan seperti `openpyxl`.

## Membuat Grafik Sederhana

DataFrame punya metode `plot()` untuk grafik cepat. Pandas memakai Matplotlib di belakangnya. Contoh berikut memakai data curah hujan bulanan fiktif yang meniru dua pola musim berbeda.

```{code-cell} ipython3
bulan = ["Jan", "Feb", "Mar", "Apr", "Mei", "Jun", "Jul", "Agu", "Sep", "Okt", "Nov", "Des"]

hujan = pd.DataFrame(
    {
        "Wilayah A": [650, 520, 440, 250, 140, 90, 40, 20, 35, 120, 280, 540],
        "Wilayah B": [200, 190, 230, 250, 230, 200, 170, 160, 170, 200, 230, 220],
    },
    index=bulan,
)

hujan.plot(figsize=(8, 4), title="Curah hujan bulanan (data contoh)", ylabel="mm");
```

Grafik batang cocok untuk membandingkan kategori:

```{code-cell} ipython3
hujan["Wilayah A"].plot.bar(figsize=(8, 4), title="Wilayah A", ylabel="mm");
```

Grafik sebar menunjukkan hubungan antara dua kolom:

```{code-cell} ipython3
hujan.plot.scatter(x="Wilayah A", y="Wilayah B", alpha=0.7, figsize=(5, 4));
```

Untuk grafik yang lebih rinci, Matplotlib dipakai langsung. Metode `plot()` cukup untuk pengecekan visual yang cepat.

## Memadukan Pandas dan NumPy

Kolom DataFrame adalah array NumPy yang diberi label, sehingga fungsi NumPy bisa langsung dipakai pada kolom. Contoh berikut menghitung jarak setiap kota ke Makassar memakai rumus Haversine dalam bentuk vektor (sama seperti pada halaman NumPy).

```{code-cell} ipython3
def haversine_np(lat1, lon1, lat2, lon2, radius=6371.0):
    """Jarak lingkaran besar (km). Menerima angka tunggal, array, maupun kolom."""
    lat1, lon1, lat2, lon2 = map(np.radians, (lat1, lon1, lat2, lon2))
    a = np.sin((lat2 - lat1) / 2) ** 2 + np.cos(lat1) * np.cos(lat2) * np.sin((lon2 - lon1) / 2) ** 2
    return radius * 2 * np.arcsin(np.sqrt(a))
```

```{code-cell} ipython3
makassar = df[df["kota"] == "Makassar"].iloc[0]

df["jarak_ke_makassar_km"] = haversine_np(
    makassar["latitude"], makassar["longitude"],
    df["latitude"], df["longitude"],
).round(1)

df.sort_values("jarak_ke_makassar_km")[["kota", "jarak_ke_makassar_km"]]
```

### Matriks Jarak Antar Semua Kota

Dengan trik broadcasting dari halaman NumPy, matriks jarak semua pasangan kota bisa dibuat sekali jalan, lalu dibungkus menjadi DataFrame berlabel supaya mudah dibaca.

```{code-cell} ipython3
lat = df["latitude"].to_numpy()
lon = df["longitude"].to_numpy()

matriks = haversine_np(lat[:, np.newaxis], lon[:, np.newaxis], lat, lon)

df_jarak = pd.DataFrame(matriks, index=df["kota"], columns=df["kota"]).round(0)
df_jarak
```

Rata-rata jarak tiap kota ke kota lainnya bisa ditambahkan sebagai kolom baru:

```{code-cell} ipython3
df["rata_jarak_km"] = matriks.mean(axis=1).round(1)
df[["kota", "rata_jarak_km"]]
```

## Panduan Singkat

| Kebutuhan | Alat Pandas |
|---|---|
| Membuat tabel dari dictionary | `pd.DataFrame()` |
| Mengenal isi tabel | `shape`, `dtypes`, `head()`, `info()`, `describe()` |
| Memilih kolom | `df["kolom"]`, `df[["a", "b"]]` |
| Memilih baris dan kolom | `loc` (label), `iloc` (posisi) |
| Menyaring baris | `df[syarat]` dengan `&`, `\|`, `~` |
| Kolom baru | `df["baru"] = ...`, `np.where`, `pd.cut`, `.str` |
| Merangkum per kelompok | `groupby(...).agg(...)` |
| Menggabungkan tabel | `pd.merge(..., on=..., how=...)` |
| Nilai kosong | `isna()`, `dropna()`, `fillna()` |
| Baca dan simpan CSV | `read_csv(sep=..., decimal=...)`, `to_csv(index=False)` |
| Grafik cepat | `df.plot()`, `.plot.bar()`, `.plot.scatter()` |

## Note Penting

- DataFrame adalah tabel berlabel yang bisa memuat kolom bertipe berbeda, sedangkan Series adalah satu kolomnya.
- Biasakan memeriksa data dengan `shape`, `dtypes`, `head()`, dan `describe()` sebelum menganalisis.
- `loc` memakai label dan menyertakan batas akhir, `iloc` memakai posisi dan tidak menyertakan batas akhir.
- Gabungkan syarat dengan `&`, `|`, `~`, dan bungkus tiap syarat dengan kurung.
- `groupby` merangkum per kelompok, sedangkan `merge` menyambung dua tabel lewat kolom kunci.
- Keputusan menangani nilai kosong memengaruhi hasil analisis. Pilih dengan sadar dan catat.
- Untuk CSV berformat Indonesia, gunakan `sep=";"` dan `decimal=","`.
- Fungsi NumPy bisa dipakai langsung pada kolom DataFrame, sehingga kedua pustaka saling melengkapi.