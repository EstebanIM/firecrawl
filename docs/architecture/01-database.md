# 01 - Base de Datos (PostgreSQL)

## Overview

PostgreSQL 16 como unica base de datos. Aprovecha full-text search nativo,
vistas materializadas para queries frecuentes, y JSONB para metadata flexible.

ORM: Drizzle (type-safe, SQL-first, lightweight).

## Diagrama Entidad-Relacion

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│   stores     │     │   categories     │     │   brands     │
│──────────────│     │──────────────────│     │──────────────│
│ id (PK)      │     │ id (PK)          │     │ id (PK)      │
│ name         │     │ name             │     │ name         │
│ slug         │     │ slug             │     │ slug         │
│ url_base     │     │ parent_id (FK)   │──┐  │ logo_url     │
│ logo_url     │     │ icon             │  │  └──────┬───────┘
│ scraper_key  │     └────────┬─────────┘  │         │
│ is_active    │              │            │         │
│ config (JSON)│              │ (self-ref) │         │
└──────┬───────┘              └────────────┘         │
       │                           │                 │
       │         ┌─────────────────▼─────────────────▼──┐
       │         │            products                   │
       │         │───────────────────────────────────────│
       │         │ id (PK)                               │
       │         │ name                                  │
       │         │ slug                                  │
       │         │ brand_id (FK) ────────────────────────┘
       │         │ category_id (FK) ─────────────────────┘
       │         │ image_url
       │         │ description
       │         │ sku (nullable, para matching)
       │         │ ean (nullable, codigo de barras)
       │         │ search_vector (tsvector)
       │         │ created_at
       │         │ updated_at
       │         └──────────────────┬────────────────────┘
       │                            │
       │    ┌───────────────────────▼───────────────────┐
       │    │            product_urls                     │
       │    │────────────────────────────────────────────│
       └────│ store_id (FK)                              │
            │ product_id (FK)                            │
            │ id (PK)                                    │
            │ url                                        │
            │ is_active                                  │
            │ last_scraped_at                            │
            │ last_price                                 │
            │ last_offer_price                           │
            │ css_selectors (JSON, override por URL)     │
            │ created_at                                 │
            └───────────────────┬────────────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │        prices           │
                   │─────────────────────────│
                   │ id (PK)                 │
                   │ product_url_id (FK)     │
                   │ price (integer, CLP)    │
                   │ offer_price (integer)   │
                   │ is_available (boolean)  │
                   │ currency (default CLP)  │
                   │ scraped_at (timestamp)  │
                   └─────────────────────────┘

┌────────────────────────────────┐
│         scrape_jobs            │
│────────────────────────────────│
│ id (PK)                        │
│ store_id (FK, nullable)        │
│ status (enum)                  │
│ total_urls (integer)           │
│ success_count (integer)        │
│ error_count (integer)          │
│ errors (JSONB)                 │
│ started_at                     │
│ finished_at                    │
│ duration_ms                    │
│ triggered_by (cron/manual)     │
└────────────────────────────────┘
```

## Schema SQL (Drizzle)

### Tabla: stores

```sql
CREATE TABLE stores (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    url_base    VARCHAR(500) NOT NULL,
    logo_url    VARCHAR(500),
    scraper_key VARCHAR(50) NOT NULL UNIQUE,  -- identifica el scraper: 'falabella', 'ripley', etc.
    is_active   BOOLEAN NOT NULL DEFAULT true,
    config      JSONB NOT NULL DEFAULT '{}',   -- config especifica: headers, cookies, delays
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Datos iniciales
INSERT INTO stores (name, slug, url_base, scraper_key) VALUES
('Falabella', 'falabella', 'https://www.falabella.com/falabella-cl', 'falabella'),
('Ripley', 'ripley', 'https://simple.ripley.cl', 'ripley'),
('Paris', 'paris', 'https://www.paris.cl', 'paris'),
('Lider', 'lider', 'https://www.lider.cl', 'lider'),
('Sodimac', 'sodimac', 'https://www.sodimac.cl', 'sodimac'),
('PCFactory', 'pcfactory', 'https://www.pcfactory.cl', 'pcfactory'),
('Hites', 'hites', 'https://www.hites.com', 'hites'),
('AbcDin', 'abcdin', 'https://www.abcdin.cl', 'abcdin'),
('Easy', 'easy', 'https://www.easy.cl', 'easy'),
('Mercado Libre', 'mercadolibre', 'https://www.mercadolibre.cl', 'mercadolibre'),
('SP Digital', 'spdigital', 'https://www.spdigital.cl', 'spdigital'),
('Microplay', 'microplay', 'https://www.microplay.cl', 'microplay');
```

### Tabla: categories

```sql
CREATE TABLE categories (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    parent_id   INTEGER REFERENCES categories(id) ON DELETE SET NULL,
    icon        VARCHAR(50),  -- nombre de icono (lucide-react)
    sort_order  INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_categories_parent ON categories(parent_id);

-- Categorias iniciales
INSERT INTO categories (name, slug, icon, sort_order) VALUES
('Tecnologia', 'tecnologia', 'laptop', 1),
('Electrohogar', 'electrohogar', 'refrigerator', 2),
('Celulares', 'celulares', 'smartphone', 3),
('Television', 'television', 'tv', 4),
('Gaming', 'gaming', 'gamepad-2', 5),
('Audio', 'audio', 'headphones', 6),
('Hogar', 'hogar', 'home', 7),
('Deportes', 'deportes', 'dumbbell', 8);

-- Subcategorias ejemplo
INSERT INTO categories (name, slug, parent_id, sort_order) VALUES
('Notebooks', 'notebooks', 1, 1),
('Monitores', 'monitores', 1, 2),
('Componentes PC', 'componentes-pc', 1, 3),
('Smartphones', 'smartphones', 3, 1),
('Smartwatch', 'smartwatch', 3, 2);
```

### Tabla: brands

```sql
CREATE TABLE brands (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    logo_url    VARCHAR(500),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_brands_slug ON brands(slug);
```

### Tabla: products

```sql
CREATE TABLE products (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(500) NOT NULL,
    slug          VARCHAR(500) NOT NULL UNIQUE,
    brand_id      INTEGER REFERENCES brands(id) ON DELETE SET NULL,
    category_id   INTEGER REFERENCES categories(id) ON DELETE SET NULL,
    image_url     VARCHAR(1000),
    description   TEXT,
    sku           VARCHAR(100),  -- SKU universal si existe
    ean           VARCHAR(20),   -- codigo de barras para matching cross-store
    metadata      JSONB NOT NULL DEFAULT '{}',  -- specs: RAM, storage, color, etc.
    search_vector TSVECTOR,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indices de busqueda
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand_id);
CREATE INDEX idx_products_sku ON products(sku) WHERE sku IS NOT NULL;
CREATE INDEX idx_products_ean ON products(ean) WHERE ean IS NOT NULL;
CREATE INDEX idx_products_slug ON products(slug);

-- Trigram index para busqueda fuzzy (typos)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);

-- Trigger para mantener search_vector actualizado
CREATE OR REPLACE FUNCTION products_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('spanish', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('spanish', COALESCE(NEW.description, '')), 'C') ||
        setweight(to_tsvector('spanish', COALESCE(NEW.sku, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_search_vector
    BEFORE INSERT OR UPDATE OF name, description, sku
    ON products
    FOR EACH ROW
    EXECUTE FUNCTION products_search_vector_update();
```

### Tabla: product_urls

```sql
CREATE TABLE product_urls (
    id               SERIAL PRIMARY KEY,
    product_id       INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    store_id         INTEGER NOT NULL REFERENCES stores(id) ON DELETE CASCADE,
    url              VARCHAR(2000) NOT NULL,
    is_active        BOOLEAN NOT NULL DEFAULT true,
    last_scraped_at  TIMESTAMPTZ,
    last_price       INTEGER,         -- cache del ultimo precio (CLP)
    last_offer_price INTEGER,         -- cache del ultimo precio oferta
    css_selectors    JSONB,           -- override de selectores para esta URL especifica
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(product_id, store_id)      -- un producto por tienda
);

CREATE INDEX idx_product_urls_store ON product_urls(store_id);
CREATE INDEX idx_product_urls_product ON product_urls(product_id);
CREATE INDEX idx_product_urls_active ON product_urls(is_active) WHERE is_active = true;
CREATE INDEX idx_product_urls_last_scraped ON product_urls(last_scraped_at);
```

### Tabla: prices (historial - tabla mas grande)

```sql
CREATE TABLE prices (
    id              BIGSERIAL PRIMARY KEY,
    product_url_id  INTEGER NOT NULL REFERENCES product_urls(id) ON DELETE CASCADE,
    price           INTEGER NOT NULL,            -- precio normal en CLP (sin decimales)
    offer_price     INTEGER,                     -- precio oferta/descuento
    is_available    BOOLEAN NOT NULL DEFAULT true,
    currency        VARCHAR(3) NOT NULL DEFAULT 'CLP',
    scraped_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indice principal: buscar precios de un producto en rango de fechas
CREATE INDEX idx_prices_product_url_date ON prices(product_url_id, scraped_at DESC);

-- Particionamiento por mes (opcional, recomendado si crece mucho)
-- CREATE TABLE prices (
--     ...
-- ) PARTITION BY RANGE (scraped_at);
-- CREATE TABLE prices_2026_03 PARTITION OF prices
--     FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

### Tabla: scrape_jobs

```sql
CREATE TYPE scrape_job_status AS ENUM ('pending', 'running', 'completed', 'failed');
CREATE TYPE scrape_trigger AS ENUM ('cron', 'manual');

CREATE TABLE scrape_jobs (
    id            SERIAL PRIMARY KEY,
    store_id      INTEGER REFERENCES stores(id) ON DELETE SET NULL,
    status        scrape_job_status NOT NULL DEFAULT 'pending',
    total_urls    INTEGER NOT NULL DEFAULT 0,
    success_count INTEGER NOT NULL DEFAULT 0,
    error_count   INTEGER NOT NULL DEFAULT 0,
    errors        JSONB NOT NULL DEFAULT '[]',   -- [{url, error, timestamp}]
    triggered_by  scrape_trigger NOT NULL DEFAULT 'cron',
    started_at    TIMESTAMPTZ,
    finished_at   TIMESTAMPTZ,
    duration_ms   INTEGER,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_scrape_jobs_status ON scrape_jobs(status);
CREATE INDEX idx_scrape_jobs_store ON scrape_jobs(store_id);
CREATE INDEX idx_scrape_jobs_created ON scrape_jobs(created_at DESC);
```

## Vistas Materializadas

### Vista: precio_actual (precio mas reciente por producto por tienda)

```sql
CREATE MATERIALIZED VIEW mv_current_prices AS
SELECT DISTINCT ON (pu.id)
    pu.id AS product_url_id,
    pu.product_id,
    pu.store_id,
    pu.url,
    p.price,
    p.offer_price,
    p.is_available,
    p.scraped_at,
    s.name AS store_name,
    s.slug AS store_slug,
    s.logo_url AS store_logo
FROM product_urls pu
JOIN prices p ON p.product_url_id = pu.id
JOIN stores s ON s.id = pu.store_id
WHERE pu.is_active = true
ORDER BY pu.id, p.scraped_at DESC;

CREATE UNIQUE INDEX idx_mv_current_prices_pu ON mv_current_prices(product_url_id);
CREATE INDEX idx_mv_current_prices_product ON mv_current_prices(product_id);
CREATE INDEX idx_mv_current_prices_store ON mv_current_prices(store_id);

-- Refrescar despues de cada ciclo de scraping:
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_current_prices;
```

### Vista: mejor_precio (mejor precio por producto)

```sql
CREATE MATERIALIZED VIEW mv_best_prices AS
SELECT DISTINCT ON (product_id)
    product_id,
    store_id,
    product_url_id,
    COALESCE(offer_price, price) AS best_price,
    price AS normal_price,
    offer_price,
    is_available,
    store_name,
    store_slug,
    scraped_at
FROM mv_current_prices
WHERE is_available = true
ORDER BY product_id, COALESCE(offer_price, price) ASC;

CREATE UNIQUE INDEX idx_mv_best_prices_product ON mv_best_prices(product_id);

-- Refrescar despues de mv_current_prices:
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_best_prices;
```

## Queries Frecuentes

### Buscar productos

```sql
-- Full-text search con ranking
SELECT p.*, bp.best_price, bp.store_name,
       ts_rank(p.search_vector, query) AS rank
FROM products p
LEFT JOIN mv_best_prices bp ON bp.product_id = p.id
CROSS JOIN plainto_tsquery('spanish', 'notebook lenovo') query
WHERE p.search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Fuzzy search (typos) como fallback
SELECT p.*, bp.best_price, bp.store_name,
       similarity(p.name, 'notbook lenov') AS sim
FROM products p
LEFT JOIN mv_best_prices bp ON bp.product_id = p.id
WHERE similarity(p.name, 'notbook lenov') > 0.2
ORDER BY sim DESC
LIMIT 20;
```

### Obtener precios de un producto en todas las tiendas

```sql
SELECT
    cp.store_name,
    cp.store_slug,
    cp.store_logo,
    cp.price,
    cp.offer_price,
    cp.is_available,
    cp.scraped_at,
    cp.url
FROM mv_current_prices cp
WHERE cp.product_id = $1
ORDER BY COALESCE(cp.offer_price, cp.price) ASC;
```

### Historial de precios de un producto en una tienda

```sql
SELECT
    p.price,
    p.offer_price,
    p.is_available,
    p.scraped_at
FROM prices p
WHERE p.product_url_id = $1
  AND p.scraped_at >= NOW() - INTERVAL '90 days'
ORDER BY p.scraped_at ASC;
```

### Mejores ofertas (mayor descuento %)

```sql
SELECT
    pr.id AS product_id,
    pr.name,
    pr.slug,
    pr.image_url,
    cp.store_name,
    cp.price,
    cp.offer_price,
    ROUND(100.0 * (cp.price - cp.offer_price) / cp.price) AS discount_pct
FROM mv_current_prices cp
JOIN products pr ON pr.id = cp.product_id
WHERE cp.offer_price IS NOT NULL
  AND cp.offer_price < cp.price
  AND cp.is_available = true
ORDER BY discount_pct DESC
LIMIT 50;
```

## Politica de Retencion de Datos

| Periodo | Granularidad | Accion |
|---------|-------------|--------|
| 0-30 dias | Todos los data points | Mantener tal cual |
| 30-180 dias | 1 por dia (min del dia) | Agregar con cron job |
| 180+ dias | 1 por semana | Agregar con cron job |

```sql
-- Job de agregacion mensual (eliminar duplicados intra-dia antiguos)
DELETE FROM prices
WHERE scraped_at < NOW() - INTERVAL '30 days'
  AND id NOT IN (
      SELECT DISTINCT ON (product_url_id, DATE(scraped_at))
             id
      FROM prices
      WHERE scraped_at < NOW() - INTERVAL '30 days'
      ORDER BY product_url_id, DATE(scraped_at), scraped_at ASC
  );
```

## Migraciones

Usar Drizzle Kit para gestionar migraciones:

```bash
# Generar migracion desde schema
npx drizzle-kit generate

# Aplicar migraciones
npx drizzle-kit migrate

# Abrir studio (GUI de DB)
npx drizzle-kit studio
```
