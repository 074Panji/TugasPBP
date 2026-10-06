# Tugas Mandiri 5: SQL Mentah vs ORM

Operasi yang dipilih: **mengambil satu data berdasarkan ID** dari tabel `jadwal`.

---

## Struktur Tabel Contoh

```sql
CREATE TABLE jadwal (
  id INT PRIMARY KEY AUTO_INCREMENT,
  mata_kuliah VARCHAR(100) NOT NULL,
  hari VARCHAR(20) NOT NULL,
  jam_mulai TIME NOT NULL
);

INSERT INTO jadwal (mata_kuliah, hari, jam_mulai)
VALUES ('Basis Data', 'Senin', '08:00:00');
```

---

## A. SQL Mentah

**Query SQL:**

```sql
SELECT * FROM jadwal WHERE id = ?;
```

**Memakai Node.js dengan `mysql2`:**

```javascript
const mysql = require('mysql2/promise');

async function getJadwalById(id) {
  const connection = await mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: '',
    database: 'kampus'
  });

  const [rows] = await connection.execute(
    'SELECT * FROM jadwal WHERE id = ?',
    [id]
  );

  await connection.end();
  return rows[0]; // undefined jika tidak ditemukan
}

getJadwalById(1).then(console.log);
```

Tanda `?` adalah placeholder, dan nilai `id` dikirim terpisah lewat array `[id]`.

---

## B. ORM (Prisma)

**Definisi model di `schema.prisma`:**

```prisma
model jadwal {
  id          Int      @id @default(autoincrement())
  mata_kuliah String   @db.VarChar(100)
  hari        String   @db.VarChar(20)
  jam_mulai   DateTime @db.Time
}
```

**Kode Node.js:**

```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function getJadwalById(id) {
  const jadwal = await prisma.jadwal.findUnique({
    where: { id: id }
  });
  return jadwal; // null jika tidak ditemukan
}

getJadwalById(1).then(console.log);
```

Kedua pendekatan menghasilkan data yang sama, yaitu satu baris jadwal dengan `id = 1`.

---

## Perbandingan Singkat

| Aspek | SQL Mentah (`mysql2`) | ORM (Prisma) |
|---|---|---|
| Cara menulis | String SQL | Method JavaScript |
| Jika data tidak ada | `undefined` | `null` |
| Pengecekan kesalahan | Saat dijalankan | Sebagian saat menulis kode (autocomplete dan tipe) |
| Kontrol atas query | Penuh | Dibatasi oleh fitur ORM |

---

## Jawaban Pertanyaan

### 1. Apa perbedaan SQL mentah dan ORM?

SQL mentah berarti programmer menulis perintah SQL sendiri sebagai teks lalu mengirimkannya ke database. ORM (*Object-Relational Mapping*) adalah library yang memetakan tabel database menjadi objek atau model di kode program, sehingga programmer mengakses data lewat method seperti `findUnique()` tanpa menulis SQL secara langsung. Di balik layar, ORM tetap mengubah method tersebut menjadi SQL.

### 2. Apa kelebihan SQL mentah?

Programmer punya kontrol penuh atas query, sehingga mudah mengoptimalkan query yang kompleks seperti join berlapis, subquery, atau fungsi khusus database tertentu. Performanya lebih mudah diprediksi karena tidak ada lapisan tambahan, dan tidak ada library besar yang perlu dipelajari. Selain itu, kemampuan SQL bisa dipakai di bahasa pemrograman apa pun.

### 3. Apa kelebihan ORM?

Kode lebih ringkas dan mudah dibaca karena memakai bahasa pemrograman yang sama dengan aplikasi. Prisma menyediakan autocomplete dan pengecekan tipe, sehingga salah nama kolom bisa ketahuan sebelum program dijalankan. ORM juga menyediakan fitur migrasi skema, relasi antar tabel yang mudah, dan umumnya bisa berpindah jenis database (misalnya MySQL ke PostgreSQL) dengan perubahan kode yang kecil. Dari sisi keamanan, ORM secara default memakai query berparameter.

### 4. Apa risiko SQL injection?

SQL injection adalah serangan ketika penyerang menyisipkan perintah SQL melalui input yang diberikan ke aplikasi, sehingga struktur query berubah. Contoh kode yang rentan:

```javascript
const query = "SELECT * FROM jadwal WHERE id = " + idDariUser;
```

Jika pengguna memasukkan `1 OR 1=1`, query menjadi:

```sql
SELECT * FROM jadwal WHERE id = 1 OR 1=1
```

sehingga seluruh data terambil. Dengan input yang lebih berbahaya, penyerang bisa membaca data rahasia seperti password, mengubah atau menghapus data (misalnya `; DROP TABLE jadwal`), melewati proses login, bahkan mengambil alih database.

### 5. Mengapa penggunaan parameter query dapat mengurangi risiko SQL injection?

Pada query berparameter, struktur SQL dan nilai input dikirim secara terpisah. Database terlebih dahulu memahami bentuk query (`SELECT * FROM jadwal WHERE id = ?`), lalu nilai input dimasukkan sebagai **data murni**, bukan sebagai bagian dari perintah SQL. Jadi jika pengguna mengirim `1 OR 1=1`, database memperlakukannya sebagai satu nilai teks utuh, bukan sebagai kondisi `OR`. Karena itu input tidak bisa mengubah logika query.

> **Catatan:** ini hanya berlaku jika nilai benar-benar dikirim lewat parameter, bukan digabung ke string query dengan `+` atau template string.

### 6. Bagaimana ORM membantu programmer dalam mengakses database?

- Programmer tidak perlu menulis SQL untuk operasi umum seperti ambil, tambah, ubah, dan hapus data, cukup memanggil method seperti `findUnique`, `create`, `update`, dan `delete`.
- Hasil query langsung berupa objek JavaScript, tanpa perlu memetakan baris secara manual.
- Skema tabel didefinisikan di satu tempat (`schema.prisma`) dan bisa dipakai untuk membuat migrasi database.
- Relasi antar tabel bisa diakses dengan mudah, misalnya lewat opsi `include`.
- Parameter query ditangani otomatis, sehingga risiko SQL injection berkurang dibanding menggabungkan string secara manual.

Namun ORM bukan pengganti pemahaman SQL. Untuk query yang sangat kompleks, ORM bisa menghasilkan query yang kurang efisien, dan pada kondisi tertentu programmer tetap perlu menulis SQL mentah. Prisma sendiri menyediakan `$queryRaw` untuk keperluan itu, dan fungsi ini juga harus dipakai dengan parameter agar tetap aman dari SQL injection.