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
title: Klasifikasi Penutup Lahan Menggunakan Random Forest (RF)
authors:
  - name: Diniyarti
---


:::{image} https://colab.research.google.com/assets/colab-badge.svg
:alt: Buka di Colab
:align: left
:target: https://colab.research.google.com/github/USERNAME/NAMA-REPO/blob/main/book/python/variables.ipynb
:::

# Klasifikasi Penutup Lahan Menggunakan Sentinel-2 dan Random Forest di Google Earth Engine

Analisis ini membahas alur klasifikasi penutup lahan menggunakan Sentinel-2 dan Random Forest di Google Earth Engine (GEE). Kode dijelaskan secara singkat untuk membantu memahami setiap tahap.

## Alur Klasifikasi

1. Tahapan Pemilihan Citra
2. Penentuan Skema Klasifikasi dan Data Latih
3. Feature Engineering
4. Hyperparameter Tuning Random Forest
5. Uji Akurasi Tematik

# 1. Tahapan Pemilihan Citra

Tahap ini menyeleksi dan menyiapkan citra Sentinel-2 sebelum pembuatan data latih.

**Wilayah kajian:** Kecamatan Kambera, Kabupaten Sumba Timur, Nusa Tenggara Timur  
**Sumber citra:** Sentinel-2 Level-2A Surface Reflectance  
**Platform:** Google Earth Engine Code Editor  
**Tujuan:** menyeleksi citra, memeriksa metadata, mengurangi pengaruh awan, memilih band, membuat komposit, dan mengekspor citra.

> **Catatan:** Skrip memuat dua pendekatan: (1) memilih satu citra (*single scene*) dengan persentase awan terendah dan (2) membuat komposit median dari beberapa citra setelah *cloud masking*. Pilih pendekatan yang sesuai dengan tujuan analisis. Untuk pembelajaran, jalankan satu pendekatan pada satu waktu agar variabel yang didefinisikan ulang tidak membingungkan.

---

## 1.1 Konsep Dasar

### Sentinel-2 Level-2A

Sentinel-2 Level-2A menyediakan data reflektansi permukaan (*surface reflectance*) yang telah melalui koreksi atmosfer. Koleksi `COPERNICUS/S2_SR_HARMONIZED` digunakan untuk menjaga konsistensi nilai reflektansi pada koleksi Sentinel-2.

### Area of Interest (AOI)

AOI (*Area of Interest*) adalah wilayah yang menjadi fokus analisis. Dalam tutorial ini, AOI berupa batas wilayah Kecamatan Kambera yang sudah diunggah sebagai asset di GEE.

### Pemilihan Band

Setiap band Sentinel-2 merekam rentang panjang gelombang yang berbeda. Kombinasi band dapat membantu membedakan objek seperti vegetasi, air, tanah terbuka, dan kawasan terbangun.

| Band | Nama umum | Resolusi asli |
|---|---|---:|
| B2 | Blue | 10 m |
| B3 | Green | 10 m |
| B4 | Red | 10 m |
| B5 | Red Edge 1 | 20 m |
| B6 | Red Edge 2 | 20 m |
| B7 | Red Edge 3 | 20 m |
| B8 | NIR | 10 m |
| B8A | Red Edge 4 / narrow NIR | 20 m |
| B11 | SWIR 1 | 20 m |
| B12 | SWIR 2 | 20 m |

**Perhatian:** B8 memiliki resolusi asli 10 m, bukan 20 m. Band B5, B6, B7, B8A, B11, dan B12 memiliki resolusi asli 20 m. Jika semua band akan digunakan bersama, tentukan resolusi kerja yang konsisten. Menyamakan resolusi tidak menambah detail asli pada band yang resolusinya lebih kasar.

---

## 1.2 Memilih Satu Citra (*Single Scene*)

Pendekatan ini memilih satu scene yang memiliki persentase tutupan awan paling rendah menurut metadata Sentinel-2. Persentase tersebut dihitung untuk cakupan scene, bukan hanya untuk AOI. Jadi, scene dengan persentase awan terendah belum tentu paling bersih tepat di wilayah kajian.

## 1.2.1 Menentukan Wilayah Kajian

```javascript
// 1.1 Tentukan area kajian: Kecamatan Kambera, Sumba Timur, NTT
var aoi = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/aoi_kambera'
);
```

## 1.2.2 Menyeleksi Koleksi Sentinel-2

```javascript
var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)
  .filterDate('2025-01-01', '2025-10-30')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 30))
  .map(function(img) {
    return img.divide(10000)
      .copyProperties(img, img.propertyNames());
  });
```

## 1.2.3 Memeriksa Koleksi dan Metadata

```javascript
print('Jumlah citra tersedia:', s2.size());

var leastCloudy = s2.sort('CLOUDY_PIXEL_PERCENTAGE').first();
print('Citra dengan tutupan awan minimal:', leastCloudy);

var date = ee.Date(leastCloudy.get('system:time_start'))
  .format('YYYY-MM-dd');
print('Tanggal perekaman citra:', date);

print(
  'Persentase awan:',
  leastCloudy.get('CLOUDY_PIXEL_PERCENTAGE')
);
```

Jika jumlah scene adalah nol, periksa kembali ID asset, rentang tanggal, AOI, dan ambang persentase awan.

## 1.2.4 Menampilkan Citra

```javascript
var viz = {
  min: 0,
  max: 0.3,
  bands: ['B4', 'B3', 'B2']
};

Map.centerObject(aoi, 10);
Map.addLayer(leastCloudy, viz, 'Citra Sentinel-2 dengan Awan Terendah');
Map.addLayer(aoi, {color: 'red'}, 'AOI');
```

Gunakan peta untuk memeriksa apakah wilayah kajian tertutup awan, bayangan awan, atau kabut. Metadata awan saja tidak cukup untuk memastikan citra bersih.

## 1.2.5 Memotong Citra dan Memilih Band

```javascript
var single_s2 = leastCloudy.clip(aoi);

var opticalBands = [
  'B2', 'B3', 'B4',
  'B5', 'B6', 'B7',
  'B8', 'B8A', 'B11', 'B12'
];

var s2_selectedband = single_s2.select(opticalBands);

print('Band yang dipilih:', s2_selectedband.bandNames());
Map.addLayer(single_s2, viz, 'Single Scene Sentinel-2');
```

## 1.2.6 Menyiapkan Resolusi Spasial 20 Meter

```javascript
var resample20 = function(image) {
  var projection = image.select('B12').projection();

  var resample = image.reproject({
    crs: projection,
    scale: 20
  });

  return ee.Image(
    resample.copyProperties(
      image,
      ['system:time_start', 'system:time_end']
    )
  );
};

var s2_20m = resample20(s2_selectedband);
```

## 1.2.7 Mengekspor Citra

Contoh berikut mengekspor 10 band terpilih pada skala 20 m.

```javascript
Export.image.toDrive({
  image: s2_20m,
  description: 'Sentinel2_2025_Kambera_20m',
  folder: 'Earth Engine',
  region: aoi.geometry(),
  scale: 20,
  crs: 'EPSG:32751',
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  },
  maxPixels: 1e13
});
```
`cloudOptimized: true` = File GeoTIFF disusun agar aplikasi dapat membaca bagian tertentu secara efisien, terutama ketika file diakses melalui jaringan.

Setelah kode dijalankan, buka tab **Tasks**, klik **Run**, periksa parameter ekspor, lalu jalankan tugasnya.

---

## 1.3 Membuat Komposit Median dengan Cloud Masking

Pendekatan kedua menggabungkan beberapa citra dari periode waktu tertentu menjadi satu komposit. Sebelum membuat komposit, piksel yang terindikasi awan atau kondisi tidak jernih dimask menggunakan Cloud Score+.

Pendekatan ini dapat mengurangi pengaruh awan yang muncul pada scene tertentu. Namun, komposit median juga dapat menggabungkan pengamatan dari tanggal yang berbeda. Periode waktu harus dipilih sesuai tujuan pemetaan dan kondisi penutup lahan.

## 1.3.1 Menyeleksi Koleksi Citra

```javascript
var aoi = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/aoi_kambera'
);

var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)
  .filterDate('2025-01-01', '2025-07-01')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 50))
  .map(function(img) {
    return img.divide(10000)
      .copyProperties(img, img.propertyNames());
  });
```

## 1.3.2 Memeriksa Koleksi

```javascript
print('Jumlah citra tersedia:', s2.size());

var leastCloudy = s2.sort('CLOUDY_PIXEL_PERCENTAGE').first();
print('Citra dengan tutupan awan minimal:', leastCloudy);

var date = ee.Date(leastCloudy.get('system:time_start'))
  .format('YYYY-MM-dd');
print('Tanggal perekaman citra:', date);

print(
  'Persentase awan:',
  leastCloudy.get('CLOUDY_PIXEL_PERCENTAGE')
);

var viz = {
  min: 0,
  max: 0.3,
  bands: ['B4', 'B3', 'B2']
};

Map.centerObject(aoi, 10);
Map.addLayer(leastCloudy, viz, 'Scene dengan Awan Terendah');
Map.addLayer(aoi, {color: 'red'}, 'AOI');
```

Bagian ini menggunakan perintah `print()` untuk memeriksa jumlah scene, tanggal scene dengan metadata awan terendah, dan persentase awannya. Peta menampilkan scene tersebut sebagai pemeriksaan visual awal.

## 1.3.3 Memuat Cloud Score+

```javascript
var csPlus = ee.ImageCollection(
  'GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED'
);

var QA_BAND = 'cs_cdf';
var CLEAR_THRESHOLD = 0.65;
```

- `csPlus` memanggil koleksi Cloud Score+ yang berpasangan dengan citra Sentinel-2.
- `QA_BAND = 'cs_cdf'` memilih band Cloud Score+ yang menggambarkan tingkat kejernihan piksel.
- `CLEAR_THRESHOLD = 0.65` menetapkan ambang kejernihan. 
- Ambang batas untuk masking; nilai antara 0,50 dan 0,65 umumnya memberikan hasil yang baik.Nilai yang lebih tinggi akan menghilangkan awan tipis, kabut, serta bayangan awan sirus.

Dokumentasi [Cloud Score+](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_CLOUD_SCORE_PLUS_V1_S2_HARMONIZED#description)

## 1.3.4 Melakukan Cloud Masking

```javascript
var s2_cloudmask = s2
  .linkCollection(csPlus, [QA_BAND])
  .map(function(img) {
    var cloudMask = img.select(QA_BAND).gte(CLEAR_THRESHOLD);

    return img
      .updateMask(cloudMask)
      .copyProperties(img, img.propertyNames());
  });
```

**Catatan:** Mask ini mempertahankan piksel dengan skor kejernihan tinggi. Hasilnya tetap perlu diperiksa karena ambang yang terlalu ketat dapat menghilangkan terlalu banyak piksel, sedangkan ambang terlalu longgar dapat menyisakan awan atau kabut.

## 1.3.5 Membuat Komposit Median

```javascript
var s2_composite = s2_cloudmask
  .median()
  .clip(aoi);

Map.addLayer(s2_composite, viz, 'Komposit Median setelah Cloud Masking');
```

**Mengapa median?** Median relatif tahan terhadap nilai ekstrem dibandingkan rata-rata. 

## 1.3.6 Menyiapkan Resolusi Spasial 20 Meter

```javascript
var resample20 = function(image) {
  var projection = image.select('B12').projection();

  var resample = image.reproject({
    crs: projection,
    scale: 20
  });

  return ee.Image(
    resample.copyProperties(
      image,
      ['system:time_start', 'system:time_end']
    )
  );
};

var s2_20m = resample20(s2_composite);
```

Fungsi ini meminta komposit dihitung pada skala 20 m berdasarkan proyeksi band B12. Untuk alur kerja yang lebih sederhana, Anda dapat langsung mengekspor `s2_composite` dengan `scale: 20`, lalu membiarkan GEE menentukan perhitungan skala saat ekspor. Jika memakai `reproject()`, pastikan memang diperlukan untuk analisis Anda.

## 1.3.7 Mengekspor Komposit

```javascript
Export.image.toDrive({
  image: s2_20m,
  description: 'Sentinel2_Median_2025_Kambera_20m',
  folder: 'Earth Engine',
  region: aoi.geometry(),
  scale: 20,
  crs: 'EPSG:32751',
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  },
  maxPixels: 1e13
});
```

`cloudOptimized: true` = File GeoTIFF disusun agar aplikasi dapat membaca bagian tertentu secara efisien, terutama ketika file diakses melalui jaringan.

Parameter ekspor memiliki fungsi yang sama dengan Bagian A. Perbedaannya, `image` sekarang berisi komposit median yang sudah melalui proses cloud masking.

---

## 1.4 Memilih Pendekatan

| Aspek | Single scene | Komposit median |
|---|---|---|
| Sumber citra | Satu scene dengan metadata awan terendah | Beberapa scene dalam rentang waktu |
| Cloud masking piksel | Belum dilakukan secara eksplisit pada skrip Bagian A | Menggunakan Cloud Score+ |
| Keuntungan | Sederhana dan tanggal pengamatan jelas | Dapat mengurangi pengaruh awan jika tersedia cukup pengamatan bersih |
| Keterbatasan | Awan masih mungkin menutupi AOI | Nilai median dapat menggabungkan tanggal pengamatan yang berbeda |
| Cocok digunakan saat | Satu scene cukup bersih dan kondisi waktu perlu spesifik | Ingin membuat komposit untuk periode tertentu |

Untuk klasifikasi penutup lahan, komposit median sering menjadi pilihan yang berguna, tetapi pastikan periode komposit tidak mencampurkan kondisi penutup lahan yang berubah secara signifikan. Jika perubahan musiman penting, pertimbangkan membuat komposit untuk musim atau periode yang lebih spesifik.

---

## 1.5 Pemeriksaan Citra

Sebelum beralih ke pengumpulan sampel training, lakukan pemeriksaan berikut:

1. **Periksa AOI.** Pastikan batas wilayah benar dan tidak salah asset.
2. **Periksa jumlah scene.** Jika nol, longgarkan filter tanggal atau ambang metadata awan setelah mempertimbangkan kebutuhan analisis.
3. **Periksa awan dan bayangan.** Jangan hanya mengandalkan persentase awan pada metadata.
4. **Periksa band.** Pastikan semua band yang diperlukan tersedia dan nama band ditulis dengan benar.
5. **Pastikan resolusi konsisten.** Band 10 m dan 20 m perlu diproses pada grid kerja yang sesuai untuk analisis berikutnya.
6. **Pastikan skala reflektansi benar.** Jangan membagi data dua kali dengan 10.000.
7. **Periksa proyeksi dan ukuran piksel hasil ekspor.** Gunakan sistem koordinat yang sesuai untuk wilayah kajian dan kebutuhan analisis.
8. **Periksa hasil GeoTIFF.** Buka di QGIS dan pastikan cakupan, band, nilai piksel, serta NoData sesuai harapan.


---

# 2. Penentuan Skema Klasifikasi dan Data Latih

Tahap ini menetapkan kelas penutup lahan dan menyiapkan sampel yang akan digunakan untuk memeriksa karakteristik spektral serta melatih model klasifikasi. Sebelum membuat sampel, tentukan terlebih dahulu nama kelas, definisi setiap kelas, dan kode numeriknya.

## 2.1 Menentukan Skema Klasifikasi

Skema klasifikasi adalah daftar kelas yang ingin dipetakan. Dalam contoh ini, digunakan sembilan kelas penutup lahan.

| ID kelas | Nama kelas |
|---:|---|
| 1 | Hutan Lahan Kering |
| 2 | Hutan Mangrove |
| 3 | Kebun Campur |
| 4 | Padang Rumput |
| 5 | Pertanian Lahan Kering |
| 6 | Sawah |
| 7 | Lahan Terbangun |
| 8 | Lahan Terbuka |
| 9 | Badan Air |

`ID_Kelas` adalah label numerik yang digunakan dalam proses komputasi, sedangkan `KelasTupla` menyimpan nama kelas yang lebih mudah diinterpretasi oleh manusia. Gunakan pasangan ID dan nama kelas yang konsisten pada seluruh sampel. Setiap ID harus merujuk pada satu kelas saja.

## 2.2 Membuat Data Latih

Data latih dapat dibuat melalui beberapa cara. Pilih metode yang paling sesuai dengan kebutuhan, ketersediaan perangkat, dan sumber informasi referensi.

### Penentuan Jumlah Sampel Pelatihan

Salah satu heuristik yang umum digunakan untuk menentukan jumlah sampel pelatihan dikemukakan oleh van Niel et al. (2005), yaitu:

$$
n \approx 30 \times N \times K
$$

dengan:

- $n$ = jumlah minimum sampel pelatihan;
- $N$ = dimensi data atau jumlah variabel prediktor;
- $K$ = jumlah kelas klasifikasi.


Apabila sepertiga dari total sampel dialokasikan untuk pengujian (*testing*), jumlah sampel yang diperlukan perlu ditingkatkan. Dengan asumsi pembagian data sebesar dua pertiga untuk pelatihan dan sepertiga untuk pengujian, total sampel yang dibutuhkan adalah:

Selain itu, Jensen (2005) mengemukakan aturan praktis (*rule of thumb*) bahwa jumlah sampel pelatihan untuk setiap kelas setidaknya 10 kali jumlah saluran citra (*band*). Misalnya, jika citra yang digunakan memiliki 10 saluran, jumlah minimum sampel yang disarankan adalah 100 piksel per kelas.

Dalam penerapannya, sampel sebaiknya tersebar pada beberapa lokasi yang mewakili variasi kondisi dalam kelas tersebut. Sebagai contoh, 100 piksel yang dikumpulkan dari 10 lokasi berbeda umumnya lebih representatif dibandingkan 100 piksel yang hanya berasal dari satu lokasi.

**Catatan:** Kedua pendekatan tersebut merupakan pedoman awal, bukan ketentuan mutlak. Jumlah sampel yang dibutuhkan bergantung pada kompleksitas wilayah, jumlah kelas, karakteristik data prediktor, dan variasi spektral setiap kelas. Sampel pengujian sebaiknya dipisahkan dari sampel pelatihan dan tidak digunakan untuk melatih model.

### Opsi 1. Membuat sampel di QGIS

QGIS cocok digunakan ketika interpretasi dilakukan dengan membandingkan citra Sentinel-2 True Color ataupun Fales Color, basemap, dan batas wilayah secara bersamaan serta lebih fleksibel.

Alur umumnya:

1. Muat citra Sentinel-2 yang sudah disiapkan, AOI, dan basemap seperti Esri Satellite.
2. Siapkan layer shp titik atau poligon untuk data latih.
3. Buat atribut `ID_Kelas` dan `KelasTupla`.
4. Aktifkan mode edit, lalu digitasi lokasi atau area yang mewakili setiap kelas bisa menggunakan point ataupun polygon. Cek jumlah point atau polygon dengan menggunakan `Statistics by categories`.
5. Simpan hasil digitasi dan periksa kembali atribut serta geometri sampel.
6. Ekspor data sebagai shapefile, lalu unggah ke GEE sebagai asset untuk digunakan pada tahap berikutnya.

```{image} ../images/pic1.jpg
:align: center
:width: 600px
``` 600px
```

### Opsi 2. Membuat sampel melalui Google Earth

Google Earth Web dapat membantu mengenali objek di permukaan bumi melalui citra resolusi tinggi dan, jika tersedia terdapat Street View untuk mengidentifikasi kelas penutup lahan. Selain itu, salah satu keunggulan Google Earth adalah kemampuannya untuk share proyek dengan orang lain. Melalui fitur share proyek, kita dapat berkolaborasi dengan anggota tim untuk mengidentifikasi dan memetakan kelas tutupan lahan secara bersama-sama.

Alur umumnya:

1. Buka [Google Earth Web](https://earth.google.com/web/).
2. Impor AOI, misalnya dalam format KML atau shapefile yang sudah dikompresi menjadi ZIP.
3. Telusuri lokasi yang akan dijadikan sampel.
4. Gunakan fitur **Add placemark** untuk menandai lokasi referensi.
5. Catat kelas penutup lahan untuk setiap titik.
6. Ekspor atau pindahkan titik referensi ke perangkat lunak GIS untuk melengkapi atribut kelas dan menyiapkan data yang akan diunggah ke GEE.

Google Earth digunakan sebagai sumber pendukung interpretasi, bukan jaminan bahwa label selalu benar. Periksa kemungkinan perbedaan tanggal perekaman citra, perubahan lahan, dan ketidakjelasan objek.

### Opsi 3. Membuat sampel melalui di Google Earth Engine

Jika sampel sederhana diperlukan untuk latihan, geometri dapat digambar langsung di Code Editor GEE menggunakan alat gambar pada peta. Geometri yang dibuat akan muncul pada bagian **Imports**. Geometri tersebut kemudian dapat di import sebagai `FeatureCollection`.


```{image} ../images/pic2.png
:align: center
:width: 600px
``` 600px
```

```javascript
//Training Sample
// Gabungkan sesuai dengan jumlah penutup lahan yang digunakan
var roi_sample_training = hutan_lahan_kering.merge(hutan_mangrove).merge(kebun_campuran).merge(padang_rumput); 
```

Untuk analisis yang lebih luas, pengelolaan sampel di QGIS atau perangkat GIS lain lalu mengunggahnya ke GEE biasanya lebih mudah daripada menggambar seluruh sampel di Code Editor.

## 2.3 Memuat Data Latih dan Memeriksa Kelas

Setelah data latih selesai dibuat dan diunggah sebagai asset, panggil AOI dan data latih di GEE. Ganti ID asset dengan asset milik sendiri jika berbeda.

```javascript
// Memuat AOI dan data latih dari Assets
var aoi = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/aoi_kambera'
);

var trainingData = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/sampel_latih_kambera'
);

// Ambil kombinasi ID dan nama kelas yang unik.
var uniqueClasses = trainingData
  .distinct(['ID_Kelas', 'KelasTupla'])
  .sort('ID_Kelas');

var classIds = uniqueClasses.aggregate_array('ID_Kelas');
var classNames = uniqueClasses.aggregate_array('KelasTupla');

print('Jumlah sampel per ID kelas:', trainingData.aggregate_histogram('ID_Kelas'));
print('ID kelas:', classIds);
print('Nama kelas:', classNames);
```

Blok ini memuat data vektor dan memeriksa kelas yang tersedia. `aggregate_histogram()` menampilkan jumlah fitur untuk setiap ID kelas. Periksa hasilnya di **Console** sebelum melanjutkan.

## 2.4 Menentukan Warna dan Menampilkan Distribusi Sampel

Warna digunakan untuk membedakan kelas pada peta. Warna hanya membantu visualisasi, bukan bagian dari atribut yang digunakan Random Forest untuk belajar.

```javascript
// Warna visualisasi untuk setiap ID kelas.
var colorDict = ee.Dictionary({
  1: '#023f02', // Hutan Lahan Kering
  2: '#058e72', // Hutan Mangrove
  3: '#1E8C1E', // Kebun Campur
  4: '#A8C040', // Padang Rumput
  5: '#D4E800', // Pertanian Lahan Kering
  6: '#F5C518', // Sawah
  7: '#e80707', // Lahan Terbangun
  8: '#a65f04', // Lahan Terbuka
  9: '#0000FF'  // Badan Air
});

var trainingColored = trainingData.map(function(feat) {
  return feat.set('style', {
    color: colorDict.get(ee.Number(feat.get('ID_Kelas')).format())
  });
});

Map.addLayer(
  trainingColored.style({styleProperty: 'style'}),
  {},
  'Distribusi Data Latih'
);

Map.centerObject(aoi, 10);
```

**Interpretasi:** Periksa apakah sampel tersebar di lokasi yang sesuai dan apakah setiap kelas memiliki sampel yang cukup. Hindari mengumpulkan semua sampel dari satu lokasi kecil jika kelas tersebut memiliki variasi kondisi yang luas.

## 2.5 Memeriksa Keterpisahan Spektral Data Latih

Setelah sampel siap, periksa nilai spektral citra pada lokasi sampel. Grafik sebar (*scatter plot*) membantu melihat apakah kelas-kelas memiliki pola reflektansi yang berbeda atau saling tumpang tindih.

Contoh berikut menggunakan citra Sentinel-2 tanggal 22 September 2025. Pastikan citra tersebut mencakup AOI dan sesuai dengan periode interpretasi data latih.

```javascript
// Memuat citra Sentinel-2 dan memilih band yang akan diperiksa.
var s2 = ee.Image(
  'COPERNICUS/S2_SR_HARMONIZED/20250922T021221_20250922T021332_T50LRQ'
)
  .select([
    'B2', 'B3', 'B4', 'B5', 'B6',
    'B7', 'B8', 'B8A', 'B11', 'B12'
  ])
  .divide(10000)
  .clip(aoi);

// Gunakan proyeksi B12 sebagai acuan grid kerja 20 m.
var resample20 = function(image) {
  var projection = image.select('B12').projection();
  return image.reproject({
    crs: projection,
    scale: 20
  });
};

var s2_resample = resample20(s2);

var viz = {
  min: 0,
  max: 0.3,
  bands: ['B4', 'B3', 'B2']
};

Map.addLayer(s2_resample, viz, 'Sentinel-2 RGB 20 m');
Map.addLayer(
  trainingColored.style({styleProperty: 'style'}),
  {},
  'Sampel Data Latih'
);

// Ambil nilai band pada setiap fitur sampel.
var samples = s2_resample.sampleRegions({
  collection: trainingData,
  properties: ['ID_Kelas', 'KelasTupla'],
  scale: 20,
  geometries: false
});

print('Sampel dengan nilai spektral:', samples.limit(10));
```

Blok ini mengambil nilai band pada sampel training agar dapat dibandingkan dalam grafik. Parameter `geometries: false` menghindari penyertaan geometri pada setiap fitur keluaran karena grafik hanya memerlukan atribut kelas dan nilai band.

### Grafik sebar Red dan NIR

```javascript
var chart_red_nir = ui.Chart.feature.groups({
  features: samples,
  xProperty: 'B4',
  yProperty: 'B8',
  seriesProperty: 'KelasTupla'
})
.setChartType('ScatterChart')
.setOptions({
  title: 'Grafik Sebar Saluran Merah dan Inframerah Dekat',
  hAxis: {title: 'Pantulan Kanal Merah (B4)'},
  vAxis: {title: 'Pantulan Kanal NIR (B8)'},
  pointSize: 3,
  dataOpacity: 0.6
});

print('Scatter plot Red dan NIR', chart_red_nir);
```

Grafik ini membandingkan reflektansi band merah (B4) dan NIR (B8). Vegetasi yang sehat umumnya menyerap cahaya merah dan memantulkan NIR dengan kuat, tetapi pola aktual dapat berbeda karena kondisi vegetasi, kelembapan, bayangan, dan campuran piksel.

### Grafik sebar Red Edge dan SWIR

```javascript
var chart_rededge_swir = ui.Chart.feature.groups({
  features: samples,
  xProperty: 'B5',
  yProperty: 'B11',
  seriesProperty: 'KelasTupla'
})
.setChartType('ScatterChart')
.setOptions({
  title: 'Grafik Sebar Red Edge 1 dan SWIR 1',
  hAxis: {title: 'Pantulan Red Edge 1 (B5)'},
  vAxis: {title: 'Pantulan SWIR 1 (B11)'},
  pointSize: 3,
  dataOpacity: 0.6
});

print('Scatter plot Red Edge dan SWIR', chart_rededge_swir);
```

Grafik ini membandingkan band Red Edge 1 (B5) dan SWIR 1 (B11). Kombinasi ini dapat membantu memeriksa perbedaan respons spektral vegetasi, lahan terbuka, dan kelas lain, tetapi tidak menjamin seluruh kelas akan terpisah jelas.

### Grafik sebar NIR dan SWIR 2

```javascript
var chart_nir_swir2 = ui.Chart.feature.groups({
  features: samples,
  xProperty: 'B8',
  yProperty: 'B12',
  seriesProperty: 'KelasTupla'
})
.setChartType('ScatterChart')
.setOptions({
  title: 'Grafik Sebar NIR dan SWIR 2',
  hAxis: {title: 'Pantulan NIR (B8)'},
  vAxis: {title: 'Pantulan SWIR 2 (B12)'},
  pointSize: 3,
  dataOpacity: 0.6
});

print('Scatter plot NIR dan SWIR 2', chart_nir_swir2);
```

### Cara membaca grafik sebar

- Setiap titik merepresentasikan sampel dengan nilai reflektansi pada dua band.
- Warna/seri menunjukkan kelas yang berasal dari atribut `KelasTupla`.
- Jika titik suatu kelas berkumpul terpisah dari kelas lain, kedua band tersebut mungkin membantu membedakan kelas itu.
- Jika banyak kelas saling tumpang tindih, pasangan band tersebut belum cukup untuk memisahkan kelas secara jelas. Model masih dapat menggunakan band lain atau fitur tambahan.

Grafik sebar merupakan pemeriksaan awal, bukan pengganti evaluasi model. Keterpisahan yang tampak pada dua band tidak menjamin akurasi klasifikasi yang tinggi.

## 2.6 Catatan Penting Sebelum Melanjutkan

1. **Periksa atribut data latih.** Pastikan `ID_Kelas` numerik dan `KelasTupla` berisi nama kelas yang konsisten.
2. **Periksa jumlah sampel per kelas.** Jumlah fitur tidak selalu sama dengan jumlah piksel sampel, terutama jika data latih berbentuk poligon.
3. **Periksa lokasi sampel pada citra.** Sampel harus berada pada objek yang benar dan tidak mencakup batas beberapa kelas tanpa sengaja.
4. **Perhatikan tanggal referensi.** Citra Sentinel-2 yang digunakan untuk memeriksa spektral sebaiknya cukup sesuai waktunya dengan kondisi yang diberi label.
5. **Jangan menilai keterpisahan hanya dari satu grafik.** Periksa beberapa pasangan band dan, bila diperlukan, indeks spektral.
6. **Warna grafik hanya untuk tampilan.** Urutan warna grafik dapat berbeda dari warna pada `colorDict`; jangan menafsirkan warna seri sebagai ID kelas kecuali pemetaannya diatur secara eksplisit.
7. **Keterpisahan spektral bukan uji akurasi.** Tahap ini membantu menemukan sampel yang salah atau kelas yang tumpang tindih. Akurasi peta perlu diuji dengan data referensi independen pada tahap evaluasi.

**Tahap berikutnya:** Setelah skema kelas dan sampel diperiksa, lanjutkan ke *feature engineering* untuk menentukan band dan indeks yang akan menjadi masukan Random Forest.


# 3. Feature Engineering

*Feature engineering* merupakan proses membentuk atau menambahkan variabel prediktor dari data yang sudah tersedia agar model klasifikasi dapat membedakan kelas penutup lahan dengan lebih baik. Selain menggunakan nilai reflektansi dari band Sentinel-2, kita dapat menambahkan informasi topografi, indeks spektral, atau tekstur citra.

Pada praktikum ini, fitur yang digunakan terdiri atas dua kelompok:

1. **Fitur topografi**, untuk merepresentasikan kondisi elevasi dan bentuk permukaan bumi.
2. **Indeks spektral**, untuk menonjolkan karakteristik vegetasi, badan air, serta lahan terbuka atau terbangun.

Seluruh fitur tersebut nantinya digabungkan menjadi satu citra prediktor yang digunakan pada tahap klasifikasi Random Forest.

## 3.1 Menyiapkan Citra dan Wilayah Studi

Pertama, muat kembali batas wilayah studi (*Area of Interest* atau AOI) dan citra Sentinel-2 yang telah dipilih pada tahap sebelumnya.

```javascript
// Muat kembali AOI
var aoi = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/aoi_kambera'
);

// Muat citra Sentinel-2 dan pilih band yang digunakan
var s2 = ee.Image(
  'COPERNICUS/S2_SR_HARMONIZED/20250922T021221_20250922T021332_T50LRQ'
)
.select([
  'B2', 'B3', 'B4', 'B5', 'B6',
  'B7', 'B8', 'B8A', 'B11', 'B12'
])
.divide(10000)
.clip(aoi);
```

Band yang digunakan mencakup spektrum visible, red-edge, NIR, dan SWIR. Kombinasi spektrum ini diharapkan mampu mengenai karakteristik permukaan bumi.

## 3.2 Fitur Topografi

Karakteristik topografi dapat membantu membedakan kelas penutup lahan yang memiliki kondisi spektral serupa, tetapi berada pada lingkungan fisik yang berbeda. Pada praktikum ini, data topografi diperoleh dari NASADEM dan ALOS Landforms.

Fitur topografi yang digunakan meliputi:

| Fitur | Keterangan |
|---|---|
| Elevasi (*elevation*) | Ketinggian permukaan terhadap datum vertikal DEM. |
| Kemiringan lereng (*slope*) | Besarnya kemiringan permukaan tanah dalam derajat. |
| Arah hadap lereng (*aspect*) | Arah orientasi lereng dalam derajat. |
| Bentuk lahan (*landform*) | Kategori bentuk permukaan bumi berdasarkan produk ALOS Landforms. |

#### A. Mengambil Data Elevasi

Data elevasi diperoleh dari produk NASADEM dengan memilih band `elevation`.

```javascript
var elev = ee.Image('NASA/NASADEM_HGT/001')
  .select('elevation')
  .clip(aoi);
```

Elevasi dapat membantu membedakan karakteristik wilayah berdasarkan ketinggiannya.

#### B. Menghitung Kemiringan dan Arah Hadap Lereng

Kemiringan dan arah hadap lereng dihitung berdasarkan data elevasi menggunakan fungsi `ee.Terrain`.

```javascript
var slope = ee.Terrain.slope(elev);
var aspect = ee.Terrain.aspect(elev);
```

- **Slope** menunjukkan kemiringan permukaan dalam derajat.
- **Aspect** menunjukkan arah hadap lereng dalam derajat, dengan nilai yang merepresentasikan orientasi lereng.

Kedua fitur tersebut dapat digunakan sebagai informasi tambahan untuk membedakan kondisi bentang alam.

#### C. Mengambil Data Bentuk Lahan

Selain elevasi dan karakteristik lereng, kita menggunakan produk ALOS Landforms untuk memperoleh kategori bentuk lahan.

```javascript
var landform = ee.Image(
  'CSP/ERGo/1_0/Global/ALOS_landforms'
)
.select('constant')
.clip(aoi)
.rename('landform');
```

Berbeda dengan elevasi dan slope yang berupa nilai numerik kontinu, `landform` berisi kategori bentuk lahan. Kategori ini dapat memberikan informasi mengenai posisi dan karakteristik bentang alam.

#### D. Menggabungkan Fitur Topografi

Setelah semua fitur diperoleh, gabungkan band-band topografi menjadi satu citra.

```javascript
var topo = elev
  .addBands(slope)
  .addBands(aspect)
  .addBands(landform)
  .toFloat();
```

Fungsi `addBands()` digunakan untuk menggabungkan beberapa band dalam satu objek citra. Hasilnya adalah citra topografi dengan empat band prediktor.

Kita dapat memeriksa hasilnya melalui peta:

```javascript
Map.addLayer(elev, {
  min: 0,
  max: 1000
}, 'Elevation');

Map.addLayer(slope, {
  min: 0,
  max: 90
}, 'Slope');

Map.addLayer(aspect, {
  min: 0,
  max: 360
}, 'Aspect');

var landformvis = {
  min: 11,
  max: 42,
  palette: [
    '141414', '383838', '808080', 'ebeb8f',
    'f7d311', 'aa0000', 'd89382', 'ddc9c9',
    'dccdce', '1c6330', '68aa63', 'b5c98e',
    'e1f0e5', 'a975ba', '6f198c'
  ]
};

Map.addLayer(landform, landformvis, 'Bentuk Lahan');
```

Visualisasi membantu kita memeriksa distribusi spasial setiap fitur sebelum menggunakannya dalam klasifikasi.

## 3.3 Transformasi Spektral

Transformasi spektral dilakukan dengan mengombinasikan nilai reflektansi dari beberapa band untuk menghasilkan informasi yang lebih spesifik mengenai karakteristik permukaan bumi. Indeks spektral dapat membantu menonjolkan karakteristik tertentu, seperti vegetasi, badan air, dan lahan terbuka.

Pada praktikum ini, indeks spektral dihitung menggunakan dua pendekatan:
1. Menghitung indeks secara langsung menggunakan operasi matematika.
2. Menggunakan *library* indeks spektral untuk menghitung beberapa indeks sekaligus.

#### A. Menghitung Indeks Secara Langsung

**1. Normalized Difference Vegetation Index (NDVI)**

NDVI digunakan untuk menonjolkan karakteristik vegetasi berdasarkan perbedaan reflektansi pada kanal NIR dan kanal merah.

Rumus NDVI:

$$
NDVI = \frac{NIR - Red}{NIR + Red}
$$

Pada Sentinel-2, band B8 digunakan sebagai NIR dan B4 sebagai band merah.

**2. Normalized Difference Water Index (NDWI)**

NDWI yang digunakan pada contoh ini dihitung dari kanal hijau dan NIR.

$$
NDWI = \frac{Green - NIR}{Green + NIR}
$$


Implementasi kedua indeks tersebut di Google Earth Engine:

```javascript
var NDVI = s2
  .normalizedDifference(['B8', 'B4'])
  .rename('NDVI');

var NDWI = s2
  .normalizedDifference(['B3', 'B8'])
  .rename('NDWI');
```

Fungsi `normalizedDifference()` menghitung perbedaan ternormalisasi dari dua band. Hasilnya berupa band baru yang dapat digunakan sebagai prediktor.

**3. Enhanced Vegetation Index (EVI)**

EVI dirancang untuk meningkatkan sensitivitas terhadap vegetasi dan mengurangi pengaruh beberapa faktor latar belakang melalui penggunaan band biru, merah, dan NIR.

Rumus EVI:

$$
EVI = 2.5 \times
\frac{NIR - Red}
{NIR + 6Red - 7.5Blue + 1}
$$

Implementasinya:

```javascript
var EVI = s2.expression(
  '2.5 * ((NIR - RED) / (NIR + 6 * RED - 7.5 * BLUE + 1))',
  {
    BLUE: s2.select('B2'),
    NIR: s2.select('B8'),
    RED: s2.select('B4')
  }
).rename('EVI');
```

**Perhatikan:** Rumus EVI pada kode menggunakan B8 sebagai NIR. Hal ini perlu konsisten dengan definisi indeks yang digunakan.

#### B. Menggunakan Library Indeks Spektral

Jika kita perlu menghitung banyak indeks, penulisan rumus satu per satu dapat menjadi kurang praktis. Untuk mempermudah proses tersebut, kita dapat menggunakan *library* [Awesome Spectral Indices](https://github.com/awesome-spectral-indices/spectral).

Dokumentasi daftar indeks tersedia di [Awesome Spectral Indices Documentation](https://awesome-spectral-indices.github.io/awesome-spectral-indices/indices/).

Pertama, muat *library* tersebut:

```javascript
var spectral = require(
  'users/dmlmont/spectral:spectral'
);

// Periksa daftar indeks yang tersedia
print(spectral.indices);
```

Library ini menyediakan fungsi untuk menghitung berbagai indeks spektral. Nama indeks dan parameter band perlu disesuaikan dengan dokumentasinya.

**Menentukan band yang diperlukan**

Setiap indeks membutuhkan kombinasi band tertentu. Oleh karena itu, kita perlu memetakan nama band yang diharapkan oleh *library* ke band Sentinel-2.

Contoh untuk NDVI:

```javascript
var bands = {
  N: s2.select('B8'),
  R: s2.select('B4')
};

var index = spectral.computeIndex(
  s2,
  ['NDVI'],
  bands
);

print('Band hasil indeks:', index.bandNames());
```

Pada contoh ini:
- `N` merepresentasikan kanal NIR, yaitu B8.
- `R` merepresentasikan kanal merah, yaitu B4.
- `computeIndex()` menghitung NDVI berdasarkan citra dan parameter band yang diberikan.

Periksa nama band hasil perhitungan melalui `bandNames()` untuk memastikan indeks berhasil dibuat.

**Menghitung beberapa indeks sekaligus**

Untuk klasifikasi penutup lahan, kita akan menambahkan beberapa indeks yang diharapkan dapat membantu membedakan kelas-kelas yang memiliki karakteristik spektral berbeda.

| Indeks | Tujuan umum |
|---|---|
| NDVI | Menonjolkan karakteristik vegetasi. |
| NDREI | Memanfaatkan kanal red-edge untuk menganalisis karakteristik vegetasi. |
| DBSI | Menonjolkan karakteristik lahan kering atau terbuka. |
| MBI | Membantu membedakan karakteristik lahan terbuka dan area terbangun. |
| BLFEI | Indeks yang digunakan untuk membantu membedakan karakteristik lahan terbangun dan lahan terbuka. |
| MNDWI | Membantu menonjolkan karakteristik badan air. |

Perlu diingat bahwa kegunaan setiap indeks bergantung pada definisi rumus, band yang digunakan, dan kondisi wilayah penelitian. Indeks tidak selalu mampu memisahkan dua kelas secara sempurna.

Tentukan parameter band yang dibutuhkan:

```javascript
var param = {
  N: s2.select('B8'),
  R: s2.select('B4'),
  RE1: s2.select('B5'),
  S1: s2.select('B11'),
  G: s2.select('B3'),
  S2: s2.select('B12')
};
```

Selanjutnya, hitung indeks yang dipilih:

```javascript
var spectral_index = spectral.computeIndex(
  s2,
  ['NDVI', 'NDREI', 'DBSI', 'MBI', 'BLFEI', 'MNDWI'],
  param
);

print(
  'Indeks spektral untuk klasifikasi:',
  spectral_index.bandNames()
);
```

Pemeriksaan nama band penting untuk memastikan indeks yang dihasilkan tersedia sebelum digunakan pada tahapan selanjutnya.

Untuk memvisualisasikan NDVI:

```javascript
var ndvi = spectral_index.select('NDVI');

Map.addLayer(ndvi, {
  min: -1,
  max: 1,
  palette: ['blue', 'white', 'green']
}, 'NDVI');
```

Rentang visualisasi -1 hingga 1 merupakan rentang umum untuk NDVI. Warna biru, putih, dan hijau membantu memperlihatkan variasi nilai indeks pada wilayah studi.

## 3.4 Menggabungkan Semua Prediktor

Setelah fitur topografi dan indeks spektral, seluruhnya digabungkan menjadi satu citra prediktor.

```javascript
var final_predictor = spectral_index
  .addBands(topo)
  .toFloat();

// Periksa seluruh nama band prediktor
print(
  'Gabungan prediktor untuk klasifikasi:',
  final_predictor.bandNames()
);
```

Citra `final_predictor` merupakan kumpulan variabel yang akan digunakan untuk melatih model klasifikasi.

Dengan menggabungkan beberapa jenis informasi, model Random Forest memiliki lebih banyak informasi yang dapat dipertimbangkan ketika memisahkan kelas penutup lahan.

**Penting:** Penambahan jumlah prediktor tidak selalu meningkatkan akurasi. Prediktor yang tidak relevan atau saling membawa informasi serupa dapat menambah kompleksitas model. Kinerja kombinasi prediktor perlu dievaluasi melalui pengujian akurasi.

## 3.5 Mengekspor Prediktor (Opsional)

Jika diperlukan, salah satu band prediktor dapat diekspor ke Google Drive untuk diperiksa atau digunakan pada perangkat lunak GIS.

Contoh mengekspor NDVI:

```javascript
Export.image.toDrive({
  image: final_predictor.select('NDVI'),
  description: 'NDVI_Prediktor_Klasifikasi',
  scale: 20,
  region: aoi.geometry(),
  folder: 'Earth Engine',
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});
```

# 4. Hyperparameter Tuning Random Forest

## 4.1 Menyiapkan Data Prediktor

Pada tahap sebelumnya, kita telah membentuk beberapa kelompok prediktor yang berasal dari indeks spektral, data topografi, dan fitur tekstur. Seluruh prediktor perlu digabungkan menjadi satu citra agar dapat digunakan oleh model Random Forest.

```javascript
// Gabungkan indeks spektral, fitur topografi, dan tekstur
var final_predictor = spectral_index
  .addBands(topo)
  .addBands(glcm_filter)
  .toFloat();

// Periksa daftar band prediktor
print(
  'Daftar prediktor untuk klasifikasi:',
  final_predictor.bandNames()
);
```

Fungsi `addBands()` digunakan untuk menggabungkan band dari beberapa citra menjadi satu citra prediktor. Selanjutnya, `bandNames()` digunakan untuk memeriksa seluruh prediktor yang tersedia.

## 4.2 Membagi Data Latih dan Data Uji

Sebelum melatih model, data sampel dibagi menjadi dua kelompok, yaitu data latih (*training data*) dan data uji (*testing data*).

- **Data latih** digunakan untuk membangun model klasifikasi.
- **Data uji** digunakan untuk mengevaluasi kemampuan model dalam memprediksi sampel yang tidak digunakan selama pelatihan.

Pada praktikum ini, data dibagi dengan proporsi 70% untuk pelatihan dan 30% untuk pengujian.

```javascript
// Muat data sampel
var titik_sampel = ee.FeatureCollection(
  'projects/ee-diniyarti/assets/aoi_kambera'
);

// Tambahkan kolom acak untuk membagi data
var random_col = titik_sampel.randomColumn('random', 42);

// 70% untuk data latih
var titik_latih = random_col.filter(
  ee.Filter.lt('random', 0.7)
);

// 30% untuk data uji
var titik_tes = random_col.filter(
  ee.Filter.gte('random', 0.7)
);
```

Fungsi `randomColumn()` menambahkan nilai acak pada setiap fitur. Nilai tersebut digunakan untuk menentukan apakah sampel masuk ke kelompok pelatihan atau pengujian.

Parameter `42` merupakan *random seed*, yaitu nilai yang membantu menghasilkan pembagian acak yang dapat direproduksi kembali.

Selanjutnya, ekstrak nilai prediktor pada masing-masing kelompok sampel.

```javascript
// Ekstrak nilai prediktor untuk data latih
var training = final_predictor.sampleRegions({
  collection: titik_latih,
  properties: ['ID_Kelas'],
  scale: 20,
  tileScale: 16
});

// Ekstrak nilai prediktor untuk data uji
var test = final_predictor.sampleRegions({
  collection: titik_tes,
  properties: ['ID_Kelas'],
  scale: 20,
  tileScale: 16
});
```

Fungsi `sampleRegions()` mengambil nilai setiap band prediktor pada lokasi sampel dan menggabungkannya dengan atribut kelas yang terdapat pada data sampel. Dalam contoh ini, `ID_Kelas` digunakan sebagai label referensi untuk pelatihan dan evaluasi model.

Parameter `scale: 20` menentukan skala nominal pengambilan sampel sebesar 20 meter, sedangkan `tileScale` dapat membantu mengatasi keterbatasan memori pada proses komputasi.

**Penting:** Pembagian acak berdasarkan titik tidak selalu menjamin independensi spasial. Jika beberapa titik berasal dari lokasi yang berdekatan, data latih dan data uji dapat memiliki karakteristik yang sangat mirip. Untuk evaluasi yang lebih kuat, pembagian spasial dapat dipertimbangkan.

## 4.3 Hyperparameter Tuning dengan Grid Search

Random Forest memiliki sejumlah parameter yang dapat memengaruhi hasil klasifikasi. Pada praktikum ini, kita menguji dua parameter utama:

| Parameter | Fungsi |
|---|---|
| `numberOfTrees` | Menentukan jumlah pohon keputusan yang digunakan dalam Random Forest. |
| `variablesPerSplit` | Menentukan jumlah variabel prediktor yang dipertimbangkan pada setiap proses pemisahan node. |

Secara umum, jumlah pohon yang lebih banyak dapat membuat prediksi lebih stabil, tetapi juga meningkatkan waktu komputasi. Sementara itu, jumlah variabel yang dipertimbangkan pada setiap pemisahan dapat memengaruhi keragaman pohon dan kemampuan model membedakan kelas.

Tidak ada satu kombinasi parameter yang selalu terbaik untuk semua wilayah dan dataset. Karena itu, parameter perlu diuji dan dibandingkan menggunakan data evaluasi.

#### A. Menentukan Rentang Parameter

Pertama, tentukan nilai parameter yang akan diuji.

```javascript
// Rentang jumlah pohon
var numTreesList = ee.List.sequence(100, 500, 100);

// Rentang jumlah variabel pada setiap pemisahan
var variablesplit = ee.List.sequence(2, 10, 2);
```

Rentang tersebut menghasilkan lima nilai jumlah pohon, yaitu 100, 200, 300, 400, dan 500. Untuk parameter `variablesPerSplit`, nilai yang diuji adalah 2, 4, 6, 8, dan 10.

Dengan demikian, terdapat 5 × 5 = 25 kombinasi parameter yang akan diuji.

#### B. Melakukan Grid Search

Google Earth Engine tidak menyediakan satu fungsi khusus yang secara otomatis menjalankan seluruh proses *Grid Search* untuk kasus ini. Oleh karena itu, kita dapat membuat perulangan menggunakan `map()` untuk membangun dan mengevaluasi model pada setiap kombinasi parameter.

```javascript
var accuracies = numTreesList.map(function(numTrees) {
  return variablesplit.map(function(variablesPerSplit) {

    // Bangun model Random Forest
    var classifier = ee.Classifier.smileRandomForest({
      numberOfTrees: numTrees,
      variablesPerSplit: variablesPerSplit
    }).train({
      features: training,
      classProperty: 'ID_Kelas',
      inputProperties: final_predictor.bandNames()
    });

    // Evaluasi model menggunakan data uji
    var accuracy = test
      .classify(classifier)
      .errorMatrix('ID_Kelas', 'classification')
      .accuracy();

    // Simpan hasil pengujian
    return ee.Feature(null, {
      accuracy: accuracy,
      numberOfTrees: numTrees,
      variableSplit: variablesPerSplit
    });
  });
}).flatten();

// Ubah hasil menjadi FeatureCollection
var accuraciesFC = ee.FeatureCollection(accuracies);

print('Hasil Grid Search:', accuraciesFC);
```

Kode tersebut menguji setiap kombinasi parameter dengan melatih model menggunakan data latih, kemudian menghitung *overall accuracy* pada data uji.

Fungsi `errorMatrix()` membandingkan label referensi dengan hasil prediksi. Selanjutnya, `accuracy()` menghitung proporsi sampel yang diklasifikasikan dengan benar.

Hasil dari setiap kombinasi disimpan dalam `FeatureCollection` agar dapat dibandingkan dan divisualisasikan.

**Catatan metodologis:** Pada contoh ini, data uji digunakan untuk memilih parameter terbaik sekaligus mengevaluasi model.

#### C. Membandingkan Hasil Hyperparameter Tuning

Untuk memahami pengaruh parameter, kita dapat membuat grafik yang menunjukkan hubungan antara jumlah pohon dan akurasi model.

```javascript
var chart1 = ui.Chart.feature.groups({
  features: accuraciesFC,
  xProperty: 'numberOfTrees',
  yProperty: 'accuracy',
  seriesProperty: 'variableSplit'
}).setOptions({
  title: 'Hyperparameter Tuning: Akurasi vs Jumlah Pohon',
  vAxis: {
    title: 'Overall Accuracy'
  },
  hAxis: {
    title: 'Jumlah Pohon (numberOfTrees)'
  },
  lineWidth: 2,
  pointSize: 4
});

print(chart1);
```

Pada grafik tersebut, sumbu X menunjukkan jumlah pohon, sedangkan sumbu Y menunjukkan *overall accuracy*. Setiap seri mewakili nilai `variablesPerSplit` yang berbeda.

Grafik ini membantu kita mengamati apakah peningkatan jumlah pohon diikuti oleh peningkatan akurasi, serta apakah pengaruhnya berbeda pada setiap nilai `variablesPerSplit`.

Kita juga dapat membandingkan akurasi berdasarkan jumlah variabel yang dipertimbangkan pada setiap pemisahan.

```javascript
var chart2 = ui.Chart.feature.groups({
  features: accuraciesFC,
  xProperty: 'variableSplit',
  yProperty: 'accuracy',
  seriesProperty: 'numberOfTrees'
}).setOptions({
  title: 'Hyperparameter Tuning: Akurasi vs Variables per Split',
  vAxis: {
    title: 'Overall Accuracy'
  },
  hAxis: {
    title: 'Variables per Split'
  },
  lineWidth: 2,
  pointSize: 4
});

print(chart2);
```

Gunakan kedua grafik tersebut untuk mengidentifikasi kombinasi parameter yang menghasilkan akurasi tertinggi. Perlu diingat bahwa selisih akurasi yang kecil belum tentu menunjukkan peningkatan kinerja yang berarti secara statistik.

## 4.4 Memilih Kombinasi Parameter Terbaik

Setelah seluruh kombinasi diuji, urutkan hasil berdasarkan akurasi dari nilai tertinggi ke terendah. Kombinasi pada urutan pertama dipilih sebagai kandidat parameter terbaik.

```javascript
// Urutkan berdasarkan akurasi tertinggi
var bestCombo = accuraciesFC
  .sort('accuracy', false)
  .first();

// Periksa parameter terbaik
print('Kombinasi Parameter Terbaik:', bestCombo);
```

Hasilnya memuat nilai akurasi serta kombinasi `numberOfTrees` dan `variableSplit` yang menghasilkan akurasi tertinggi dalam pengujian.

**Catatan:** Parameter yang terpilih merupakan kombinasi terbaik di antara nilai yang diuji, bukan jaminan bahwa kombinasi tersebut merupakan parameter optimal untuk semua kemungkinan nilai.

## 4.5 Melatih Model Final dan Melakukan Klasifikasi

Setelah kombinasi parameter dipilih, latih kembali model Random Forest menggunakan parameter tersebut.

```javascript
// Ambil parameter terbaik
var bestNumTrees = bestCombo.getNumber('numberOfTrees');
var bestVarSplit = bestCombo.getNumber('variableSplit');

// Latih model final
var model_rf_final = ee.Classifier.smileRandomForest({
  numberOfTrees: bestNumTrees,
  variablesPerSplit: bestVarSplit
}).train({
  features: training,
  classProperty: 'ID_Kelas',
  inputProperties: final_predictor.bandNames()
});
```

Model yang dihasilkan kemudian diterapkan pada seluruh citra prediktor.

```javascript
// Terapkan model pada seluruh wilayah studi
var klasifikasi_final = final_predictor.classify(model_rf_final);
```

Fungsi `classify()` menerapkan model Random Forest pada setiap piksel yang memiliki nilai prediktor. Hasilnya adalah citra klasifikasi yang berisi label kelas untuk setiap piksel.

#### Menampilkan Hasil Klasifikasi

Gunakan warna yang konsisten dengan skema klasifikasi yang telah ditetapkan sebelumnya.

```javascript
var visParams = {
  min: 1,
  max: 9,
  palette: [
    '#023f02',
    '#058e72',
    '#1E8C1E',
    '#A8C040',
    '#D4E800',
    '#F5C518',
    '#e80707',
    '#a65f04',
    '#0000FF'
  ]
};

Map.centerObject(aoi, 11);

Map.addLayer(
  klasifikasi_final,
  visParams,
  'Peta Klasifikasi Final'
);
```

Peta yang dihasilkan memperlihatkan distribusi spasial sembilan kelas penutup lahan pada wilayah studi. Hasil ini selanjutnya perlu dievaluasi menggunakan data referensi yang tidak digunakan untuk melatih model.

## 4.6 Evaluasi Akurasi Klasifikasi

Evaluasi dilakukan untuk mengetahui seberapa baik model mengklasifikasikan sampel yang tidak digunakan selama pelatihan.

Pada tahap ini, model final diterapkan pada data uji, kemudian hasil prediksinya dibandingkan dengan label referensi.

```javascript
// Prediksi kelas pada data uji
var validated = test.classify(model_rf_final);

// Buat confusion matrix
var finalErrorMatrix = validated.errorMatrix(
  'ID_Kelas',
  'classification'
);

// Tampilkan confusion matrix
print('Confusion Matrix:', finalErrorMatrix);

// Overall Accuracy
print(
  'Overall Accuracy:',
  finalErrorMatrix.accuracy()
);

// Kappa
print(
  'Kappa:',
  finalErrorMatrix.kappa()
);
```

*Confusion matrix* menunjukkan hubungan antara kelas referensi dan kelas hasil prediksi. Matriks ini dapat digunakan untuk melihat kelas yang sering tertukar dengan kelas lainnya.

Beberapa metrik yang dapat digunakan untuk mengevaluasi hasil klasifikasi adalah:

- **Overall Accuracy (OA):** proporsi seluruh sampel uji yang diklasifikasikan dengan benar.
- **Producer's Accuracy (PA):** menunjukkan proporsi sampel referensi suatu kelas yang berhasil dipetakan dengan benar.
- **User's Accuracy (UA):** menunjukkan proporsi sampel yang dipetakan ke suatu kelas dan benar-benar termasuk kelas tersebut berdasarkan data referensi.
- **Kappa:** ukuran kesepakatan klasifikasi yang memperhitungkan kesepakatan yang mungkin terjadi secara kebetulan.

Untuk menghitung akurasi pada tingkat kelas:

```javascript
var producersAccuracy = finalErrorMatrix
  .producersAccuracy()
  .toList()
  .flatten();

var consumersAccuracy = finalErrorMatrix
  .consumersAccuracy()
  .toList()
  .flatten();

print(
  'Producer Accuracy:',
  producersAccuracy
);

print(
  'User Accuracy:',
  consumersAccuracy
);
```

Pada Google Earth Engine, `consumersAccuracy()` digunakan untuk memperoleh nilai yang lazim disebut *User's Accuracy*. Nilai tersebut membantu mengidentifikasi kelas yang hasil pemetaannya memiliki proporsi kesalahan relatif tinggi.

**Penting:** Hasil evaluasi hanya dapat dianggap representatif apabila data uji benar-benar independen dari proses pelatihan dan pemilihan parameter. Jika data uji telah digunakan untuk *Grid Search*, akurasi yang diperoleh dapat memberikan gambaran yang terlalu optimistis mengenai kemampuan generalisasi model.

## 4.7 Menganalisis Kontribusi Prediktor

Selain mengevaluasi akurasi, kita dapat memeriksa tingkat kepentingan relatif setiap prediktor dalam model Random Forest.

Informasi ini dapat diperoleh melalui fungsi `explain()`.

```javascript
// Ambil informasi model
var explain = model_rf_final.explain();

// Ambil nilai importance setiap prediktor
var featureImportance = ee.Dictionary(
  explain.get('importance')
);

// Ubah menjadi FeatureCollection
var importanceFC = ee.FeatureCollection(
  featureImportance.keys().map(function(key) {
    return ee.Feature(null, {
      band_name: key,
      importance: featureImportance.get(key)
    });
  })
);

// Urutkan dari importance tertinggi
var importanceCollection = importanceFC
  .sort('importance', false);

print(
  'Kontribusi relatif prediktor:',
  importanceCollection
);
```

Nilai *importance* memberikan gambaran mengenai kontribusi relatif setiap prediktor dalam model. Prediktor dengan nilai lebih tinggi memiliki kontribusi lebih besar menurut ukuran kepentingan yang digunakan oleh implementasi Random Forest tersebut.

Untuk memudahkan interpretasi, hasilnya dapat ditampilkan sebagai grafik batang.

```javascript
var importanceChart = ui.Chart.feature.byFeature({
  features: importanceCollection,
  xProperty: 'band_name',
  yProperties: ['importance']
}).setOptions({
  title: 'Kontribusi Relatif Prediktor',
  hAxis: {
    title: 'Prediktor'
  },
  vAxis: {
    title: 'Importance'
  },
  legend: {
    position: 'none'
  }
}).setChartType('BarChart');

print(importanceChart);
```

Grafik tersebut membantu kita mengidentifikasi prediktor yang paling berkontribusi dalam model. Namun, nilai *importance* tidak membuktikan hubungan sebab-akibat dan dapat dipengaruhi oleh korelasi antarvariabel serta karakteristik data.

## 4.8 Mengekspor Hasil Klasifikasi

Setelah proses klasifikasi dan evaluasi selesai, hasil klasifikasi dapat diekspor ke Google Drive dalam format GeoTIFF.

```javascript
Export.image.toDrive({
  image: klasifikasi_final,
  description: 'Klasifikasi_Kambera_S2_2025',
  scale: 20,
  region: aoi.geometry(),
  folder: 'Earth Engine',
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});
```

Parameter `scale: 20` menentukan resolusi spasial nominal keluaran sebesar 20 meter. Parameter `region` menentukan wilayah yang diekspor, sedangkan `crs` menentukan sistem koordinat file keluaran.

Opsi `cloudOptimized: true` menghasilkan GeoTIFF yang dioptimalkan untuk akses data secara efisien pada perangkat lunak yang mendukung Cloud Optimized GeoTIFF (COG).

Setelah kode dijalankan, periksa tab **Tasks** pada Code Editor GEE dan jalankan tugas ekspor. File hasil klasifikasi dapat digunakan untuk visualisasi, pemeriksaan kualitas, dan analisis lanjutan di perangkat lunak GIS.

# 5. Uji Akurasi Tematik


Uji akurasi tematik dilakukan untuk mengetahui seberapa baik hasil klasifikasi penutup lahan merepresentasikan kondisi aktual di lapangan atau pada data referensi. Pada praktikum ini, hasil klasifikasi Random Forest akan dibandingkan dengan data penguji akurasi yang telah disiapkan secara terpisah.

Evaluasi dilakukan menggunakan *confusion matrix* untuk memperoleh beberapa indikator, yaitu **Overall Accuracy (OA)**, **Producer's Accuracy (PA)**, **User's Accuracy (UA)**, dan **Kappa Coefficient**. Selain itu, F1-score dihitung untuk merangkum keseimbangan antara Producer's Accuracy dan User's Accuracy pada setiap kelas.

## 5.1 Menyiapkan Data Penguji Akurasi

Langkah pertama adalah memuat data penguji akurasi yang telah disiapkan sebelumnya dan disimpan sebagai *asset* di Google Earth Engine (GEE). Data ini harus memiliki atribut `ID_Kelas` sebagai kode kelas dan `KelasTupla` sebagai nama kelas penutup lahan.

```javascript
// Memuat data penguji akurasi
var ground_ref = ee.FeatureCollection(
  'projects/ee-agilfahrezy60/assets/Sesi2_Data_Penguji_Akurasi'
);

// Memeriksa jumlah sampel untuk setiap kelas
print(
  'Jumlah Sampel Per Kelas:',
  ground_ref.aggregate_histogram('ID_Kelas')
);

// Memeriksa ID dan nama kelas
var uniqueClasses = ground_ref.distinct('ID_Kelas');

print('ID Kelas:', uniqueClasses.aggregate_array('ID_Kelas'));
print('Nama Kelas:', uniqueClasses.aggregate_array('KelasTupla'));
```

Pemeriksaan ini bertujuan untuk memastikan bahwa data penguji akurasi memiliki atribut kelas yang sesuai dengan skema klasifikasi. Distribusi sampel setiap kelas juga perlu diperhatikan agar kelas yang jumlah sampelnya terlalu sedikit dapat diidentifikasi sebelum evaluasi dilakukan.

**Catatan:** `aggregate_histogram('ID_Kelas')` menghitung jumlah fitur untuk setiap nilai kelas. Sementara itu, `distinct('ID_Kelas')` mengambil satu fitur untuk setiap ID kelas yang unik. Kedua perintah tersebut membantu memeriksa data referensi sebelum digunakan dalam evaluasi.

## 5.2 Memuat Hasil Klasifikasi

Selanjutnya, muat peta penutup lahan hasil klasifikasi yang akan diuji. Pada contoh ini, peta klasifikasi Kambera tahun 2025 telah disimpan sebagai *asset* di GEE.

```javascript
// Memuat hasil klasifikasi
var lulcRaw = ee.Image(
  'projects/ee-agilfahrezy60/assets/Klasifikasi_Kambera_2025_S2'
);

// Menghilangkan piksel NoData yang memiliki nilai 0
var lulc = lulcRaw.updateMask(lulcRaw.neq(0));
```

Piksel bernilai `0` dikeluarkan dari analisis karena pada contoh ini nilai tersebut digunakan sebagai penanda NoData, bukan sebagai kelas penutup lahan.

Penggunaan `updateMask()` memastikan bahwa piksel tersebut tidak ikut diperlakukan sebagai hasil klasifikasi ketika nilai kelas diambil untuk proses evaluasi.

**Penting:** Pastikan nilai `0` memang merupakan NoData pada peta klasifikasi. Jika `0` merupakan kelas yang valid, penggunaan mask tersebut akan menghilangkan kelas itu dari evaluasi.

## 5.3 Memvisualisasikan Hasil Klasifikasi dan Data Referensi

Sebelum menghitung akurasi, tampilkan peta klasifikasi dan lokasi sampel penguji pada peta GEE. Visualisasi ini membantu memeriksa apakah lokasi sampel berada pada wilayah studi yang sesuai dan apakah sebarannya cukup mewakili kelas-kelas penutup lahan yang akan dievaluasi.

Skema klasifikasi yang digunakan terdiri atas sembilan kelas.

| ID Kelas | Nama Kelas | Warna |
|---:|---|---|
| 1 | Hutan Lahan Kering | `#023f02` |
| 2 | Hutan Mangrove | `#058e72` |
| 3 | Kebun Campur | `#1E8C1E` |
| 4 | Padang Rumput | `#A8C040` |
| 5 | Pertanian Lahan Kering | `#D4E800` |
| 6 | Sawah | `#F5C518` |
| 7 | Lahan Terbangun | `#e80707` |
| 8 | Lahan Terbuka | `#a65f04` |
| 9 | Badan Air | `#0000FF` |

Daftar kelas tersebut digunakan untuk mengatur warna hasil klasifikasi dan memberikan warna yang sesuai pada sampel referensi.

```javascript
// Skema kelas penutup lahan
var kelas = [
  {id: 1, name: 'Hutan Lahan Kering',     color: '#023f02'},
  {id: 2, name: 'Hutan Mangrove',         color: '#058e72'},
  {id: 3, name: 'Kebun Campur',           color: '#1E8C1E'},
  {id: 4, name: 'Padang Rumput',          color: '#A8C040'},
  {id: 5, name: 'Pertanian Lahan Kering', color: '#D4E800'},
  {id: 6, name: 'Sawah',                  color: '#F5C518'},
  {id: 7, name: 'Lahan Terbangun',        color: '#e80707'},
  {id: 8, name: 'Lahan Terbuka',          color: '#a65f04'},
  {id: 9, name: 'Badan Air',              color: '#0000FF'}
];

// Menyiapkan daftar ID, nama, dan warna kelas
var jumlah_kelas = kelas.length;
var id_kelas = kelas.map(function(c) {
  return c.id;
});
var nama_kelas = kelas.map(function(c) {
  return c.name;
});
var warna = kelas.map(function(c) {
  return c.color;
});

// Membuat dictionary warna berdasarkan ID kelas
var color_dict = ee.Dictionary(
  kelas.reduce(function(acc, c) {
    acc[c.id.toString()] = c.color;
    return acc;
  }, {})
);

// Menampilkan hasil klasifikasi
Map.addLayer(
  lulc,
  {min: 1, max: 9, palette: warna},
  'Klasifikasi LULC 2025'
);

// Memberikan warna pada sampel referensi berdasarkan kelas
var groundStyled = ground_ref.map(function(f) {
  var key = f.getNumber('ID_Kelas').int().format('%d');
  var color = color_dict.getString(key);

  return f.set('style', {
    color: color,
    fillColor: color,
    pointSize: 5,
    width: 1.2
  });
});

// Menampilkan sampel referensi
Map.addLayer(
  groundStyled.style({styleProperty: 'style'}),
  {},
  'Data Validasi (per kelas)'
);

Map.centerObject(ground_ref, 10);
```

Untuk memudahkan interpretasi peta, tambahkan legenda yang menjelaskan warna setiap kelas.

```javascript
// Membuat kotak warna untuk legenda
var makeColorBox = function(hexColor) {
  return ui.Label('', {
    backgroundColor: hexColor,
    padding: '8px',
    margin: '0 6px 2px 0',
    border: '1px solid rgba(0,0,0,0.25)'
  });
};

// Membuat panel legenda
var legend = ui.Panel({
  style: {
    position: 'bottom-right',
    padding: '8px 12px',
    backgroundColor: 'white',
    border: '1px solid #bbb'
  }
});

// Menambahkan judul legenda
legend.add(ui.Label('Kelas Penutup Lahan', {
  fontWeight: 'bold',
  fontSize: '11px',
  margin: '0 0 6px 0'
}));

// Menambahkan nama dan warna setiap kelas
for (var li = 0; li < jumlah_kelas; li++) {
  legend.add(ui.Panel({
    widgets: [
      makeColorBox(warna[li]),
      ui.Label(nama_kelas[li], {
        fontSize: '10px',
        margin: '2px 0'
      })
    ],
    layout: ui.Panel.Layout.Flow('horizontal')
  }));
}

legend.add(ui.Label('', {
  border: '0.5px solid #ddd',
  margin: '4px 0'
}));

Map.add(legend);
```

Setelah peta ditampilkan, periksa kesesuaian posisi sampel referensi dengan wilayah studi dan hasil klasifikasi. Visualisasi ini merupakan pemeriksaan awal, bukan pengganti perhitungan akurasi secara kuantitatif.

## 5.4 Menghitung Akurasi Tematik

Pada tahap ini, nilai kelas hasil klasifikasi diambil dari lokasi sampel referensi. Nilai tersebut kemudian dibandingkan dengan label kelas aktual yang terdapat pada atribut `ID_Kelas`.

Proses pengambilan nilai piksel dilakukan menggunakan `sampleRegions()`, sedangkan perbandingan kelas dilakukan menggunakan `errorMatrix()`.

```javascript
// Mengambil nama band hasil klasifikasi
var classBand = lulc.bandNames().get(0);

// Mengambil nilai piksel pada lokasi sampel referensi
var sampled = lulc.sampleRegions({
  collection: ground_ref,
  properties: ['ID_Kelas'],
  scale: 10,
  tileScale: 4
});

// Menentukan urutan ID kelas untuk confusion matrix
var order = ee.List(id_kelas);

// Membuat confusion matrix
var confMatrix = sampled.errorMatrix({
  actual: 'ID_Kelas',
  predicted: classBand,
  order: order
});

// Menampilkan hasil evaluasi
print('Uji Akurasi Tematik');
print('Confusion Matrix:', confMatrix);
print('Overall Accuracy:', confMatrix.accuracy());
print('Kappa Coefficient:', confMatrix.kappa());
print("Producer's Accuracy:", confMatrix.producersAccuracy());
print("User's Accuracy:", confMatrix.consumersAccuracy());
```

**Memahami indikator akurasi**

| Indikator | Penjelasan |
|---|---|
| Confusion Matrix | Tabel perbandingan kelas aktual dan kelas hasil klasifikasi. |
| Overall Accuracy (OA) | Proporsi seluruh sampel yang diklasifikasikan dengan benar. |
| Producer's Accuracy (PA) | Menggambarkan seberapa baik sampel aktual suatu kelas berhasil dikenali oleh klasifikasi. PA berkaitan dengan *omission error*. |
| User's Accuracy (UA) | Menggambarkan peluang bahwa suatu sampel yang dipetakan sebagai kelas tertentu memang termasuk kelas tersebut menurut data referensi. UA berkaitan dengan *commission error*. |
| Kappa Coefficient | Ukuran kesesuaian klasifikasi dan data referensi yang memperhitungkan kesesuaian yang mungkin terjadi secara kebetulan. |

OA dihitung dari jumlah sampel yang terklasifikasi benar dibagi jumlah seluruh sampel yang dievaluasi. Sementara itu, PA dan UA dihitung untuk setiap kelas berdasarkan nilai dalam confusion matrix.

Perlu diperhatikan bahwa `sampleRegions()` hanya menghasilkan sampel pada piksel yang tidak termask. Oleh karena itu, sampel yang berada pada piksel NoData tidak akan masuk ke dalam confusion matrix.

## 5.5 Menghitung F1-Score dan Mengekspor Hasil Evaluasi

Selain OA, PA, UA, dan Kappa, evaluasi juga dapat dilengkapi dengan F1-score. Metrik ini merupakan rata-rata harmonik antara PA dan UA sehingga membantu menilai keseimbangan kedua indikator tersebut pada setiap kelas.

Rumus F1-score adalah:

\[
F1 = 2 \times \frac{PA \times UA}{PA + UA}
\]

Jika PA dan UA dinyatakan dalam persen, F1-score yang dihasilkan juga berada dalam persen.

Berikutnya, hasil evaluasi disusun menjadi tabel yang memuat metrik untuk setiap kelas, kemudian diekspor sebagai file CSV agar dapat dianalisis lebih lanjut.

```javascript
// Mengambil metrik akurasi
var oa = confMatrix.accuracy();
var kappa = confMatrix.kappa();

var pa = confMatrix.producersAccuracy().toList().flatten();
var ua = confMatrix.consumersAccuracy().toList().flatten();

// Menyusun tabel metrik untuk setiap kelas
var barisKelas = order.map(function(id) {
  var idx = ee.Number(id).subtract(1);

  var paVal = ee.Number(pa.get(idx)).multiply(100);
  var uaVal = ee.Number(ua.get(idx)).multiply(100);

  // Menghitung F1-score
  var jumlahAkurasi = paVal.add(uaVal);

  var f1Val = ee.Algorithms.If(
    jumlahAkurasi.gt(0),
    ee.Number(2)
      .multiply(paVal)
      .multiply(uaVal)
      .divide(jumlahAkurasi),
    0
  );

  // Mengambil nama kelas
  var nama = ee.List(nama_kelas).get(idx);

  return ee.Feature(null, {
    'Indikator_atau_Kelas': nama,
    'ID_Kelas': id,
    'Producer_Accuracy_Persen': paVal,
    'User_Accuracy_Persen': uaVal,
    'F1_Score_Persen': f1Val
  });
});

// Menambahkan baris Overall Accuracy dan Kappa
var barisTotal = ee.List([
  ee.Feature(null, {
    'Indikator_atau_Kelas': 'OVERALL ACCURACY',
    'ID_Kelas': null,
    'Producer_Accuracy_Persen': oa.multiply(100),
    'User_Accuracy_Persen': null,
    'F1_Score_Persen': null
  }),
  ee.Feature(null, {
    'Indikator_atau_Kelas': 'KAPPA COEFFICIENT',
    'ID_Kelas': null,
    'Producer_Accuracy_Persen': kappa,
    'User_Accuracy_Persen': null,
    'F1_Score_Persen': null
  })
]);

// Menggabungkan hasil metrik per kelas dan metrik keseluruhan
var tabelAkurasi = ee.FeatureCollection(
  barisKelas.cat(barisTotal)
);

// Mengekspor hasil evaluasi ke Google Drive
Export.table.toDrive({
  collection: tabelAkurasi,
  description: 'Ringkasan_Akurasi_Kambera_2025',
  fileFormat: 'CSV',
  selectors: [
    'Indikator_atau_Kelas',
    'ID_Kelas',
    'Producer_Accuracy_Persen',
    'User_Accuracy_Persen',
    'F1_Score_Persen'
  ]
});
```

Setelah kode dijalankan, buka tab **Tasks** di GEE Code Editor dan jalankan tugas ekspor. File CSV akan disimpan ke Google Drive sesuai dengan pengaturan ekspor.

**Catatan interpretasi:** Nilai Kappa pada kode di atas diekspor dalam skala aslinya, bukan dikalikan 100. Dengan demikian, kolom `Producer_Accuracy_Persen` berisi nilai OA dalam persen pada baris *OVERALL ACCURACY*, tetapi berisi nilai Kappa pada skala aslinya pada baris *KAPPA COEFFICIENT*. Perhatikan perbedaan skala ini saat membaca tabel hasil.



---

## Referensi dokumentasi

- [Google Earth Engine: Sentinel-2 Surface Reflectance Harmonized](https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR_HARMONIZED)
- [Google Earth Engine: Cloud Score+](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_CLOUD_SCORE_PLUS_V1_S2_HARMONIZED)
- [Google Earth Engine: Export.image.toDrive](https://developers.google.com/earth-engine/apidocs/export-image-todrive)
- [Google Earth Engine: ee.Classifier.smileRandomForest](https://developers.google.com/earth-engine/apidocs/ee-classifier-smilerandomforest)
