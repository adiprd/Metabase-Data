# Apache Superset - Panduan Konfigurasi Akses Data dan Dataset

## Daftar Isi
- [Setup dan Konfigurasi Dataset](#setup-dan-konfigurasi-dataset)
- [Kontrol Akses Data](#kontrol-akses-data)
- [Keamanan Level Baris (Row Level Security)](#keamanan-level-baris-row-level-security)
- [Koneksi Database](#koneksi-database)
- [Konfigurasi SQL Lab](#konfigurasi-sql-lab)
- [Pembaruan Data dan Caching](#pembaruan-data-dan-caching)
- [Pemecahan Masalah](#pemecahan-masalah)

## Setup dan Konfigurasi Dataset

### Menambahkan Koneksi Database

#### Melalui Antarmuka Pengguna:
1. **Navigasi** ke `Sources` → `Databases`
2. **Klik** `+ DATABASE`
3. **Pilih** PostgreSQL
4. **Konfigurasi** parameter koneksi:

```
Database: PostgreSQL
Host: your-postgres-host
Port: 5432
Database Name: your_database
Username: superset_user
Password: ********
```

#### Melalui Command Line:
```bash
# Export database URI
export DATABASE_URI="postgresql://superset_user:password@host:5432/your_database"

# Atau menggunakan superset CLI
superset set-database-uri \
    --database-name "Production DB" \
    --uri postgresql://superset_user:password@host:5432/your_database
```

### Konfigurasi Database Lanjutan

```python
# Dalam superset_config.py
PREVENT_UNSAFE_DB_CONNECTIONS = False

# Parameter koneksi tambahan
ADDITIONAL_DATABASE_ENGINE_PARAMETERS = {
    'postgresql': {
        'connect_args': {
            'sslmode': 'require',
            'application_name': 'superset'
        }
    }
}
```

## Kontrol Akses Data

### Peran dan Izin Pengguna

#### Peran Default di Superset:
- **Admin**: Akses penuh
- **Alpha**: Akses ke semua sumber data
- **Gamma**: Akses terbatas (perlu izin spesifik)
- **sql_lab**: Hanya akses SQL Lab
- **Public**: Tidak ada akses secara default

### Membuat Peran Kustom

```python
# Dalam superset_config.py
from superset.security import SupersetSecurityManager

class CustomSecurityManager(SupersetSecurityManager):
    def __init__(self, appbuilder):
        super(CustomSecurityManager, self).__init__(appbuilder)
        
        # Definisikan peran kustom
        self.define_custom_roles()

    def define_custom_roles(self):
        self.appbuilder.sm.add_role("Analis Data")
        self.appbuilder.sm.add_role("Manajer Penjualan")
        self.appbuilder.sm.add_role("Viewer Cabang")
```

### Menetapkan Izin

#### Melalui Antarmuka Pengguna:
1. **Navigasi** ke `Security` → `List Roles`
2. **Pilih** peran yang ingin dikonfigurasi
3. **Atur** izin:

**Untuk Analis Data:**
- `can explore on Superset`
- `can explore json on Superset`
- `can csv on Superset`
- `can share dashboard`
- `can share chart`

**Untuk Manajer Penjualan:**
- `can dashboard on Superset`
- `can explore on Superset`
- `can list dashboard`
- `can chart on Superset`

## Keamanan Level Baris (Row Level Security)

### Setup Aturan RLS

#### Melalui Antarmuka Pengguna:
1. **Navigasi** ke `Security` → `Row Level Security`
2. **Klik** `+ ADD RULE`
3. **Konfigurasi** parameter aturan:

```python
# Contoh Aturan RLS untuk Akses Berbasis Cabang
Nama Aturan: "Akses Cabang - Jakarta"
Tabel: "data_transaksi_sparepart"
Klausa: "cabang = 'JAKARTA'"
Peran: ["Sales Jakarta", "Gamma"]

Nama Aturan: "Akses Cabang - Bandung" 
Tabel: "data_transaksi_sparepart"
Klausa: "cabang = 'BANDUNG'"
Peran: ["Sales Bandung", "Gamma"]
```

#### Melalui API:
```python
import requests

def create_rls_rule(rule_data):
    headers = {
        'Authorization': 'Bearer YOUR_ACCESS_TOKEN',
        'Content-Type': 'application/json'
    }
    
    response = requests.post(
        'http://superset-host/api/v1/rowlevelsecurity/',
        headers=headers,
        json=rule_data
    )
    return response.json()

# Contoh data aturan
rule_data = {
    "name": "Akses Cabang - Surabaya",
    "tables": [{"table_name": "data_transaksi_sparepart"}],
    "filter_type": "Regular",
    "clause": "cabang = 'SURABAYA'",
    "roles": [{"id": 3}]  # ID Peran untuk Sales Surabaya
}
```

### RLS Dinamis dengan Template Jinja

```sql
-- Menggunakan Jinja untuk konteks pengguna dinamis
SELECT *
FROM mb.data_transaksi_sparepart
WHERE cabang IN (
    {% for branch in current_user.branches %}
        '{{ branch }}'{% if not loop.last %},{% endif %}
    {% endfor %}
)

-- Atau menggunakan makro
SELECT *
FROM mb.data_transaksi_sparepart
WHERE cabang = '{{ current_user.get_username().upper() }}'
```

### RLS untuk Skenario Kompleks

```python
# Dalam superset_config.py
from flask import g
import json

def get_user_branches():
    """Fungsi dinamis untuk mendapatkan cabang pengguna"""
    user_branches = {
        'user_jakarta': ['JAKARTA'],
        'user_bandung': ['BANDUNG'], 
        'user_regional': ['JAKARTA', 'BANDUNG', 'SURABAYA']
    }
    return user_branches.get(g.user.username, [])

# Daftarkan fungsi untuk digunakan di Jinja
JINJA_CONTEXT_ADDONS = {
    'user_branches': get_user_branches
}
```

## Manajemen Koneksi Database

### Koneksi Database Berganda

```python
# Konfigurasi database berganda
DATABASES = {
    'production': {
        'sqlalchemy_uri': 'postgresql://user:pass@prod-host:5432/prod_db',
        'cache_timeout': 300,
        'metadata_params': {},
        'engine_params': {
            'pool_size': 10,
            'max_overflow': 20,
            'pool_pre_ping': True
        }
    },
    'staging': {
        'sqlalchemy_uri': 'postgresql://user:pass@staging-host:5432/staging_db',
        'cache_timeout': 600,
        'engine_params': {
            'pool_size': 5,
            'max_overflow': 10
        }
    }
}
```

### Konfigurasi Connection Pool

```python
# Dalam superset_config.py
SQLALCHEMY_DATABASE_URI = 'postgresql://superset:password@localhost/superset'
SQLALCHEMY_ENGINE_OPTIONS = {
    'pool_size': 10,
    'max_overflow': 20,
    'pool_recycle': 3600,
    'pool_pre_ping': True
}

# Untuk database sumber
DATA_DB_ENGINE_PARAMS = {
    'pool_size': 20,
    'max_overflow': 30,
    'pool_recycle': 1800,
    'pool_pre_ping': True
}
```

## Konfigurasi SQL Lab

### Pengaturan SQL Lab

```python
# Dalam superset_config.py
FEATURE_FLAGS = {
    'ENABLE_TEMPLATE_PROCESSING': True,
    'SQL_VALIDATORS_BY_ENGINE': {
        'postgresql': 'PostgreSQLValidator',
    }
}

# Batasan SQL Lab
SQLLAB_CTAS_NO_LIMIT = True
SQLLAB_TIMEOUT = 3600  # 1 jam
SQLLAB_VALIDATION_TIMEOUT = 300  # 5 menit
SQLLAB_ASYNC_TIME_LIMIT_SEC = 3600

# Batasan hasil query
DISPLAY_SQL_MAX_ROW = 100000
SQL_MAX_ROW = 100000
```

### Variabel Template untuk SQL Lab

```sql
-- Menggunakan variabel template
SELECT *
FROM mb.data_transaksi_sparepart
WHERE tgl_invoice BETWEEN '{{ from_dttm }}' AND '{{ to_dttm }}'
  AND cabang IN ({{ "'" + "','".join(filter_values('cabang')) + "'" }})
  AND total_harga > {{ numeric_filter('min_price') }}

-- Klausa WHERE dinamis
SELECT *
FROM mb.data_transaksi_sparepart
WHERE 1=1
{% if filter_values('cabang') %}
  AND cabang IN ({{ "'" + "','".join(filter_values('cabang')) + "'" }})
{% endif %}
{% if from_dttm is defined %}
  AND tgl_invoice >= '{{ from_dttm }}'
{% endif %}
```

## Pembaruan Data dan Caching

### Konfigurasi Pembaruan Dataset

```python
# Konfigurasi cache
CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
}

DATA_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 86400,
    'CACHE_KEY_PREFIX': 'data_',
    'CACHE_REDIS_URL': 'redis://localhost:6379/1'
}

# Konfigurasi Celery untuk query async
class CeleryConfig(object):
    broker_url = 'redis://localhost:6379/0'
    imports = ('superset.sql_lab',)
    result_backend = 'redis://localhost:6379/0'
    worker_prefetch_multiplier = 1
    task_acks_late = False

CELERY_CONFIG = CeleryConfig
```

### Pembaruan Data Otomatis

```python
# Pemanasan cache terjadwal
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    'cache-warmup-daily': {
        'task': 'superset.tasks.cache_warmup',
        'schedule': crontab(hour=0, minute=30),
        'kwargs': {
            'strategy_name': 'top_n_dashboards',
            'top_n': 5,
            'since': '7 days ago',
        },
    },
    'refresh-datasets': {
        'task': 'superset.tasks.refresh_datasources',
        'schedule': crontab(hour=1, minute=0),
    }
}
```

## Contoh Praktis

### Setup Dataset Lengkap untuk Data Sparepart

```python
# Skrip untuk setup dataset awal
def setup_sparepart_datasets():
    """Setup semua dataset yang diperlukan"""
    
    datasets = [
        {
            'name': 'Transaksi Sparepart',
            'table_name': 'data_transaksi_sparepart',
            'schema': 'mb',
            'metrics': [
                {
                    'metric_name': 'total_penjualan',
                    'expression': 'SUM(total_harga)'
                },
                {
                    'metric_name': 'total_transaksi', 
                    'expression': 'COUNT(DISTINCT no_invoice)'
                },
                {
                    'metric_name': 'nilai_pesanan_rata_rata',
                    'expression': 'AVG(total_harga)'
                }
            ],
            'columns': [
                'cabang', 'tgl_invoice', 'no_invoice', 'kode_barang',
                'nama_barang', 'tipe_sparepart', 'total_harga', 'qty'
            ]
        },
        {
            'name': 'Kinerja Produk',
            'table_name': 'vw_product_performance', 
            'schema': 'mb',
            'metrics': [
                {
                    'metric_name': 'pendapatan',
                    'expression': 'SUM(revenue)'
                },
                {
                    'metric_name': 'kuantitas_terjual',
                    'expression': 'SUM(quantity_sold)'
                }
            ]
        }
    ]
    
    return datasets
```

### Contoh Konfigurasi RLS

```python
# Contoh aturan RLS untuk organisasi
RLS_RULES = [
    {
        'name': 'Manajer Regional - Barat',
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "cabang IN ('JAKARTA', 'BANDUNG', 'BOGOR')",
        'roles': ['Manajer Regional Barat']
    },
    {
        'name': 'Manajer Regional - Timur', 
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "cabang IN ('SURABAYA', 'MALANG', 'BALI')",
        'roles': ['Manajer Regional Timur']
    },
    {
        'name': 'Direktur Nasional',
        'tables': ['data_transaksi_sparepart', 'vw_product_performance'],
        'clause': "1=1",  # Akses ke semua data
        'roles': ['Direktur Nasional']
    }
]
```

## Pemecahan Masalah Akses Data

### Kesalahan Umum dan Solusi

```python
# Error: Izin ditolak untuk tabel
"""
Solusi: 
1. Pastikan pengguna database memiliki izin SELECT
2. Periksa aturan RLS tidak bertentangan
3. Verifikasi izin schema
"""

# Error: Tidak dapat menjelajahi dataset
"""
Solusi:
1. Periksa pengguna memiliki izin 'can explore'  
2. Verifikasi dataset dipublikasikan
3. Periksa koneksi database valid
"""

# Error: Timeout query
"""
Solusi:
1. Tingkatkan SQLLAB_TIMEOUT
2. Optimalkan query database
3. Tambahkan indeks yang sesuai
"""
```

### Debugging Masalah RLS

```sql
-- Uji klausa RLS secara langsung
SELECT * FROM mb.data_transaksi_sparepart 
WHERE cabang = 'JAKARTA'  -- Klausa RLS

-- Periksa konteks pengguna
SELECT current_user, session_user;

-- Verifikasi izin tabel
SELECT grantee, privilege_type 
FROM information_schema.role_table_grants 
WHERE table_name = 'data_transaksi_sparepart';
```

### Memantau Akses Data

```python
# Log upaya akses data
import logging
from flask import request, g

def log_data_access(dataset, user, query):
    logging.info(f"Akses data - Pengguna: {user}, Dataset: {dataset}, Query: {query}")

# Logger query kustom
class QueryLogger:
    def before_cursor_execute(self, conn, cursor, statement, parameters, context, executemany):
        if hasattr(g, 'user') and g.user is not None:
            logging.info(f"Query oleh {g.user.username}: {statement}")
```

## Praktik Terbaik untuk Akses Data

### Praktik Terbaik Keamanan
- Selalu gunakan pengguna database read-only
- Implementasikan RLS untuk data multi-tenant
- Audit izin pengguna secara berkala
- Pantau pola query yang mencurigakan

### Praktik Terbaik Performa
- Gunakan materialized views untuk query kompleks
- Implementasikan indeks database yang sesuai
- Konfigurasikan connection pooling
- Setup query timeouts

### Praktik Terbaik Manajemen
- Dokumentasikan semua aturan RLS
- Version control konfigurasi dataset
- Pembersihan dataset tidak terpakai secara berkala
- Pantau metrik penggunaan dataset

Dengan konfigurasi ini, Anda dapat mengontrol akses data secara granular di Apache Superset dan memastikan keamanan data sesuai dengan kebutuhan bisnis organisasi.
