# reflection

## 1. key differences between unary, server streaming, and bidirectional streaming rpc

unary server request adalah tipe request paling simpel. client mengirim satu request dan server mengembalikan satu response. contoh unary server request adalah payment service, login-logout request atau retrieve satu item dari db.

server streaming rpc adalah tipe request dimana client mengirim satu request dan server mengembalikan beberapa response dalam selang waktu. tipe ini cocok untuk continous updates atau data yang ukurannya lumayan besar. contohnya seperti transaction history, harga stock, logs, atau feed notifikasi.

bidirectional streaming rpc memberikan server dan client kemampuan untuk saling mengirim message ke satu sama lain secara keberlanjutan. ini cocok untul real-time communication seperti aplikasi chat, live collaboration, game live service, atau sistem monitoring yang harus update terus menerus.

## 2. security considerations dalam implementasi grpc service di rust

dalam implementasi grpc service, hal penting yang perlu diperhatikan adalah authentication, authorization, dan encription. authentication dipakai untuk memastikan siapa user atau service yang mengirim request. authorization dipakai untuk memastikan user tersebut memang boleh melakukan action tertentu.

misalnya, user biasa seharusnya tidak bisa melihat transaction history milik user lain. kalau authorization tidak dicek di server, client bisa saja mengganti user_id di request dan mencoba mengakses data lain.

encryption penting karena grpc berjalan melalui network. untuk production, service harus memakai tls agar data yang dikirim tidak mudah dibaca oleh pihak lain. ini penting ketika data yang dikirim berisi informasi sensitif seperti payment, user profile, atau transaction data.

input dari client juga harus divalidasi. server tidak boleh langsung percaya pada request. contoh: amount payment tidak boleh negatif, user_id harus dicek, dan request yang tidak valid sebaiknya dikembalikan sebagai error.

## 3. potential challenges dalam bidirectional streaming

bidirectional streaming lebih complicated dibanding unary atau server streaming karena client dan server sama-sama bisa mengirim message secara terus menerus. dalam aplikasi live service, masalah yang bisa muncul misalnya client disconnect tiba-tiba, server gagal mengirim response, atau message masuk terlalu cepat.

masalah lain adalah pengelolaan stream. kalau stream sudah tidak digunakan tapi task async masih berjalan, service bisa membuang resource. ini bisa menjadi masalah kalau jumlah client cukup banyak.

concurrency juga perlu diperhatikan. karena beberapa message bisa diproses dalam waktu yang berdekatan, server harus menghindari blocking operation. kalau ada operasi yang terlalu lama, stream lain bisa ikut terganggu. 

ketiga masalah ini bisa menimbulkan package loss yang berbahaya terutama untuk aplikasi live service yang sering melakukan micro-update. contohnya game multiplayer.

## 4. advantages and disadvantages dari receiverstream

receiverstream berguna karena dapat mengubah receiver dari tokio channel menjadi stream yang bisa dipakai oleh tonic. dengan cara ini, server bisa membuat response secara bertahap lalu mengirimkannya ke client.

pada contoh transaction service, server membuat beberapa transaction record dan mengirimnya satu per satu. ini lebih sesuai untuk data yang jumlahnya banyak dibanding mengumpulkan semuanya dulu lalu baru dikirim dalam satu response.

kekurangannya adalah penggunaan channel menambah hal yang harus diatur. buffer size harus dipilih dengan cukup hati-hati. kalau terlalu kecil, proses pengiriman bisa tertahan. kalau terlalu besar, penggunaan memory bisa naik . server juga harus menangani kondisi ketika client sudah disconnect dan channel tidak bisa dikirim lagi.

## 5. cara struktur code agar lebih reusable dan modular

code grpc bisa dibuat lebih rapi dengan memisahkan setiap service ke file yang berbeda. paymentservice bisa ada di file payment_service.rs, transactionservice di transaction_service.rs, dan chatservice di chat_service.rs. dengan begitu grpc_server.rs tidak terlalu panjang.

business logic juga sebaiknya tidak dicampur terlalu banyak ke handler grpc. handler cukup menerima request, memanggil function yang sesuai, lalu mengembalikan response. misalnya, validasi payment bisa dibuat di function atau module payment sendiri.

struktur seperti ini membuat code lebih mudah dites. kalau ada perubahan pada payment, bagian chat tidak perlu ikut terganggu. ini juga membantu kalau nanti service ditambah atau logic dibuat lebih kompleks.

## 6. additional steps untuk payment processing yang lebih kompleks

pada tutorial ini, payment service langsung mengembalikan success true. pada sistem yang lebih realistis, langkahnya lebih banyak. server harus validasi user, validasi amount, mengecek metode pembayaran, lalu mencatat transaksi.

service juga harus menangani kondisi gagal. contohnya payment gateway tidak merespons, amount tidak valid, atau user tidak punya izin. response dari server menjelaskan status payment dengan cukup jelas tanpa reveal business logic aplikasi.

payment processing juga harus memakai idempotency. ini berguna agar request yang sama tidak diproses dua kali karena retry dari client. mencegah kemungkinan transaksi yang sama tercatat atau terbayar lebih dari sekali.

## 7. impact grpc terhadap architecture distributed system

grpc membuat komunikasi antar service menjadi lebih terstruktur karena service dan message sudah didefinisikan dalam file proto. client dan server jadi memiliki contract yang jelas tentang format request dan response.

dalam distributed system, ini berguna karena service bisa dibuat dengan bahasa pemrograman yang berbeda. selama mengikuti proto definition yang sama, service rust bisa berkomunikasi dengan service go, java, python, atau bahasa lain.

tapi grpc juga tidak selalu paling praktis untuk semua kasus. untuk public api sederhana, rest dengan json kadang lebih mudah digunakan dan dites. grpc lebih cocok untuk komunikasi internal antar service yang butuh struktur dan performa.

## 8. advantages and disadvantages http/2 dibanding http/1.1 atau websocket

grpc menggunakan http/2. http/2 mendukung multiplexing dan streaming, sehingga beberapa komunikasi bisa berjalan dalam satu connection. 

dibanding rest biasa di http/1.1, grpc dengan protobuf bisa mengirim data dengan ukuran yang lebih kecil. karena datanya binary dan schema-nya jelas, proses komunikasi bisa lebih cepat dalam banyak kasus.

kekurangannya, grpc tidak senyaman rest untuk testing manual. rest bisa langsung dicoba dengan browser, postman, atau curl. websocket juga lebih umum dipakai untuk real-time communication di browser. jadi pilihan protokol tetap tergantung kebutuhan sistemnya.

## 9. rest request-response dibanding bidirectional streaming grpc

rest api biasanya memakai model request-response. client mengirim request, lalu server mengembalikan satu response. model ini cocok untuk operasi seperti mengambil data, membuat data baru, update data, atau delete data.

untuk komunikasi real-time, rest kurang ideal karena client biasanya harus melakukan polling. artinya client mengirim request berulang kali untuk mengecek apakah ada data baru. cara ini bisa bekerja, tapi memakan resource.

bidirectional streaming grpc lebih cocok untuk komunikasi real-time. client dan server bisa saling mengirim message dalam satu connection yang sama. ini membuat sistem seperti chat atau live monitoring menjadi lebih responsif.

## 10. schema-based protocol buffers dibanding json yang schema-less

protocol buffers menggunakan pendekatan schema-based. struktur data harus didefinisikan dulu dalam file proto. dengan begitu, field dan tipe data yang digunakan menjadi lebih jelas.

keuntungannya adalah komunikasi antar service menjadi lebih konsisten. kesalahan tipe data bisa lebih mudah dihindari karena format message sudah ditentukan. protobuf juga biasanya lebih compact dibanding json.

json lebih fleksibel karena tidak harus selalu memakai schema. ini membuat json lebih mudah digunakan untuk api sederhana. tetapi fleksibilitas ini juga bisa membuat client dan server memiliki ekspektasi data yang berbeda. karena itu, protobuf lebih cocok untuk sistem internal yang butuh contract jelas, sedangkan json lebih praktis untuk api yang ingin mudah dibaca dan dicoba.