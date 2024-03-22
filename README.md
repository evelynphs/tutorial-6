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
