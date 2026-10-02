## Lingkungan yang Digunakan

| Komponen | Versi / Keterangan |
|---|---|
| Sistem operasi | Windows 11 |
| Python | 3.14.3 |
| Flask | 3.1.3 |
| Editor | Visual Studio Code |
| Alat uji | `curl.exe` pada Windows PowerShell |

Versi lain kemungkinan besar tetap bisa dipakai, tetapi hasil di atas diuji pada versi tersebut.

## Langkah 0: Persiapan

1. Pastikan Python sudah terpasang:

   ```
   python --version
   ```

2. Pasang library yang dibutuhkan:

   ```
   pip install flask requests
   ```

3. Siapkan sebuah folder kerja, lalu buat tiga file berikut di dalamnya, atau salin dari repositori ini.


---

## Bagian 1: Aplikasi Monolith

Seluruh fitur (buku dan pesanan) digabung dalam satu file dan berjalan pada **port 5000**.

### Langkah 1: Buat `monolith_app.py`

```python
from flask import Flask, jsonify, request
app = Flask(__name__)

# Database bohongan (In-memory)
books = [{"id": 1, "title": "Belajar Flask", "stock": 5}]
orders = []

# --- FITUR BUKU ---
@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

# --- FITUR PESANAN ---
@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')
    # Logika bisnis: Cek stok buku langsung dari variabel global
    for b in books:
        if b['id'] == book_id and b['stock'] > 0:
            b['stock'] -= 1
            order = {"id": len(orders)+1, "book_id": book_id, "status": "berhasil"}
            orders.append(order)
            return jsonify(order), 201
    return jsonify({"error": "Buku tidak ditemukan atau stok habis"}), 400

if __name__ == '__main__':
    # Aplikasi berjalan di port 5000
    app.run(port=5000, debug=True)
```

### Langkah 2: Jalankan aplikasi

Di **terminal 1**:

```
python monolith_app.py
```

### Langkah 3: Uji aplikasi

Buka **terminal 2** (PowerShell), lalu jalankan tiga perintah berikut secara berurutan.

```powershell
# 1. Lihat daftar buku (stok awal 5)
curl.exe http://localhost:5000/books

# 2. Buat pesanan
curl.exe -X POST -H "Content-Type: application/json" -d '{\"book_id\":1}' http://localhost:5000/orders

# 3. Lihat daftar buku lagi
curl.exe http://localhost:5000/books
```

**Hasil yang diharapkan:**

- Perintah 1 menampilkan buku dengan `"stock": 5`.
- Perintah 2 menampilkan `{"book_id": 1, "id": 1, "status": "berhasil"}`.
- Perintah 3 menampilkan `"stock": 4`. **Stok berkurang**, karena fitur pesanan mengakses variabel `books` yang sama dengan fitur buku.

### Langkah 4: Hentikan server

Tekan `Ctrl+C` pada terminal 1 sebelum lanjut ke Bagian 2, agar port 5000 bebas.

---

## Bagian 2: Microservices

Fitur dipecah menjadi dua layanan terpisah. **Order Service tidak bisa membaca variabel `books` secara langsung**, sehingga harus menghubungi Book Service lewat HTTP.

### Langkah 1: Buat `book_service.py` (port 5001)

```python
from flask import Flask, jsonify
app = Flask(__name__)

# Database khusus Book Service
books = [{"id": 1, "title": "Belajar Flask", "stock": 5}]

@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

@app.route('/books/<int:book_id>', methods=['GET'])
def get_book(book_id):
    for b in books:
        if b['id'] == book_id:
            return jsonify(b)
    return jsonify({"error": "Not found"}), 404

if __name__ == '__main__':
    # Book service berjalan di port 5001
    app.run(port=5001, debug=True)
```

### Langkah 2: Buat `order_service.py` (port 5002)

```python
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)
orders = []
BOOK_SERVICE_URL = "http://localhost:5001"  # Alamat Book Service

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')
    # Komunikasi antar service: Tanya Book Service apakah buku ada
    try:
        response = requests.get(f"{BOOK_SERVICE_URL}/books/{book_id}")
        if response.status_code == 200:
            book_data = response.json()
            if book_data['stock'] > 0:
                order = {"id": len(orders)+1, "book_id": book_id, "status": "berhasil"}
                orders.append(order)
                return jsonify(order), 201
        return jsonify({"error": "Buku tidak tersedia"}), 400
    except requests.exceptions.ConnectionError:
        return jsonify({"error": "Book Service sedang down!"}), 500

if __name__ == '__main__':
    # Order service berjalan di port 5002
    app.run(port=5002, debug=True)
```

### Langkah 3: Jalankan kedua layanan

Di **terminal 1**:

```
python book_service.py
```

Di **terminal 2**:

```
python order_service.py
```

### Langkah 4: Periksa kondisi awal Book Service

Buka **terminal 3** (PowerShell):

```powershell
curl.exe http://localhost:5001/books
curl.exe http://localhost:5001/books/1
```

**Hasil yang diharapkan:** keduanya menampilkan buku dengan id 1 dan `"stock": 5`.

### Langkah 5: Buat pesanan lewat Order Service

Jalankan perintah ini **dua kali**:

```powershell
curl.exe -X POST -H "Content-Type: application/json" -d '{\"book_id\":1}' http://localhost:5002/orders
```

**Hasil yang diharapkan:** pesanan pertama berisi `"id": 1` dan pesanan kedua `"id": 2`, keduanya dengan `"status": "berhasil"`.

Lalu periksa stok di Book Service:

```powershell
curl.exe http://localhost:5001/books
```

Stok **tetap 5**. Pada kode ini Order Service hanya *membaca* data buku dan tidak pernah mengubah stok. Ini berbeda dengan Monolith, dan menunjukkan bahwa perubahan data lintas layanan harus dirancang secara eksplisit lewat API.

---

## Bagian 3: Eksperimen Kegagalan (Fault Isolation)

1. Pada terminal tempat Book Service berjalan, tekan `Ctrl+C` untuk mematikannya. Biarkan Order Service tetap hidup.
2. Kirim pesanan lagi ke Order Service:

   ```powershell
   curl.exe -X POST -H "Content-Type: application/json" -d '{\"book_id\":1}' http://localhost:5002/orders
   ```

**Hasil yang diharapkan:**

```json
{
  "error": "Book Service sedang down!"
}
```

Order Service **tidak ikut mati**. Ia menangkap `ConnectionError` dan mengembalikan pesan kesalahan yang terkendali. Inilah yang disebut **Fault Isolation**.

---

## Pertanyaan Diskusi

1. Di kode Monolith, jika fitur Pesanan mengalami crash/bug yang fatal, apa yang terjadi pada fitur Buku? Bandingkan dengan versi Microservices.
2. Pada versi Microservices, proses pengecekan stok buku menjadi sedikit lebih lambat karena membutuhkan HTTP Request. Apakah ada solusi untuk mengatasi masalah latensi jaringan ini di industri nyata?
3. Jika saat peluncuran toko buku ini traffic pencarian buku melonjak drastis sedangkan traffic pesanan biasa saja, layanan mana (dan port mana) yang akan Anda scale-up atau perbanyak servernya?

Jawaban lengkap ada pada laporan praktikum.

---

## Pemecahan Masalah

| Masalah | Penyebab dan solusi |
|---|---|
| `curl` mengeluarkan error aneh di PowerShell | Di PowerShell, `curl` adalah alias dari `Invoke-WebRequest`. Gunakan **`curl.exe`**. |
| `Address already in use` / port sudah terpakai | Server sebelumnya masih berjalan. Hentikan dengan `Ctrl+C`, atau tutup terminal lama. |
| `ModuleNotFoundError: No module named 'flask'` | Jalankan `pip install flask requests`. |
| `ModuleNotFoundError: No module named 'requests'` | Jalankan `pip install flask requests`. |
| Order Service selalu menjawab "Book Service sedang down!" | Pastikan `book_service.py` sedang berjalan di port 5001. |
| Perintah `curl` dengan JSON gagal di Linux atau macOS | Tulis JSON tanpa backslash: `-d '{"book_id":1}'`, dan pakai `curl` biasa. |
| Perintah `python` tidak dikenali | Coba `python3`, atau pastikan Python sudah masuk ke PATH. |

## Catatan

- Data disimpan di memori (variabel Python), jadi akan kembali ke kondisi awal setiap server dijalankan ulang.
- Praktikum ini tidak mengukur latensi dan tidak menguji penskalaan. Pembahasan keduanya pada laporan hanya bersifat konsep.
