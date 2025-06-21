# backend_golang

//AssignmentDay23: Manajemen inventaris Database (using mysql)

1. Create database manajemen inventaris:
create database manajemen_invebtaris;

-----------------------------------------
2. Buat Tabel-tabel:
-- Tabel Produk (MySQL)
CREATE TABLE Produk (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nama VARCHAR(100) NOT NULL,
    deskripsi TEXT,
    harga DECIMAL(10,2) NOT NULL,
    kategori VARCHAR(50) NOT NULL
);

-- Tabel Inventaris (MySQL)
CREATE TABLE Inventaris (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produk_id INT NOT NULL,
    jumlah INT NOT NULL CHECK(jumlah >= 0),
    lokasi VARCHAR(50) NOT NULL,
    
    FOREIGN KEY (produk_id) 
        REFERENCES Produk(id) 
        ON DELETE CASCADE
);

-- Tabel Pesanan (MySQL) - versi fixed
CREATE TABLE Pesanan (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produk_id INT NOT NULL,
    jumlah INT NOT NULL CHECK(jumlah > 0),
    tanggal_pesanan DATE NOT NULL DEFAULT (CURDATE()),
    
    FOREIGN KEY (produk_id) 
        REFERENCES Produk(id) 
        ON DELETE RESTRICT
);

-----------------------------------------------------------------
3. Insert Data Sampel
sql
-- Data produk
INSERT INTO Produk (nama, deskripsi, harga, kategori) VALUES
('Laptop AX', 'Laptop 15 inci RAM 8GB', 8500000, 'Elektronik'),
('Buku Tulis', 'Buku 100 halaman', 12000, 'Alat Tulis'),
('Mouse Wireless', 'Mouse Bluetooth', 150000, 'Aksesoris');

-- Data inventaris
INSERT INTO Inventaris (produk_id, jumlah, lokasi) VALUES
(1, 15, 'Gudang Utara'),
(2, 200, 'Gudang Selatan'),
(3, 50, 'Gudang Barat'),
(1, 5, 'Display Toko');

-- Data pesanan
INSERT INTO Pesanan (produk_id, jumlah, tanggal_pesanan) VALUES
(1, 2, '2025-06-01'),
(3, 5, '2025-06-03'),
(2, 10, '2025-06-05'),
(1, 1, '2025-06-10');

-----------------------------------------------------------------
4. Query Dasar
sql
-- Ambil semua produk
SELECT * FROM Produk;

-- Ambil inventaris dengan detail produk
SELECT p.nama, i.jumlah, i.lokasi
FROM Inventaris i
JOIN Produk p ON i.produk_id = p.id;

-- Ambil pesanan dengan detail produk
SELECT o.id, p.nama, o.jumlah, o.tanggal_pesanan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id;

------------------------------------------------
5. Query Agregasi
sql
-- Total pesanan per produk
SELECT 
    p.nama,
    SUM(o.jumlah) AS total_pesanan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id
GROUP BY p.nama;

-- Total stok per lokasi untuk produk tertentu
SELECT 
    p.nama,
    i.lokasi,
    SUM(i.jumlah) AS total_stok
FROM Inventaris i
JOIN Produk p ON i.produk_id = p.id
WHERE p.id = 1 -- Ganti ID produk sesuai kebutuhan
GROUP BY p.nama, i.lokasi;

--------------------------------------------------
6. Query Optimisasi Operasional
sql
-- Cek ketersediaan stok produk
SELECT 
    p.nama,
    SUM(i.jumlah) AS total_stok,
    COALESCE(SUM(o.jumlah), 0) AS total_pesanan,
    (SUM(i.jumlah) - COALESCE(SUM(o.jumlah), 0)) AS stok_tersedia
FROM Produk p
LEFT JOIN Inventaris i ON p.id = i.produk_id
LEFT JOIN (
    SELECT produk_id, SUM(jumlah) AS jumlah
    FROM Pesanan
    GROUP BY produk_id
) o ON p.id = o.produk_id
WHERE p.id = 1 -- Ganti ID produk sesuai kebutuhan
GROUP BY p.nama;

-- Riwayat penjualan per kategori
SELECT 
    p.kategori,
    COUNT(o.id) AS jumlah_transaksi,
    SUM(o.jumlah * p.harga) AS total_pendapatan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id
GROUP BY p.kategori;

------------------------------------------------------
