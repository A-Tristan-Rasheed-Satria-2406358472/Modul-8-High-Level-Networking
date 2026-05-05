# Refleksi Modul 8

## 1. Perbedaan unary, server streaming, dan bi-directional streaming RPC, serta kapan paling cocok dipakai

Menurut saya perbedaan utamanya ada di pola komunikasi antara client dan server. Unary itu paling sederhana karena client kirim satu request lalu server balas satu response. Cocok untuk operasi yang sifatnya langsung dan hasilnya juga sekali jadi, misalnya proses payment, login, validasi data, atau ambil detail satu data tertentu. Di project ini `process_payment` cocok pakai unary karena request-nya cuma butuh `user_id` dan `amount`, lalu hasilnya juga cukup satu status sukses atau gagal.

Server streaming itu client tetap kirim satu request, tapi server bisa balas banyak response secara bertahap. Model ini cocok kalau data yang dikirim banyak atau memang lebih enak dikonsumsi per bagian, misalnya riwayat transaksi, notifikasi log, progress proses panjang, atau monitoring data realtime ringan. Di project ini `get_transaction_history` cocok pakai server streaming karena client cukup kirim satu request user, lalu server mengirim banyak data transaksi satu per satu tanpa harus menunggu semuanya terkumpul dulu.

Bi-directional streaming lebih fleksibel lagi karena client dan server sama-sama bisa saling kirim banyak pesan secara terus menerus dalam satu koneksi. Ini paling cocok untuk skenario yang benar-benar realtime seperti chat, collaborative editing, live dashboard interaktif, atau komunikasi device yang butuh event dua arah. Pada project ini fitur `chat` paling masuk akal pakai bidirectional streaming karena user bisa kirim beberapa pesan dan server bisa membalas setiap pesan tanpa harus membuat request baru terus-menerus.

## 2. Pertimbangan security pada implementasi gRPC di Rust, terutama authentication, authorization, dan encryption

Kalau bicara security, hal pertama yang menurut saya wajib dipikirin itu authentication. Server harus bisa memastikan siapa yang memanggil service, misalnya lewat token JWT, API key, atau mTLS kalau mau lebih kuat di level antar service. Di gRPC Rust, token biasanya bisa diambil dari metadata request, lalu diverifikasi dulu sebelum request diproses lebih jauh.

Setelah authentication, berikutnya authorization. Ini beda dengan auth, karena meskipun user sudah terbukti valid, belum tentu dia boleh akses semua method atau data. Misalnya user A tidak boleh lihat transaction history user B, dan admin bisa punya hak akses yang berbeda. Jadi idealnya ada pengecekan role, permission, atau ownership data di level service sebelum response dikirim.

Soal data encryption, gRPC idealnya dijalankan di atas TLS supaya data yang lewat jaringan tidak gampang disadap atau dimodifikasi. Ini penting apalagi kalau request membawa data sensitif seperti nominal payment, identity user, token, atau isi chat. Selain itu, kita juga perlu hati-hati dengan logging. Jangan sampai log debug malah menyimpan token, isi pesan sensitif, atau detail internal error yang terlalu banyak. Menurut saya secure coding di gRPC bukan cuma soal enkripsi, tapi juga soal membatasi akses, memvalidasi input, dan menjaga data sensitif supaya tidak bocor lewat log atau error message.

## 3. Tantangan saat menangani bidirectional streaming di Rust gRPC, terutama untuk chat

Tantangan paling terasa di bidirectional streaming itu ada di concurrency dan lifecycle koneksi. Karena client dan server bisa saling kirim data terus, kita harus mengatur kapan stream dibuka, kapan ditutup, dan bagaimana menangani kalau salah satu sisi disconnect mendadak. Kalau ini tidak di-handle dengan baik, bisa muncul task yang menggantung, channel yang tidak pernah ditutup, atau resource leak.

Di Rust sendiri ada tambahan tantangan karena harus disiplin dengan ownership, borrowing, dan async task. Saat stream dibaca dan response dikirim secara paralel, kita harus memastikan data yang dipindah ke task async memang aman dan sesuai aturan ownership. Untuk aplikasi chat, tantangan lain adalah menjaga urutan pesan, menangani backpressure kalau pesan datang terlalu cepat, dan memastikan broadcast atau reply tidak bikin bottleneck. Belum lagi kalau mau skenario lebih realistis seperti multiple room, user presence, retry, reconnect, atau penyimpanan history chat.

Menurut saya, bidirectional streaming itu kuat banget untuk realtime communication, tapi implementasinya memang lebih kompleks dibanding unary. Karena itu struktur kode, error handling, dan monitoring harus lebih rapi dari awal.

## 4. Kelebihan dan kekurangan `tokio_stream::wrappers::ReceiverStream` untuk streaming response

Kelebihannya, `ReceiverStream` cukup praktis karena mudah menjembatani `tokio::sync::mpsc` channel menjadi stream yang bisa dipakai gRPC. Jadi kalau server ingin menghasilkan data secara bertahap dari task async lain, pola ini enak dan cukup natural. Kodenya juga relatif sederhana dan mudah dibaca untuk kasus seperti transaction streaming atau balasan chat.

Selain itu, `mpsc` channel memberi pemisahan yang lumayan jelas antara producer dan consumer. Producer cukup fokus mengirim item ke channel, lalu gRPC response tinggal membungkus `Receiver`-nya jadi `ReceiverStream`. Untuk project pembelajaran atau service yang belum terlalu kompleks, ini menurut saya solusi yang cukup bagus.

Kekurangannya, pendekatan ini menambah lapisan buffering yang kalau tidak diatur bisa menimbulkan backpressure atau konsumsi memory yang tidak efisien. Ukuran channel juga perlu dipikirkan, karena terlalu kecil bisa bikin producer sering nunggu, tapi terlalu besar juga bisa membuat pesan numpuk. Selain itu, kalau error handling dan penutupan channel tidak rapi, stream bisa berhenti dengan cara yang membingungkan atau task async tetap jalan walaupun consumer-nya sudah tidak ada. Jadi enak dipakai, tapi tetap butuh disiplin di sisi flow control dan cleanup.

## 5. Cara menyusun kode Rust gRPC supaya reusable, modular, dan mudah di-maintain

Menurut saya, kode gRPC akan lebih enak dirawat kalau dipisah per tanggung jawab. Misalnya definisi proto dan hasil generate tetap terpisah, implementasi service diletakkan di module sendiri, business logic dipisah dari transport layer, dan model/domain object tidak bercampur dengan detail tonic atau transport. Jadi handler gRPC jangan langsung menampung semua logic di dalam satu file besar.

Contohnya, `MyPaymentService`, `MyTransactionService`, dan `MyChatService` bisa dipisah ke module masing-masing seperti `services/payment.rs`, `services/transaction.rs`, dan `services/chat.rs`. Lalu kalau ada logic yang lebih murni bisnis, misalnya validasi payment, penyimpanan transaksi, atau formatting response chat, itu dipindah lagi ke layer service/domain helper. Dengan begitu kalau nanti ada perubahan logic, kita tidak perlu bongkar file transport utama.

Selain itu, penggunaan trait untuk abstraksi juga membantu extensibility. Misalnya payment processor bisa dibuat trait, sehingga implementasi mock, sandbox, atau provider payment sungguhan bisa diganti tanpa banyak mengubah handler gRPC. Pendekatan seperti ini menurut saya bikin project lebih siap berkembang daripada semuanya ditulis langsung di satu implementasi service.

## 6. Hal tambahan yang dibutuhkan agar `MyPaymentService` bisa menangani payment yang lebih kompleks

Implementasi sekarang masih sangat sederhana karena setiap request langsung dibalas `success: true`. Kalau payment logic-nya dibuat lebih realistis, ada beberapa langkah tambahan yang menurut saya perlu. Pertama, validasi input yang lebih lengkap, misalnya amount tidak boleh nol atau negatif, user harus valid, dan format request harus benar. Kedua, perlu integrasi ke sumber data atau payment gateway sungguhan, jadi server tidak hanya memberi response dummy.

Ketiga, perlu status yang lebih kaya. Dalam dunia nyata, payment tidak selalu cuma sukses atau gagal, tapi bisa pending, rejected, timeout, duplicate, atau butuh retry. Jadi response idealnya memuat transaction id, status detail, error code, dan mungkin timestamp. Keempat, perlu idempotency supaya request yang terkirim ulang tidak menyebabkan pembayaran ganda. Ini penting kalau ada retry dari client atau gangguan jaringan.

Selain itu, logging, auditing, observability, dan error handling juga harus lebih serius. Payment termasuk area sensitif, jadi setiap proses penting biasanya perlu jejak audit, monitoring, dan pengamanan terhadap fraud atau request yang mencurigakan. Jadi menurut saya, `MyPaymentService` saat ini cocok untuk demo konsep unary RPC, tapi kalau untuk kasus production masih butuh banyak lapisan tambahan.

## 7. Dampak penggunaan gRPC terhadap arsitektur distributed system, terutama interoperabilitas

Adopsi gRPC punya dampak yang cukup besar ke desain arsitektur karena komunikasi antar service jadi lebih contract-first lewat `.proto`. Ini bagus karena kontrak API jadi lebih jelas, typed, dan konsisten lintas service. Dalam sistem terdistribusi, pendekatan ini membantu sinkronisasi antar tim karena struktur request dan response sudah didefinisikan dengan tegas dari awal.

Di sisi performa, gRPC juga mendorong arsitektur yang lebih efisien untuk komunikasi internal antar service karena Protocol Buffers lebih ringkas daripada JSON dan HTTP/2 mendukung multiplexing. Ini cocok untuk microservices yang butuh throughput tinggi atau komunikasi sering. Tapi di sisi interoperabilitas, kadang ada trade-off. Walaupun gRPC mendukung banyak bahasa, integrasi dengan browser, tool REST biasa, atau pihak ketiga kadang tidak sesederhana HTTP JSON yang lebih universal. Jadi gRPC sering lebih nyaman untuk service-to-service internal dibanding API publik yang langsung dikonsumsi banyak klien umum.

Menurut saya, penggunaan gRPC membuat sistem lebih terstruktur dan efisien, tapi juga menuntut disiplin lebih tinggi dalam versioning schema, tooling, dan compatibility antar platform.

## 8. Kelebihan dan kekurangan HTTP/2 untuk gRPC dibanding HTTP/1.1 atau HTTP/1.1 + WebSocket pada REST API

Kelebihan HTTP/2 yang paling terasa adalah multiplexing, header compression, dan dukungan streaming yang lebih natural untuk gRPC. Banyak request bisa berjalan bersamaan dalam satu koneksi tanpa pola blocking seperti di HTTP/1.1. Buat service yang ramai atau butuh realtime-ish communication, ini cukup membantu performa dan efisiensi koneksi.

Kalau dibanding HTTP/1.1 REST biasa, gRPC di atas HTTP/2 biasanya lebih efisien untuk payload dan latency, terutama pada komunikasi internal antar service. Kalau dibanding HTTP/1.1 + WebSocket, gRPC punya model RPC dan schema yang lebih terstruktur, jadi developer experience untuk service contract bisa lebih rapi. Tapi kekurangannya, HTTP/2 dan gRPC tooling kadang lebih kompleks, debugging manual tidak sesederhana REST JSON, dan dukungan langsung di browser juga lebih terbatas tanpa grpc-web atau proxy tambahan.

Jadi menurut saya HTTP/2 memberi fondasi teknis yang lebih kuat untuk komunikasi modern, tapi tidak selalu jadi pilihan paling praktis kalau kebutuhan utamanya cuma CRUD sederhana yang ingin gampang dites dan gampang dikonsumsi dari banyak platform umum.

## 9. Perbedaan model request-response REST dengan bidirectional streaming gRPC dari sisi realtime communication dan responsiveness

REST umumnya memakai pola request-response yang lebih diskrit. Client kirim request, server balas, habis itu selesai. Kalau butuh update terus-menerus, client biasanya harus polling atau memakai mekanisme tambahan seperti WebSocket atau SSE. Artinya untuk komunikasi realtime, REST murni cenderung kurang responsif dan lebih boros request kalau update-nya sering.

Sebaliknya, bidirectional streaming gRPC memungkinkan koneksi tetap hidup dan kedua sisi bisa kirim pesan kapan saja dalam alur yang sama. Ini membuat interaksi terasa lebih realtime dan latencynya lebih rendah untuk skenario seperti chat, live event, atau sinkronisasi data dua arah. Server tidak perlu menunggu request baru untuk mengirim update, dan client juga tidak harus buka koneksi baru setiap kali mengirim pesan.

Menurut saya, untuk operasi bisnis biasa REST masih sangat cocok karena sederhana dan mudah diintegrasikan. Tapi untuk komunikasi yang butuh respons cepat dan dua arah, bidirectional streaming gRPC jauh lebih natural.

## 10. Implikasi pendekatan schema-based gRPC dengan Protocol Buffers dibanding JSON yang lebih fleksibel pada REST

Pendekatan schema-based pada gRPC membuat kontrak data jadi eksplisit. Ini keuntungan besar karena type safety lebih kuat, validasi struktur lebih jelas, dan code generation membantu mengurangi banyak error manual. Protocol Buffers juga lebih hemat ukuran payload dibanding JSON, jadi efisien untuk jaringan dan komunikasi antar service yang intensif.

Tapi konsekuensinya, perubahan schema harus dikelola dengan lebih hati-hati. Kita perlu memikirkan compatibility, field numbering, deprecation, dan versioning agar perubahan tidak merusak client lama. Fleksibilitasnya memang lebih rendah dibanding JSON yang bisa lebih cepat diubah atau ditambah field secara ad-hoc.

JSON di REST lebih enak untuk debugging manual, lebih human-readable, dan lebih gampang dipakai oleh banyak tool umum. Tapi karena sifatnya lebih longgar, ada risiko kontrak data jadi kurang disiplin dan inkonsistensi lebih mudah muncul kalau dokumentasi atau validasinya lemah. Jadi menurut saya, gRPC dengan protobuf unggul untuk sistem yang butuh kontrak kuat dan performa, sedangkan JSON REST lebih unggul untuk fleksibilitas dan kemudahan integrasi luas.
