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
1. konfigurasi _source database_, dan
2. pembangunan _data warehouse_

### Source Database Logical Design
File csv tidak cocok untuk kebutuhan analisis dan akses data secara cepat dan kompleks. Oleh karena itu pada tahap pertama dalam _data engineer_ ini saya akan membuat basis data untuk menyimpan data untuk kebutuhan pemrosesan. Keunggulan dari penggunaan basis data adalah kemudahan pengaksesan data, manajemen akses, dan kemampuan penyajian data dengan transformasi kompleks menggunakan SQL.

Sistem manajemen basis data yang saya gunakan adalah PostgreSQL versi 13.

<cimg></cimg>
![Source Database Logical Design (ERD)](https://storage.googleapis.com/dionricky-blog/erd_db_updated.png)
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
<!-- what is dimension -->
<!-- what is fact -->

<!--
/ fact order
/ fact feedback
/ mart customer rfm
langsung join ke dimension oke-->

#### ETL Process


## Data Analysis
<!--
/ tampilkan BQ yg related ke next part
/ bq2 dan bq4 -->

## Data Science
<!-- / tampilkan semwa -->


## Summary


