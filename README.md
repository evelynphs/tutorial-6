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