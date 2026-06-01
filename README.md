🌿 HanaPrice — Cek Harga Pasar

Aplikasi web komunitas untuk memantau dan melaporkan harga bahan pokok di pasar secara real-time, dilengkapi asisten AI untuk membantu mencari harga termurah.


📋 Daftar Isi

Tentang Proyek
Fitur Utama
Demo & Screenshot
Tech Stack
Struktur Database
Cara Instalasi & Setup
Konfigurasi
Cara Penggunaan
Struktur File
Kontribusi


Tentang Proyek
HanaPrice adalah aplikasi web single-file berbasis komunitas yang memungkinkan warga untuk melaporkan dan memantau harga bahan pokok di pasar-pasar Banda Aceh secara real-time. Data harga dikumpulkan dari laporan pengguna dan disimpan di Supabase, sementara asisten AI (Claude) siap membantu merekomendasikan tempat belanja dengan harga terbaik.
Aplikasi ini dirancang dengan tampilan mobile-first menyerupai aplikasi native (phone-frame), namun tetap berjalan di browser biasa tanpa perlu instalasi apapun.

Fitur Utama
📊 Dashboard Harga

Menampilkan daftar harga bahan pokok terkini dari berbagai toko dan pasar
Filter berdasarkan kategori: Semua, Sayuran, Bumbu, Protein, Pokok, Snack, Minuman, Buah
Pencarian real-time berdasarkan nama produk, nama toko, atau kecamatan
Statistik ringkas: jumlah produk terpantau dan jumlah lokasi tercatat
Skeleton loading saat data sedang dimuat
Tampilan kosong yang informatif jika belum ada data

🤖 Tanya AI (Chat)

Asisten AI berbasis Claude (Anthropic) yang memahami konteks data harga lokal
Memberikan rekomendasi lokasi termurah untuk produk yang dicari
Menyertakan link Google Maps jika tersedia
Mode offline otomatis jika API key belum dikonfigurasi — tetap memberikan jawaban dari data yang ada
Riwayat percakapan disimpan selama sesi berlangsung

📝 Laporan Harga

Form pelaporan harga yang mudah digunakan oleh siapa saja (tanpa login)
Input: nama produk, nama toko, kecamatan, harga, kategori, catatan, dan link Google Maps
Foto harga (opsional) — tersedia preview sebelum dikirim
Nama pelapor opsional (default: Anonim)
Sistem otomatis: jika produk/lokasi belum ada, akan dibuat secara otomatis
Validasi form dan notifikasi sukses/gagal

🗂️ Detail Modal

Klik produk untuk melihat detail lengkap: harga, satuan, toko, kecamatan, pelapor, catatan, dan tanggal
Tombol langsung ke Google Maps jika tersedia
Dukungan gambar harga


Tech Stack
KomponenTeknologiFrontendHTML5, CSS3, Vanilla JavaScriptDatabase & BackendSupabase (PostgreSQL + REST API)AI / ChatAnthropic Claude API (claude-sonnet-4-20250514)FontPlus Jakarta Sans, DM Sans (Google Fonts)HostingBisa di-host di mana saja (GitHub Pages, Netlify, Vercel, dsb.)

Tidak ada framework JS, tidak ada build tool — cukup satu file index.html.


Struktur Database
Aplikasi menggunakan tiga tabel utama di Supabase dan satu view gabungan.
Tabel products
KolomTipeKeteranganiduuid / serialPrimary keynametextNama produk (contoh: Bawang Merah)categorytextKategori (Sayuran, Bumbu, Protein, dll.)unittextSatuan (kg, liter, buah, dll.)
Tabel locations
KolomTipeKeteranganiduuid / serialPrimary keynametextNama toko / pasarareatextKecamatan / areagmaps_linktextLink Google Maps (opsional)
Tabel prices
KolomTipeKeteranganiduuid / serialPrimary keyproduct_idfk → productsReferensi produklocation_idfk → locationsReferensi lokasipriceintegerHarga dalam Rupiahreported_bytextNama pelapor (default: Anonim)notetextCatatan tambahanprice_imagetextURL gambar harga (opsional)reported_attimestamptzWaktu pelaporan (default: now())
View price_summary
View yang menggabungkan ketiga tabel di atas, berisi kolom:
product_name, category, unit, price_image, location_name, area, gmaps_link, price, reported_by, note, reported_at
Contoh SQL untuk membuat view:
sqlCREATE VIEW price_summary AS
SELECT
  p.name       AS product_name,
  p.category,
  p.unit,
  pr.price_image,
  l.name       AS location_name,
  l.area,
  l.gmaps_link,
  pr.price,
  pr.reported_by,
  pr.note,
  pr.reported_at
FROM prices pr
JOIN products p  ON pr.product_id  = p.id
JOIN locations l ON pr.location_id = l.id;

Cara Instalasi & Setup
Prasyarat

Akun Supabase (gratis)
API key Anthropic (opsional, untuk fitur chat AI)
Web server atau hosting statis (atau buka langsung di browser)

Langkah-langkah
1. Siapkan Database Supabase
Buat project baru di Supabase, lalu jalankan SQL berikut di SQL Editor:
sql-- Tabel produk
CREATE TABLE products (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL UNIQUE,
  category TEXT DEFAULT 'Pokok',
  unit     TEXT DEFAULT 'kg'
);

-- Tabel lokasi
CREATE TABLE locations (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  area       TEXT,
  gmaps_link TEXT
);

-- Tabel harga
CREATE TABLE prices (
  id          SERIAL PRIMARY KEY,
  product_id  INTEGER REFERENCES products(id),
  location_id INTEGER REFERENCES locations(id),
  price       INTEGER NOT NULL,
  reported_by TEXT DEFAULT 'Anonim',
  note        TEXT,
  price_image TEXT,
  reported_at TIMESTAMPTZ DEFAULT NOW()
);

-- View gabungan
CREATE VIEW price_summary AS
SELECT
  p.name       AS product_name,
  p.category,
  p.unit,
  pr.price_image,
  l.name       AS location_name,
  l.area,
  l.gmaps_link,
  pr.price,
  pr.reported_by,
  pr.note,
  pr.reported_at
FROM prices pr
JOIN products p  ON pr.product_id  = p.id
JOIN locations l ON pr.location_id = l.id;
2. Aktifkan Akses Publik (Row Level Security)
Di Supabase, aktifkan RLS lalu buat policy untuk mengizinkan SELECT dan INSERT oleh anon role pada tabel products, locations, prices, dan view price_summary.
3. Download dan Konfigurasi File
bash# Clone atau download file index.html
# Tidak perlu npm install atau build apapun
4. Isi Kredensial
Buka index.html, temukan bagian berikut di awal file, dan isi dengan kredensial milikmu:
javascriptconst SUPABASE_URL  = 'https://xxxxx.supabase.co';
const SUPABASE_ANON = 'eyJhbGci...';   // anon public key
const ANTHROPIC_API_KEY = '';          // opsional, untuk fitur chat AI
5. Buka di Browser
Cukup buka index.html langsung di browser, atau upload ke hosting statis pilihanmu.

Konfigurasi
VariabelWajibKeteranganSUPABASE_URL✅ YaURL project Supabase kamuSUPABASE_ANON✅ YaAnon/public key dari SupabaseANTHROPIC_API_KEY❌ OpsionalAPI key Anthropic untuk fitur Tanya AI. Tanpa ini, chat tetap berjalan dalam mode offline menggunakan data lokal

⚠️ Catatan Keamanan: Karena ini adalah aplikasi client-side, ANTHROPIC_API_KEY akan terekspos di source code. Untuk produksi, sebaiknya buat backend proxy sederhana untuk menyembunyikan API key.


Cara Penggunaan
Melihat Harga

Buka aplikasi — halaman Dashboard langsung tampil dengan daftar harga terkini
Gunakan tab kategori untuk menyaring jenis produk
Gunakan kotak pencarian di atas untuk mencari produk atau toko tertentu
Klik kartu produk untuk melihat detail lengkap

Bertanya ke AI

Klik tab Tanya AI di navigasi bawah
Ketik pertanyaan, contoh: "Bawang merah paling murah di mana?"
AI akan menjawab berdasarkan data harga terkini yang tersedia

Melaporkan Harga

Klik tab Lapor di navigasi bawah
Isi form:

Nama Produk (wajib) — contoh: Cabai Merah Besar
Nama Toko / Pasar (wajib) — contoh: Pasar Peunayong
Kecamatan — pilih dari dropdown
Harga (wajib) — dalam Rupiah, tanpa titik/koma
Kategori — pilih tag yang sesuai
Catatan — opsional
Link Google Maps — opsional, untuk memudahkan pembeli menemukan toko
Foto Harga — opsional


Klik Kirim Laporan Harga


Struktur File
hanaprice/
└── index.html          # Seluruh aplikasi dalam satu file
    ├── <style>         # CSS lengkap dengan design tokens
    ├── <body>
    │   ├── .header     # Header dengan logo dan search bar
    │   ├── #vDash      # View: Dashboard harga
    │   ├── #vChat      # View: Chat AI
    │   ├── #vReport    # View: Form laporan
    │   ├── .bnav       # Bottom navigation
    │   └── .overlay    # Modal detail produk
    └── <script>
        ├── Supabase client (sb.get / sb.post)
        ├── State management
        ├── loadPrices / renderList
        ├── Filter & Search
        ├── Modal detail
        ├── Form laporan (doReport)
        ├── Chat AI (doChat / offlineReply)
        └── Utilities (rp, rt, x)

Kategori Produk
EmojiKategoriContoh Produk🌾PokokBeras, Gula, Minyak Goreng🥬SayuranKangkung, Bayam, Tomat🧄BumbuBawang Merah, Cabai, Jahe🥩ProteinDaging Sapi, Ayam, Ikan🍉BuahSemangka, Jeruk, Pisang🍪SnackKeripik, Biskuit🥤MinumanAir Mineral, Sirup

Kontribusi
Kontribusi sangat disambut! Cara berkontribusi:

Fork repositori ini
Buat branch fitur baru (git checkout -b fitur/nama-fitur)
Commit perubahan (git commit -m 'Tambah fitur X')
Push ke branch (git push origin fitur/nama-fitur)
Buat Pull Request

Ide Pengembangan

 Autentikasi pengguna untuk mencegah laporan palsu
 Grafik tren harga per produk
 Notifikasi ketika harga turun drastis
 Export data ke CSV/Excel
 Backend proxy untuk menyembunyikan API key
 Dukungan multi-kota selain Banda Aceh
 Upload gambar ke Supabase Storage


Lisensi
Proyek ini bersifat open-source. Silakan gunakan dan modifikasi sesuai kebutuhan.

Dibuat dengan ❤️ untuk membantu masyarakat Banda Aceh berbelanja lebih hemat.
