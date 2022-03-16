# NDVI dan Exporting Data
## NDVI: Normalized Difference Vegetation Index
### 1. Perhitungan manual
Perhitungan NDVI dapat dilakukan secara manual dengan mengikuti rumus NDVI untuk seluruh pixel sehingga menghasilkan peta NDVI.
Pada GEE, kita dapat melakukan operasi matematika seperti yang dibutuhkan untuk perhitungan ini.

Rumus NDVI:
```
NDVI = (NIR - RED)/(NIR + RED)
```

**Tahap pertama** adalah memunculkan _image_ yang ingin dianalisis. Metode dan _script_ yang digunakan sama dengan pertemuan-pertemuan sebelumnya. Filter yang digunakan juga masih sama, dengan filter awan, tanggal antar Januari 2021 sampai Desember 2021, dan area of interest menggunakan [shapefile](https://github.com/geraldyudha/EBA2022/blob/bef8af46159b07e323cc20b93051565d853c6071/CA_dan_TWA_Papandayan.zip) batas wilayah CA dan TWA Papandayan. Silahkan jika ingin merubah tanggal untuk melihat perubahan indeks vegetasi. 


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
var citra2021 = L8
    .filterDate('2021-01-01', '2021-12-31')
    .map(maskL8)
    .median();

var citra2021 = citra2021.clip(aoi);

// Display the results in a cloudy place.
Map.centerObject(aoi,12);
Map.addLayer(citra2021 ,{bands: ['B4', 'B3', 'B2'], max: 0.3},'Natural Colour');
```

**Tahap kedua** adalah melakukan kalkulasi sesuai dengan rumus NDVI
```javascript
// Compute the Normalized Difference Vegetation Index (NDVI).
var nir = citra2021.select('B5');
var red = citra2021.select('B4');
var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
```
Pada script ini, kita membuat dua variabel, yaitu variabel `nir` dan `red` yang masing-masing menggambarkan _image_ yang hanya terdiri dari band tersebut saja. Kemudian kita melakukan operasi matematika pada kedua variabel ini dengan mengurangi, lalu menambahkan, dan membagi hasil pengurangan dan penambahan tersebut.

**Tahap ketiga** setelah kita mendapatkan nilai NDVI, kita bisa langsung memvisualisasi dalam bentuk peta dengan `Map.addLayer`
```javascript
// Display the result
Map.addLayer(ndvi, {min: -1, max: 1, palette:['blue', 'white', 'green']}, 'Citra NDVI');
```
Dari hasil ini, terlihat bahwa daerah yang lebih hijau memiliki indeks vegetasi yang lebih tinggi
![ndvi1](https://github.com/geraldyudha/EBA2022/blob/99c1468de60e634c74a27844f08eac5a86a715c3/ContohNDVI1.png)

### 2. Perhitungan langsung
Selain menggunakan cara manual seperti diatas, karena NDVI sangat sering dipakai dalam _remote sensing_, GEE memiliki cara singkat untuk melakukan kalkulasi ini, yaitu dengan menggunakan fungsi `ee.image` seperti dibawah ini

```javascript
var ndvi2 = citra2021.normalizedDifference(['B5', 'B4']).rename('Citra NDVI');
```

Jika hasil ini di visualisasi, hasil yang didapatkan akan sama dengan hasil dengan metode manual
![ndvi2](https://github.com/geraldyudha/EBA2022/blob/99c1468de60e634c74a27844f08eac5a86a715c3/ContohNDVI2.png)

### 3. Hasil NDVI
Kita bisa melihat nilai NDVI untuk setiap pixel pada menu inspector hanya dengan mengklik titik pada peta, kemudian akan terliha nilai NDVI-nya. Nilai NDVI ini dapat membantu mengklasifikasi tingkat kehijauan, atau jenis penutupan lahan seperti yang dapat dilihat pada tabel dibawah ini

![tabel](https://github.com/lindypriyanka/EBA2020/blob/master/15.png)

### 4. Export hasil
Setelah selesai melakukan analisis NDVI, silahkan export gambar dalam format GeoTiff

```javascript
//Mendownload Data Citra Natural Color ke Google Drive masing-masing
citra2021 = citra2021.select('B4', 'B3', 'B2'); //memilih band mana saja yang akan dimasukkan gambar
Export.image.toDrive({ 
image:citra2021, //variabel yang akan diunduh 
description:'Natural_Color_Landsat_8', //nama file terunduh, jangan pake spasi 
region: aoi, 
folder: 'Praktikum EBA', //nama folder pada G-Drive 
scale:30, //ukuran pixel Landsat 8 30x30 m
}); 
 
//Mendownload Data NDVI ke Google Drive masing-masing
Export.image.toDrive({ 
image:ndvi2, //variabel yang akan diunduh
description:'NDVI_Landsat_8', //nama file terunduh, jangan pake spasi
region: aoi,
folder: 'Praktikum EBA', //nama folder GDrive
scale:30, //ukuran pixel Landsat 8 30x30 m
});
```

##(Bisa dicoba secara mandiri)
### 5. NDVI menggunakan citra Sentinel-2
Perbedaan utama dalam processing image pada sentinel-2 dibandingkan dengan landsat 8 adalah function yang berbeda untuk cloud masking dan nomor band yang digunakan karena memiliki urutan band berbeda untuk red dan near-infrared. Keuntungan menggunakan dataset sentinel 2 adalah resolusi spasial yang lebih tinggi yaitu 10x10m dibandingkan landsat 8  30x30m.

```javascript
//memanggil dataset Sentinel-2A
var sentinel = (ee.ImageCollection("COPERNICUS/S2_SR"))
//menentukan range data citra yang akan diambil
.filterDate('2021-01-01', '2021-12-01')

//Fungsi untuk menyeleksi tutupan awan pada citra
//5 berarti persentase awan, jadi hanya akan dipilih citra yg memiliki tutupan awan <5%
.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 5))
//ini adalah fungsi yang kita buat di atas untuk mereduksi awan
.map(maskS2clouds)

//metode yang digunakan dalam mozaik, dengan nilai median pada setiap pixel,
//bisa juga dengan mean, min, max, dll
.median()
//untuk memotong citra agar data yang ditampilkan hanya sebesar wilayah yang kita inginkan
.clip(aoi);

//fungsi untuk menghapus awan
function maskS2clouds(image) {
var qa = image.select('QA60');
// Bits 10 and 11 are clouds and cirrus, respectively.
var cloudBitMask = 1 << 10;
var cirrusBitMask = 1 << 11;
// Both flags should be set to zero, indicating clear conditions.
var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
.and(qa.bitwiseAnd(cirrusBitMask).eq(0));
return image.updateMask(mask).divide(1);
}

//Menampilkan tampilan citra True Color atau Natural Color
var Citra2021Sentinel = sentinel.select(['B4', 'B3', 'B2']); //kombinasi band natural color
//Mengatur parameter visualisasi Citra Natural Color
var imageparameter = { min: 0, max: 3000};
//Jika max semakin kecil, brightness semakin tinggi (terang-putih)
//Jika max semakin besar, brightness semakin rendah (gelap-hitam)

//Menambah layer Peta bernama ITB Nangor_Sentinel 2A
Map.addLayer(Citra2021Sentinel, imageparameter, 'Papandayan Sentinel 2A');

//Pengambilan Band untuk menghitung NDVI
var nir = sentinel.select('B8');
var red = sentinel.select('B4');
//Menghitung nilai NDVI dengan rumus
var ndvi = (nir.subtract(red)).divide(nir.add(red)).rename('NDVIS2A');
//Mengatur parameter visualisasi NDVI
//palette bisa dipilih warna-warnanya tinggal diganti saja
var NDVIparam = {min: -1, max: 1, palette:['blue', 'white', 'green']};
//Menampilkan peta hasil visualisasi acuan center extent
Map.centerObject(aoi,12.5); //peta akan ditampilkan berdasarkan wilayah AOI dengan zoom 12.5
//rentang nilai zoom 1-20, semakin besar semakin dekat

//Menambahkan layer peta NDVI Nangor 2021
Map.addLayer(ndvi, NDVIparam, 'NDVI Papandayan 2021');
```
**Perbandingan visualisasi ndvi dengan citra landsat (kiri) dan sentinel (kanan)**
![Landsat vs Sentinel](https://github.com/geraldyudha/EBA2022/blob/4ff482f49dfd7f12d61a3db93867fc8fe83adc38/ndvilandsatvssentinel.png)
