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

Dokumentasi daftar indeks tersedia di [Awesome Spectral Indices Documentation](https://awesome-ee-spectral-indices.readthedocs.io/en/latest/list.html).

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

Random Forest menggabungkan banyak pohon keputusan. Hyperparameter tuning membandingkan pengaturan model untuk mencari konfigurasi yang sesuai.

## 4.1 Menentukan Parameter
Menentukan parameter yang diuji, misalnya `numberOfTrees`, `variablesPerSplit`, dan `minLeafPopulation`.

## 4.2 Membandingkan Konfigurasi
Melatih dan mengevaluasi beberapa kombinasi parameter menggunakan data validasi.

## 4.3 Memilih Model Terbaik
Memilih konfigurasi berdasarkan metrik validasi. Data uji akhir tidak digunakan dalam proses tuning.

> Kode tahap ini akan dilengkapi setelah skrip tuning diberikan.

# 5. Uji Akurasi Tematik

Tahap ini mengukur kesesuaian hasil klasifikasi dengan data referensi yang independen.

## 5.1 Membuat Confusion Matrix
Membandingkan kelas hasil prediksi dengan kelas referensi.

## 5.2 Menghitung Metrik Akurasi
Menghitung Overall Accuracy, Producer's Accuracy, dan User's Accuracy. F1-score dapat ditambahkan jika diperlukan.

## 5.3 Menginterpretasikan Hasil
Menilai kelas yang sering tertukar dan keterbatasan peta klasifikasi.

> Kode tahap ini akan dilengkapi setelah skrip uji akurasi diberikan. Gunakan data uji yang tidak dipakai untuk pelatihan atau tuning.

---

## Referensi dokumentasi

- [Google Earth Engine: Sentinel-2 Surface Reflectance Harmonized](https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR_HARMONIZED)
- [Google Earth Engine: Cloud Score+](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_CLOUD_SCORE_PLUS_V1_S2_HARMONIZED)
- [Google Earth Engine: Export.image.toDrive](https://developers.google.com/earth-engine/apidocs/export-image-todrive)
- [Google Earth Engine: ee.Classifier.smileRandomForest](https://developers.google.com/earth-engine/apidocs/ee-classifier-smilerandomforest)
