# Klasifikasi Tutupan Lahan 1
## Unsupervised Classification
Praktikum hari ini kita akan mempelajari bagaimana cara membuat geometri zona tepi, menjalankan klasifikasi tutupan lahan berdasarkan algoritma earth engine, 
mereklasifikasi klasifikasi tersebut, dan menghitung luas tiap kelas tutupan lahan pada zona tepi.

### 1. Membuat zona buffer
Untuk memudahkan, aoi (feature table) diubah menjadi geometri. Dibuat juga area buffer dan dioperasikan sehingga mendapat area tepi
```javascript
// merubah aoi menjadi geometry
var region = aoi.geometry();

// membuat buffer pada aoi
var bufferOut = region.buffer(1000)
var bufferIn = region.buffer(-500)

Map.addLayer(bufferOut, {color: 'red'}, 'Buffer out');
Map.addLayer(region, {color: 'blue'}, 'No buffer');
Map.addLayer(bufferIn, {color: 'yellow'}, 'Buffer in');

// Mengurangi area bufferout dengan bufferin untuk mendapatkan edge area
var edgearea = bufferOut.difference({'right': bufferIn, 'maxError': 1});

Map.addLayer(edgearea, {'color': 'green'}, 'edge area');
```
### 2. Memanggil data Sentinel-2 dan cloudmasking
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
.clip(bufferOut);


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

print(sentinel)

//Menambah layer Peta bernama Papandayan Sentinel 2A
Map.addLayer(sentinel, {bands: ['B4','B3','B2'], max:3000}, 'Papandayan Sentinel 2A');
Map.centerObject(aoi,12.5)

```
### 3. Membuat training sample
Setelah foto yang akan dilakukan analisis didapatkan, kita bisa memulai proses klasifikasi dengan metode unsupervised. Dalam proses klasifikasi diperlukan training sample sebagai bahan ajar untuk classifier. Training untuk classifier dilakukan pertama-tama dengan mendefinisikan daerah yang akan di training menggunakan script ini.

```javascript
// training region in the full scene
var training = sentinel.sample({
  region: bufferOut, //daerah yang menjadi sumber training
  scale: 10, //resolusi sentinel-2
  numPixels: 5000 //jumlah sampel
});
```
Pada script ini, `region` menggambarkan daerah yang menjadi sumber training, dan scale menggambarkan resolusi, yang kali ini digunakan 10x10m sesuai dengan resolusi sentinel 2. 

Data dicluster dengan algoritma K Means, 10 berarti jumlah kelas yang dihasilkan (bisa dipilih sesuai keperluan).

```javascript
// train cluster on image
var clusterer = ee.Clusterer.wekaKMeans(10).train(training);
```
### 4. Menjalankan klasifikasi
Setelah classifier berhasil di training kita bisa memjalankan klasifikasi dengan script berikut dan hasilnya dapat kemudian dilihat dengan visualisasi menggunakan `Map.addLayer`

```javascript
// cluster the complete image
var hasilklasifikasi = sentinel.cluster(clusterer);

// Display the clusters with random colors.
Map.addLayer(hasilklasifikasi.clip(bufferOut).randomVisualizer(),{} , 'hasil klasifikasi');
```

### 5. Reklasifikasi Peta
Klasifikasi tutupan lahan yang dihasilkan bisa saja menghasilkan dua atau lebih kelas yang sebenarnya memiliki tutupan lahan yang sama namun dicluster dalam kelompok berbeda oleh algoritma yang kita gunakan (ex. hutan masuk ke cluster 2 dan 4). Maka itu perlu dilakukan reklasifikasi untuk mendapat hasil klasifikasi yang lebih baik. 
Ini kelas tutupan lahan yang akan kita gunakan ![kelas](https://github.com/lindypriyanka/EBA2020/blob/1caf000f9e3f2578dc5cfc25481ad63f155aa123/6.png)
```javascript
//Menentukan cluster tiap kelas
//fromCrater = 9 // ini nilai originalnya
//toCrater = [5]; // ini nilai yang akan kita berikan (bebas)
  
//fromForest = [1,4,0,5,8];
//toForest = [1,1,1,1,1];
  
//fromShrubs = [6,7]
//toShrubs = [2,2];

//fromAgricultural = [2]
//toAgricultural = [3];
  
//fromTea = [3]
//toTea = [4];

  //Reclassify the map
var reklasifikasi = hasilklasifikasi
.remap([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], 
[1, 1, 3, 4, 1, 1, 2, 1, 1, 5])
```
### 6. Membuat legenda
```javascript
//Create Legend
// set position of panel
var legend = ui.Panel({
  style: {
    position: 'bottom-left',
    padding: '8px 15px'
  }
});
// Create legend title
var legendTitle = ui.Label({
  value: 'Legend',
  style: {
    fontWeight: 'bold',
    fontSize: '18px',
    margin: '0 0 4px 0',
    padding: '0'
    }
});
// Add the title to the panel
legend.add(legendTitle);
// Creates and styles 1 row of the legend.
var makeRow = function(color, name) {
      
      // Create the label that is actually the colored box.
      var colorBox = ui.Label({
        style: {
          backgroundColor: color,
          // Use padding to give the box height and width.
          padding: '8px',
          margin: '0 0 4px 0'
        }
      });
      
      // Create the label filled with the description text.
      var description = ui.Label({
        value: name,
        style: {margin: '0 0 4px 6px'}
      });
      
      // return the panel
      return ui.Panel({
        widgets: [colorBox, description],
        layout: ui.Panel.Layout.Flow('horizontal')
      });
};
//  Palette with the colors
var names = ['Forest', 'Shrubs', 'Agricultural', 'Tea', 'Crater']
var palette = ['green', 'peru', 'palegreen', 'teal', 'cornsilk']
// Add color and and names
for (var i = 0; i < 5; i++) {
  legend.add(makeRow(palette[i], names[i]));
  }
// add legend to map
Map.add(legend);  

Map.addLayer(
    reklasifikasi, {'min': 1, 'max': 5, 'palette': palette}, 'reklasifikasi'
)
```

### 7. Export Data
```javascript
//Mendownload Data klasifikasi ke Google Drive masing-masing
Export.image.toDrive({ 
image:reklasifikasi, //variabel yang akan diunduh
description:'unsupervised', //nama file terunduh, jangan pake spasi
region: bufferOut,
folder: 'Praktikum EBA', //nama folder GDrive
scale:10,
});
```

### 8. Menghitung luas tiap kelas
```javascript
//CALCULATE AREA FOR EACH CLASS

//Find area in Square meter
var options = {
  title: 'Luas Area (Sqm)',
  hAxis: {title: 'Kelas Tutupan Lahan'},
  vAxis: {title: 'Area (Sqm)'},
  lineWidth: 1,
  series: {
    0: {color: 'green'},
    1: {color: 'peru'}, 
    2: {color: 'palegreen'},
    3: {color: 'teal'},
    4: {color: 'cornsilk'},
}};

print(reklasifikasi)

var areaChart = ui.Chart.image.byClass({
  image: ee.Image.pixelArea().addBands(reklasifikasi),
  classBand: 'remapped', 
  region: bufferOut,
  reducer: ee.Reducer.sum(),
  scale: 20,
}).setOptions(options);
print(areaChart);
```
