# backend_golang

//AssignmentDay23: 
Soal 1:
Manajemen inventaris Database (using mysql)

1. Create database manajemen inventaris:
create database manajemen_inventaris;

-----------------------------------------
2. Buat Tabel-tabel:
-- tabel produk:
CREATE TABLE Produk (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nama VARCHAR(100) NOT NULL,
    deskripsi TEXT,
    harga DECIMAL(10,2) NOT NULL,
    kategori VARCHAR(50) NOT NULL
);

-- tabel inventaris:
CREATE TABLE Inventaris (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produk_id INT NOT NULL,
    jumlah INT NOT NULL CHECK(jumlah >= 0),
    lokasi VARCHAR(50) NOT NULL,
    
    FOREIGN KEY (produk_id) 
        REFERENCES Produk(id) 
        ON DELETE CASCADE
);

-- tabel pesanan:
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
3. Insert Data Sampel:
   
-- data produk:
INSERT INTO Produk (nama, deskripsi, harga, kategori) VALUES
('Laptop AX', 'Laptop 15 inci RAM 8GB', 8500000, 'Elektronik'),
('Buku Tulis', 'Buku 100 halaman', 12000, 'Alat Tulis'),
('Mouse Wireless', 'Mouse Bluetooth', 150000, 'Aksesoris');

-- data inventaris:
INSERT INTO Inventaris (produk_id, jumlah, lokasi) VALUES
(1, 15, 'Gudang Utara'),
(2, 200, 'Gudang Selatan'),
(3, 50, 'Gudang Barat'),
(1, 5, 'Display Toko');

-- data pesanan:
INSERT INTO Pesanan (produk_id, jumlah, tanggal_pesanan) VALUES
(1, 2, '2025-06-01'),
(3, 5, '2025-06-03'),
(2, 10, '2025-06-05'),
(1, 1, '2025-06-10');

-----------------------------------------------------------------
4. Query Dasar:
   
-- ambil semua produk:
SELECT * FROM Produk;

-- ambil inventaris dengan detail produk:
SELECT p.nama, i.jumlah, i.lokasi
FROM Inventaris i
JOIN Produk p ON i.produk_id = p.id;

-- ambil pesanan dengan detail produkny:
SELECT o.id, p.nama, o.jumlah, o.tanggal_pesanan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id;

------------------------------------------------
5. Query Agregasi:

-- total pesanan per produk:
SELECT 
    p.nama,
    SUM(o.jumlah) AS total_pesanan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id
GROUP BY p.nama;

-- total stok per lokasi untuk produk tertentu:
SELECT 
    p.nama,
    i.lokasi,
    SUM(i.jumlah) AS total_stok
FROM Inventaris i
JOIN Produk p ON i.produk_id = p.id
WHERE p.id = 1 -- ganti ID produk sesuai kita mau
GROUP BY p.nama, i.lokasi;

--------------------------------------------------
6. Query Optimisasi Operasional:

-- cek dulu ketersediaan stok produk:
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
WHERE p.id = 1 -- ganti ID produk sesuai kita mau
GROUP BY p.nama;

-- riwayat penjualan per kategori:
SELECT 
    p.kategori,
    COUNT(o.id) AS jumlah_transaksi,
    SUM(o.jumlah * p.harga) AS total_pendapatan
FROM Pesanan o
JOIN Produk p ON o.produk_id = p.id
GROUP BY p.kategori;

------------------------------------------------------
Soal 2:
API Deployment:

1. Menginstall Gin dan Driver MySQL:
go get -u github.com/gin-gonic/gin
go get -u github.com/go-sql-driver/mysql

---------------------------------------
2. Struktur Project:
├── main.go
├── go.mod
├── handlers/
│   ├── product.go
│   ├── inventory.go
│   └── external.go
├── database/
│   └── db.go
└── models/
    ├── product.go
    └── order.go

-----------------------------------------------
3. Queries:

a. main.go :

package main
import (
	"database/sql"
	"log"
	
	"github.com/gin-gonic/gin"
	"github.com/yourname/inventaris/handlers"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// inisialisasi database:
	db, err := sql.Open("mysql", "user:sony123@tcp(localhost:3306)/inventaris")
	if err != nil {
		log.Fatal("Database connection failed:", err)
	}
	defer db.Close()

	// Buat router Gin
	router := gin.Default()

	// Setup routes
	api := router.Group("/api")
	{
		productHandler := handlers.NewProductHandler(db)
		api.GET("/products", productHandler.GetProducts)
		api.POST("/products", productHandler.CreateProduct)
		
		inventoryHandler := handlers.NewInventoryHandler(db)
		api.GET("/inventory/:productId", inventoryHandler.GetProductInventory)
		api.PUT("/inventory/:id", inventoryHandler.UpdateStock)
		
		externalHandler := handlers.NewExternalHandler()
		api.GET("/external-products", externalHandler.FetchExternalProducts)
	}

	// Jalankan server
	router.Run(":8080")
}

--------------------------------------------
2. models/product.go (Model Data):

package models

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
}

type Inventory struct {
    ID        int    `json:"id"`
    ProductID int    `json:"product_id"`
    Quantity  int    `json:"quantity"`
    Location  string `json:"location"`
}

type ExternalProduct struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Category    string  `json:"category"`
}

-------------------------------------------------
3. handlers/product.go (Produk Handler):

package handlers
import (
	"database/sql"
	"net/http"
	
	"github.com/gin-gonic/gin"
	"github.com/yourname/inventaris/models"
)

type ProductHandler struct {
	DB *sql.DB
}

func NewProductHandler(db *sql.DB) *ProductHandler {
	return &ProductHandler{DB: db}
}

// GetProducts - GET /api/products
func (h *ProductHandler) GetProducts(c *gin.Context) {
	rows, err := h.DB.Query("SELECT id, nama, deskripsi, harga, kategori FROM Produk")
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	defer rows.Close()

	var products []models.Product
	for rows.Next() {
		var p models.Product
		if err := rows.Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Category); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		products = append(products, p)
	}

	c.JSON(http.StatusOK, products)
}

// CreateProduct - POST /api/products
func (h *ProductHandler) CreateProduct(c *gin.Context) {
	var product models.Product
	if err := c.ShouldBindJSON(&product); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	result, err := h.DB.Exec("INSERT INTO Produk (nama, deskripsi, harga, kategori) VALUES (?, ?, ?, ?)",
		product.Name, product.Description, product.Price, product.Category)
		
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	id, _ := result.LastInsertId()
	product.ID = int(id)

	c.JSON(http.StatusCreated, product)
}

------------------------------------------------------------
4. handlers/inventory.go (Inventaris Handler):

package handlers
import (
	"database/sql"
	"net/http"
	"strconv"
	
	"github.com/gin-gonic/gin"
	"github.com/yourname/inventaris/models"
)

type InventoryHandler struct {
	DB *sql.DB
}

func NewInventoryHandler(db *sql.DB) *InventoryHandler {
	return &InventoryHandler{DB: db}
}

// GetProductInventory - GET /api/inventory/:productId
func (h *InventoryHandler) GetProductInventory(c *gin.Context) {
	productID, err := strconv.Atoi(c.Param("productId"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid product ID"})
		return
	}

	rows, err := h.DB.Query(`
		SELECT i.id, i.produk_id, i.jumlah, i.lokasi
		FROM Inventaris i
		WHERE i.produk_id = ?`, productID)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	defer rows.Close()

	var inventory []models.Inventory
	for rows.Next() {
		var inv models.Inventory
		if err := rows.Scan(&inv.ID, &inv.ProductID, &inv.Quantity, &inv.Location); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		inventory = append(inventory, inv)
	}

	if len(inventory) == 0 {
		c.JSON(http.StatusNotFound, gin.H{"message": "Inventory not found"})
		return
	}

	c.JSON(http.StatusOK, inventory)
}

// UpdateStock - PUT /api/inventory/:id
func (h *InventoryHandler) UpdateStock(c *gin.Context) {
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid inventory ID"})
		return
	}

	var update struct {
		Quantity int `json:"quantity"`
	}
	if err := c.ShouldBindJSON(&update); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	_, err = h.DB.Exec("UPDATE Inventaris SET jumlah = ? WHERE id = ?", 
		update.Quantity, id)
		
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{"message": "Inventory updated"})
}

--------------------------------------------------------------
5. handlers/external.go (External API Handler):

package handlers
import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	
	"github.com/gin-gonic/gin"
	"github.com/yourname/inventaris/models"
)

type ExternalHandler struct{}

func NewExternalHandler() *ExternalHandler {
	return &ExternalHandler{}
}

// FetchExternalProducts - GET /api/external-products
func (h *ExternalHandler) FetchExternalProducts(c *gin.Context) {
	resp, err := http.Get("https://dummyjson.com/products")
	if err != nil {
		c.JSON(http.StatusServiceUnavailable, gin.H{"error": "External API unreachable"})
		return
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to read response"})
		return
	}

	var result struct {
		Products []models.ExternalProduct `json:"products"`
	}
	if err := json.Unmarshal(body, &result); err != nil {
		fmt.Println("Error unmarshalling:", err)
		c.JSON(http.StatusInternalServerError, gin.H{"error": "Invalid API response"})
		return
	}

	c.JSON(http.StatusOK, result.Products)
}

----------------------------------
Cara Menjalankan Project:
# Inisialisasi modul
go mod init inventaris-api

# Jalankan server
go run main.go

----------------------------------------
