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
title: Regresi AGBD Menggunakan Random Forest
authors:
  - name: Diniyarti
---

# Regresi Aboveground Biomass Density (AGBD) Menggunakan Random Forest

:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::


# 1. Pendahuluan

Analisis ini membahas penerapan **Random Forest Regression** untuk memperkirakan *Aboveground Biomass Density* (AGBD) pada area hutan. Pemodelan memanfaatkan data **Google Satellite Embedding** sebagai variabel prediktor dan data **GEDI** sebagai variabel target atau referensi.

Berbeda dari klasifikasi penggunaan lahan yang memprediksi label kelas, regresi digunakan untuk memperkirakan nilai numerik yang kontinu. Dalam praktikum ini, nilai yang diprediksi adalah kerapatan biomassa atas permukaan tanah (*aboveground biomass density*), yang umumnya dinyatakan dalam satuan megagram per hektare (Mg/ha).

Alur analisis mengintegrasikan beberapa lingkungan dan pustaka pemrograman:

- **Google Earth Engine (GEE)** untuk mengakses dan mengolah data penginderaan jauh berbasis cloud.
- **Google Satellite Embedding** sebagai data prediktor.
- **GEDI** sebagai sumber data AGBD.
- **Xarray dan Xee** untuk membaca data Earth Engine sebagai dataset berkoordinat spasial.
- **Pandas dan NumPy** untuk menyiapkan data tabular.
- **Scikit-learn** untuk membangun, melakukan tuning, dan mengevaluasi model Random Forest.
- **Matplotlib** untuk membuat visualisasi hasil.
- **rioxarray** untuk menyimpan hasil prediksi dalam format raster GeoTIFF.


# 2. Persiapan lingkungan kerja

Praktikum menggunakan Google Colab agar mahasiswa dapat menjalankan Python tanpa perlu melakukan instalasi seluruh pustaka pada komputer lokal. Google Drive digunakan untuk menyimpan data dan hasil pemrosesan.

## 2.1 Menghubungkan Google Drive

Google Drive dihubungkan ke Google Colab melalui `drive.mount()`. Selanjutnya, `ROOT_PATH` digunakan sebagai direktori utama untuk menyimpan hasil praktikum.

```python
# Mount Google Drive
import os
from google.colab import drive

# Menghubungkan Google Drive ke lingkungan Google Colab
drive.mount('/content/gdrive')

# Direktori utama untuk menyimpan seluruh input dan output praktikum
ROOT_PATH = '/content/gdrive/MyDrive/Bahan Ajar Semester Ganjil 2026/Praktek Spasial Programming/Praktek_Regresi_RF'

# Membuat folder secara otomatis jika belum tersedia
os.makedirs(ROOT_PATH, exist_ok=True)
```

- `import os` memanggil pustaka Python untuk berinteraksi dengan sistem direktori dan berkas.
- `drive.mount()` menghubungkan Google Drive sehingga dapat diakses melalui direktori `/content/gdrive`.
- `ROOT_PATH` mendefinisikan lokasi kerja utama. Dengan menggunakan satu variabel direktori, lokasi penyimpanan keluaran dapat diubah tanpa mengedit setiap perintah penyimpanan.
- `os.makedirs(..., exist_ok=True)` membuat folder jika belum tersedia. Argumen `exist_ok=True` mencegah error ketika folder tersebut sudah ada.

## 2.2 Instalasi pustaka

Beberapa pustaka tambahan diperlukan untuk menghubungkan Google Earth Engine dengan Xarray serta mengolah data raster.

```python
# Install Dependencies
!pip install --upgrade xee
!pip install -U geemap
!pip install rioxarray
```

- `xee` menyediakan engine untuk membaca objek Earth Engine sebagai dataset Xarray.
- `geemap` mempermudah interaksi dengan Google Earth Engine, terutama untuk visualisasi peta di lingkungan Jupyter atau Colab.
- `rioxarray` menambahkan kemampuan pengelolaan informasi spasial dan ekspor raster melalui Xarray.

> Catatan: Instalasi pustaka biasanya cukup dilakukan satu kali pada sesi runtime Colab. Jika runtime di-reset, pustaka perlu dipasang kembali.

## 2.3 Inisialisasi Google Earth Engine

Google Earth Engine memerlukan autentikasi akun dan inisialisasi project sebelum dapat digunakan.

```python
# Initialize Google Earth Engine
import ee
import geemap

# Autentikasi akun Google yang memiliki akses ke Earth Engine
ee.Authenticate()

# Inisialisasi Earth Engine menggunakan Google Cloud Project
ee.Initialize(
    project='diniyrti',
    opt_url='https://earthengine-highvolume.googleapis.com'
)
```

- `import ee` memanggil API Python Google Earth Engine.
- `ee.Authenticate()` menjalankan proses autentikasi akun Google.
- `ee.Initialize()` menghubungkan notebook dengan Earth Engine menggunakan project yang telah terdaftar.
- `project` perlu disesuaikan dengan Google Cloud Project milik pengguna.
- `opt_url` menentukan endpoint Earth Engine yang digunakan. Pada script ini digunakan endpoint High Volume.

## 2.4 Membuat peta interaktif

`geemap.Map()` digunakan untuk membuat peta interaktif. Peta ini akan dipakai untuk menampilkan AOI, tutupan lahan, mask hutan, dan data biomassa.

```python
# Membuat peta interaktif dengan basemap citra satelit
Map = geemap.Map(basemap='SATELLITE')

# Menampilkan peta
Map
```

Parameter `basemap='SATELLITE'` memilih tampilan dasar berupa citra satelit. Variabel `Map` dapat digunakan kembali untuk menambahkan berbagai layer hasil pengolahan.

---

# 3. Menentukan Area of Interest (AOI)

*Area of Interest* (AOI) merupakan wilayah geografis yang menjadi fokus analisis. Pada praktikum ini, wilayah kajian adalah **Provinsi Sulawesi Tenggara**.

Batas administrasi diambil dari dataset FAO Global Administrative Unit Layers (GAUL) tahun 2015 pada level provinsi atau `level1`.

## 3.1 Memanggil data batas administrasi

```python
# Memanggil batas administrasi negara dan provinsi dari GAUL
aoi_gaul = ee.FeatureCollection('FAO/GAUL/2015/level1')

# Memeriksa nama properti yang tersedia pada data
print(aoi_gaul.first().propertyNames().getInfo())

# Menampilkan daftar nama provinsi di Indonesia
print(
    aoi_gaul
    .filter(ee.Filter.eq('ADM0_NAME', 'Indonesia'))
    .aggregate_array('ADM1_NAME')
    .getInfo()
)
```

- `ee.FeatureCollection()` memanggil data batas administrasi berbentuk kumpulan fitur vektor.
- `first()` mengambil satu fitur pertama untuk memeriksa struktur atribut.
- `propertyNames()` menampilkan nama kolom atau atribut yang tersedia.
- `filter()` menyaring fitur berdasarkan kondisi tertentu.
- `ADM0_NAME` merupakan atribut nama negara, sedangkan `ADM1_NAME` merupakan atribut nama wilayah administrasi tingkat pertama.
- `getInfo()` mengambil hasil dari server Earth Engine ke lingkungan Python. Gunakan secara bijak karena pemanggilan ini menjalankan permintaan sinkron ke server.

## 3.2 Memilih Provinsi Sulawesi Tenggara

```python
# Memilih fitur administrasi Provinsi Sulawesi Tenggara
aoi_sul_tengg = aoi_gaul.filter(
    ee.Filter.eq('ADM1_NAME', 'Sulawesi Tenggara')
)

# Mengubah FeatureCollection menjadi objek geometri
aoi_sul_tengg = aoi_sul_tengg.geometry()

# Menampilkan geometri AOI
aoi_sul_tengg
```

`filter()` memilih fitur dengan nilai `ADM1_NAME` sama dengan `Sulawesi Tenggara`. Selanjutnya, `geometry()` mengambil geometri wilayah tersebut sehingga dapat digunakan pada operasi spasial seperti `clip()`, `filterBounds()`, dan ekspor citra.

## 3.3 Visualisasi AOI

```python
# Menambahkan batas wilayah kajian ke peta
Map.addLayer(aoi_sul_tengg, {}, 'Provinsi Sulawesi Tenggara')

# Memusatkan tampilan peta pada AOI
Map.centerObject(aoi_sul_tengg, 7)

# Menampilkan peta
Map
```

- `addLayer()` menambahkan geometri AOI sebagai layer peta.
- `centerObject()` mengatur pusat dan tingkat zoom peta berdasarkan objek geometri.
- Angka `7` menunjukkan tingkat zoom yang digunakan. Nilainya dapat disesuaikan dengan luas wilayah yang ditampilkan.

---

# 4. Identifikasi area hutan menggunakan MODIS Land Cover

Sebelum membangun model biomassa, wilayah analisis dibatasi pada area yang teridentifikasi sebagai hutan. Pada notebook ini digunakan produk **MODIS MCD12Q1** dengan band `LC_Type1`.

Produk MCD12Q1 menyediakan informasi tutupan lahan tahunan. Band `LC_Type1`, dengan kelas 1 sampai 5 yang merepresentasikan tipe hutan.

## 4.1 Memanggil data MODIS Land Cover

```python
# Memanggil produk MODIS Land Cover tahunan
modis_lc = (
    ee.ImageCollection('MODIS/061/MCD12Q1')
    # Memilih periode tahun 2020
    .filterDate('2020-01-01', '2021-01-01')
    # Mengambil band klasifikasi IGBP
    .select('LC_Type1')
    # Mengambil citra pada periode yang dipilih
    .first()
    # Memotong citra berdasarkan AOI
    .clip(aoi_sul_tengg)
)

# Memeriksa objek citra
modis_lc
```

- `ImageCollection()` memanggil koleksi citra MODIS.
- `filterDate()` memilih citra berdasarkan rentang waktu. Rentang akhir bersifat eksklusif, sehingga tanggal `2021-01-01` digunakan untuk mengambil periode tahun 2020.
- `select('LC_Type1')` mengambil band klasifikasi tutupan lahan IGBP.
- `first()` mengambil citra pertama setelah penyaringan waktu.
- `clip()` membatasi cakupan citra pada wilayah kajian.

## 4.2 Visualisasi tutupan lahan

Setiap nilai kelas MODIS ditampilkan menggunakan warna yang berbeda. Palet berikut mengikuti urutan kelas IGBP pada band `LC_Type1`.

```python
# Parameter visualisasi tutupan lahan
vis_params = {
    'min': 1,
    'max': 17,
    'palette': [
        '05450a',  # 1 Evergreen Needleleaf Forest
        '086a10',  # 2 Evergreen Broadleaf Forest
        '54a708',  # 3 Deciduous Needleleaf Forest
        '78d203',  # 4 Deciduous Broadleaf Forest
        '009900',  # 5 Mixed Forest
        'c6b044',  # 6 Closed Shrublands
        'dcd159',  # 7 Open Shrublands
        'dade48',  # 8 Woody Savannas
        'fbff13',  # 9 Savannas
        'b6ff05',  # 10 Grasslands
        '27ff87',  # 11 Permanent Wetlands
        'c24f44',  # 12 Croplands
        'a5a5a5',  # 13 Urban and Built-up
        'ff6d4c',  # 14 Cropland/Natural Vegetation Mosaic
        '69fff8',  # 15 Snow and Ice
        'f9ffa4',  # 16 Barren or Sparsely Vegetated
        '1c0dff'   # 17 Water Bodies
    ]
}

# Menambahkan layer MODIS ke peta
Map.addLayer(modis_lc, vis_params, 'MODIS Land Cover 2020')

# Memusatkan peta pada AOI
Map.centerObject(aoi_sul_tengg, 7)

Map
```

`vis_params` merupakan dictionary yang menyimpan parameter visualisasi. `min` dan `max` menentukan rentang nilai kelas yang ditampilkan, sedangkan `palette` menentukan warna untuk setiap kelas.

Visualisasi ini membantu memeriksa sebaran tutupan lahan sebelum proses masking. Perlu diingat bahwa warna hanya digunakan untuk visualisasi dan tidak mengubah nilai kelas pada raster.

## 4.3 Membuat forest mask

Pada praktikum ini, kelas hutan ditentukan menggunakan kelas 1 sampai 5. Nilai piksel yang memenuhi rentang tersebut akan bernilai benar (`1`), sedangkan kelas lainnya bernilai salah (`0`).

```python
# Memilih kelas hutan IGBP, yaitu kelas 1 sampai 5
forest_mask = (
    modis_lc.gte(1)
    .And(modis_lc.lte(5))
    .rename('Forest')
)

# Menampilkan forest mask
Map.addLayer(forest_mask, {}, 'Forest Mask')
Map
```

- `gte(1)` memeriksa apakah nilai piksel lebih besar atau sama dengan 1.
- `lte(5)` memeriksa apakah nilai piksel lebih kecil atau sama dengan 5.
- `.And()` menggabungkan kedua kondisi. Piksel hanya dipilih jika memenuhi keduanya.
- `rename('Forest')` memberikan nama band agar lebih mudah dikenali.

## 4.4 Melakukan masking area non-hutan

Setelah forest mask dibuat, mask tersebut diterapkan pada citra tutupan lahan. Dengan demikian, piksel yang tidak termasuk kelas hutan akan menjadi nilai kosong (*masked*).

```python
# Mempertahankan piksel yang teridentifikasi sebagai hutan
forest_only = modis_lc.updateMask(forest_mask)

# Visualisasi area hutan
Map.addLayer(
    forest_only,
    {'palette': ['006400']},
    'Forest Only'
)

Map
```

`updateMask()` menerapkan mask pada citra. Piksel yang memiliki nilai mask 0 tidak ditampilkan dan tidak ikut dalam operasi berikutnya. Hasilnya adalah citra tutupan lahan yang hanya menampilkan kelas hutan.

## 4.5 Mengubah raster hutan menjadi geometri

Raster hutan selanjutnya diubah menjadi poligon menggunakan `reduceToVectors()`. Geometri hasil konversi digunakan untuk membatasi cakupan data GEDI dan Satellite Embedding.

```python
# Mengubah raster area hutan menjadi kumpulan poligon
forest_aoi = forest_only.reduceToVectors(
    geometry=aoi_sul_tengg,  # Batas maksimum proses vektorisasi
    scale=500,               # Skala proses mengikuti resolusi MODIS
    geometryType='polygon',  # Geometri keluaran berupa poligon
    eightConnected=True,     # Piksel diagonal dianggap saling terhubung
    labelProperty='class',   # Nama atribut kelas pada hasil vektor
    maxPixels=1e13           # Batas maksimum jumlah piksel yang diproses
)

# Menggabungkan geometri poligon menjadi satu geometri
forest_aoi = forest_aoi.geometry()

# Menampilkan area hutan
Map.addLayer(forest_aoi, {}, 'Forest AOI')
Map
```

- `reduceToVectors()` mengubah wilayah raster yang memiliki nilai kelas menjadi fitur vektor.
- `geometry` membatasi area pemrosesan.
- `scale=500` menggunakan skala sekitar 500 meter, sesuai resolusi spasial produk MODIS MCD12Q1.
- `geometryType='polygon'` menghasilkan geometri poligon.
- `eightConnected=True` menganggap piksel yang bersinggungan secara diagonal sebagai bagian dari kelompok yang sama.
- `labelProperty` menentukan nama atribut untuk menyimpan nilai kelas.
- `maxPixels` meningkatkan batas jumlah piksel yang dapat diproses.
- `geometry()` menggabungkan geometri fitur menjadi satu objek geometri.

---

# 5. Mengambil data referensi GEDI AGBD

Data GEDI digunakan sebagai variabel target (*dependent variable*) dalam pemodelan regresi. Pada notebook ini, citra dipanggil dari koleksi `LARSE/GEDI/GEDI04_B_002`, kemudian band `MU` dipilih dan diberi nama `AGBD`.

```python
# Memanggil data GEDI dan memilih band MU
gedi_ds = (
    ee.Image('LARSE/GEDI/GEDI04_B_002')
    .select(['MU'], ['AGBD'])
    # Membatasi data GEDI pada area hutan
    .clip(forest_aoi)
)

# Menambahkan data GEDI ke peta
Map.addLayer(gedi_ds, {}, 'GEDI AGBD')
Map
```

- `ee.Image()` memanggil produk GEDI sebagai citra.
- `select(['MU'], ['AGBD'])` memilih band `MU` dan mengganti namanya menjadi `AGBD`. Nama band ini kemudian digunakan sebagai nama variabel target pada tahap pemodelan.
- `clip(forest_aoi)` membatasi data pada area hutan yang telah ditentukan.

Dalam pemodelan ini, `AGBD` menjadi nilai yang akan dipelajari oleh model. Sementara itu, band-band Satellite Embedding digunakan sebagai prediktor.

---

# 6. Mengambil data Google Satellite Embedding

Google Satellite Embedding merupakan representasi numerik dari informasi permukaan bumi yang diekstrak dari data satelit. Embedding terdiri atas sejumlah band numerik yang dapat digunakan sebagai fitur masukan pada model machine learning.

Pada praktikum ini digunakan koleksi tahunan `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL` untuk periode 2020.

## 6.1 Memanggil Satellite Embedding

```python
# Memanggil Google Satellite Embedding tahunan
embedding = (
    ee.ImageCollection('GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL')
    # Memilih periode tahun 2020
    .filterDate('2020-01-01', '2021-01-01')
    # Memilih citra yang mencakup wilayah kajian
    .filterBounds(aoi_sul_tengg)
    # Menggabungkan citra yang tersedia
    .mosaic()
    # Membatasi cakupan pada area hutan
    .clip(forest_aoi)
)

# Memeriksa nama seluruh band embedding
print('Embedding Bands:', embedding.bandNames().getInfo())

# Memeriksa proyeksi salah satu band
embedding.select('A00').projection()
```

- `ImageCollection()` memanggil koleksi Satellite Embedding.
- `filterDate()` memilih data tahunan 2020.
- `filterBounds()` memilih citra yang beririsan dengan AOI.
- `mosaic()` menggabungkan citra yang tersedia menjadi satu citra.
- `clip()` membatasi citra pada area hutan.
- `bandNames()` menampilkan nama band yang tersedia. Pada produk ini, band embedding digunakan sebagai fitur numerik.
- `projection()` memeriksa informasi proyeksi band.

## 6.2 Visualisasi Satellite Embedding

Untuk visualisasi, tiga band embedding dipilih sebagai kanal tampilan. Kombinasi tersebut bukan komposit warna alami seperti RGB Sentinel-2, melainkan representasi visual dari tiga dimensi embedding.

```python
# Parameter visualisasi tiga band embedding
vis_embb = {
    'bands': ['A00', 'A01', 'A02'],
    'min': 0,
    'max': 0.3
}

# Menambahkan layer embedding ke peta
Map.addLayer(embedding, vis_embb, 'Google Satellite Embedding')
Map
```
- `bands` menentukan tiga band yang digunakan sebagai kanal visualisasi.
- `min` dan `max` mengatur rentang nilai tampilan.
- Kombinasi band `A00`, `A01`, dan `A02` hanya digunakan untuk melihat pola spasial secara visual. 

---

# 7. Menggabungkan prediktor dan target

Sebelum pemodelan, data prediktor dan data target harus berada dalam satu struktur data. Operasi `addBands()` menggabungkan band Satellite Embedding dengan band AGBD dari GEDI.

```python
# Menggabungkan seluruh band embedding dengan band AGBD
stack_ds = embedding.addBands(gedi_ds)

# Memeriksa struktur citra gabungan
stack_ds
```

- `embedding` berisi band-band prediktor.
- `gedi_ds` berisi band target bernama `AGBD`.
- `addBands()` menggabungkan keduanya menjadi satu objek citra multiband.

Secara konseptual, setiap lokasi piksel akan memiliki sejumlah nilai prediktor dari Satellite Embedding dan satu nilai target AGBD. Struktur ini diperlukan agar nilai prediktor dan target dapat dipasangkan berdasarkan lokasi spasial.

## 7.1 Mengekspor data gabungan ke Google Drive

Save data menggunakan `ee.batch.Export.image.toDrive()`. Hasil ekspor disimpan dalam format GeoTIFF.

```python
# Membuat task ekspor citra multiband ke Google Drive
task = ee.batch.Export.image.toDrive(
    image=stack_ds,
    description='Satellite_Embedding_Biomass',
    folder='praktek_regresi',
    scale=0.01,
    region=forest_aoi,
    # maxPixels=1e13,
    crs='EPSG:4326',
    fileFormat='GeoTIFF'
)

# Memulai proses ekspor pada server Earth Engine
task.start()

print('Task ekspor citra telah dikirim ke server GEE.')
print('Mengecek status proses ekspor...')

# Memantau status task setiap 15 detik
import time

while task.active():
    print(f"Status Ekspor: {task.status()['state']}...")
    time.sleep(15)

# Menampilkan status akhir ekspor
print(f"Selesai! Status Akhir Ekspor: {task.status()['state']}")
```

- `image` merupakan citra multiband yang akan diekspor.
- `description` merupakan nama task ekspor.
- `folder` menentukan folder tujuan di Google Drive.
- `scale=0.01` menentukan ukuran piksel dalam satuan derajat karena CRS yang digunakan adalah EPSG:4326. Nilai ini **bukan 0,01 meter**. Resolusi dalam meter berubah menurut lokasi dan arah lintang-bujur.
- `region` menentukan geometri wilayah ekspor.
- `crs='EPSG:4326'` menentukan sistem koordinat keluaran.
- `fileFormat='GeoTIFF'` menentukan format raster.
- `task.start()` mengirim task ke server Earth Engine.
- `task.active()` memeriksa apakah task masih berjalan.
- `time.sleep(15)` memberi jeda 15 detik sebelum pemeriksaan status berikutnya.

---

# 8. Konversi data Earth Engine ke Xarray

Xarray merupakan pustaka Python untuk mengelola data multidimensi. Data raster dapat direpresentasikan sebagai dataset yang memiliki dimensi spasial, seperti `x` dan `y`, serta variabel yang berasal dari band-band citra.

Pada tahap ini, pustaka `xee` digunakan sebagai penghubung antara objek Earth Engine dan Xarray.

## 8.1 Memanggil pustaka yang diperlukan

```python
# Library untuk konversi Earth Engine ke Xarray
from xee import helpers
from shapely.geometry import shape
import xarray as xr
from dask.diagnostics import ProgressBar
```

- `helpers` menyediakan fungsi pendukung dari Xee.
- `shape` mengubah representasi geometri menjadi objek geometri Shapely.
- `xarray` digunakan untuk membaca dan mengelola data multidimensi.
- `ProgressBar` menampilkan perkembangan komputasi Dask ketika data dimuat.

### 8.2 Menentukan grid spasial

Sebelum membuka citra sebagai Xarray, diperlukan parameter grid yang menjelaskan sistem koordinat, resolusi, dan cakupan spasial.

```python
# Menentukan grid spasial berdasarkan geometri area hutan
grid_params = helpers.fit_geometry(
    geometry=shape(forest_aoi.getInfo()),
    grid_crs='EPSG:4326',
    grid_scale=(0.01, -0.01)
)
```

- `forest_aoi.getInfo()` mengambil representasi geometri dari server Earth Engine ke Python.
- `shape()` mengubah geometri tersebut menjadi objek Shapely.
- `fit_geometry()` menyesuaikan grid agar mencakup geometri yang dianalisis.
- `grid_crs='EPSG:4326'` menentukan sistem koordinat grid.
- `grid_scale=(0.01, -0.01)` menentukan ukuran dan arah resolusi grid pada sumbu koordinat. Nilai negatif pada sumbu `y` menunjukkan arah koordinat raster dari atas ke bawah.

> Resolusi 0,01 derajat bukan resolusi tetap dalam meter. Pada EPSG:4326, jarak fisik satu derajat bujur berubah menurut lintang. Pastikan grid yang digunakan sesuai dengan tujuan dan resolusi data sumber.

## 8.3 Membuka citra Earth Engine sebagai Xarray Dataset

```python
# Membaca citra multiband Earth Engine sebagai Xarray Dataset
ds = xr.open_dataset(
    stack_ds,
    engine='ee',
    **grid_params,
    chunks='auto'
)

# Memeriksa struktur dataset
ds
```

- `xr.open_dataset()` membuka objek citra sebagai Xarray Dataset.
- `stack_ds` merupakan citra multiband hasil penggabungan embedding dan GEDI.
- `engine='ee'` menggunakan engine Earth Engine dari Xee.
- `**grid_params` menerapkan parameter grid spasial yang telah dibuat.
- `chunks='auto'` memungkinkan pembacaan data secara bertahap menggunakan mekanisme chunking.

## 8.4 Memuat data ke memori

Data yang dibuka melalui Xarray dapat bersifat *lazy*, yaitu perhitungan baru dilakukan ketika data benar-benar dibutuhkan. `compute()` digunakan untuk menjalankan komputasi dan memuat hasilnya.

```python
# Memuat data ke memori dengan indikator progres
with ProgressBar():
    ds = ds.compute()

# Menghilangkan dimensi time karena hanya satu periode waktu yang digunakan
ds = ds.squeeze('time').drop_vars('time')

# Memeriksa dataset setelah dimensi waktu dihilangkan
ds
```

- `ProgressBar()` menampilkan perkembangan proses komputasi.
- `compute()` menjalankan operasi yang sebelumnya ditunda dan memuat hasilnya ke memori.
- `squeeze('time')` menghilangkan dimensi `time` karena hanya ada satu waktu.
- `drop_vars('time')` menghapus variabel koordinat waktu yang tidak lagi diperlukan.

> **Catatan komputasi:** Memuat seluruh raster ke memori dapat membutuhkan RAM yang besar. Jika area kajian atau resolusi grid terlalu besar, pertimbangkan pemrosesan berbasis chunk atau pembatasan wilayah sebelum menjalankan `compute()`.

## 8.5 Visualisasi data AGBD

```python
# Menampilkan distribusi spasial nilai AGBD
ds.AGBD.plot(
    x='x',
    y='y',
    robust=True
)
```

`ds.AGBD` memilih variabel AGBD dari Xarray Dataset. Fungsi `plot()` menampilkan distribusi nilai berdasarkan koordinat `x` dan `y`. Parameter `robust=True` menggunakan rentang tampilan yang lebih tahan terhadap nilai ekstrem sehingga pola umum raster lebih mudah diamati.

---

# 9. Mengubah Xarray menjadi DataFrame

Model Random Forest pada Scikit-learn menerima data dalam bentuk tabel. Oleh karena itu, dataset Xarray perlu diubah menjadi DataFrame menggunakan `to_dataframe()`.

## 9.1 Konversi ke DataFrame

```python
# Mengubah seluruh variabel Xarray menjadi tabel
df_dataset = ds.to_dataframe()

# Menghapus baris yang seluruh kolomnya bernilai kosong
df_dataset = df_dataset.dropna(how='all')

# Memeriksa hasil konversi
df_dataset.head()
```

- `to_dataframe()` mengubah variabel pada Xarray menjadi kolom-kolom DataFrame.
- Koordinat spasial seperti `x` dan `y` ikut menjadi informasi indeks atau kolom sesuai struktur dataset.
- `dropna(how='all')` menghapus baris yang seluruh nilainya kosong. Baris yang hanya memiliki sebagian nilai kosong masih dapat tersisa.

## 9.2 Menyimpan DataFrame sebagai CSV

```python
# Menentukan lokasi penyimpanan CSV di Google Drive
csv_path = os.path.join(ROOT_PATH, 'Satellite_Embedding_Biomass.csv')

# Menyimpan tabel ke format CSV
df_dataset.to_csv(
    csv_path,
    index=True
)

print(f'CSV saved: {csv_path}')
```

- `os.path.join()` menggabungkan direktori utama dengan nama file.
- `to_csv()` menyimpan tabel ke format CSV.
- `index=True` menyimpan indeks DataFrame. Pada dataset spasial, indeks dapat memuat koordinat `x` dan `y`, sehingga informasi lokasi tetap dapat dipertahankan.

---

# 10. Persiapan data training dan testing

Tahap pemodelan dimulai dengan memisahkan variabel prediktor dan variabel target. Dalam praktikum ini:

- **Prediktor (X):** seluruh band Google Satellite Embedding dan informasi koordinat yang ikut terbawa ke DataFrame.
- **Target (y):** variabel `AGBD` dari GEDI.

## 10.1 Memanggil pustaka

```python
# Library pengolahan data, visualisasi, dan pembagian dataset
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.model_selection import train_test_split
```

## 10.2 Menghapus data yang memiliki nilai kosong

```python
# Menghapus baris yang memiliki setidaknya satu nilai kosong
df_model = df_dataset.dropna()

# Memeriksa jumlah dan struktur data yang digunakan
df_model.info()
```

`dropna()` tanpa argumen tambahan akan menghapus baris yang memiliki nilai kosong pada salah satu kolom. Langkah ini diperlukan karena model Scikit-learn yang digunakan pada tutorial ini tidak menerima nilai NaN secara langsung.

Periksa jumlah data sebelum dan sesudah penghapusan agar dapat diketahui seberapa banyak sampel yang hilang akibat proses tersebut.

## 10.3 Memisahkan prediktor (X)

```python
# Mengambil seluruh kolom prediktor selain variabel target AGBD
X = df_model.drop('AGBD', axis=1)

# Memeriksa variabel prediktor
X.head()
```

`drop('AGBD', axis=1)` menghapus kolom target dari tabel. Seluruh kolom yang tersisa digunakan sebagai variabel masukan model. Dalam notebook ini, koordinat `x` dan `y` dapat ikut menjadi prediktor apabila tersedia sebagai kolom DataFrame.

## 10.4 Memisahkan target (y)

```python
# Mengambil AGBD sebagai variabel target
y = df_model['AGBD']

# Memeriksa nilai target
y.head()
```

Variabel `y` berisi nilai numerik AGBD yang akan dipelajari dan diprediksi oleh model. Karena target berupa nilai kontinu, pendekatan yang digunakan adalah regresi, bukan klasifikasi.

## 10.5 Membagi data training dan testing

```python
# Membagi dataset menjadi data training dan testing
X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.3,     # 30% data untuk pengujian
    random_state=42    # Menjaga agar pembagian dapat direproduksi
)

# Memeriksa jumlah sampel pada setiap subset
print('Jumlah data training:', len(X_train))
print('Jumlah target training:', len(y_train))

print('Jumlah data testing:', len(X_test))
print('Jumlah target testing:', len(y_test))
```

- `train_test_split()` membagi data menjadi dua subset.
- `test_size=0.3` berarti 30% data digunakan untuk testing dan 70% untuk training.
- `random_state=42` menetapkan seed agar pembagian data konsisten ketika kode dijalankan kembali.
- `X_train` dan `y_train` digunakan untuk melatih model.
- `X_test` dan `y_test` digunakan untuk mengukur performa model pada data yang tidak digunakan selama training.

## 10.6 Memeriksa distribusi target

Distribusi nilai target perlu diperiksa untuk mengetahui karakteristik data training dan testing. Notebook menggunakan histogram, boxplot, dan kurva distribusi kumulatif (*cumulative distribution function* atau CDF).

```python
# Membuat tiga panel visualisasi distribusi target
fig, axes = plt.subplots(1, 3, figsize=(16, 4))

# (a) Histogram distribusi AGBD
axes[0].hist(
    y_train,
    bins=40,
    color='#2e7d32',
    edgecolor='white',
    label='Train'
)
axes[0].hist(
    y_test,
    bins=40,
    color='#81c784',
    edgecolor='white',
    alpha=0.6,
    label='Test'
)
axes[0].set_xlabel('GEDI AGBD (Mg/ha)')
axes[0].set_ylabel('Count')
axes[0].set_title('Target Distribution')
axes[0].legend()

# (b) Boxplot untuk membandingkan training dan testing
axes[1].boxplot(
    [y_train, y_test],
    labels=['Train', 'Test'],
    patch_artist=True,
    boxprops=dict(facecolor='#a5d6a7')
)
axes[1].set_ylabel('GEDI AGBD (Mg/ha)')
axes[1].set_title('Train vs Test Boxplot')

# (c) Cumulative Distribution Function (CDF)
sorted_train = np.sort(y_train)
cdf_train = np.arange(1, len(sorted_train) + 1) / len(sorted_train)
axes[2].plot(sorted_train, cdf_train, color='#2e7d32', label='Train')

sorted_test = np.sort(y_test)
cdf_test = np.arange(1, len(sorted_test) + 1) / len(sorted_test)
axes[2].plot(
    sorted_test,
    cdf_test,
    color='#81c784',
    linestyle='--',
    label='Test'
)
axes[2].set_xlabel('GEDI AGBD (Mg/ha)')
axes[2].set_ylabel('Cumulative Probability')
axes[2].set_title('Cumulative Distribution')
axes[2].legend()

# Mengatur tata letak agar panel tidak saling bertumpuk
fig.tight_layout()

# Menyimpan gambar ke Google Drive
fig.savefig(
    os.path.join(ROOT_PATH, 'gedi_target.png'),
    dpi=150,
    bbox_inches='tight'
)

plt.show()

# Menampilkan statistik deskriptif data training dan testing
print(
    f'Train -- Mean: {y_train.mean():.2f}, '
    f'Std: {y_train.std():.2f}, '
    f'Min: {y_train.min():.2f}, '
    f'Max: {y_train.max():.2f}'
)

print(
    f'Test -- Mean: {y_test.mean():.2f}, '
    f'Std: {y_test.std():.2f}, '
    f'Min: {y_test.min():.2f}, '
    f'Max: {y_test.max():.2f}'
)
```

1. **Histogram** memperlihatkan frekuensi nilai AGBD pada training dan testing. Kedua distribusi sebaiknya memiliki pola yang relatif serupa.
2. **Boxplot** membantu membandingkan median, rentang antarkuartil, dan nilai ekstrem.
3. **CDF** memperlihatkan proporsi kumulatif sampel yang memiliki nilai AGBD sampai nilai tertentu.
4. Statistik mean, standar deviasi, minimum, dan maksimum memberikan ringkasan numerik distribusi target.

Jika distribusi training dan testing sangat berbeda, hasil evaluasi model perlu ditafsirkan secara hati-hati.

---

# 11. Hyperparameter tuning Random Forest Regression

Random Forest merupakan algoritma *ensemble learning* yang menggabungkan banyak pohon keputusan (*decision trees*). Pada regresi, hasil prediksi akhir diperoleh dengan menggabungkan prediksi dari pohon-pohon tersebut, lalu nilainya di rata-rata.

Dua parameter tuning yang dipilih pada praktikum ini adalah:

- `n_estimators`: jumlah pohon keputusan yang dibangun.
- `max_features`: jumlah fitur yang dipertimbangkan ketika mencari pemisahan terbaik pada setiap node.

Pemilihan parameter yang sesuai dapat membantu meningkatkan performa model.

## 11.1 Memanggil pustaka

```python
# Library untuk tuning dan pemodelan Random Forest Regression
import time
import numpy as np
import matplotlib.pyplot as plt

from sklearn.model_selection import cross_val_score
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import r2_score, mean_squared_error
from scipy.stats import pearsonr
```

## 11.2 Membuat fungsi hyperparameter tuning

Pada notebook, pencarian parameter dilakukan dengan menguji kombinasi `max_features` dan `n_estimators`. Setiap kombinasi dievaluasi menggunakan 5-fold cross-validation dengan metrik R².

```python
def tuning_model(
    regressor,
    X_train,
    y_train,
    save_csv_path='tuning_result_reg_rf.csv'
):
    # Daftar jumlah fitur yang diuji pada setiap split
    p_list = np.array([1, 2, 3, 4, 5])

    # Daftar jumlah pohon keputusan yang diuji
    n_list = np.array([100, 200, 300, 400, 500])

    # Inisialisasi skor terbaik
    best_score = -np.inf
    best_params = {}

    # Menguji setiap kombinasi parameter
    for p in p_list:
        for n in n_list:
            start_time = time.time()

            # Mengubah parameter model sesuai kombinasi yang diuji
            regressor.set_params(
                max_features=p,
                n_estimators=n
            )

            # Evaluasi menggunakan 5-fold cross-validation
            cv_scores = cross_val_score(
                regressor,
                X_train,
                y_train,
                scoring='r2',
                cv=5,
                n_jobs=-1
            )

            # Menghitung waktu komputasi dalam menit
            end_time = time.time()
            elapsed_time = (end_time - start_time) / 60

            # Menghitung rata-rata skor R2 dari seluruh fold
            cv_avgscore = cv_scores.mean()

            # Memperbarui parameter terbaik jika skor meningkat
            if cv_avgscore > best_score:
                best_score = cv_avgscore
                best_params = {
                    'max_features': p,
                    'n_estimators': n
                }

            # Menampilkan hasil pengujian kombinasi parameter
            print(
                f'max_features={p}, '
                f'n_estimators={n}, '
                f'R2={cv_avgscore:.4f}, '
                f'time={elapsed_time:.2f} min'
            )

    print('Best Parameters:', best_params)
    print('Best CV R2 Score:', best_score)

    return best_params, best_score
```

- `p_list` berisi kandidat nilai `max_features`.
- `n_list` berisi kandidat jumlah pohon (`n_estimators`).
- Dua perulangan `for` membentuk seluruh kombinasi parameter yang akan diuji. Dalam contoh ini terdapat 5 × 5 = 25 kombinasi.
- `set_params()` mengatur parameter model sebelum evaluasi.
- `cross_val_score()` melakukan evaluasi cross-validation. Dengan `cv=5`, data training dibagi menjadi lima bagian. Model dilatih dan dievaluasi secara bergantian menggunakan bagian yang berbeda.
- `scoring='r2'` menetapkan R² sebagai metrik evaluasi.
- `n_jobs=-1` menggunakan seluruh core CPU yang tersedia.
- `best_score` menyimpan skor rata-rata terbaik, sedangkan `best_params` menyimpan kombinasi parameter yang menghasilkan skor tersebut.

### 11.3 Menjalankan proses tuning

```python
# Membuat model Random Forest Regression
rf = RandomForestRegressor(
    oob_score=True,
    random_state=42
)

# Menjalankan pencarian parameter terbaik
tuning_model(rf, X_train, y_train)
```

- `RandomForestRegressor()` membuat objek model Random Forest untuk tugas regresi.
- `oob_score=True` mengaktifkan perhitungan *out-of-bag score*. Fitur ini memanfaatkan sampel yang tidak terpilih pada proses bootstrap untuk memperkirakan performa model.
- `random_state=42` menjaga konsistensi proses acak.
- Fungsi `tuning_model()` menguji seluruh kombinasi parameter dan mengembalikan kombinasi dengan skor cross-validation tertinggi.

> `oob_score=True` tidak menggantikan evaluasi pada data testing. Keduanya memiliki mekanisme evaluasi yang berbeda.

---

# 12. Training final model dan evaluasi

Setelah memperoleh parameter terbaik, model dilatih kembali menggunakan seluruh data training. Selanjutnya, model digunakan untuk memprediksi data training dan testing.

Evaluasi dilakukan menggunakan tiga metrik utama:

1. **R² (Coefficient of Determination):** mengukur proporsi variasi target yang dapat dijelaskan oleh model.
2. **RMSE (Root Mean Squared Error):** mengukur besarnya kesalahan prediksi dalam satuan yang sama dengan target.
3. **Pearson correlation (r):** mengukur kekuatan hubungan linear antara nilai observasi dan prediksi.

## 12.1 Membuat fungsi training dan evaluasi

```python
# Library evaluasi model dan pengolahan feature importance
from sklearn.metrics import root_mean_squared_error
from scipy.stats import pearsonr
import pandas as pd
import numpy as np

def final_model(regressor, X_train, y_train, X_test, y_test):

    # Melatih model menggunakan data training
    regressor.fit(X_train, y_train)

    # Menghitung R2 pada data training dan testing
    r2_train = regressor.score(X_train, y_train)
    r2_test = regressor.score(X_test, y_test)

    # Menghasilkan prediksi pada data training dan testing
    y_train_pred = regressor.predict(X_train)
    y_test_pred = regressor.predict(X_test)

    # Menghitung RMSE pada data testing
    rmse_test = root_mean_squared_error(y_test, y_test_pred)

    # Menghitung korelasi Pearson antara observasi dan prediksi
    rxy, _ = pearsonr(y_test, y_test_pred)

    # Menampilkan hasil evaluasi
    print(f'Training set R² = {r2_train:.2f}')
    print(f'Testing set R² = {r2_test:.2f}')
    print(f'RMSE of the testing set = {rmse_test:.2f}')
    print(f'Pearson r between observation and prediction = {rxy:.2f}')

    # Membuat tabel feature importance
    df_importance = pd.DataFrame({
        'Predictor': X_train.columns,
        'Importance': regressor.feature_importances_
    })

    # Mengurutkan prediktor dari nilai importance tertinggi
    df_importance = df_importance.sort_values(
        by='Importance',
        ascending=False
    ).reset_index(drop=True)

    print('\nTop 10 Predictor Importance:')
    print(df_importance.head(10).to_string(index=False))

    return y_train_pred, y_test_pred, df_importance
```

- `fit()` melatih model menggunakan pasangan data prediktor dan target.
- `score()` pada `RandomForestRegressor` menghasilkan nilai R².
- `predict()` menghasilkan nilai prediksi AGBD.
- `root_mean_squared_error()` menghitung akar dari rata-rata kuadrat kesalahan prediksi.
- `pearsonr()` menghasilkan koefisien korelasi Pearson dan nilai signifikansinya.
- `feature_importances_` mengambil nilai kepentingan relatif setiap fitur berdasarkan model Random Forest.
- `sort_values()` mengurutkan fitur dari nilai importance tertinggi.

Interpretasi metrik perlu dilakukan bersama-sama. R² yang tinggi tidak otomatis berarti kesalahan prediksi kecil. RMSE menunjukkan besarnya kesalahan dalam satuan AGBD, sedangkan korelasi Pearson menunjukkan kekuatan hubungan linear antara observasi dan prediksi.

## 12.2 Melatih final Random Forest

Hasil tuning memberikan parameter terbaik yaitu `max_features=5` dan `n_estimators=500`. Pastikan nilai tersebut sesuai dengan hasil tuning yang diperoleh pada saat notebook dijalankan.

```python
# Membuat model Random Forest dengan parameter yang digunakan pada notebook
regr_rf = RandomForestRegressor(
    max_features=5,
    n_estimators=500,
    oob_score=True,
    random_state=42
)

# Melatih model dan mengevaluasi performanya
rf_y_train_pred, rf_y_test_pred, rf_importance = final_model(
    regr_rf,
    X_train,
    y_train,
    X_test,
    y_test
)
```

- `max_features=5` berarti model mempertimbangkan lima fitur pada proses pencarian split setiap node.
- `n_estimators=500` membangun 500 pohon keputusan.
- `oob_score=True` mengaktifkan evaluasi out-of-bag.
- `random_state=4` menetapkan seed proses acak pada model akhir.
- Fungsi `final_model()` mengembalikan prediksi training, prediksi testing, dan tabel feature importance.

---

# 13. Visualisasi hasil evaluasi model

## 13.1 Scatter plot actual versus predicted

Garis diagonal 1:1 menunjukkan kondisi ideal ketika nilai prediksi sama dengan nilai observasi. Titik yang semakin dekat dengan garis tersebut menunjukkan prediksi yang semakin mendekati nilai referensi.

```python
# Menghitung metrik berdasarkan prediksi testing yang aktual
from sklearn.metrics import r2_score, root_mean_squared_error
from scipy.stats import pearsonr

r2_val = r2_score(y_test, rf_y_test_pred)
rmse_val = root_mean_squared_error(y_test, rf_y_test_pred)
pearson_r, _ = pearsonr(y_test, rf_y_test_pred)

# Menyiapkan informasi metrik untuk ditampilkan pada legenda
stats_label = (
    f'$R^2$: {r2_val:.2f}\n'
    f'RMSE: {rmse_val:.2f} Mg/ha\n'
    f'r: {pearson_r:.2f}'
)

plt.figure(figsize=(8, 8))

# Membandingkan nilai aktual dan prediksi
plt.scatter(
    y_test,
    rf_y_test_pred,
    alpha=0.6,
    color='#2ecc71',
    edgecolor='white',
    s=50
)

# Membuat garis referensi 1:1
min_val = min(y_test.min(), rf_y_test_pred.min())
max_val = max(y_test.max(), rf_y_test_pred.max())

plt.plot(
    [min_val, max_val],
    [min_val, max_val],
    'r--',
    lw=2,
    label=stats_label
)

plt.legend(loc='upper left', fontsize=12, frameon=True, edgecolor='black')
plt.xlabel('Actual Biomass (GEDI) [Mg/ha]')
plt.ylabel('Predicted Biomass (RF) [Mg/ha]')
plt.title('Performance on Test Set: RF Model')
plt.grid(True, linestyle='--', alpha=0.7)
plt.show()
```

- Sumbu X menunjukkan nilai AGBD aktual dari GEDI.
- Sumbu Y menunjukkan nilai AGBD hasil prediksi Random Forest.
- `plt.scatter()` menampilkan pasangan nilai aktual dan prediksi sebagai titik.
- Garis 1:1 menjadi garis referensi untuk membandingkan kedua nilai.
- R², RMSE, dan Pearson r dihitung langsung dari `y_test` dan `rf_y_test_pred`.

## 13.2 Visualisasi feature importance

Random Forest menyediakan informasi mengenai kontribusi relatif setiap fitur dalam proses prediksi. Pada tutorial ini, sepuluh fitur dengan nilai importance tertinggi ditampilkan dalam diagram batang horizontal.

```python
# Memilih sepuluh prediktor dengan importance tertinggi
top10 = rf_importance.head(10).sort_values(
    by='Importance',
    ascending=True
)

fig, ax = plt.subplots(figsize=(7, 6))

# Membuat diagram batang horizontal
bars = ax.barh(
    top10['Predictor'],
    top10['Importance'] * 100,
    color='#2e7d32',
    edgecolor='white',
    height=0.7
)

# Menambahkan nilai importance pada setiap batang
for bar in bars:
    width = bar.get_width()

    ax.text(
        width + 0.05,
        bar.get_y() + bar.get_height() / 2,
        f'{width:.2f}%',
        va='center',
        fontsize=9
    )

ax.set_xlabel('Feature Importance (%)', fontsize=11)
ax.set_ylabel('Predictors', fontsize=11)
ax.set_title('Top 10 Feature Importance - Random Forest', fontsize=13, pad=15)
ax.grid(axis='x', linestyle='--', alpha=0.5)
ax.set_axisbelow(True)
ax.spines['top'].set_visible(False)
ax.spines['right'].set_visible(False)

plt.tight_layout()
plt.show()
```

- `head(10)` mengambil sepuluh prediktor dengan importance tertinggi.
- `sort_values(..., ascending=True)` mengurutkan kembali data agar batang dengan nilai tertinggi berada di bagian atas diagram.
- `barh()` membuat diagram batang horizontal.
- `feature_importances_` pada Random Forest menghasilkan nilai relatif yang jumlahnya secara umum sama dengan 1. Oleh karena itu, dikalikan 100 agar ditampilkan dalam persentase.

> Feature importance menunjukkan kontribusi relatif fitur menurut mekanisme model, bukan hubungan sebab-akibat. Nilai importance juga dapat terbagi di antara fitur yang saling berkorelasi.

## 13.3 Menggabungkan scatter plot dan feature importance

Kedua visualisasi dapat disajikan dalam satu gambar untuk memberikan ringkasan performa model dan prediktor yang paling berpengaruh.

```python
# Membuat figure dengan dua panel
fig, ax = plt.subplots(1, 2, figsize=(16, 7))

# Menghitung metrik dari prediksi testing
r2_val = r2_score(y_test, rf_y_test_pred)
rmse_val = root_mean_squared_error(y_test, rf_y_test_pred)
pearson_r, _ = pearsonr(y_test, rf_y_test_pred)

stats_label = (
    f'$R^2$: {r2_val:.2f}\n'
    f'RMSE: {rmse_val:.2f} Mg/ha\n'
    f'r: {pearson_r:.2f}'
)

# Panel (a): scatter plot actual vs predicted
ax[0].scatter(
    y_test,
    rf_y_test_pred,
    alpha=0.6,
    color='#2ecc71',
    edgecolor='white',
    s=40
)

# Garis referensi 1:1
min_val = min(y_test.min(), rf_y_test_pred.min())
max_val = max(y_test.max(), rf_y_test_pred.max())

ax[0].plot(
    [min_val, max_val],
    [min_val, max_val],
    'r--',
    lw=2,
    label=stats_label
)

ax[0].legend(loc='upper left', fontsize=10, frameon=True, edgecolor='black')
ax[0].set_xlabel('Actual Biomass (GEDI) [Mg/ha]', fontsize=10)
ax[0].set_ylabel('Predicted Biomass (RF) [Mg/ha]', fontsize=10)
ax[0].set_title('(a) Actual vs Predicted Biomass', fontsize=12)
ax[0].grid(True, linestyle='--', alpha=0.5)

# Panel (b): sepuluh fitur dengan importance tertinggi
top10 = rf_importance.head(10).sort_values(
    by='Importance',
    ascending=True
)

bars = ax[1].barh(
    top10['Predictor'],
    top10['Importance'] * 100,
    color='#2e7d32',
    edgecolor='white',
    height=0.7
)

# Menambahkan label persentase pada batang
for bar in bars:
    width = bar.get_width()
    ax[1].text(
        width + 0.05,
        bar.get_y() + bar.get_height() / 2,
        f'{width:.2f}%',
        va='center',
        fontsize=9
    )

ax[1].set_xlabel('Feature Importance (%)', fontsize=10)
ax[1].set_ylabel('Predictors', fontsize=10)
ax[1].set_title('(b) Top 10 Feature Importance', fontsize=12)
ax[1].grid(axis='x', linestyle='--', alpha=0.5)
ax[1].set_axisbelow(True)
ax[1].spines['top'].set_visible(False)
ax[1].spines['right'].set_visible(False)

# Mengatur tata letak akhir
plt.tight_layout(pad=2.0)

# Menyimpan gambar dengan resolusi 300 dpi
plt.savefig(
    os.path.join(ROOT_PATH, 'RF_Model_Performance.png'),
    dpi=300,
    bbox_inches='tight'
)

plt.show()
```

`plt.subplots(1, 2)` membuat dua panel dalam satu baris. Panel pertama menampilkan hubungan nilai aktual dan prediksi, sedangkan panel kedua menampilkan sepuluh prediktor paling penting. `dpi=300` digunakan agar gambar memiliki resolusi yang memadai untuk dokumentasi atau publikasi.

---

# 14. Spatial inference: menghasilkan peta prediksi AGBD

Setelah model selesai dilatih dan dievaluasi, model dapat diterapkan pada data prediktor di seluruh area kajian. Proses penerapan model pada data spasial untuk menghasilkan prediksi disebut *spatial inference*.

Pada tahap ini, variabel target `AGBD` tidak lagi digunakan sebagai masukan. Model hanya menerima variabel prediktor yang sama seperti ketika model dilatih.

## 14.1 Menyiapkan data prediksi

Pada notebook awal, data prediksi dibuat dengan menghapus kolom `AGBD`. Namun, data raster dapat memiliki nilai kosong pada sebagian lokasi. Oleh sebab itu, sebelum prediksi perlu dipastikan bahwa seluruh prediktor pada baris yang akan diprediksi memiliki nilai valid.

```python
# Menghapus kolom target AGBD sehingga hanya tersisa variabel prediktor
df_inference = df_dataset.drop(columns=['AGBD'])
df_inference['abgd_predic'] =regr_rf.predict(df_inference)
```

- `drop(columns=['AGBD'])` menghilangkan variabel target.
- `dropna()` memilih baris yang tidak memiliki nilai kosong.
- `predict()` menghasilkan estimasi AGBD berdasarkan fitur yang diberikan.
- Hasil prediksi disimpan pada kolom `abgd_predic`.

## 14.2 Memeriksa nilai minimum dan maksimum prediksi

```python
# Mengambil nilai minimum hasil prediksi
min_val = df_inference['abgd_predic'].min()

# Mengambil nilai maksimum hasil prediksi
max_val = df_inference['abgd_predic'].max()

print(f'Min Biomass: {min_val:.2f} Mg/ha')
print(f'Max Biomass: {max_val:.2f} Mg/ha')
```

Nilai minimum dan maksimum memberikan gambaran rentang hasil prediksi AGBD pada wilayah kajian. Nilai ini dapat digunakan untuk membantu menentukan rentang visualisasi peta.

## 14.3 Mengubah hasil prediksi kembali menjadi Xarray

DataFrame yang memiliki indeks koordinat spasial dapat dikonversi kembali menjadi Xarray agar dapat divisualisasikan sebagai raster.

```python
# Mengubah DataFrame hasil prediksi menjadi Xarray Dataset
data_inference = (
    df_inference
    .to_xarray()
    .sortby('x')
    .sortby('y')
)

# Memeriksa struktur data prediksi
data_inference
```

- `to_xarray()` mengubah DataFrame menjadi Xarray Dataset.
- `sortby('x')` dan `sortby('y')` mengurutkan koordinat spasial.
- Pengurutan koordinat membantu menyusun data secara konsisten untuk visualisasi dan ekspor raster.

### 14.4 Visualisasi peta prediksi AGBD

```python
# Visualisasi spasial hasil prediksi AGBD
data_inference.abgd_predic.plot(
    x='x',
    y='y',
    cmap='Greens',
    robust=True,
    cbar_kwargs={'label': 'AGBD (Mg/ha)'}
)

plt.title('Predicted Aboveground Biomass Density', fontsize=10)
plt.xlabel('Longitude')
plt.ylabel('Latitude')
plt.show()
```

- `data_inference.abgd_predic` memilih variabel hasil prediksi.
- `cmap='Greens'` menggunakan gradasi warna hijau.
- `robust=True` membantu mengatur rentang warna agar tidak terlalu dipengaruhi nilai ekstrem.
- `cbar_kwargs` memberikan label pada colorbar.
- Sumbu `x` dan `y` menunjukkan koordinat geografis.

Peta ini memperlihatkan variasi spasial AGBD hasil estimasi model Random Forest pada area kajian.

## 14.5 Membandingkan peta prediksi, GEDI, dan Satellite Embedding

Untuk memahami hasil pemodelan, peta prediksi dapat dibandingkan dengan data AGBD referensi dan salah satu band Satellite Embedding.

```python
# Membuat tiga panel peta
fig, ax = plt.subplots(1, 3, figsize=(18, 5))

# (a) Peta hasil prediksi Random Forest
data_inference.abgd_predic.plot(
    ax=ax[0],
    x='x',
    y='y',
    cmap='Greens',
    robust=True,
    cbar_kwargs={
        'label': 'AGBD (Mg/ha)',
        'shrink': 0.8
    }
)
ax[0].set_title('(a) Predicted Forest Aboveground Biomass', fontsize=11)
ax[0].set_xlabel('Longitude')
ax[0].set_ylabel('Latitude')

# (b) Peta AGBD referensi dari GEDI
ds.AGBD.plot(
    ax=ax[1],
    x='x',
    y='y',
    robust=True,
    cbar_kwargs={
        'label': 'AGBD (Mg/ha)',
        'shrink': 0.8
    }
)
ax[1].set_title('(b) GEDI AGBD', fontsize=11)
ax[1].set_xlabel('Longitude')
ax[1].set_ylabel('Latitude')

# (c) Peta salah satu band Satellite Embedding
ds.A01.plot(
    ax=ax[2],
    x='x',
    y='y',
    robust=True,
    cbar_kwargs={
        'label': 'Embedding Value',
        'shrink': 0.8
    }
)
ax[2].set_title('(c) Satellite Embedding A01', fontsize=11)
ax[2].set_xlabel('Longitude')
ax[2].set_ylabel('Latitude')

# Mengatur jarak antar-panel
plt.tight_layout(pad=2)

# Menyimpan hasil visualisasi
plt.savefig(
    os.path.join(ROOT_PATH, 'Biomass_Embedding_GEDI.png'),
    dpi=300,
    bbox_inches='tight'
)

plt.show()
```

- Panel (a) menunjukkan AGBD hasil prediksi Random Forest.
- Panel (b) menunjukkan AGBD dari data GEDI yang digunakan sebagai target.
- Panel (c) menunjukkan nilai salah satu band Satellite Embedding.
- Ketiga panel membantu melihat pola spasial data input, data referensi, dan hasil prediksi.

Perbandingan visual bukan pengganti evaluasi kuantitatif. Perlu diperhatikan pula bahwa rentang warna masing-masing panel dapat berbeda, sehingga kesamaan warna tidak selalu berarti kesamaan nilai.

---

# 15. Menyimpan model Random Forest

Model yang telah dilatih dapat disimpan agar tidak perlu menjalankan proses training kembali setiap kali akan melakukan prediksi. Pada praktikum ini, pustaka `joblib` digunakan untuk menyimpan model dalam format `.pkl`.

## 15.1 Menyimpan model

```python
# Memanggil pustaka untuk menyimpan objek model Python
import joblib

# Menentukan lokasi penyimpanan model
MODEL_PATH = os.path.join(ROOT_PATH, 'Best_RF_Model.pkl')

# Menyimpan model Random Forest yang telah dilatih
joblib.dump(regr_rf, MODEL_PATH)

print(f'Model saved: {MODEL_PATH}')
```

- `joblib` digunakan untuk serialisasi objek Python.
- `MODEL_PATH` menentukan lokasi dan nama file model.
- `joblib.dump()` menyimpan objek model yang telah dilatih, termasuk parameter dan struktur pohon keputusan yang terbentuk.

## 15.2 Memuat kembali model

```python
# Memuat model Random Forest dari file PKL
agb_model = joblib.load(MODEL_PATH)

# Memastikan model berhasil dimuat
print('Model successfully loaded!')

# Memeriksa parameter model
print('Model parameters:', agb_model.get_params())
```

`joblib.load()` membaca kembali model dari file. Model yang telah dimuat dapat digunakan untuk melakukan prediksi tanpa mengulang proses training, selama struktur dan urutan variabel prediktornya sama.

---

# 16. Mengekspor hasil prediksi menjadi GeoTIFF

Hasil prediksi AGBD masih berada dalam struktur Xarray. Agar dapat digunakan dalam perangkat lunak GIS seperti QGIS, hasil tersebut dapat diekspor ke format GeoTIFF menggunakan `rioxarray`.

## 16.1 Memanggil rioxarray

```python
# Memanggil rioxarray untuk pengelolaan informasi spasial raster
import rioxarray
```

## 16.2 Menyimpan raster prediksi

```python
# Memilih variabel hasil prediksi dari Xarray Dataset
ds_save = data_inference['abgd_predic']

# Menambahkan informasi CRS geografis
save_data = ds_save.rio.write_crs('EPSG:4326')

# Menentukan lokasi file keluaran
output_path = os.path.join(ROOT_PATH, 'abgd_predic.tif')

# Menyimpan hasil prediksi sebagai GeoTIFF
save_data.rio.to_raster(output_path)

print(f'GeoTIFF saved: {output_path}')
```

**Penjelasan:**

- `data_inference['abgd_predic']` memilih variabel hasil prediksi.
- `rio.write_crs('EPSG:4326')` menambahkan informasi sistem koordinat ke objek raster.
- `os.path.join()` membentuk alamat file keluaran di dalam `ROOT_PATH`.
- `rio.to_raster()` mengekspor data menjadi raster GeoTIFF.

**Pemeriksaan setelah ekspor:**

1. Buka file `abgd_predic.tif` di QGIS.
2. Periksa apakah CRS terbaca sebagai EPSG:4326.
3. Periksa posisi raster terhadap batas wilayah kajian.
4. Periksa ukuran piksel, nilai NoData, dan rentang nilai raster.
5. Bandingkan pola raster dengan hasil visualisasi di notebook.

> `write_crs()` hanya menuliskan informasi CRS. Fungsi tersebut tidak melakukan transformasi koordinat. Karena itu, pastikan koordinat `x` dan `y` pada Xarray memang sudah sesuai dengan EPSG:4326 sebelum menuliskan CRS tersebut. Periksa juga transformasi affine dan orientasi sumbu raster setelah ekspor.

---

# 17. Hal-hal yang perlu diperhatikan dalam interpretasi

Beberapa aspek metodologis perlu diperhatikan ketika menggunakan hasil praktikum:

- **Kesesuaian waktu data:** MODIS dan Satellite Embedding pada script menggunakan periode tahun 2020. Periksa periode dan karakteristik temporal data GEDI yang digunakan sebelum menarik kesimpulan.
- **Definisi target:** Pastikan band `MU` pada produk GEDI yang digunakan benar-benar sesuai dengan definisi AGBD dan satuan yang diharapkan. Nama band hasil penggantian nama tidak menggantikan pemeriksaan metadata.
- **Resolusi spasial:** `scale=0.01` dan `grid_scale=(0.01, -0.01)` menggunakan satuan derajat pada EPSG:4326. Jangan menyebutnya sebagai resolusi 0,01 meter.
- **Mask hutan:** Mask berasal dari MODIS yang memiliki resolusi relatif kasar. Batas hasilnya merupakan pendekatan area hutan.
- **Nilai kosong:** Pastikan baris yang digunakan untuk training dan inference tidak mengandung NaN pada seluruh fitur prediktor.
- **Metrik evaluasi:** Gunakan nilai metrik yang dihitung dari prediksi aktual, bukan angka yang ditulis secara manual.
- **Feature importance:** Nilai importance menunjukkan kontribusi relatif menurut model, bukan pengaruh kausal suatu variabel.
- **Validasi raster:** Pastikan CRS, koordinat, resolusi, transformasi affine, dan NoData benar sebelum menggunakan GeoTIFF untuk analisis lanjutan.

---

# 18. Referensi dan dokumentasi

Dokumentasi berikut dapat digunakan untuk mempelajari pustaka dan produk data yang digunakan dalam praktikum.

1. Google Earth Engine Documentation: https://developers.google.com/earth-engine
2. geemap Documentation: https://geemap.org/
3. Xee Documentation: https://xee.readthedocs.io/
4. Xarray Documentation: https://docs.xarray.dev/
5. Scikit-learn Random Forest Regression: https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestRegressor.html
6. Scikit-learn Model Selection: https://scikit-learn.org/stable/model_selection.html
7. rioxarray Documentation: https://corteva.github.io/rioxarray/stable/
8. MODIS MCD12Q1.061: https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MCD12Q1
9. Google Satellite Embedding V1 Annual: https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_SATELLITE_EMBEDDING_V1_ANNUAL
10. GEDI04_B_002: https://developers.google.com/earth-engine/datasets/catalog/LARSE_GEDI_GEDI04_B_002
