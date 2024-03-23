# Tutorial 6

### Commit 1 Reflection notes
#### Apa yang dilakukan oleh method `handle_connection`?

``` rust
fn handle_connection(mut stream: TcpStream)
```
Method `handle_connection` menerima parameter `stream` dengan tipe `TcpStream` yang di-pass oleh method `main`.

<br>

```rust
let buf_reader = BufReader::new(&mut stream);
```
Membuat instance `BufReader` baru yang menerima _mutable reference_ dari variabel `stream`. Tujuannya adalah untuk menambahkan _buffering_ ketika sedang membaca `stream`.

<br>

```rust
let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();
```
Sebuah variabel `http_request` dibuat untuk mengumpulkan/menyimpan _lines_ dari _request_ yang dikirimkan oleh browser ke server. _Request_ tersebut dibaca menggunakan `buf_reader` dan kemudian dikumpulkan dalam bentuk vector menggunakan `Vec<_>`. Dengan menggunakan `.lines()`, _request_ akan dipecah/dipisah per baris dan menghasilkan suatu `Result` yang mengandung string. Kemudian, `.map(|result| result.unwrap())` digunakan untuk meng-_unwrap_ masing-masing dari `Result` tersebut dan memperoleh string nya. `.take_while(|line| !line.is_empty())` akan "memberi tahu" program untuk berhenti mengambil _lines_ ketika sudah menemukan suatu line kosong. Terakhir, `.collect()` digunakan untuk mengumpulkan semua string yang telah diperoleh tadi ke dalam vector.

<br>

```rust
println!("Request: {:#?}", http_request);
```
Hasil dari kumpulan string dalam vector `http_request` tadi di-print.

---

### Commit 2 Reflection notes
#### Return HTML

Pada commit kali ini, kita menambahkan `fs` pada `use` untuk mengimport library filesystem yang nantinya digunakan untuk membaca file `hello.html`. Kemudian, kita memodifikasi method `handle_connection` agar dapat memunculkan response berupa halaman HTML.

<br>

```rust
    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();
```
Ketiga variabel ini ditambahkan untuk nantinya di-include pada response. Variabel `status_line` berisi versi HTTP `HTTP/1.1`, status code `200`, dan reason phrase `OK`. Variabel `contents` berisi konten `hello.html` yang dibaca menggunakan `fs` dan kemudian di-unwrap menjadi string. Terakhir, variabel `length` berisi length dari variabel `contents`.

<br>

```rust
    let response = format!("{status_line}\r\nContent-length: {length}\r\n\r\n{contents}");
```
Response kemudian disusun menggunakan `format!` untuk meng-include `contents` sebagai isi dari _response body_. Pada response ini, content-length ditambahkan pada header untuk mengatur length dari _response body_ yang diinginkan. Karena kita ingin seluruh isi dari `hello.html` untuk diikutkan pada _response body_, maka isi dari content-length adalah variable `contents` yang sudah menyesuaikan value nya dengan length dari `hello.html`.

<br>

```rust
    stream.write_all(response.as_bytes()).unwrap();
```
String response akan di-convert menjadi _byte_ menggunakan `as_bytes()` dan kemudian dikirimkan melalui stream oleh `write_all` sebagai response ke client.

<br>

Ketika program di-run dan `http://127.0.0.1:7878/` dibuka pada browser, akan muncul tampilan berikut.
![Commit 2 screen capture](/assets/images/commit2.png)

---

### Commit 3 Reflection notes
#### Validating request and selectively responding

Pada commit sebelumnya, program mengabaikan isi request pada `http_request` dan selalu me-return `hello.html` pada kondisi apapun. Kali ini, kita menerapkan request validation dan selective responding agar program me-return response yang sesuai dengan isi request. Program dibuat agar me-return `hello.html` ketika mendapat request path `/` dan me-return error page ketika mendapat request lainnya. Method `handle_connection` dimodifikasi sebagai berikut.

<br>

```rust
let request_line = buf_reader.lines().next().unwrap().unwrap();
```
Variabel `http_response` diganti dengan variabel `request_line` yang berisi baris pertama dari hasil pembacaan `buf_reader`. Hal ini dilakukan karena kita hanya memerlukan baris pertama dari request untuk validasi. `buf_reader.lines().next().unwrap().unwrap()` digunakan untuk meng-unwrap item pertama dari iterator, kemudian meng-unwrap lagi hasil iterasi tersebut menjadi string.

<br>

```rust
if request_line == "GET / HTTP/1.1" {
    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```
Penyusunan dan pengiriman response dipindahkan ke dalam suatu blok `if`. Blok tersebut memeriksa apakah `request_line` berisi request GET dengan path `/`. Penyusunan dan pengiriman response `hello.html` hanya akan dijalankan jika `reuest_line` nya sesuai.

<br>

```rust
else {
    let status_line = "HTTP/1.1 404 NOT FOUND";
    let contents = fs::read_to_string("404.html").unwrap();
    let length = contents.len();

    let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```
Ditambahkan suatu blok `else` yang akan meng-handle response jika request yang diterima BUKAN `GET /`. Pada blok ini, `status_line` akan mengirimkan status code `400` dan reason phrase `NOT FOUND`, sedangkan `content` akan berisi hasil pembacaan `404.html`. Response ini menandakan bahwa terjadi error karena penerimaan request yang dianggap _unknown_.

<br>

Kini, program sudah bisa memvalidasi request dan mengirimkan response yang berbeda sesuai dengan request yang didapat. Tetapi, kode yang dibuat masih memiliki kekurangan. Masih terdapat beberapa duplikasi kode pada bagian blok `if` dan `else`. Isi dari kedua blok tersebut hampir sama persis, perbedaan hanya terletak pada `status_line` dan `contents`. Perlu dilakukan refactoring agar penulisan kode menjadi lebih ringkas. Refactoring dilakukan dengan memodifikasi `handle_connection` menjadi sebagai berikut.

<br>

```rust
fn handle_connection(mut stream: TcpStream){
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    }
    else{
        ("HTTP/1.1 400 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();
    let response = format!("{status_line}\r\nContent-length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```
Pada kode di atas, blok `if` dan `else` hanya bertugas menentukan isi dari variabel `status_line` dan `filename` sesuai dengan `request_line` yang didapat. Penyusunan dan pengiriman response kini dilakukan di luar scope blok `if` dan `else` tersebut. Variabel `content` kini bergantung pada variabel `filename` yang sudah dideklarasikan pada validasi request. Dengan begitu, tidak ada duplikasi kode sehingga penulisan menjadi lebih ringkas.

<br>

Berikut tampilan browser ketika program di-run dan path request invalid, misalnya `http://127.0.0.1:7878/bad`
![Commit 3 screen capture](/assets/images/commit3.png)

---

### Commit 4 Reflection notes
#### Simulating slow request

Saat ini, server yang telah dibuat bersifat _single-threaded_. Jika server menerima lebih dari satu request, masing-masing request tersebut akan diproses secara berurutan. Suatu request hanya dapat diproses setelah request sebelumnya selesai diproses. Jika terdapat request yang perlu waktu lama untuk diproses, maka request berikutnya juga harus menunggu lama untuk diproses.

<br>

Kita akan mensimulasikan dampak dari adanya request yang _slow_. Bagian pengisian variabel `status_line` dan `filename` pada method `handle_connection` diubah menjadi sebagai berikut.

<br>

```rust
    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };
```
Validasi request kini menggunakan `match` untuk mencocokkan _pattern_ dari `request_line`. Kita menambahkan satu kasus validasi request, yaitu untuk path `/sleep`. Menggunakan `thread::sleep(Duration::from_secs(5));`, thread akan dijeda selama 5 detik sebelum me-set variabel `status_line` dan `filename` sehingga pengiriman response untuk path `/sleep` juga akan terjeda 5 detik.

<br>

Simulasi dilakukan dengan membuka dua tab, satunya untuk path `/` dan yang satunya lagi untuk path `/sleep`. Ketika browser hanya me-load tab dengan path `/`, tab tersebut dapat di-load dengan sangat cepat. Tetapi, ketika me-load tab dengan path `/sleep`, browser membutuhkan waktu 5 detik sebelum memunculkan `hello.html`. Ketika tab `/sleep` di-load terlebih dahulu, kemudian tab `/` ikut di-load sebelum tab `/sleep` menyelesaikan load nya, tab `/` harus menunggu sampai tab `/sleep` menyelesaikan load nya yang berdurasi 5 detik. Akibat tab `/sleep` yang lebih _slow_ dibanding tab `/`, loading untuk tab `/` ikut terpengaruh dan terjeda. Simulasi ini menunjukkan bahwa pemrosesan pada _single-threaded server_ kurang efisien dan akan sangat memakan waktu ketika diakses oleh banyak client.

---

### Commit 5 Reflection notes
#### Creating a multithreaded server

Multithreaded server pada program ini diimplementasikan menggunakan ThreadPool, yaitu semacam "wadah" atau "kolam" yang berisi lebih dari satu thread dengan jumlah tertentu. Threads tersebut akan mengambil masing-masing satu "tugas" ketika sedang "menganggur". Setiap kali suatu thread selesai mengerjakan suatu tugas, thread tersebut akan kembali "menganggur" dan siap menerima tugas berikutnya. Multithreaded server ini dapat dimanfaatkan untuk meng-handle beberapa request dalam satu waktu yang sama.

<br>

Dalam satu ThreadPool, terdapat beberapa `worker` dengan jumlah tertentu. Para `worker` inilah yang nantinya akan menerima tugas dan mengeksekusinya dalam suatu thread. Agar `ThreadPool` dapat berkomunikasi dengan `worker` untuk mengirimkan tugas, dibuatlah `channel` yang memiliki `sender` dan `receiver`. Selain itu, dibuat juga suatu _struct_ bernama `Job` yang digunakan untuk menyimpan data dari tugas yang akan dieksekusi. `ThreadPool` akan memiliki `sender` yang berisi "antrean" dari `Job`, dan `worker` memiliki `receiver` yang akan menerima kiriman `Job` tersebut.

<br>

Para `worker` menggunakan `Arc` untuk dapat "berbagi" receiver, dan masing-masing dari mereka menggunakan `mutex` untuk membatasi akses ke receiver ketika pengambilan `Job`. Setiap `worker` akan melakukan looping terhadap `receiver` untuk mencari `Job`. Ketika suatu `worker` mengambil suatu `Job`, worker tersebut akan menjalankan suatu thread, di mana thread tersebut akan mengunci receiver dengan menggunakan `lock()` untuk menerapkan _mutual exclusion_. Kemudian, `thread` tersebut akan mengambil `Job` menggunakan `recv()` dari channel untuk dieksekusi. Setelah itu, barulah receiver akan ter-unlock dan worker lain dapat kembali mengakses receiver untuk mencari `Job`.