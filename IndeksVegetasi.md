# NDVI dan Exporting Data
## NDVI: Normalized Difference Vegetation Index
### 1. Perhitungan manual
Perhitungan NDVI dapat dilakukan secara manual dengan mengikuti rumus NDVI untuk seluruh pixel sehingga menghasilkan peta NDVI.
Pada GEE, kita dapat melakukan operasi matematika seperti yang dibutuhkan untuk perhitungan ini.

Rumus NDVI:
```
NDVI = (NIR - RED)/(NIR + RED)
```

**Tahap pertama** adalah memunculkan _image_ yang ingin dianalisis. Metode dan _script_ yang digunakan sama dengan pertemuan-pertemuan sebelumnya. Filter yang digunakan juga masih sama, dengan filter awan, tanggal antar januari 2019 sampai desember 2019, dan ROI di daerah Gunung Papandayan. Silahkan jika ingin merubah tanggal untuk melihat perubahan indeks vegetasi.

```javascript
// Function to mask clouds using the quality band of Landsat 8.
var maskL8 = function(image) {
  var qa = image.select('BQA');
  /// Check that the cloud bit is off.
  // See https://www.usgs.gov/land-resources/nli/landsat/landsat-collection-1-level-1-quality-assessment-band
  var mask = qa.bitwiseAnd(1 << 4).eq(0);
  return image.updateMask(mask);
}

// Map the function over one year of Landsat 8 TOA data and take the median.
var composite = L8
    .filterDate('2018-01-01', '2018-12-31')
    .map(maskL8)
    .median();

var roicomposite = composite.clip(roi);

// Display the results in a cloudy place.
Map.setCenter(107.722, -7.3181);
Map.addLayer(roicomposite);
```

**Tahap kedua** adalah melakukan kalkulasi sesuai dengan rumus NDVI
```javascript
// Compute the Normalized Difference Vegetation Index (NDVI).
var nir = roicomposite.select('B5');
var red = roicomposite.select('B4');
var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
```
Pada script ini, kita membuat dua variabel, yaitu variabel `nir` dan `red` yang masing-masing menggambarkan _image_ yang hanya terdiri dari band tersebut saja. Kemudian kita melakukan operasi matematika pada kedua variabel ini dengan mengurangi, lalu menambahkan, dan membagi hasil pengurangan dan penambahan tersebut.

**Tahap ketiga** setelah kita mendapatkan nilai NDVI, kita bisa langsung memvisualisasi dalam bentuk peta dengan `Map.addLayer`
```javascript
// Display the result
Map.addLayer(ndvi, {min: -1, max: 1, palette:['blue', 'white', 'green']}, 'NDVI image');
```
Dari hasil ini, terlihat bahwa daerah yang lebih hijau memiliki indeks vegetasi yang lebih tinggi
![ndvi1](https://github.com/lindypriyanka/EBA2020/blob/master/13.png)

### 2. Perhitungan langsung
Selain menggunakan cara manual seperti diatas, karena NDVI sangat sering dipakai dalam _remote sensing_, GEE memiliki cara singkat untuk melakukan kalkulasi ini, yaitu dengan menggunakan fungsi `ee.image` seperti dibawah ini

```javascript
var ndvi2 = roicomposite.normalizedDifference(['B5', 'B4']).rename('NDVI');
```

Jika hasil ini di visualisasi, hasil yang didapatkan akan sama dengan hasil dengan metode manual
![ndvi2](https://github.com/lindypriyanka/EBA2020/blob/master/14.png)

### 3. Hasil NDVI
Kita bisa melihat nilai NDVI untuk setiap pixel pada menu inspector hanya dengan mengklik titik pada peta, kemudian akan terliha nilai NDVI-nya. Nilai NDVI ini dapat membantu mengklasifikasi tingkat kehijauan, atau jenis penutupan lahan seperti yang dapat dilihat pada tabel dibawah ini

![tabel](https://github.com/lindypriyanka/EBA2020/blob/master/15.png)

### 4. Export hasil
Setelah selesai melakukan analisis NDVI, silahkan export gambar dalam format GeoTiff seperti minggu kemarin

```javascript
//export map
Export.image.toDrive({
  image: ndvi,
  fileNamePrefix:"ndvi",
  region: roi,
  scale: 30
})
