---
layout: post
title:  "E-commerce Data Analysis using Statistics and Machine Learning"
date:   2021-08-22 18:33:00 +0700
categories: experience
---
<style>
    p.img-caption {
        text-align: center;
        font-size: 0.9rem;
    }

    cimg + p {
        margin: 0;
        text-align: center;
    }
</style>
## Background
> "Data is the new oil." - Wired.com

Pada tahun 2014, situs berita wired.com mempublikasikan sebuah berita dengan judul "_Data Is the New Oil of the Digital Economy_". Judul ini seolah seperti memberikan otoritas kepada data untuk menentukan arah perkembangan ekonomi digital. Data memang memiliki sifat yang serupa seperti minyak. Ketika minyak masih dalam bentuk mentah jumlahnya sangat banyak, tetapi tidak bisa dimanfaatkan. Namun setelah melalui proses ekstraksi, pengolahan, dan penelitian pada akhirnya minyak dapat digunakan untuk bahan bakar, bahan produksi material seperti plastik, hingga produk-produk kecantikan. Data dalam bentuk mentah jumlahnya juga sangat banyak, namun tidak bermanfaat karena sulit untuk menemukan informasi yang berharga. Proses ekstraksi, pengolahan, dan analisis juga diperlukan untuk dapat menemukan informasi yang berharga pada kumpulan data.

Pengolahan dan analisis data dapat dilakukan secara manual oleh manusia. Jika data berukuran besar atau diperbarui dengan sangat cepat, proses pengolahan dan analisis yang akan memakan waktu yang lama. Pada kondisi seperti ini komputer adalah solusinya. Komputer dapat mengolah dan melakukan analisis pada data dengan sangat cepat dan akurat. Tentu, manusia masih memegang peranan penting untuk merancang alur dan prosedur pengolahan serta analisis yang dilakukan pada data. Namun dengan kecepatan dan akurasi komputasi yang dimiliki oleh komputer, beban pekerjaan manusia menjadi jauh lebih ringan. Proses analisis dengan memanfaatkan komputer ini disebut dengan istilah _Machine Learning_.

Informasi hasil analisis data kemudian digunakan untuk membantu dalam proses pengambilan keputusan, perencanaan, personalisasi, dan banyak lainnya. Pada konteks e-commerce keputusan yang dibuat meliputi ranah pemasaran, penjualan, pengiriman barang, hingga akuisisi pengguna.

Tahapan analisis data e-commerce yang saya lakukan dibagi kedalam tiga bagian. Bagian pertama adalah tahap _data engineer_ untuk melakukan ekstraksi dan pengolahan data. Bagian selanjutnya adalah tahap _data analysis_ untuk melakukan analisis data menggunakan grafik dan statistik untuk menghasilkan informasi yang bermanfaat. Tahap ketiga adalah _data science_ untuk melakukan analisis menggunakan bantuan komputer. Ketiga tahapan ini tidak senantiasa dilakukan secara linear dan berurutan melainkan secara non-linear dan iteratif.

## Data Engineer
<!-- / tampilkan overview data dan fact yg dipake utk part da + ds -->
Seperti yang sudah saya sebutkan pada bagian sebelumnya, data dalam bentuk mentah dianggap tidak berharga karena sulit untuk menghasilkan informasi penting dari jutaan angka dan huruf. Pada bagian pertama ini, saya akan memberikan _treatment_ kepada data mentah untuk menghasilkan struktur dan hubungan yang lebih jelas.

Data mentah saya dapatkan dari file csv (_Comma Separated Value_) yang diberikan oleh mentor. Katanya sih data ini sengaja dibuat _"kotor"_ sebagai bentuk tantangan untuk melakukan pembersihan atau pengolahan. Data tersebut merupakan data penjualan dari suatu e-commerce yang memuat informasi terkait pengguna, penjual, produk, pesanan, ulasan, dan pembayaran.

Pada bagian ini saya tidak memiliki urutan pengerjaan dengan jelas, namun yang pasti tahap _data engineer_ ini saya bagi kedalam dua bagian yaitu:
1. Konfigurasi _source database_, dan
2. Pembangunan _data warehouse_

### Source Database Logical Design
File csv tidak cocok untuk kebutuhan analisis dan akses data secara cepat dan kompleks. Oleh karena itu pada tahap pertama dalam _data engineer_ ini saya akan membuat basis data untuk menyimpan data untuk kebutuhan pemrosesan. Keunggulan dari penggunaan basis data adalah kemudahan pengaksesan data, manajemen akses, dan kemampuan penyajian data dengan transformasi kompleks menggunakan SQL.

Sistem manajemen basis data yang saya gunakan adalah PostgreSQL versi 13.

<cimg></cimg>
![Source Database Logical Design (ERD)](https://storage.googleapis.com/dionricky-blog/ecommerce-data-analysis/erd_db_updated.png)
<p class='img-caption'>Gambar 1.1 Database ERD</p>

Gambar 1.1 menunjukkan _logical design_ dari basis data sumber. Keseluruhannya terdapat 7 tabel yang terhubung menggunakan asosiasi _one-to-many_ yang dilambangkan dengan `[1, 1] <-> [1, *]` atau _many-to-many_ yang dilambangkan dengan `[1, *] <-> [1, *]`.

Data pengguna, penjual, dan produk disimpan dalam tabel `user`, `seller`, dan `products`. Tabel-tabel tersebut menyimpan informasi yang dapat digunakan untuk mengidentifikasi entitas yang satu dengan yang lain. Entitas ini nantinya yang akan digunakan sebagai "aktor" dalam transaksi dan disimpan dalam tabel lain sebagai _foreign key_.

Data transaksi pengguna disimpan pada tabel `order`. Tabel tersebut menyimpan informasi terkait status transaksi, tanggal transaksi, dan pengguna yang melakukan transaksi.

Dalam sebuah transaksi, pengguna dapat membeli satu atau lebih produk. Semua data terkait produk yang dibeli dalam satu transaksi disimpan dalam tabel `order-item`. Tabel `order` dengan `order-item` memiliki hubungan _one-to-many_ dimana satu `order` dapat diasosiasikan dengan satu atau lebih `order-item`.

Saat melakukan transaksi, pengguna perlu membayar apa yang ia beli. Data pembayaran yang digunakan oleh pengguna disimpan dalam tabel `payment`. Tabel `order` memiliki hubungan _one-to-many_ dengan tabel `payment` sehingga satu `order` dapat memiliki satu atau lebih pembayaran.

Setelah melakukan transaksi, pengguna dapat memberikan ulasan terhadap kualitas dari produk yang dibeli. Tabel `feedback` digunakan untuk menyimpan ulasan-ulasan dari pengguna. Tabel ini juga memiliki hubungan _one-to-many_ dengan tabel `order`.

### Pembangunan Data Warehouse
Setelah melakukan perancangan pada basis data sumber, langkah selanjutnya adalah pembuatan _data warehouse_.

Untuk keperluan analitik diperlukan suatu _repository_ yang dapat menyimpan data secara historikal, berorientasi subjek, dan terintegrasi. Basis data tidak tepat digunakan karena fokusnya adalah untuk kebutuhan operasional. _Data warehouse_ adalah pilihan yang banyak digunakan. Konsep dasar _data warehouse_ yaitu:
1. Berorientasi subjek: Artinya _data warehouse_ memiliki fokus atau tema tertentu seperti _sales_, _marketing_, atau _production_.
2. Terintegrasi: _Data warehouse_ dapat menggabungkan data dari berbagai sumber dalam satu tempat dengan struktur yang telah disetarakan. Data mentah dapat berasal dari file json, csv, atau bahkan dari log suatu komputer.
3. Historikal: _Data warehouse_ menyimpan seluruh perubahan data yang terjadi pada sumber. Kebutuhan analitik untuk mengetahui kondisi masa lalu dapat diatasi dengan menggunakan _data warehouse_.
4. _Non-volatile_: Operasi _update_ jarang atau hampir tidak dilakukan sama sekali pada _data warehouse_. Apabila ada perubahan pada data sumber, _data warehouse_ dapat mengatasinya dengan menyimpan sebagai data baru atau dengan strategi tertentu misalnya SCD (_Slowly Changing Dimension_).

Ralph Kimball, bapak _data warehouse_, menciptakan konsep pemodelan _data warehouse_ yang dirancang khusus untuk memenuhi kebutuhan analisis data numerik seperti nilai, jumlah, berat, dan lainnya yang disebut dengan _dimensional modelling_.

#### Warehouse Dimensional Modelling
Bagian penting dari dimensional modelling adalah pembagian tabel kedalam dua peran yaitu sebagai _fact table_ dan _dimension table_. Pembagian peran tabel ini mempermudah dalam proses pengambilan informasi dan untuk menghasilkan laporan.

_Fact_ adalah ukuran atau metrik dari proses bisnis. Misalnya pada proses bisnis penjualan, maka faktanya adalah jumlah penjualan. Fakta ini dapat diberi dimensi untuk menghasilkan informasi tambahan. Seperti jumlah penjualan per bulan, jumlah penjualan per kuartal, atau jumlah penjualan pada wilayah tertentu.

_Fact table_ adalah tabel yang berisikan setidaknya satu fakta, meskipun ada juga jenis _fact table_ yang tidak memiliki fakta, dan dimensi sebagai konteks dari fakta tersebut.

_Dimension table_ memberikan konteks terhadap kejadian-kejadian pada proses bisnis. Sederhananya, dimensi memberikan informasi mengenai siapa, apa, dimana, dan kapan dari fakta. Pada proses bisnis penjualan, dimensinya adalah:

- Siapa - Nama pembeli
- Dimana - Lokasi
- Apa - Nama produk
- Kapan - Tanggal dan jam

Dimensi pada tabel fakta tidak selalu memiliki semua komponen Siapa-Dimana-Apa-Kapan, namun idealnya semakin banyak informasi yang dapat ditambahkan maka semakin baik.

_Data mart_ merupakan _data warehouse_ yang lebih spesifik ke satu subjek misalnya sales, HR, atau marketing.

Tabel-tabel fakta yang saya buat dalam tahap ini adalah:
1. Fact order
2. Fact feedback

Data mart yang saya buat dalam tahap ini adalah:
1. Data mart customer RFM

<!--
/ fact order -->
#### 1. Fact Order
Penjualan adalah aspek penting pada e-commerce. Analisis penjualan bermanfaat untuk menilai performa periodik, memprediksi permintaan masa depan, dan untuk menyusun strategi pemasaran. Untuk mendukung analisis penjualan, saya membuat tabel untuk menyimpan fakta penjualan (Gambar 1.2).

<cimg></cimg>
![Entity relationship diagram of fact order](https://storage.googleapis.com/dionricky-blog/ecommerce-data-analysis/fact_order_erd.jpg)
<p class='img-caption'>Gambar 1.2 Fact order ERD</p>

Tabel fakta order ini digunakan pada bagian data analisis dan untuk melatih model _machine learning_ jenis _supervised learning_.

Dimensi yang digunakan adalah tanggal dan jam pesanan, penjual, pengguna yang membeli, produk, lokasi penjual, lokasi pembeli, dan status pesanan. Dimensi-dimensi tersebut sekaligus menentukan _granularity_ pada tabel fakta order ini. Sehingga _granularity_-nya adalah pada level produk dalam setiap transaksi.

<!--
/ fact feedback -->
#### 2. Fact Feedback
Ukuran kepuasan pelanggan dari produk dan layanan yang diberikan pada e-commerce ini adalah ulasan. Pengguna yang kurang puas akan memilih untuk meninggalkan e-commerce ini. Sehingga penting untuk melakukan analisis terhadap ulasan pelanggan. Analisis tersebut ditujukan untuk mengetahui aspek apa yang perlu diperbaiki dari layanan ataupun produk yang ditawarkan. Untuk mendukung analisis kepuasan pelanggan, saya membuat tabel fakta ulasan (Gambar 1.3).

<cimg></cimg>
![Entity relationship diagram of fact feedback](https://storage.googleapis.com/dionricky-blog/ecommerce-data-analysis/fact_feedback_erd.jpg)
<p class='img-caption'>Gambar 1.3 Fact feedback ERD</p>

Bagian data analisis selanjutnya juga menggunakan tabel ini untuk mengetahui pengaruh penggunaan voucher terhadap kepuasan pelanggan.

Dimensi yang digunakan adalah tanggal ulasan dan ID ulasan sehingga memiliki _granularity_ pada level tiap ulasan.

<!--
/ mart customer rfm
langsung join ke dimension oke-->
#### 3. Data Mart Customer RFM
Analisis RFM (_Recency-Frequency-Monetary_) merupakan teknik segmentasi pelanggan yang dapat memberikan informasi untuk membuat keputusan strategis. Pelanggan akan dibagi kedalam kelompok yang homogen berdasarkan tiga ukuran yaitu:
- _Recency_, berapa lama sejak pelanggan membeli atau mengunjungi situs
- _Frequency_, berapa banyak jumlah pembelian atau kunjungan pelanggan
- _Monetary_, berapa banyak uang yang dikeluarkan oleh pelanggan

Tabel data mart customer RFM (Gambar 1.4) dibuat khusus untuk analisis tersebut.

<cimg></cimg>
![Entity relationship diagram of data mart customer RFM](https://storage.googleapis.com/dionricky-blog/ecommerce-data-analysis/marts_erd.jpg)
<p class='img-caption'>Gambar 1.4 Data mart customer RFM ERD</p>

Semua tabel pada data mart customer memiliki dimensi pengguna sehingga level _granularity_-nya adalah pada setiap pengguna.

#### ETL Process
Setelah data warehouse selesai dirancang dan dibuat langkah selanjutnya adalah melakukan proses ETL. ETL (Extract-Transform-Load) adalah tahapan untuk mengisi data kedalam data warehouse. Sesuai namanya, ada tiga tahapan dalam proses ETL yaitu:
1. _Extract_, tahap untuk mengambil data dari sumber seperti database, file, atau API
2. _Transform_, melakukan perubahan struktur untuk menyesuaikan dengan kebutuhan pada data warehouse
3. _Load_, memasukkan data kedalam tabel-tabel dalam data warehouse

Proses ETL biasanya menggunakan bantuan tools _workflow executor_ seperti Talend atau Airflow. Saya menggunakan Airflow karena lebih mudah digunakan dan lebih fleksibel. _Workflow_ pada Airflow dapat dibuat menggunakan bahasa pemrograman Python.

## Data Analysis


### Apakah Terdapat Peningkatan Transaksi pada Akhir Pekan?


### Apa Pengaruh Penggunaan Voucher Terhadap Kepuasan Pelanggan?


## Data Science

### Sales Prediction

### Customer Segmentation

## Summary


