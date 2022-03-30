# Klasifikasi Tutupan Lahan 2
## Supervised Classification
Praktikum ini kita  mempelajari bagaimana membuat klasifikasi dengan sample training yang kita tentukan sendiri.

### 1. Membuat zona buffer
Untuk memudahkan, aoi (feature table) diubah menjadi geometri. Dibuat juga area buffer dan dioperasikan sehingga mendapat area tepi
```javascript
// merubah aoi menjadi geometry
var region = aoi.geometry();

// membuat buffer pada aoi
var bufferOut = region.buffer(1000) //jarak buffer dalam meter, bisa diatur sesuai konteks
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
```
Citra ditampilkan pada peta dengan kombinasi band natural colour dan kombinasi khusus agrikultur untuk memudahkan
menentukan training sample.
```
//Menambah layer Peta bernama Papandayan Sentinel 2A
Map.addLayer(sentinel, {bands: ['B4','B3','B2'], max:3000}, 'Papandayan Sentinel 2A');
Map.addLayer(sentinel, {bands: ['B11','B8','B2'], max:3000}, 'Kombinasi');
Map.centerObject(aoi,12.5)

```
### 2. Mengumpulkan _training data_
Pada klasifikasi kali ini kita akan menggunakan 7 kelas seperti pada gambar ini:

![class](https://github.com/lindypriyanka/EBA2020/blob/master/6.png)

Seperti metode sebelumnya, kita perlu melakukan _training_ untuk classifier yang akan kita gunakan. Namun dalam metode ini, kita menentukan sendiri data sampel training yang akan digunakan. Cara membuat data _training sample_ adalah dengan menambahkan ***geometry*** 

Kita perlu membuat satu layer geometry untuk setiap kelas dan memberi nama geometry sesuai nama kelas. Kita juga perlu mengganti jenis geometry menjadi `featureCollection` pada settings dan menambahkan properties. Properties ini menjadi penanda setiap kelompok kelas ketika nanti disatukan. Silahkan beri nama dan beri nomor dimulai dari 0 untuk setiap kelas.

![feature](https://github.com/geraldyudha/EBA2022/blob/89eaf8658ce8108eff6508d7cd873d1023cc4941/1.png)

Training sample setiap kelas dipilih dengan membuat polygon, rectangle, atau juga bisa point pada daerah yang merepresentasikan kelas tersebut. Semakin banyak geometry yang dibuat akan semakin baik hasil yang didapatkan. Geometry yang dibuat sebisa mungkin harus beragam dan representatif terhadap kelas tersebut.

![trainingsample](https://github.com/geraldyudha/EBA2022/blob/89eaf8658ce8108eff6508d7cd873d1023cc4941/2.png)

Setelah training sample seluruh kelas terbuat, kita harus menyatukan data sehingga menjadi satu variable, data disatukan dengan
script berikut:
```javascript
//Merge training sample
var sample = forest.merge(shrubs).merge(agricultural_fields).merge(buildings)
              .merge(tea_plantation).merge(crater);
```

### 3. Membuat _training data_
Dalam klasifikasi, classifier memerlukan data dari berbagai jenis band yang dimiliki citra. Untuk memunculkan data ini, kita harus membuat training data. Cara membuat training data ini hampir sama dengan metode sebelumnya
```javascript
// Specify and select bands that will be used in the classification.
var bands = [
  'B1', 'B2', 'B3', 'B4', 'B5', 'B6', 'B7', 'B8', 'B8A', 'B9', 'B11',
  'B12']
  
  //Create training from geometry to image
  var training = sentinel.select(bands).sampleRegions({
collection: sample,
properties: ['lc'],
scale: 10
});
```

### 4. _Training classifier_
Langkah selanjutnya adalah untuk men-_train_ classifier kita dengan script berikut. Pastikan bahwa `classProperty` sesuai dengan nama properties tiap kelas yang kita buat di awal. `features` diisi dengan training data yang telah kira miliki, dan `inputProperties` diisi oleh band-band yang sudah kita pilih di awal
```javascript
//Train the classifier
var classifier = ee.Classifier.smileCart().train({
features: training,
classProperty: 'lc',
inputProperties: bands
});
```

### 5. Melakukan klasifikasi
Langkah terakhir adalah melakukan klasifikasi kepada gambar yang telah kita pilih diawal. Hasil kemudian dapat divisualisasi dengan command `Map.addLayer` kita juga dapat memilih palette dari setiap kelas dengan menggunakan kode HTML.

```javascript
//Run the classification
var classified = sentinel.select(bands).classify(classifier);
Map.addLayer(classified,
{min: 0, max: 6, palette:['#28c310','#ecaf4a','#4ad9ff','#1043c2','#1043c2','#ff5319','#2bff8e','#fcffae']},
'classification');
```

### 6. Membuat legenda
Disesuaikan nama, palette, dan min max kelas yang masuk dalam legenda

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
var names = ['Forest', 'Shrubs', 'Agricultural','Water Bodies','Buildings', 'Tea', 'Crater']
var palette = ['#28c310','#ecaf4a','#4ad9ff','#1043c2','#ff5319','#2bff8e','#fcffae']
// Add color and and names
for (var i = 0; i < 7; i++) { //diatur menyesuaikan jumlah kelas
  legend.add(makeRow(palette[i], names[i]));
  }
// add legend to map
Map.add(legend);  
```

### 7. Menghitung luas tiap kelas
Luas area dapat dihitung dari hasil klasifikasi, dapat diatur region yang akan dihitung luas areanya, kali ini kita akan mempelajari luas tutupan lahan pada `edgearea`
```javascript
//CALCULATE AREA FOR EACH CLASS

//Find area in Square meter
var options = {
  title: 'Luas Area (Sqm)',
  hAxis: {title: 'Kelas Tutupan Lahan'},
  vAxis: {title: 'Area (Sqm)'},
  lineWidth: 1,
  series: {
    0: {color: '#28c310'},
    1: {color: '#ecaf4a'}, 
    2: {color: '#4ad9ff'},
    3: {color: '#1043c2'},
    4: {color: '#ff5319'},
    5: {color: '#2bff8e'},
    6: {color: '#fcffae'},
}};

var areaChart = ui.Chart.image.byClass({
  image: ee.Image.pixelArea().addBands(classified),
  classBand: 'classification', 
  region: edgearea,
  reducer: ee.Reducer.sum(),
  scale: 10,
}).setOptions(options);
print(areaChart);
```
