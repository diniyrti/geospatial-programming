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
title: Aplikasi Geospasial Interaktif GEE
authors:
  - name: Diniyarti
---


:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

# Pengembangan Antarmuka Aplikasi Geospasial Interaktif untuk Estimasi Aboveground Biomass Density (AGBD)

Praktikum ini memperkenalkan pengembangan aplikasi geospasial interaktif
menggunakan Google Earth Engine (GEE). Studi kasus yang digunakan adalah
pemetaan dan estimasi *Aboveground Biomass Density* (AGBD) pada area
hutan di Provinsi Sulawesi Tenggara tahun 2020.

> **Catatan:** Kode sumber praktikum menggunakan JavaScript pada Google
> Earth Engine Code Editor. Kode lengkap tetap menjadi bahan utama
> praktikum. Penjelasan di bawah mengikuti urutan blok pada kode
> tersebut.

------------------------------------------------------------------------

# 1. Menentukan Area of Interest (AOI)
Menentukan batas wilayah analisis, yaitu Provinsi Sulawesi Tenggara. Batas administrasi diambil dari dataset FAO GAUL 2015 level 1.

```javascript
var aoi_gaul = ee.FeatureCollection('FAO/GAUL/2015/level1');

print('Daftar properti:', aoi_gaul.first().propertyNames());

var indonesia = aoi_gaul.filter(
  ee.Filter.eq('ADM0_NAME', 'Indonesia')
);

print(
  'Daftar Provinsi di Indonesia:',
  indonesia.aggregate_array('ADM1_NAME')
);

var aoi_sul_tengg = aoi_gaul.filter(
  ee.Filter.eq('ADM1_NAME', 'Sulawesi Tenggara')
).geometry();

print('Geometri AOI Sulawesi Tenggara:', aoi_sul_tengg);

Map.addLayer(aoi_sul_tengg, {}, 'Southeast Sulawesi Province');
Map.centerObject(aoi_sul_tengg, 7);
```

### 1.1 Memanggil dan memeriksa dataset administrasi

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.FeatureCollection()` | Memanggil koleksi fitur geografis dari katalog Earth Engine. |
| `'FAO/GAUL/2015/level1'` | ID dataset batas administrasi GAUL 2015 level 1. |
| `aoi_gaul` | Variabel yang menyimpan fitur batas administrasi. |
| `aoi_gaul.first()` | Mengambil fitur pertama untuk pemeriksaan atribut. |
| `.propertyNames()` | Mengambil daftar nama properti atau atribut pada fitur. |
| `print()` | Menampilkan hasil pemeriksaan pada Console. |

### 1.2 Memfilter Indonesia dan menampilkan daftar provinsi

| Kode atau komponen | Penjelasan |
|---|---|
| `aoi_gaul.filter()` | Menyaring fitur berdasarkan kondisi tertentu. |
| `ee.Filter.eq()` | Membuat kondisi filter sama dengan (*equal*). |
| `'ADM0_NAME'` | Properti yang menyimpan nama negara. |
| `'Indonesia'` | Nama negara yang dipilih. |
| `aggregate_array('ADM1_NAME')` | Mengumpulkan nilai nama provinsi di Indonesia dari seluruh fitur menjadi array. |

Blok ini menampilkan daftar provinsi agar nama wilayah yang digunakan pada proses filter berikutnya dapat diperiksa.

### 1.3 Memilih AOI Sulawesi Tenggara

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.Filter.eq('ADM1_NAME', 'Sulawesi Tenggara')` | Memilih fitur yang nilai `ADM1_NAME`-nya sama dengan Sulawesi Tenggara. |
| `.geometry()` | Mengambil geometri dari fitur hasil filter. |
| `aoi_sul_tengg` | Variabel yang menyimpan geometri AOI. |
| `Map.addLayer()` | Menambahkan geometri AOI ke peta. |
| `{}` | Parameter visualisasi kosong sehingga GEE menggunakan tampilan default. |
| `'Southeast Sulawesi Province'` | Nama layer pada daftar layer peta. |
| `Map.centerObject()` | Memusatkan tampilan peta pada objek tertentu. |
| `7` | Tingkat zoom peta. |

---

# 2. Mengidentifikasi Area Hutan Menggunakan MODIS Land Cover

Mengambil produk MODIS Land Cover tahunan untuk tahun 2020, memilih band klasifikasi `LC_Type1`, dan membatasi citra pada AOI.

```javascript
var modis_lc = ee.ImageCollection('MODIS/061/MCD12Q1')
  .filterDate('2020-01-01', '2021-01-01')
  .select('LC_Type1')
  .first()
  .clip(aoi_sul_tengg);

var vis_params = {
  min: 1,
  max: 17,
  palette: [
    '05450a',  // 1 Evergreen Needleleaf Forest
    '086a10',  // 2 Evergreen Broadleaf Forest
    '54a708',  // 3 Deciduous Needleleaf Forest
    '78d203',  // 4 Deciduous Broadleaf Forest
    '009900',  // 5 Mixed Forest
    'c6b044',  // 6 Closed Shrublands
    'dcd159',  // 7 Open Shrublands
    'dade48',  // 8 Woody Savannas
    'fbff13',  // 9 Savannas
    'b6ff05',  // 10 Grasslands
    '27ff87',  // 11 Permanent Wetlands
    'c24f44',  // 12 Croplands
    'a5a5a5',  // 13 Urban and Built-up
    'ff6d4c',  // 14 Cropland/Natural Vegetation Mosaic
    '69fff8',  // 15 Snow and Ice
    'f9ffa4',  // 16 Barren or Sparsely Vegetated
    '1c0dff'   // 17 Water Bodies
  ]
};

Map.addLayer(modis_lc, vis_params, 'MODIS Land Cover 2020');
Map.centerObject(aoi_sul_tengg, 7);
```

### 2.1 Memanggil dan memfilter koleksi MODIS

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.ImageCollection()` | Memanggil koleksi citra dari katalog Earth Engine. |
| `'MODIS/061/MCD12Q1'` | ID produk MODIS Land Cover Type Yearly versi 6.1. |
| `.filterDate('2020-01-01', '2021-01-01')` | Memilih citra dalam rentang tahun 2020. Tanggal akhir tidak termasuk dalam rentang. |
| `.select('LC_Type1')` | Memilih band `LC_Type1`, yaitu klasifikasi tutupan lahan berdasarkan skema IGBP. |
| `.first()` | Mengambil citra pertama dari koleksi yang telah difilter. |
| `.clip(aoi_sul_tengg)` | Memotong citra mengikuti geometri AOI. |
| `modis_lc` | Menyimpan citra MODIS hasil pemilihan band dan pemotongan wilayah. |

### 2.2 Menentukan parameter visualisasi

| Parameter | Penjelasan |
|---|---|
| `min: 1` | Nilai minimum kelas yang divisualisasikan. |
| `max: 17` | Nilai maksimum kelas yang divisualisasikan. |
| `palette` | Daftar warna untuk merepresentasikan kelas tutupan lahan. Urutan warna mengikuti urutan nilai kelas pada kode. |

Komentar pada setiap warna menunjukkan nama kelas IGBP yang direpresentasikan.

### 2.3 Menampilkan layer

`Map.addLayer()` menambahkan citra MODIS ke peta. Argumen pertama adalah citra, argumen kedua parameter visualisasi, dan argumen ketiga nama layer. `Map.centerObject()` memusatkan peta pada AOI.

---

# 3. Membuat Forest Mask

Membuat mask biner yang mempertahankan kelas hutan IGBP 1 sampai 5.

```javascript
var forest_mask = modis_lc.gte(1)
  .and(modis_lc.lte(5))
  .rename('Forest');

Map.addLayer(forest_mask, {}, 'Forest Mask');
```

### 3.1 Penjelasan blok dan baris kode

| Kode atau komponen | Penjelasan |
|---|---|
| `modis_lc.gte(1)` | Menghasilkan kondisi benar untuk piksel dengan nilai kelas lebih besar atau sama dengan 1. |
| `.lte(5)` | Menghasilkan kondisi benar untuk piksel dengan nilai kelas lebih kecil atau sama dengan 5. |
| `.and()` | Menggabungkan dua kondisi. Keduanya harus terpenuhi. |
| `.rename('Forest')` | Mengganti nama band hasil mask menjadi `Forest`. |
| `forest_mask` | Citra Boolean yang menandai piksel kelas 1 sampai 5. |
| `Map.addLayer()` | Menampilkan hasil mask pada peta. |

---

# 4. Melakukan Masking Area Non-Hutan

Menerapkan forest mask pada citra MODIS sehingga piksel non-hutan menjadi tidak valid atau tersembunyi.

```javascript
var forest_only = modis_lc.updateMask(forest_mask);

Map.addLayer(forest_only, {'palette': '006400'}, 'Forest Only');
```

### 4.1 Penjelasan blok dan baris kode

| Kode atau komponen | Penjelasan |
|---|---|
| `modis_lc` | Citra klasifikasi yang akan diberi mask. |
| `.updateMask(forest_mask)` | Memperbarui mask citra berdasarkan nilai forest mask. |
| `forest_only` | Citra MODIS yang hanya mempertahankan piksel kelas hutan. |
| `Map.addLayer()` | Menampilkan hasil masking pada peta. |
| `palette: '006400'` | Menggunakan warna hijau tua untuk visualisasi piksel yang tersisa. |

**Perbedaan:** `forest_mask` merupakan citra Boolean penanda lokasi hutan, sedangkan `forest_only` merupakan citra kelas MODIS yang telah diberi mask. Nilai kelas asli pada piksel hutan tetap dipertahankan.

---

# 5. Mengubah Raster Hutan Menjadi Vektor
Mengubah raster area hutan menjadi poligon. Geometri hasilnya digunakan untuk membatasi data GEDI dan embedding pada area hutan.

```javascript
var forest_aoi = forest_only.reduceToVectors({
  geometry: aoi_sul_tengg,
  scale: 500,
  geometryType: 'polygon',
  eightConnected: true,
  labelProperty: 'class',
  maxPixels: 1e13
});

var forest_aoi = forest_aoi.geometry();

Map.addLayer(forest_aoi, {}, 'Forest AOI');
```

### 5.1 Parameter reduceToVectors

| Parameter | Fungsi |
|---|---|
| `reduceToVectors()` | Mengubah piksel raster yang memiliki nilai valid menjadi fitur vektor berdasarkan kelompok piksel yang terhubung. |
| `geometry` | Membatasi wilayah tempat proses vektorisasi dilakukan. |
| `scale` | Menentukan skala nominal pemrosesan dalam meter. Nilai 500 digunakan pada kode ini. |
| `geometryType` | Menentukan jenis geometri keluaran, yaitu `polygon`. |
| `eightConnected` | Jika `true`, piksel yang bersinggungan secara diagonal dianggap terhubung. |
| `labelProperty` | Menentukan nama atribut yang menyimpan nilai kelas raster pada fitur keluaran. |
| `maxPixels` | Membatasi jumlah maksimum piksel yang dapat diproses. |

`reduceToVectors()` menghasilkan `FeatureCollection` yang berisi poligon hasil pengelompokan piksel yang terhubung.

### 5.2 Menggabungkan geometri poligon

| Kode atau komponen | Penjelasan |
|---|---|
| `forest_aoi.geometry()` | Mengambil geometri dari fitur-fitur dalam FeatureCollection dan mengembalikannya sebagai satu objek geometri gabungan. |
| `forest_aoi` | Setelah baris tersebut, variabel menyimpan geometri gabungan, bukan lagi FeatureCollection. |
| `Map.addLayer()` | Menampilkan geometri area hutan pada peta. |

**Catatan teknis:** Vektorisasi wilayah luas dapat menghasilkan banyak poligon dan membutuhkan sumber daya komputasi besar. Perhatikan nilai `scale`, `maxPixels`, dan kompleksitas geometri.

---

# 6. Mengambil Data Referensi GEDI AGBD

Mengambil produk GEDI L4B, memilih band `MU`, mengganti nama band menjadi `AGBD`, lalu membatasi data pada geometri area hutan.

```javascript
var gedi_ds = ee.Image('LARSE/GEDI/GEDI04_B_002')
  .select('MU')
  .rename('AGBD')
  .clip(forest_aoi);

var gedi_layer = ui.Map.Layer(
  gedi_ds,
  {min: 0, max: 600, palette: ['ffffb2','fecc5c','fd8d3c','f03b20','bd0026']},
  'GEDI ABGD (Mg/ha)', true, 1
);

Map.layers().add(gedi_layer);

print(gedi_ds, 'gedi-ds');
print(gedi_ds.projection(), 'GEDI Projection');
```

### 6.1 Memanggil dan menyiapkan GEDI

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.Image()` | Memanggil objek citra dari katalog Earth Engine. |
| `'LARSE/GEDI/GEDI04_B_002'` | ID produk GEDI L4B yang digunakan. |
| `.select('MU')` | Memilih band `MU` dari citra GEDI. |
| `.rename('AGBD')` | Mengganti nama band menjadi `AGBD` agar mudah dikenali sebagai variabel target. |
| `.clip(forest_aoi)` | Membatasi citra pada geometri area hutan. |
| `gedi_ds` | Menyimpan citra GEDI hasil pemilihan band dan pemotongan area. |

Penggantian nama band tidak mengubah nilai piksel maupun satuannya. Periksa metadata dan dokumentasi produk untuk memastikan definisi band `MU` dan satuan yang benar sebelum menginterpretasikan hasil.

### 6.2 Membuat layer GEDI

| Kode atau komponen | Penjelasan |
|---|---|
| `ui.Map.Layer()` | Membuat objek layer peta yang dapat dikendalikan visibilitas dan opacity-nya. |
| `gedi_ds` | Citra yang akan ditampilkan. |
| `min: 0` | Nilai minimum rentang visualisasi. |
| `max: 600` | Nilai maksimum rentang visualisasi. |
| `palette` | Palet warna untuk merepresentasikan nilai citra dari rentang minimum ke maksimum. |
| `'GEDI ABGD (Mg/ha)'` | Nama layer pada daftar layer. Teks ini merupakan label yang ditulis pada kode. |
| `true` | Menampilkan layer secara default. |
| `1` | Mengatur opacity penuh. |
| `Map.layers().add(gedi_layer)` | Menambahkan objek layer ke daftar layer peta. |

### 6.3 Memeriksa citra dan proyeksi

- `print(gedi_ds, ...)` menampilkan informasi citra pada Console.
- `gedi_ds.projection()` mengambil informasi proyeksi citra, termasuk CRS dan transformasi grid.
- Informasi proyeksi ini digunakan sebagai acuan pada tahap penyamaan grid spasial.

---

# 7. Mengambil Google Satellite Embedding

Mengambil koleksi Google Satellite Embedding V1 Annual untuk tahun 2020, memilih citra yang mencakup AOI, menggabungkan citra, dan membatasi hasil pada area hutan.

```javascript
var embedding = ee.ImageCollection('GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL')
  .filterDate('2020-01-01', '2021-01-01')
  .filterBounds(aoi_sul_tengg)
  .mosaic()
  .clip(forest_aoi);

print(embedding, 'Embedding Bands');

var vis_embb = {
  bands: ['A00', 'A01', 'A02'],
  min: 0,
  max: 0.3
};

Map.addLayer(embedding, vis_embb, 'Google Satellite Embedding');
```

### 7.1 Memanggil dan memfilter embedding

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.ImageCollection()` | Memanggil koleksi citra dari katalog Earth Engine. |
| `'GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL'` | ID koleksi Google Satellite Embedding V1 Annual. |
| `.filterDate()` | Memilih citra berdasarkan rentang tanggal. |
| `.filterBounds(aoi_sul_tengg)` | Memilih citra yang beririsan dengan AOI. |
| `.mosaic()` | Menggabungkan citra yang tersedia menjadi satu citra mosaik. |
| `.clip(forest_aoi)` | Membatasi citra mosaik pada area hutan. |
| `embedding` | Variabel yang menyimpan citra embedding hasil pemrosesan. |

### 7.2 Memeriksa band dan visualisasi

| Kode atau komponen | Penjelasan |
|---|---|
| `print(embedding, ...)` | Menampilkan informasi citra dan nama band pada Console. |
| `bands: ['A00', 'A01', 'A02']` | Memilih tiga band untuk komposit visualisasi. |
| `min: 0` | Nilai minimum visualisasi. |
| `max: 0.3` | Nilai maksimum visualisasi. |
| `vis_embb` | Variabel yang menyimpan parameter visualisasi. |
| `Map.addLayer()` | Menambahkan citra embedding ke peta. |

Tiga band pada `vis_embb` hanya digunakan untuk tampilan peta. Pada pemodelan, seluruh band dipilih melalui `bandNames()`.

---

# 8. Menyamakan Proyeksi dan Skala Spasial Embedding dengan GEDI

Menyiapkan citra embedding agar menggunakan proyeksi dan skala nominal yang mengacu pada data GEDI.

```javascript
var resample1km = function(image) {

  var projection = gedi_ds.projection();

  var resample = image.reproject({
    crs: projection,
    scale: gedi_ds.projection().nominalScale()
  });

  return ee.Image(
    resample.copyProperties(
      image,
      ['system:time_start', 'system:time_end']
    )
  );
};

var embedding_1km = resample1km(embedding);

Map.addLayer(
  embedding_1km,
  vis_embb,
  'Satellite Embedding 1km'
);
```

### 8.1 Membuat fungsi reproyeksi

| Kode atau komponen | Penjelasan |
|---|---|
| `function(image)` | Membuat fungsi yang menerima sebuah citra sebagai input. |
| `gedi_ds.projection()` | Mengambil objek proyeksi dari citra GEDI. |
| `projection` | Menyimpan proyeksi GEDI sebagai proyeksi target. |
| `image.reproject()` | Meminta Earth Engine memproyeksikan citra ke CRS dan skala yang ditentukan. |
| `crs: projection` | Menggunakan proyeksi GEDI sebagai CRS target. |
| `scale: gedi_ds.projection().nominalScale()` | Menggunakan skala nominal proyeksi GEDI sebagai skala target. |
| `resample` | Menyimpan citra hasil permintaan reproyeksi. |

### 8.2 Mempertahankan properti citra

| Kode atau komponen | Penjelasan |
|---|---|
| `copyProperties(image, ...)` | Menyalin properti terpilih dari citra input ke citra hasil. |
| `system:time_start` | Properti waktu awal citra. |
| `system:time_end` | Properti waktu akhir citra. |
| `ee.Image()` | Memastikan hasil dikembalikan sebagai objek citra. |
| `return` | Mengembalikan citra dari fungsi. |

### 8.3 Menjalankan fungsi

- `resample1km(embedding)` menjalankan fungsi pada citra embedding.
- `embedding_1km` menyimpan hasil reproyeksi.
- `Map.addLayer()` menambahkan hasil ke peta menggunakan parameter visualisasi yang sama.

---

# 9. Menyiapkan Data Training dan Testing

Menggabungkan band embedding sebagai predictor dengan band AGBD GEDI sebagai target, mengambil sampel piksel pada area hutan, lalu membagi sampel menjadi data training dan testing.

```javascript
var predictor_bands = embedding_1km.bandNames();
var predictor = embedding_1km.select(predictor_bands);
var target = gedi_ds;

var stack = predictor.addBands(target);
print('Stack dataset:', stack);

var samples = stack.sample({
  region: forest_aoi,
  projection: gedi_ds.projection(),
  numPixels: 10000,
  seed: 42,
  geometries: false,
  tileScale: 8,
  dropNulls: true
});

print('Total samples:', samples.size());
print('Sample data:', samples.limit(5));

var samplesRandom = samples.randomColumn('random', 42);

var training = samplesRandom.filter(ee.Filter.lt('random', 0.7));
var testing = samplesRandom.filter(ee.Filter.gte('random', 0.7));

print('Training samples:', training.size());
print('Testing samples:', testing.size());
```
### 9.1 Menentukan predictor dan target

| Kode atau komponen | Penjelasan |
|---|---|
| `embedding_1km.bandNames()` | Mengambil nama seluruh band pada citra embedding. |
| `predictor_bands` | Menyimpan daftar nama band predictor. |
| `embedding_1km.select(predictor_bands)` | Memilih seluruh band predictor dari citra embedding. |
| `predictor` | Citra yang berisi variabel input model. |
| `target = gedi_ds` | Menentukan citra GEDI sebagai variabel target. |
| `predictor.addBands(target)` | Menggabungkan predictor dan target menjadi satu citra multiband. |
| `stack` | Citra gabungan yang digunakan untuk sampling. |

Setiap sampel dari `stack` akan berisi nilai band embedding sebagai input dan nilai `AGBD` sebagai target.

### 9.2 Mengambil sampel piksel

| Parameter | Fungsi |
|---|---|
| `region` | Geometri wilayah tempat sampling dilakukan. Pada kode ini menggunakan `forest_aoi`. |
| `projection` | Proyeksi yang digunakan saat sampling, yaitu proyeksi GEDI. |
| `numPixels` | Jumlah piksel sampel yang diminta, yaitu 10.000. |
| `seed` | Nilai awal generator angka acak agar proses dapat direproduksi. |
| `geometries` | Menentukan apakah geometri titik sampel disertakan. `false` berarti geometri tidak disertakan. |
| `tileScale` | Faktor skala tile pemrosesan untuk membantu mengatasi keterbatasan memori. |
| `dropNulls` | Jika `true`, sampel yang memiliki nilai null pada band yang digunakan akan dibuang. |

`stack.sample()` menghasilkan FeatureCollection. Setiap fitur berisi nilai band pada piksel sampel. Jumlah sampel aktual dapat berbeda dari jumlah yang diminta karena mask dan ketersediaan data. Periksa `samples.size()` untuk mengetahui jumlah aktual.

### 9.3 Memeriksa sampel

- `samples.size()` menghitung jumlah fitur hasil sampling.
- `samples.limit(5)` mengambil maksimal lima sampel untuk memeriksa struktur dan atribut.
- `print()` menampilkan hasil pada Console.

### 9.4 Membuat kolom acak

| Kode atau komponen | Penjelasan |
|---|---|
| `randomColumn('random', 42)` | Menambahkan properti acak bernama `random` pada setiap fitur. |
| `'random'` | Nama kolom yang menyimpan nilai acak. |
| `42` | Seed agar proses pengacakan dapat direproduksi. |
| `samplesRandom` | Koleksi sampel yang telah memiliki kolom acak. |

### 9.5 Membagi training dan testing

| Kode atau komponen | Penjelasan |
|---|---|
| `filter()` | Menyaring fitur berdasarkan kondisi. |
| `ee.Filter.lt('random', 0.7)` | Memilih sampel dengan nilai acak kurang dari 0,7, sekitar 70%. |
| `training` | Koleksi sampel untuk melatih model. |
| `ee.Filter.gte('random', 0.7)` | Memilih sampel dengan nilai acak lebih besar atau sama dengan 0,7, sekitar 30%. |
| `testing` | Koleksi sampel untuk menguji model. |
| `training.size()` | Menghitung jumlah sampel training. |
| `testing.size()` | Menghitung jumlah sampel testing. |

---

# 10. Melatih Random Forest Regression

Melatih model Random Forest untuk memprediksi nilai AGBD berdasarkan band-band Google Satellite Embedding. Karena target berupa nilai kontinu, model diatur ke mode regresi.

```javascript
var nTrees = 500;
var variablesPerSplit = 8;

var rfModel = ee.Classifier.smileRandomForest({
  numberOfTrees: nTrees,
  variablesPerSplit: variablesPerSplit,
  seed: 42
}).setOutputMode('REGRESSION');

var trainedRF = rfModel.train({
  features: training,
  classProperty: 'AGBD',
  inputProperties: predictor_bands
});

print('Random Forest Model:', trainedRF);
```
### 10.1 Menentukan parameter model

| Parameter | Fungsi |
|---|---|
| `nTrees` | Variabel yang menyimpan jumlah pohon keputusan dalam ensemble. Nilainya 500. |
| `variablesPerSplit` | Jumlah variabel kandidat yang dipertimbangkan pada setiap split. Nilainya 8. |
| `seed` | Seed untuk proses acak dalam algoritma. |

### 10.2 Membuat konfigurasi Random Forest

| Kode atau komponen | Penjelasan |
|---|---|
| `ee.Classifier.smileRandomForest()` | Membuat model Random Forest menggunakan implementasi SMILE di Earth Engine. |
| `numberOfTrees` | Menentukan jumlah pohon keputusan. |
| `variablesPerSplit` | Menentukan jumlah variabel kandidat pada setiap split. |
| `seed: 42` | Menentukan seed proses acak. |
| `.setOutputMode('REGRESSION')` | Mengatur model untuk menghasilkan nilai kontinu. |
| `rfModel` | Menyimpan konfigurasi model sebelum proses training. |

### 10.3 Melatih model

| Kode atau komponen | Penjelasan |
|---|---|
| `rfModel.train()` | Melatih model menggunakan data training. |
| `features: training` | Menentukan koleksi sampel training. |
| `classProperty: 'AGBD'` | Menentukan properti target numerik yang akan diprediksi. |
| `inputProperties: predictor_bands` | Menentukan daftar band yang digunakan sebagai predictor. |
| `trainedRF` | Menyimpan model Random Forest yang telah dilatih. |
| `print()` | Menampilkan informasi model pada Console. |

Nama parameter API adalah `classProperty`, tetapi pada mode regresi properti yang ditunjuk berisi nilai numerik target `AGBD`, bukan label kelas kategorikal.

---

# 11. Evaluasi Model

Menguji model menggunakan data testing dan menghitung R², RMSE, serta korelasi Pearson.

```javascript
var testPrediction = testing.classify(trainedRF, 'predicted');

var meanObserved = ee.Number(testPrediction.aggregate_mean('AGBD'));

var evaluation = testPrediction.map(function(feature) {
  var observed = ee.Number(feature.get('AGBD'));
  var predicted = ee.Number(feature.get('predicted'));
  var error = observed.subtract(predicted);

  return feature.set({
    squaredError: error.pow(2),
    squaredTotal: observed.subtract(meanObserved).pow(2)
  });
});

var SSE = ee.Number(evaluation.aggregate_sum('squaredError'));
var SST = ee.Number(evaluation.aggregate_sum('squaredTotal'));

var R2 = ee.Number(1).subtract(SSE.divide(SST));

var RMSE = SSE.divide(evaluation.size()).sqrt();

var correlation = evaluation.reduceColumns({
  reducer: ee.Reducer.pearsonsCorrelation(),
  selectors: ['AGBD', 'predicted']
});

print('Number of Trees:', nTrees);
print('Variables per Split:', variablesPerSplit);
print('Testing R²:', R2);
print('Testing RMSE:', RMSE);
print('Pearson r:', correlation.get('correlation'));
```

### 11.1 Menghasilkan prediksi testing

| Kode atau komponen | Penjelasan |
|---|---|
| `testing.classify()` | Menerapkan model pada setiap fitur dalam koleksi testing. |
| `trainedRF` | Model Random Forest yang telah dilatih. |
| `'predicted'` | Nama properti untuk menyimpan hasil prediksi. |
| `testPrediction` | Koleksi testing yang berisi nilai observasi dan prediksi. |

### 11.2 Menghitung rata-rata observasi

| Kode atau komponen | Penjelasan |
|---|---|
| `aggregate_mean('AGBD')` | Menghitung rata-rata nilai properti AGBD pada data testing. |
| `ee.Number()` | Memastikan hasil direpresentasikan sebagai objek angka Earth Engine. |
| `meanObserved` | Menyimpan rata-rata AGBD observasi pada data testing. |

### 11.3 Menghitung residual

| Kode atau komponen | Penjelasan |
|---|---|
| `testPrediction.map()` | Memproses setiap fitur pada koleksi hasil prediksi. |
| `feature.get('AGBD')` | Mengambil nilai AGBD observasi. |
| `feature.get('predicted')` | Mengambil nilai AGBD hasil prediksi. |
| `ee.Number()` | Mengubah nilai menjadi objek angka Earth Engine. |
| `observed.subtract(predicted)` | Menghitung selisih nilai observasi dan prediksi atau residual. |
| `error.pow(2)` | Mengkuadratkan residual. |
| `observed.subtract(meanObserved).pow(2)` | Menghitung kuadrat selisih observasi terhadap rata-rata observasi. |
| `feature.set()` | Menambahkan properti evaluasi pada fitur. |
| `squaredError` | Menyimpan kuadrat kesalahan prediksi. |
| `squaredTotal` | Menyimpan kuadrat selisih observasi terhadap rata-rata. |
| `evaluation` | Koleksi yang telah dilengkapi properti untuk perhitungan metrik. |

### 11.4 Menghitung SSE dan SST

| Kode atau komponen | Penjelasan |
|---|---|
| `aggregate_sum('squaredError')` | Menjumlahkan seluruh kuadrat kesalahan prediksi. |
| `SSE` | *Sum of Squared Errors*, yaitu jumlah kuadrat residual. |
| `aggregate_sum('squaredTotal')` | Menjumlahkan kuadrat selisih observasi terhadap rata-rata observasi. |
| `SST` | *Total Sum of Squares*, yaitu jumlah kuadrat variasi nilai observasi terhadap rata-ratanya. |

### 11.5 Menghitung R²

Rumus:

$$
R^2 = 1 - \frac{SS_{res}}{SS_{tot}}
$$

`SSE.divide(SST)` menghitung rasio SSE terhadap SST. `ee.Number(1).subtract(...)` menghitung satu dikurangi rasio tersebut.

R² mendekati 1 menunjukkan bahwa model menjelaskan variasi data dengan baik. R² dapat bernilai negatif jika kesalahan model lebih besar daripada prediksi menggunakan rata-rata observasi sebagai baseline.

### 11.6 Menghitung RMSE

Rumus:

$$
RMSE = \sqrt{\frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2}
$$

`evaluation.size()` menghitung jumlah sampel evaluasi. `SSE.divide(evaluation.size())` menghitung rata-rata kuadrat kesalahan, sedangkan `.sqrt()` menghitung akar kuadratnya. RMSE memiliki satuan yang sama dengan variabel target.

### 11.7 Menghitung korelasi Pearson

| Kode atau komponen | Penjelasan |
|---|---|
| `reduceColumns()` | Melakukan reduksi pada properti fitur dalam koleksi. |
| `ee.Reducer.pearsonsCorrelation()` | Menghitung korelasi Pearson antara dua variabel. |
| `selectors: ['AGBD', 'predicted']` | Menentukan properti observasi dan prediksi yang dibandingkan. |
| `correlation` | Menyimpan hasil perhitungan korelasi. |
| `correlation.get('correlation')` | Mengambil nilai koefisien korelasi dari hasil reducer. |

Korelasi Pearson memiliki rentang -1 hingga 1. Nilai positif menunjukkan hubungan linear searah, nilai negatif menunjukkan hubungan berlawanan arah, dan nilai mendekati nol menunjukkan hubungan linear yang lemah.

### 11.8 Menampilkan hasil evaluasi

| Kode | Penjelasan |
|---|---|
| `print('Number of Trees:', nTrees)` | Menampilkan jumlah pohon keputusan. |
| `print('Variables per Split:', variablesPerSplit)` | Menampilkan jumlah variabel kandidat per split. |
| `print('Testing R²:', R2)` | Menampilkan R² data testing. |
| `print('Testing RMSE:', RMSE)` | Menampilkan RMSE data testing. |
| `print('Pearson r:', ...)` | Menampilkan koefisien korelasi Pearson. |

---

# 12. Menghitung Feature Importance

Mengambil nilai importance dari model Random Forest untuk mengetahui predictor yang memiliki kontribusi relatif paling besar menurut mekanisme importance model.

```javascript
var importance = ee.Dictionary(
  trainedRF.explain().get('importance')
);

print('Feature Importance:', importance);
```

| Kode atau komponen | Penjelasan |
|---|---|
| `trainedRF.explain()` | Mengambil informasi penjelasan dari model Random Forest yang telah dilatih. |
| `.get('importance')` | Mengambil bagian informasi yang berisi nilai importance setiap predictor. |
| `ee.Dictionary()` | Mengubah hasil menjadi objek dictionary Earth Engine. |
| `importance` | Menyimpan pasangan nama predictor dan nilai importance. |
| `print()` | Menampilkan dictionary pada Console. |

Nilai importance menunjukkan kontribusi relatif predictor menurut mekanisme yang digunakan model. Nilai yang lebih tinggi tidak otomatis menunjukkan hubungan sebab-akibat.

---

# 13. Spatial Inference

Menerapkan model Random Forest terlatih pada seluruh citra predictor untuk menghasilkan peta estimasi AGBD.

```javascript
var biomassPrediction = predictor
  .classify(trainedRF)
  .rename('AGBD_Prediction');

var agb_layer = ui.Map.Layer(
  biomassPrediction,
  {min: 0, max: 600, palette: ['ffffcc','c2e699','78c679','31a354','006837']},
  'AGBD Estimate (Mg/ha)', false, 1
);

Map.layers().add(agb_layer);
```

### 13.1 Menerapkan model pada citra predictor

| Kode atau komponen | Penjelasan |
|---|---|
| `predictor` | Citra multiband yang berisi seluruh variabel predictor. |
| `.classify(trainedRF)` | Menerapkan model Random Forest terlatih pada piksel citra predictor. Karena mode model adalah regresi, hasilnya berupa nilai kontinu. |
| `.rename('AGBD_Prediction')` | Mengganti nama band hasil prediksi. |
| `biomassPrediction` | Citra hasil prediksi spasial AGBD. |

### 13.2 Membuat dan menambahkan layer hasil estimasi

| Kode atau komponen | Penjelasan |
|---|---|
| `ui.Map.Layer()` | Membuat objek layer peta. |
| `biomassPrediction` | Citra hasil prediksi yang akan ditampilkan. |
| `min: 0` | Nilai minimum rentang visualisasi. |
| `max: 600` | Nilai maksimum rentang visualisasi. |
| `palette` | Palet warna untuk merepresentasikan nilai prediksi. |
| `'AGBD Estimate (Mg/ha)'` | Nama layer pada daftar layer. |
| `false` | Layer tidak ditampilkan secara default. |
| `1` | Opacity penuh. |
| `Map.layers().add(agb_layer)` | Menambahkan layer hasil estimasi ke peta. |

**Catatan:** Parameter `min` dan `max` hanya mengatur rentang visualisasi warna, bukan membatasi nilai prediksi yang dihasilkan model.

---

# 14. Menghitung Statistik Distribusi AGBD

Menghitung statistik deskriptif dari citra hasil prediksi AGBD pada area hutan. Statistik yang dihitung meliputi rata-rata, standar deviasi, nilai minimum, nilai maksimum, dan median.

```javascript
var agbd_stats = biomassPrediction.reduceRegion({
  reducer: ee.Reducer.mean()
    .combine({
      reducer2: ee.Reducer.stdDev(),
      sharedInputs: true
    })
    .combine({
      reducer2: ee.Reducer.minMax(),
      sharedInputs: true
    })
    .combine({
      reducer2: ee.Reducer.median(),
      sharedInputs: true
    }),
  geometry: forest_aoi,
  scale: gedi_ds.projection().nominalScale(),
  maxPixels: 1e10,
  tileScale: 8
});

print('AGBD stock distribution stats:', agbd_stats);
```

### 14.1 Menghitung statistik dengan reduceRegion()

| Kode atau komponen | Penjelasan |
|---|---|
| `biomassPrediction` | Citra hasil prediksi AGBD yang akan dihitung statistiknya. |
| `.reduceRegion()` | Menghitung statistik nilai piksel pada wilayah geometri yang ditentukan dan mengembalikan hasil sebagai dictionary. |
| `reducer` | Menentukan operasi statistik yang akan diterapkan pada piksel. |
| `ee.Reducer.mean()` | Menghitung nilai rata-rata piksel. |
| `ee.Reducer.stdDev()` | Menghitung standar deviasi nilai piksel. |
| `ee.Reducer.minMax()` | Menghitung nilai minimum dan maksimum piksel. |
| `ee.Reducer.median()` | Menghitung nilai median piksel. |
| `.combine()` | Menggabungkan beberapa reducer agar statistik dihitung dalam satu operasi. |
| `reducer2` | Menentukan reducer tambahan yang akan digabungkan. |
| `sharedInputs: true` | Menunjukkan bahwa reducer yang digabungkan menggunakan input band yang sama. |

### 14.2 Parameter wilayah dan pemrosesan

| Parameter | Fungsi |
|---|---|
| `geometry` | Area tempat statistik dihitung. Pada kode ini menggunakan geometri `forest_aoi`. |
| `scale` | Skala nominal analisis dalam meter, mengikuti skala nominal proyeksi GEDI. |
| `maxPixels` | Batas maksimum jumlah piksel yang diproses. |
| `tileScale` | Faktor skala tile pemrosesan untuk membantu pengelolaan memori. |

### 14.3 Menampilkan hasil statistik

`reduceRegion()` menghasilkan dictionary yang berisi nilai statistik dari band `AGBD_Prediction`. `print()` menampilkan dictionary tersebut pada Console dengan label `AGBD stock distribution stats`.

Nama key pada dictionary mengikuti nama band dan keluaran reducer Earth Engine. Periksa hasil aktual di Console untuk melihat penamaan key yang dikembalikan.

------------------------------------------------------------------------

# 15. Menampilkan Grafik pada Console

Blok ini membuat ScatterChart untuk membandingkan AGBD observasi dengan AGBD hasil prediksi pada data testing. Grafik ditampilkan pada Console Code Editor.

```javascript
print(ui.Chart.feature.byFeature({
  features: testPrediction, xProperty: 'AGBD', yProperties: ['predicted']
}).setChartType('ScatterChart').setOptions({
  title: 'Observed vs Predicted AGBD (Test Set)',
  hAxis: {title: 'Observed AGBD (Mg/ha)'}, vAxis: {title: 'Predicted AGBD (Mg/ha)'},
  legend: {position: 'none'}, pointSize: 4, colors: ['#1a73e8'],
  trendlines: {0: {type: 'linear', color: '#e53935', lineWidth: 1.5, opacity: 0.8}}
}));
```

| Baris atau komponen | Fungsi |
|---|---|
| `print()` | Menampilkan hasil atau objek pada Console Code Editor. |
| `ui.Chart.feature.byFeature()` | Membuat grafik berdasarkan properti setiap fitur pada `FeatureCollection`. |
| `features: testPrediction` | Menentukan koleksi data testing yang berisi nilai observasi dan prediksi. |
| `xProperty: 'AGBD'` | Menggunakan properti `AGBD` sebagai sumbu X (observasi). |
| `yProperties: ['predicted']` | Menggunakan properti `predicted` sebagai sumbu Y (hasil prediksi). |
| `.setChartType('ScatterChart')` | Menentukan jenis grafik sebagai diagram pencar. |
| `title` | Menentukan judul grafik. |
| `hAxis` dan `vAxis` | Mengatur judul sumbu horizontal dan vertikal. |
| `legend: {position: 'none'}` | Menyembunyikan legenda grafik. |
| `pointSize` dan `colors` | Mengatur ukuran dan warna titik data. |
| `trendlines` | Menambahkan garis tren linear untuk memperlihatkan kecenderungan hubungan observasi dan prediksi. |
| `lineWidth` dan `opacity` | Mengatur ketebalan dan transparansi garis tren. |

**Interpretasi:** Titik yang semakin dekat dengan pola hubungan ideal antara observasi dan prediksi menunjukkan kesesuaian prediksi yang semakin baik. Garis tren membantu memperlihatkan kecenderungan hubungan, tetapi tidak menggantikan metrik evaluasi seperti R² dan RMSE.

---
# 16. Memahami konsep UI pada Google Earth Engine
## 16.2 Membuat Left Sidebar

Left Sidebar merupakan panel di sisi kiri aplikasi. Panel ini berisi identitas aplikasi, daftar layer, legenda, pengaturan opacity, dan fitur *Pixel Inspector*.

### 16.2.1 Membuat panel utama

```javascript
var leftPanel = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'),
  style : {width: '300px', padding: '12px', backgroundColor: 'white'}
});
```

| Baris atau komponen | Fungsi |
|---|---|
| `var leftPanel` | Menyimpan objek panel ke variabel agar dapat diisi dan ditambahkan ke antarmuka. |
| `ui.Panel()` | Membuat wadah untuk komponen antarmuka seperti label, checkbox, slider, dan panel lain. |
| `layout: ui.Panel.Layout.flow('vertical')` | Menyusun komponen di dalam panel dari atas ke bawah. |
| `width: '300px'` | Mengatur lebar panel menjadi 300 piksel. |
| `padding: '12px'` | Memberikan ruang antara batas panel dan komponen di dalamnya. |
| `backgroundColor: 'white'` | Mengatur warna latar belakang panel. |

### 16.2.2 Menambahkan identitas aplikasi

```javascript
leftPanel.add(ui.Label('Forest Aboveground Biomass Density (ABGD)',
              {fontWeight: 'bold', fontSize: '16px', margin: '0 0 2px 0'}));
leftPanel.add(ui.Label('Southeast Sulawesi — 2020',
              {fontSize: '12px', color: '#444', margin: '0 0 2px 0'}));
leftPanel.add(ui.Label('Satellite Embedding V1 + GEDI L4B + MCD12Q1.061 MODIS | RF Regressor',
              {fontSize: '11px', color: '#666', margin: '0 0 6px 0'}));
leftPanel.add(ui.Label('Diniyarti',
              {fontSize: '11px', margin: '0'}));
leftPanel.add(ui.Label('github.com/diniyrti',
              {fontSize: '11px', color: '#1a73e8', margin: '0 0 4px 0'}));
```

`leftPanel.add()` memasukkan komponen ke panel. `ui.Label()` membuat elemen teks. Lima label tersebut masing-masing menampilkan judul aplikasi, wilayah dan tahun kajian, sumber data dan metode, nama pembuat, serta alamat GitHub.

| Properti style | Fungsi |
|---|---|
| `fontWeight: 'bold'` | Membuat teks tebal. |
| `fontSize` | Mengatur ukuran teks. |
| `color` | Mengatur warna teks. |
| `margin` | Mengatur jarak luar komponen label. |

### 16.2.3 Membuat garis pemisah

```javascript
leftPanel.add(ui.Label('', {height: '1px', backgroundColor: '#ccc',
              margin: '8px 0', stretch: 'horizontal'}));
```

Label kosong digunakan sebagai elemen visual berbentuk garis. `height` menentukan ketebalan garis, `backgroundColor` menentukan warnanya, `margin` memberi jarak vertikal, dan `stretch: 'horizontal'` membuat garis mengikuti lebar panel. Pola ini digunakan berulang kali untuk memisahkan kelompok informasi.

### 16.2.3 Membuat daftar layer dengan checkbox

```javascript
leftPanel.add(ui.Label('Layers',
              {fontWeight: 'bold', fontSize: '13px', margin: '0 0 4px 0'}));

for (var i = 0; i < Map.layers().length(); i++) {
  (function(idx) {
    var layer = Map.layers().get(idx);
    var cb = ui.Checkbox({
      label: layer.getName(), value: layer.getShown(),
      style: {fontSize: '11px', margin: '2px 0'}
    });
    cb.onChange(function(checked) { layer.setShown(checked); });
    leftPanel.add(cb);
  })(i);
}
```

| Baris atau komponen | Fungsi |
|---|---|
| `Map.layers()` | Mengakses daftar layer pada peta. |
| `Map.layers().length()` | Mengambil jumlah layer yang tersedia. |
| `for (...)` | Mengulangi proses untuk setiap indeks layer. |
| `(function(idx) {...})(i)` | Menjalankan fungsi untuk indeks saat itu. Pola ini mengikat setiap checkbox ke objek layer yang sesuai. |
| `Map.layers().get(idx)` | Mengambil objek layer berdasarkan indeks. |
| `layer.getName()` | Mengambil nama layer untuk label checkbox. |
| `layer.getShown()` | Mengambil status visibilitas layer saat ini. |
| `ui.Checkbox()` | Membuat kontrol pilihan aktif/nonaktif. |
| `cb.onChange(function(checked) {...})` | Menjalankan fungsi ketika pengguna mengubah checkbox. `checked` berisi status terbaru. |
| `layer.setShown(checked)` | Menampilkan atau menyembunyikan layer sesuai status checkbox. |
| `leftPanel.add(cb)` | Menambahkan checkbox ke Left Sidebar. |

**Alur interaksi:** Kode membaca seluruh layer pada peta, kemudian membuat satu checkbox untuk setiap layer. Checkbox yang dicentang akan menampilkan layer; checkbox yang tidak dicentang akan menyembunyikannya.

### 16.2.4 Membuat legenda AGBD hasil estimasi

```javascript
leftPanel.add(ui.Label('AGBD (Mg/ha)',
              {fontWeight: 'bold', fontSize: '12px', margin: '0 0 4px 0'}));
leftPanel.add(ui.Thumbnail({
  image: ee.Image.pixelLonLat().select('longitude').unitScale(-180, 180)
            .visualize({min: 0, max: 1,
                        palette: ['ffffcc','c2e699','78c679','31a354','006837']}),
  params: {bbox: [-180, -1, 180, 1], dimensions: '256x16'},
  style: {stretch: 'horizontal', height: '16px', margin: '0', padding: '0'}
}));
leftPanel.add(ui.Panel([
  ui.Label('0', {fontSize: '10px', margin: '2px 0'}),
  ui.Label('150', {fontSize: '10px', margin: '2px 0', stretch: 'horizontal', textAlign: 'center'}),
  ui.Label('300', {fontSize: '10px', margin: '2px 0', stretch: 'horizontal', textAlign: 'center'}),
  ui.Label('600+', {fontSize: '10px', margin: '2px 0'})
], ui.Panel.Layout.flow('horizontal'), {stretch: 'horizontal'}));
```

| Baris atau komponen | Fungsi |
|---|---|
| `ui.Label('AGBD (Mg/ha)')` | Menampilkan judul legenda dan satuan. |
| `ui.Thumbnail()` | Membuat gambar mini yang digunakan sebagai legenda warna. |
| `ee.Image.pixelLonLat()` | Membuat citra koordinat longitude dan latitude. |
| `.select('longitude')` | Memilih band longitude. |
| `.unitScale(-180, 180)` | Menormalisasi longitude dari rentang -180 sampai 180 menjadi 0 sampai 1. |
| `.visualize()` | Mengubah citra menjadi tampilan visual berdasarkan rentang dan palet warna. |
| `min`, `max`, `palette` | Menentukan rentang nilai visualisasi dan urutan warna. |
| `params.bbox` | Menentukan batas wilayah untuk pembuatan thumbnail. |
| `dimensions: '256x16'` | Menentukan ukuran thumbnail dalam piksel. |
| `style` | Mengatur ukuran dan tata letak thumbnail pada panel. |
| `ui.Panel([...])` | Membuat panel kecil yang menampung label nilai legenda. |
| `ui.Panel.Layout.flow('horizontal')` | Menyusun label angka secara horizontal. |
| `textAlign: 'center'` | Meratakan teks ke tengah. |

**Catatan metodologis:** Thumbnail ini dibuat dari longitude yang dinormalisasi, bukan dari nilai piksel `biomassPrediction` secara langsung. Dengan demikian, thumbnail berfungsi sebagai contoh gradasi palet. Pastikan rentang angka 0, 150, 300, dan 600+ sesuai dengan rentang visualisasi layer AGBD yang sebenarnya.

### 16.2.5 Membuat legenda GEDI

```javascript
leftPanel.add(ui.Label('GEDI AGBD (Mg/ha)',
              {fontWeight: 'bold', fontSize: '12px', margin: '0 0 4px 0'}));
leftPanel.add(ui.Thumbnail({
  image: ee.Image.pixelLonLat().select('longitude').unitScale(-180, 180)
            .visualize({min: 0, max: 1,
                        palette: ['ffffb2','fecc5c','fd8d3c','f03b20','bd0026']}),
  params: {bbox: [-180, -1, 180, 1], dimensions: '256x16'},
  style: {stretch: 'horizontal', height: '16px', margin: '0', padding: '0'}
}));
leftPanel.add(ui.Panel([
  ui.Label('0', {fontSize: '10px', margin: '2px 0'}),
  ui.Label('150', {fontSize: '10px', margin: '2px 0', stretch: 'horizontal', textAlign: 'center'}),
  ui.Label('300', {fontSize: '10px', margin: '2px 0', stretch: 'horizontal', textAlign: 'center'}),
  ui.Label('600+', {fontSize: '10px', margin: '2px 0'})
], ui.Panel.Layout.flow('horizontal'), {stretch: 'horizontal'}));
```

Struktur kode ini sama dengan legenda AGBD sebelumnya, tetapi menggunakan palet GEDI yang berbeda. `ui.Thumbnail()` membuat gradasi warna, sedangkan panel horizontal menampilkan penanda angka.

**Catatan:** Sama seperti legenda AGBD, thumbnail ini dibentuk dari longitude yang dinormalisasi. Palet dan angka legenda perlu diperiksa agar konsisten dengan parameter visualisasi layer GEDI pada peta.

### 16.2.6 Membuat slider untuk mengatur opacity layer

```javascript
leftPanel.add(ui.Label('AGBD (Mg/ha) Layer Opacity',
              {fontWeight: 'bold', fontSize: '12px', margin: '0 0 2px 0'}));
leftPanel.add(ui.Slider({
  min: 0, max: 1, value: 1, step: 0.05,
  onChange: function(v) { agb_layer.setOpacity(v); },
  style: {stretch: 'horizontal'}
}));
leftPanel.add(ui.Label('GEDI AGBD Layer Opacity',
              {fontWeight: 'bold', fontSize: '12px', margin: '0 0 2px 0'}));
leftPanel.add(ui.Slider({
  min: 0, max: 1, value: 1, step: 0.05,
  onChange: function(v) { gedi_layer.setOpacity(v); },
  style: {stretch: 'horizontal'}
}));
```

| Baris atau komponen | Fungsi |
|---|---|
| `ui.Slider()` | Membuat kontrol geser untuk mengubah nilai secara kontinu. |
| `min: 0` | Menentukan opacity minimum, yaitu transparan. |
| `max: 1` | Menentukan opacity maksimum, yaitu tidak transparan. |
| `value: 1` | Menentukan nilai awal slider. |
| `step: 0.05` | Mengatur interval perubahan nilai sebesar 0,05. |
| `onChange: function(v)` | Menjalankan fungsi saat slider berubah; `v` adalah nilai terbaru. |
| `agb_layer.setOpacity(v)` | Mengatur opacity layer hasil estimasi AGBD. |
| `gedi_layer.setOpacity(v)` | Mengatur opacity layer GEDI. |
| `stretch: 'horizontal'` | Membuat slider memenuhi ruang horizontal panel. |

Kedua slider mempunyai fungsi yang sama, tetapi terhubung ke objek layer yang berbeda. Slider pertama mengendalikan `agb_layer`, sedangkan slider kedua mengendalikan `gedi_layer`.

### 16.2.7 Membuat Pixel Inspector

*Pixel Inspector* menampilkan nilai raster pada lokasi yang dipilih pengguna melalui klik peta. Fitur ini menggabungkan interaksi peta, pengambilan sampel citra, dan pembaruan komponen UI.

Full Code 

```javascript
//Membuat panel informasi
leftPanel.add(ui.Label('Pixel Inspector',
              {fontWeight: 'bold', fontSize: '13px', margin: '0 0 4px 0'}));

var inspectorContent = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'), style: {margin: '0'}
});
inspectorContent.add(ui.Label('Click anywhere on the map to inspect.',
              {fontSize: '11px', color: '#888'}));
leftPanel.add(inspectorContent);

// Mengaktifkan klik pada peta
Map.style().set('cursor', 'crosshair');
Map.onClick(function(coords) {
  inspectorContent.clear();
  inspectorContent.add(ui.Label(
    'Lon: ' + coords.lon.toFixed(5) + '  Lat: ' + coords.lat.toFixed(5),
    {fontSize: '10px', color: '#555', margin: '0 0 4px 0'}
  ));
  inspectorContent.add(ui.Label('Sampling...', {fontSize: '11px', color: '#888'}));

  // Membuat titik dan menggabungkan citra
  var point     = ee.Geometry.Point([coords.lon, coords.lat]);
  var sampleImg = embedding_1km
                    .addBands(biomassPrediction)
                    .addBands(gedi_ds);

  // Mengambil nilai piksel dengan reduceRegion
  sampleImg.reduceRegion({
    reducer: ee.Reducer.first(), geometry: point, crs: gedi_ds.projection(), scale: gedi_ds.projection().nominalScale()
  }).evaluate(function(vals) {
    // Memperbarui panel dan memeriksa ketersediaan data
    inspectorContent.clear();
    inspectorContent.add(ui.Label(
      'Lon: ' + coords.lon.toFixed(5) + '  Lat: ' + coords.lat.toFixed(5),
      {fontSize: '10px', color: '#555', margin: '0 0 4px 0'}
    ));
    if (!vals || vals['AGBD_Prediction'] === null || vals['AGBD_Prediction'] === undefined) {
      inspectorContent.add(ui.Label('No AGBD data at this location.',
                    {fontSize: '11px', color: '#c00'}));
      return;
    }
    // Menampilkan AGBD hasil estimasi
    inspectorContent.add(ui.Label(
      'AGBD Estimation: ' + (vals['AGBD_Prediction'] !== null ? vals['AGBD_Prediction'].toFixed(2) : 'n/a') + ' Mg/ha',
      {fontSize: '12px', fontWeight: 'bold', color: '#980043', margin: '0 0 2px 0'}
    ));

    // Menampilkan nilai GEDI AGBD
    inspectorContent.add(ui.Label(
      'GEDI AGBD: ' +
      (vals['AGBD'] !== null && vals['AGBD'] !== undefined
        ? vals['AGBD'].toFixed(2) + ' Mg/ha'
        : 'No Data'),
      {fontSize: '12px', fontWeight: 'bold', color: '#31a354', margin: '0 0 2px 0'}
    ));
    
    // Menampilkan nilai seluruh band Satellite Embedding
    var bands = embedding_1km.bandNames();

    bands.evaluate(function(bandNames) {

      bandNames.forEach(function(b) {

        var v = vals[b];

        var display = (v === null || v === undefined) ? 'n/a' :
                      (Math.abs(v) < 0.001 && v !== 0 ?
                      v.toExponential(2) : v.toFixed(4));

        inspectorContent.add(ui.Label(
          b + ': ' + display,
          {fontSize: '10px', margin: '1px 0'}
        ));

      });

    });

  });
});

```

#### Penelasan Code pada setiap parth nya
#### Membuat panel informasi

```javascript
leftPanel.add(ui.Label('Pixel Inspector',
              {fontWeight: 'bold', fontSize: '13px', margin: '0 0 4px 0'}));
var inspectorContent = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'), style: {margin: '0'}
});
inspectorContent.add(ui.Label('Click anywhere on the map to inspect.',
              {fontSize: '11px', color: '#888'}));
leftPanel.add(inspectorContent);
```

`ui.Label()` membuat judul fitur. `inspectorContent` merupakan panel khusus untuk menampung hasil pemeriksaan. Layout vertikal menyusun hasil dari atas ke bawah. Label awal memberi instruksi kepada pengguna, kemudian `leftPanel.add()` memasukkan panel tersebut ke sidebar.

#### Mengaktifkan klik pada peta

```javascript
Map.style().set('cursor', 'crosshair');
Map.onClick(function(coords) {
  inspectorContent.clear();
  inspectorContent.add(ui.Label(
    'Lon: ' + coords.lon.toFixed(5) + '  Lat: ' + coords.lat.toFixed(5),
    {fontSize: '10px', color: '#555', margin: '0 0 4px 0'}
  ));
  inspectorContent.add(ui.Label('Sampling...', {fontSize: '11px', color: '#888'}));
```

| Baris atau komponen | Fungsi |
|---|---|
| `Map.style().set('cursor', 'crosshair')` | Mengubah bentuk kursor menjadi tanda silang saat berada di atas peta. |
| `Map.onClick(function(coords) {...})` | Mendaftarkan fungsi yang dijalankan saat peta diklik. |
| `coords` | Menyimpan koordinat lokasi klik, termasuk longitude dan latitude. |
| `inspectorContent.clear()` | Menghapus hasil pemeriksaan sebelumnya. |
| `coords.lon` dan `coords.lat` | Mengambil longitude dan latitude lokasi klik. |
| `.toFixed(5)` | Menampilkan koordinat dengan lima angka di belakang koma. |
| Label `Sampling...` | Memberi tahu pengguna bahwa pengambilan nilai sedang berlangsung. |

#### Membuat titik dan menggabungkan citra

```javascript
var point = ee.Geometry.Point([coords.lon, coords.lat]);
var sampleImg = embedding_1km
  .addBands(biomassPrediction)
  .addBands(gedi_ds);
```

- `ee.Geometry.Point()` membuat geometri titik dari koordinat klik. Urutan koordinatnya adalah longitude kemudian latitude.
- `embedding_1km` merupakan citra predictor Satellite Embedding yang telah disiapkan pada resolusi sekitar 1.000 meter.
- `.addBands(biomassPrediction)` menambahkan band hasil estimasi AGBD.
- `.addBands(gedi_ds)` menambahkan band AGBD dari data GEDI.
- `sampleImg` menjadi citra gabungan yang akan diperiksa pada lokasi klik.

#### Mengambil nilai piksel dengan reduceRegion

```javascript
sampleImg.reduceRegion({
  reducer: ee.Reducer.first(),
  geometry: point,
  crs: gedi_ds.projection(),
  scale: gedi_ds.projection().nominalScale()
}).evaluate(function(vals) {
```

| Baris atau komponen | Fungsi |
|---|---|
| `reduceRegion()` | Mengambil nilai atau menghitung statistik citra pada geometri tertentu. |
| `ee.Reducer.first()` | Mengambil nilai pertama yang ditemukan pada wilayah geometri. Pada geometri titik, reducer ini digunakan untuk memperoleh nilai band di lokasi tersebut. |
| `geometry: point` | Menentukan titik hasil klik sebagai lokasi pengambilan nilai. |
| `crs: gedi_ds.projection()` | Menggunakan proyeksi data GEDI sebagai acuan. |
| `scale: gedi_ds.projection().nominalScale()` | Menggunakan skala nominal proyeksi GEDI. |
| `.evaluate(function(vals) {...})` | Mengambil hasil komputasi dari server ke sisi klien agar dapat digunakan untuk memperbarui UI. |
| `vals` | Objek JavaScript yang menampung nilai setiap band yang berhasil diambil. |

`.evaluate()` bersifat asynchronous. Kode di dalam callback dijalankan setelah hasil komputasi diterima dari server.

#### Memperbarui panel dan memeriksa ketersediaan data

```javascript
inspectorContent.clear();
inspectorContent.add(ui.Label(
  'Lon: ' + coords.lon.toFixed(5) + '  Lat: ' + coords.lat.toFixed(5),
  {fontSize: '10px', color: '#555', margin: '0 0 4px 0'}
));
if (!vals || vals['AGBD_Prediction'] === null || vals['AGBD_Prediction'] === undefined) {
  inspectorContent.add(ui.Label('No AGBD data at this location.',
                {fontSize: '11px', color: '#c00'}));
  return;
}
```

Setelah hasil sampling diterima, panel dibersihkan dan koordinat ditampilkan kembali. Kondisi `if` memeriksa apakah objek hasil sampling tersedia dan apakah nilai `AGBD_Prediction` bernilai `null` atau `undefined`. Jika tidak tersedia, aplikasi menampilkan pesan dan `return` menghentikan callback agar kode selanjutnya tidak memproses nilai kosong.

#### Menampilkan AGBD hasil estimasi

```javascript
inspectorContent.add(ui.Label(
  'AGBD Estimation: ' +
  (vals['AGBD_Prediction'] !== null ? vals['AGBD_Prediction'].toFixed(2) : 'n/a') + ' Mg/ha',
  {fontSize: '12px', fontWeight: 'bold', color: '#980043', margin: '0 0 2px 0'}
));
```

`vals['AGBD_Prediction']` mengambil nilai prediksi. Operator ternary memilih nilai yang ditampilkan: jika nilai tidak `null`, `.toFixed(2)` membatasi tampilan menjadi dua angka desimal; jika `null`, teks `n/a` digunakan. Satuan `Mg/ha` ditambahkan pada label. Style mengatur ukuran, ketebalan, warna, dan jarak teks.

#### Menampilkan GEDI AGBD

```javascript
inspectorContent.add(ui.Label(
  'GEDI AGBD: ' +
  (vals['AGBD'] !== null && vals['AGBD'] !== undefined
    ? vals['AGBD'].toFixed(2) + ' Mg/ha'
    : 'No Data'),
  {fontSize: '12px', fontWeight: 'bold', color: '#31a354', margin: '0 0 2px 0'}
));
```

`vals['AGBD']` mengambil nilai band GEDI. Kondisi memeriksa `null` dan `undefined`. Jika nilai tersedia, angka ditampilkan dengan dua desimal dan satuan Mg/ha; jika tidak, panel menampilkan `No Data`. Warna hijau membedakan nilai GEDI dari hasil estimasi.

### Menampilkan seluruh band Satellite Embedding

```javascript
var bands = embedding_1km.bandNames();
bands.evaluate(function(bandNames) {
  bandNames.forEach(function(b) {
    var v = vals[b];
    var display = (v === null || v === undefined) ? 'n/a' :
                  (Math.abs(v) < 0.001 && v !== 0 ?
                  v.toExponential(2) : v.toFixed(4));
    inspectorContent.add(ui.Label(
      b + ': ' + display,
      {fontSize: '10px', margin: '1px 0'}
    ));
  });
});
```

| Baris atau komponen | Fungsi |
|---|---|
| `embedding_1km.bandNames()` | Mengambil daftar nama band citra embedding. |
| `bands.evaluate(...)` | Mengambil daftar nama band dari server agar dapat diproses di sisi klien. |
| `bandNames.forEach(...)` | Mengulangi proses untuk setiap nama band. |
| `var v = vals[b]` | Mengambil nilai band yang bersesuaian dari hasil sampling. |
| Pemeriksaan `null` dan `undefined` | Memastikan nilai tersedia sebelum diformat. |
| `Math.abs(v) < 0.001 && v !== 0` | Memeriksa apakah nilai absolut sangat kecil tetapi bukan nol. |
| `v.toExponential(2)` | Menampilkan nilai dalam notasi ilmiah dengan dua angka desimal. |
| `v.toFixed(4)` | Menampilkan nilai dengan empat angka desimal. |
| `inspectorContent.add(ui.Label(...))` | Menambahkan nama band dan nilai ke panel. |

Notasi ilmiah digunakan agar nilai embedding yang sangat kecil tetap mudah dibaca. Nilai lainnya ditampilkan dengan empat angka di belakang koma.

### 16.2.7 Menambahkan Left Sidebar ke antarmuka utama

```javascript
ui.root.insert(0, leftPanel);
```

`ui.root` mengakses susunan utama antarmuka Code Editor. `insert(0, leftPanel)` menambahkan Left Sidebar pada posisi indeks 0 dalam susunan tersebut.

---

## 16.3 Membuat Right Sidebar

Right Sidebar menyajikan statistik hasil estimasi, metrik evaluasi model, informasi training, grafik observasi dan prediksi, serta *feature importance*.

### 16.3.1 Membuat panel utama

```javascript
var rightPanel = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'),
  style: {width: '300px', padding: '12px', backgroundColor: 'white'}
});
```

`ui.Panel()` membuat wadah Right Sidebar. `layout` menyusun komponen secara vertikal. `width` menentukan lebar 300 piksel, `padding` memberi ruang di dalam panel, dan `backgroundColor` menentukan warna latar belakang.

### 16.3.2 Menambahkan judul dan informasi model

```javascript
rightPanel.add(ui.Label('RF Model Performance', {
  fontWeight: 'bold', fontSize: '17px', margin: '0 0 2px 0'
}));
rightPanel.add(ui.Label(
  'Satellite Embedding V1 + GEDI L4B | 2020',
  {fontSize: '11px', color: '#666', margin: '0 0 2px 0'}
));
rightPanel.add(ui.Label(
  'Random Forest Regression | 70% Training - 30% Testing',
  {fontSize: '11px', color: '#666', margin: '0 0 6px 0'}
));
```

`rightPanel.add()` menambahkan label ke panel. Label pertama merupakan judul performa model, label kedua menjelaskan data dan tahun yang dicantumkan, sedangkan label ketiga menunjukkan metode regresi dan proporsi training-testing. Properti `fontWeight`, `fontSize`, `color`, dan `margin` mengatur tampilan teks.

### 16.3.3  Menampilkan statistik hasil estimasi AGBD

#### Membuat label statistik

```javascript
rightPanel.add(ui.Label('ABGD Estimation',
              {fontWeight: 'bold', fontSize: '14px', margin: '4px 0 4px 0', color: '#222'}));
var cMean   = ui.Label('Mean      : computing...', {fontSize: '12px', margin: '2px 0'});
var cMedian = ui.Label('Median    : computing...', {fontSize: '12px', margin: '2px 0'});
var cStd    = ui.Label('Std Dev   : computing...', {fontSize: '12px', margin: '2px 0'});
var cMin    = ui.Label('Min       : computing...', {fontSize: '12px', margin: '2px 0'});
var cMax    = ui.Label('Max       : computing...', {fontSize: '12px', margin: '2px 0'});
rightPanel.add(cMean);
rightPanel.add(cMedian);
rightPanel.add(cStd);
rightPanel.add(cMin);
rightPanel.add(cMax);
```

`ui.Label()` membuat lima label untuk mean, median, standar deviasi, minimum, dan maksimum. Teks `computing...` merupakan status sementara sebelum hasil diterima. Setiap label disimpan dalam variabel agar isinya dapat diperbarui, kemudian dimasukkan ke Right Sidebar dengan `rightPanel.add()`.

### Mengisi label dengan hasil statistik

```javascript
agbd_stats.evaluate(function(s) {
  if (!s) return;
  cMean.setValue('Mean      : ' + s['AGBD_Prediction_mean'].toFixed(2) + ' Mg/ha');
  cMedian.setValue('Median    : ' + s['AGBD_Prediction_median'].toFixed(2) + ' Mg/ha');
  cStd.setValue('Std Dev   : ' + s['AGBD_Prediction_stdDev'].toFixed(2) + ' Mg/ha');
  cMin.setValue('Min       : ' + s['AGBD_Prediction_min'].toFixed(2) + ' Mg/ha');
  cMax.setValue('Max       : ' + s['AGBD_Prediction_max'].toFixed(2) + ' Mg/ha');
});
```

`agbd_stats` merupakan objek statistik yang dibuat pada bagian sebelumnya. `.evaluate()` mengambil hasil dari server ke sisi klien dan menyimpannya pada parameter `s`. `if (!s) return` menghentikan callback jika hasil tidak tersedia. Properti `AGBD_Prediction_mean`, `median`, `stdDev`, `min`, dan `max` mengambil statistik masing-masing. `.toFixed(2)` membatasi tampilan menjadi dua angka desimal, sedangkan `.setValue()` memperbarui label yang sebelumnya menampilkan `computing...`.

### 16.3.4 Menampilkan metrik evaluasi model

#### Menyiapkan judul dan style

```javascript
rightPanel.add(ui.Label('Model Evaluation', {
  fontWeight: 'bold', fontSize: '13px', margin: '4px 0 5px 0'
}));
var metricStyle = {fontSize: '12px', margin: '3px 0'};
```

Label pertama menjadi judul bagian evaluasi. `metricStyle` menyimpan format yang digunakan bersama oleh label R², RMSE, dan Pearson r.

#### R-squared

```javascript
var r2Label = ui.Label('R²: Calculating...', metricStyle);
rightPanel.add(r2Label);
R2.evaluate(function(value) {
  r2Label.setValue(
    'R²: ' + (value !== null && value !== undefined ? value.toFixed(4) : 'N/A')
  );
});
```

`r2Label` membuat label sementara. `R2.evaluate()` mengambil nilai R² dari server. Kondisi memeriksa apakah nilai bukan `null` dan bukan `undefined`. Jika tersedia, `.toFixed(4)` menampilkan empat angka desimal; jika tidak, teks `N/A` ditampilkan. `setValue()` memperbarui label.

### Root Mean Square Error

```javascript
var rmseLabel = ui.Label('RMSE: Calculating...', metricStyle);
rightPanel.add(rmseLabel);
RMSE.evaluate(function(value) {
  rmseLabel.setValue(
    'RMSE: ' +
    (value !== null && value !== undefined ? value.toFixed(3) + ' Mg/ha' : 'N/A')
  );
});
```

`rmseLabel` merupakan label sementara untuk RMSE. `RMSE.evaluate()` mengambil hasil dari server. Jika tersedia, `.toFixed(3)` menampilkan tiga angka desimal dan satuan `Mg/ha` ditambahkan karena RMSE mengikuti satuan variabel target. Jika tidak tersedia, label menampilkan `N/A`.

#### Pearson Correlation

```javascript
var pearsonLabel = ui.Label('Pearson r: Calculating...', metricStyle);
rightPanel.add(pearsonLabel);
correlation.get('correlation').evaluate(function(value) {
  pearsonLabel.setValue(
    'Pearson r: ' +
    (value !== null && value !== undefined ? value.toFixed(4) : 'N/A')
  );
});
```

`correlation.get('correlation')` mengambil properti `correlation` dari objek hasil perhitungan korelasi. `.evaluate()` mengambil nilainya dari server. Nilai yang tersedia ditampilkan dengan empat angka desimal; jika tidak tersedia, label menampilkan `N/A`.

**Perbedaan metrik:** R² mengukur proporsi variasi observasi yang dapat dijelaskan model; RMSE mengukur besarnya kesalahan prediksi dalam satuan variabel target; Pearson r mengukur kekuatan dan arah hubungan linear antara dua variabel. Ketiganya memberikan informasi berbeda.

### 16.3.5 Menampilkan informasi pelatihan model

#### Judul dan style informasi

```javascript
rightPanel.add(ui.Label('Training Information', {
  fontWeight: 'bold', fontSize: '13px', margin: '4px 0 5px 0'
}));
var infoStyle = {fontSize: '11px', margin: '3px 0'};
```

Label menampilkan judul bagian informasi training. `infoStyle` menyimpan format tampilan untuk label informasi selanjutnya.

#### Parameter Random Forest

```javascript
rightPanel.add(ui.Label('Number of Trees: ' + nTrees, infoStyle));
rightPanel.add(ui.Label('Variables per Split: ' + variablesPerSplit, infoStyle));
```

`nTrees` berisi jumlah pohon keputusan yang digunakan. `variablesPerSplit` berisi jumlah variabel kandidat yang dipertimbangkan pada setiap split. Operator `+` menggabungkan teks dengan nilai variabel agar keduanya tampil dalam satu label.

#### Proporsi training dan testing

```javascript
rightPanel.add(ui.Label('Training Split: 70%', infoStyle));
rightPanel.add(ui.Label('Testing Split: 30%', infoStyle));
```

Kedua label menampilkan proporsi pembagian data yang ditulis langsung pada kode. Ubah teks ini apabila pembagian pada proses pemodelan tidak lagi 70% training dan 30% testing.

#### Jumlah predictor bands

```javascript
var predictorLabel = ui.Label('Predictor Bands: Calculating...', infoStyle);
rightPanel.add(predictorLabel);
predictor_bands.size().evaluate(function(value) {
  predictorLabel.setValue('Predictor Bands: ' + value);
});
```

`predictorLabel` menampilkan status sementara. `predictor_bands.size()` menghitung jumlah elemen pada objek `predictor_bands`. `.evaluate()` mengambil hasil dari server, kemudian `setValue()` memperbarui label. Interpretasi sebagai jumlah band tepat apabila `predictor_bands` memang berisi daftar band predictor.

#### Jumlah sampel training

```javascript
var trainingLabel = ui.Label('Training Samples: Calculating...', infoStyle);
rightPanel.add(trainingLabel);
training.size().evaluate(function(value) {
  trainingLabel.setValue('Training Samples: ' + value);
});
```

`training.size()` menghitung jumlah fitur pada koleksi training. Hasilnya diambil menggunakan `.evaluate()` dan ditampilkan dengan `trainingLabel.setValue()`.

#### Jumlah sampel testing

```javascript
var testingLabel = ui.Label('Testing Samples: Calculating...', infoStyle);
rightPanel.add(testingLabel);
testing.size().evaluate(function(value) {
  testingLabel.setValue('Testing Samples: ' + value);
});
```

Strukturnya sama dengan bagian training, tetapi objek yang dihitung adalah `testing`. `testing.size()` menghitung jumlah fitur pada koleksi testing.

#### Variabel target dan satuan

```javascript
rightPanel.add(ui.Label('Target: GEDI L4B AGBD', infoStyle));
rightPanel.add(ui.Label('Unit: Mg/ha', infoStyle));
```

Label pertama menunjukkan variabel target pemodelan, sedangkan label kedua menunjukkan satuan nilai AGBD.

### 16.3.6 Membuat grafik Observed vs Predicted AGBD

Grafik ini membandingkan AGBD observasi dan prediksi pada data testing. Berbeda dengan grafik Console pada bagian awal, grafik ini ditambahkan ke Right Sidebar.

```javascript
rightPanel.add(ui.Label('Observed vs Predicted AGBD', {
  fontWeight: 'bold', fontSize: '13px', margin: '4px 0 5px 0'
}));

var scatterChart = ui.Chart.feature.byFeature({
  features: testPrediction,
  xProperty: 'AGBD',
  yProperties: ['predicted']
})
.setChartType('ScatterChart')
.setOptions({
  title: 'AGBD Prediction (Test Set)',
  hAxis: {
    title: 'Observed AGBD (Mg/ha)',
    textStyle: {fontSize: 9},
    titleTextStyle: {fontSize: 10}
  },
  vAxis: {
    title: 'Predicted AGBD (Mg/ha)',
    textStyle: {fontSize: 9},
    titleTextStyle: {fontSize: 10}
  },
  legend: {position: 'none'},
  pointSize: 3,
  colors: ['#1a73e8'],
  trendlines: {
    0: {type: 'linear', color: '#e53935', lineWidth: 1.5, opacity: 0.8}
  },
  chartArea: {width: '75%', height: '70%'},
  width: 270,
  height: 230
});
rightPanel.add(scatterChart);
```

| Baris atau komponen | Fungsi |
|---|---|
| `ui.Chart.feature.byFeature()` | Membuat grafik berdasarkan properti setiap fitur. |
| `features: testPrediction` | Menggunakan data testing yang berisi nilai observasi dan prediksi. |
| `xProperty: 'AGBD'` | Menempatkan nilai observasi pada sumbu X. |
| `yProperties: ['predicted']` | Menempatkan nilai prediksi pada sumbu Y. |
| `.setChartType('ScatterChart')` | Memilih diagram pencar. |
| `.setOptions()` | Mengatur judul, sumbu, simbol, warna, garis tren, dan ukuran grafik. |
| `hAxis` dan `vAxis` | Mengatur judul dan tampilan teks sumbu. |
| `legend: {position: 'none'}` | Menyembunyikan legenda. |
| `pointSize` dan `colors` | Mengatur ukuran dan warna titik. |
| `trendlines` | Menambahkan garis tren linear. |
| `chartArea` | Mengatur proporsi ruang yang digunakan area grafik. |
| `width` dan `height` | Mengatur ukuran grafik dalam piksel. |
| `rightPanel.add(scatterChart)` | Memasukkan grafik ke Right Sidebar. |

Titik yang mengikuti pola linear secara rapat menunjukkan hubungan yang lebih kuat antara observasi dan prediksi. Grafik ini membantu pemeriksaan visual dan sebaiknya dibaca bersama R² serta RMSE.

### 16.3.7 Membuat grafik Feature Importance

Bagian ini mengubah nilai importance predictor menjadi persentase, mengurutkannya, mengambil sepuluh predictor tertinggi, lalu menyajikannya sebagai grafik batang.

### Menghitung total importance

```javascript
var totalImportance = ee.Number(
  importance.values().reduce(ee.Reducer.sum())
);
```

`importance.values()` mengambil seluruh nilai dari dictionary importance. `.reduce(ee.Reducer.sum())` menjumlahkan seluruh nilai tersebut. `ee.Number()` memastikan hasilnya menjadi objek angka Earth Engine dan disimpan pada `totalImportance`.

### Mengubah dictionary menjadi FeatureCollection dan menghitung persentase

```javascript
var importanceFC = ee.FeatureCollection(
  importance.keys().map(function(key) {
    var rawImportance = ee.Number(importance.get(key));
    var importancePct = rawImportance
      .divide(totalImportance)
      .multiply(100);
    return ee.Feature(null, {
      band: key,
      importance: rawImportance,
      importance_pct: importancePct
    });
  })
);
```

| Baris atau komponen | Fungsi |
|---|---|
| `importance.keys()` | Mengambil seluruh nama predictor dari dictionary. |
| `.map(function(key) {...})` | Memproses setiap nama predictor. |
| `importance.get(key)` | Mengambil nilai importance untuk predictor tersebut. |
| `ee.Number()` | Memastikan nilai direpresentasikan sebagai angka Earth Engine. |
| `.divide(totalImportance)` | Membagi importance suatu predictor dengan total importance. |
| `.multiply(100)` | Mengubah proporsi menjadi persentase. |
| `ee.Feature(null, {...})` | Membuat fitur tanpa geometri, tetapi memiliki properti atribut. |
| `band: key` | Menyimpan nama predictor. |
| `importance: rawImportance` | Menyimpan nilai importance asli. |
| `importance_pct: importancePct` | Menyimpan importance dalam persentase. |
| `ee.FeatureCollection()` | Menggabungkan fitur-fitur menjadi sebuah koleksi. |

Rumus persentase importance:

$$
Importance_i(\%) = \\frac{Importance_i}{\\sum_{j=1}^{p} Importance_j} \\times 100
$$

Keterangan: `Importance_i` adalah nilai importance predictor ke-i, sedangkan `p` adalah jumlah seluruh predictor.

#### Mengurutkan dan mengambil sepuluh predictor tertinggi

```javascript
var topImportance = importanceFC
  .sort('importance_pct', false)
  .limit(10);
```

`.sort('importance_pct', false)` mengurutkan berdasarkan persentase importance dari terbesar ke terkecil (`false` berarti urutan menurun). `.limit(10)` mengambil sepuluh fitur teratas.

#### Membuat grafik batang feature importance

```javascript
var importanceChart = ui.Chart.feature.byFeature({
  features: topImportance,
  xProperty: 'band',
  yProperties: ['importance_pct']
})
.setChartType('ColumnChart')
.setOptions({
  title: 'Top 10 Predictors',
  hAxis: {
    title: 'Embedding Bands',
    slantedText: true,
    slantedTextAngle: 45,
    textStyle: {fontSize: 9},
    titleTextStyle: {fontSize: 10}
  },
  vAxis: {
    title: 'Relative Importance (%)',
    minValue: 0,
    textStyle: {fontSize: 9},
    titleTextStyle: {fontSize: 10}
  },
  legend: {position: 'none'},
  colors: ['#2878B5'],
  chartArea: {width: '75%', height: '65%'},
  width: 270,
  height: 250
});
rightPanel.add(importanceChart);
```

| Baris atau komponen | Fungsi |
|---|---|
| `features: topImportance` | Menggunakan sepuluh predictor teratas. |
| `xProperty: 'band'` | Menggunakan nama band sebagai kategori sumbu X. |
| `yProperties: ['importance_pct']` | Menggunakan persentase importance sebagai nilai sumbu Y. |
| `.setChartType('ColumnChart')` | Membuat grafik batang vertikal. |
| `hAxis.title` | Memberi judul sumbu horizontal. |
| `slantedText: true` | Memiringkan label kategori pada sumbu X. |
| `slantedTextAngle: 45` | Mengatur kemiringan label sebesar 45 derajat. |
| `vAxis.title` | Memberi judul sumbu vertikal. |
| `minValue: 0` | Menentukan nilai minimum sumbu Y. |
| `legend.position: 'none'` | Menyembunyikan legenda. |
| `colors` | Menentukan warna batang. |
| `chartArea` | Mengatur proporsi area grafik. |
| `width` dan `height` | Mengatur ukuran grafik. |
| `rightPanel.add(importanceChart)` | Menambahkan grafik ke Right Sidebar. |

Batang yang lebih tinggi menunjukkan predictor dengan persentase importance relatif lebih besar dalam model. Nilai importance menunjukkan kontribusi relatif menurut mekanisme pengukuran importance model, bukan hubungan sebab-akibat.

### 16.3.8 Menampilkan informasi data

```javascript
rightPanel.add(ui.Label('Data Information', {
  fontWeight: 'bold', fontSize: '13px', margin: '4px 0 5px 0'
}));
rightPanel.add(ui.Label('Predictor: Google Satellite Embedding V1', infoStyle));
rightPanel.add(ui.Label('Reference: GEDI L4B Gridded AGBD', infoStyle));
rightPanel.add(ui.Label('Forest Mask: MODIS MCD12Q1.061', infoStyle));
rightPanel.add(ui.Label('Reference Year: 2020', infoStyle));
rightPanel.add(ui.Label('Spatial Resolution: ~1,000 m', infoStyle));
```

| Label | Fungsi |
|---|---|
| `Data Information` | Judul bagian informasi data. |
| `Predictor` | Menunjukkan dataset predictor. |
| `Reference` | Menunjukkan dataset referensi atau variabel target. |
| `Forest Mask` | Menunjukkan dataset yang digunakan untuk membentuk mask hutan. |
| `Reference Year` | Menampilkan tahun referensi yang dicantumkan pada aplikasi. |
| `Spatial Resolution` | Menampilkan resolusi spasial yang dicantumkan pada aplikasi. |

Nilai tahun dan resolusi pada blok ini ditulis langsung sebagai teks. Karena itu, perbarui informasinya apabila konfigurasi data pada proses sebelumnya berubah.

### 16.3.9 Menambahkan Right Sidebar ke antarmuka utama

```javascript
ui.root.add(rightPanel);
```

`ui.root` mengakses susunan utama antarmuka Code Editor. `add(rightPanel)` menambahkan panel kanan ke antarmuka setelah seluruh komponennya dibuat.

---

# 17. Ringkasan Komponen Antarmuka

| Komponen atau fungsi | Peran dalam aplikasi |
|---|---|
| `ui.Panel()` | Membuat wadah untuk mengelompokkan komponen antarmuka. |
| `ui.Label()` | Menampilkan judul, keterangan, nilai statistik, dan informasi. |
| `ui.Checkbox()` | Mengatur visibilitas layer peta. |
| `ui.Slider()` | Mengatur opacity layer secara interaktif. |
| `ui.Thumbnail()` | Menampilkan gambar mini, termasuk legenda warna. |
| `ui.Chart.feature.byFeature()` | Membuat grafik berdasarkan properti pada `FeatureCollection`. |
| `ui.Map.Layer()` | Membuat objek layer peta yang dapat dikendalikan. |
| `Map.onClick()` | Menjalankan fungsi ketika pengguna mengklik peta. |
| `reduceRegion()` | Mengambil nilai atau menghitung statistik citra pada geometri tertentu. |
| `.evaluate()` | Mengambil hasil komputasi server Earth Engine ke sisi klien. |
| `.setValue()` | Memperbarui teks pada komponen label. |
| `ui.root.insert()` | Menambahkan komponen pada posisi tertentu di susunan utama UI. |
| `ui.root.add()` | Menambahkan komponen ke susunan utama UI. |

# 18. Alur Kerja Antarmuka Aplikasi

1. **Menyiapkan panel:** Membuat Left Sidebar dan Right Sidebar menggunakan `ui.Panel()`.
2. **Menyusun informasi:** Menambahkan judul, identitas aplikasi, dan keterangan menggunakan `ui.Label()`.
3. **Mengelola layer:** Membuat checkbox untuk menampilkan atau menyembunyikan layer.
4. **Menyediakan legenda:** Membuat thumbnail dan label nilai untuk membantu interpretasi warna raster.
5. **Mengatur tampilan peta:** Menyediakan slider untuk mengubah opacity layer.
6. **Memeriksa piksel:** Menggunakan `Map.onClick()` untuk menerima lokasi klik, kemudian `reduceRegion()` untuk mengambil nilai raster.
7. **Menampilkan hasil analisis:** Menggunakan `.evaluate()` dan `.setValue()` untuk menampilkan statistik dan metrik evaluasi yang dihitung di server.
8. **Menyajikan grafik:** Menampilkan grafik observasi-prediksi dan *feature importance*.
9. **Menyusun antarmuka:** Menambahkan panel ke `ui.root` agar komponen dapat digunakan dalam aplikasi.