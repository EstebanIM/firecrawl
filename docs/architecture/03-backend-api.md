# 03 - Backend API

## Overview

API implementada con Next.js App Router (Route Handlers + Server Actions).
No hay backend separado: Next.js sirve frontend, API, y ejecuta workers.

## Estructura de Rutas

```
src/app/
├── api/
│   ├── products/
│   │   ├── search/route.ts          GET  /api/products/search?q=...&category=...&brand=...
│   │   ├── suggest/route.ts         GET  /api/products/suggest?q=...  (autocomplete)
│   │   ├── compare/route.ts         GET  /api/products/compare?slugs=a,b,c
│   │   ├── [slug]/route.ts          GET  /api/products/[slug]
│   │   ├── [slug]/history/route.ts  GET  /api/products/[slug]/history?store=...&days=90
│   │   └── [slug]/prices/route.ts   GET  /api/products/[slug]/prices
│   ├── categories/
│   │   └── route.ts                 GET  /api/categories
│   │   └── [slug]/route.ts          GET  /api/categories/[slug]
│   ├── brands/
│   │   └── route.ts                 GET  /api/brands
│   ├── stores/
│   │   └── route.ts                 GET  /api/stores
│   │   └── [slug]/route.ts          GET  /api/stores/[slug]
│   ├── deals/
│   │   └── route.ts                 GET  /api/deals?limit=50&category=...
│   └── admin/
│       ├── scrape/route.ts          POST /api/admin/scrape
│       ├── jobs/route.ts            GET  /api/admin/jobs
│       └── jobs/[id]/route.ts       GET  /api/admin/jobs/[id]
```

> **NOTA**: Las rutas de producto usan `[slug]` (no `[id]`) para coincidir con
> las URLs del frontend (`/producto/[slug]`). Paginas accedidas por SEO o links
> compartidos solo tienen el slug; no hay necesidad de un round-trip slug→ID.

## Formato de Errores Estandar

Todos los endpoints usan el mismo envelope para errores:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Descripcion legible del error",
    "details": [{ "field": "q", "message": "Search query is required" }]
  }
}
```

Codigos HTTP: `400` (validation), `401` (unauthorized), `404` (not found), `429` (rate limited), `500` (server error).

## Endpoints Detallados

### GET /api/products/search

Busqueda de productos con full-text search y filtros.

```typescript
// src/app/api/products/search/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { products } from '@/lib/db/schema';
import { sql } from 'drizzle-orm';

export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const q = searchParams.get('q') || '';
  const category = searchParams.get('category');   // slug
  const brand = searchParams.get('brand');          // slug
  const sort = searchParams.get('sort') || 'relevance'; // relevance | price_asc | price_desc
  const page = parseInt(searchParams.get('page') || '1');
  const limit = Math.min(parseInt(searchParams.get('limit') || '20'), 50);
  const offset = (page - 1) * limit;

  // ... query con full-text search, filtros, paginacion
}
```

**Parametros:**
| Param | Tipo | Default | Descripcion |
|-------|------|---------|-------------|
| q | string | '' | Texto de busqueda |
| category | string | - | Filtro por slug de categoria |
| brand | string | - | Filtro por slug de marca |
| sort | enum | relevance | relevance, price_asc, price_desc, discount |
| page | int | 1 | Pagina actual |
| limit | int | 20 | Items por pagina (max 50) |
| min_price | int | - | Precio minimo CLP |
| max_price | int | - | Precio maximo CLP |

**Respuesta:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Notebook Lenovo IdeaPad 3 15.6\" AMD Ryzen 5 8GB 256GB SSD",
      "slug": "notebook-lenovo-ideapad-3-15-amd-ryzen-5-8gb-256gb",
      "imageUrl": "https://...",
      "category": { "name": "Notebooks", "slug": "notebooks" },
      "brand": { "name": "Lenovo", "slug": "lenovo" },
      "bestPrice": 429990,
      "normalPrice": 529990,
      "storeName": "PCFactory",
      "storeSlug": "pcfactory",
      "storeCount": 5,
      "discountPct": 19,
      "lastScrapedAt": "2026-03-23T08:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 142,
    "totalPages": 8
  }
}
```

### GET /api/products/[slug]

Detalle de un producto con precios en todas las tiendas. Usa slug (no ID) para coincidir con URLs de frontend.

**Respuesta:**
```json
{
  "product": {
    "id": 1,
    "name": "Notebook Lenovo IdeaPad 3 15.6\"...",
    "slug": "notebook-lenovo-ideapad-3-15-amd-ryzen-5-8gb-256gb",
    "description": "...",
    "imageUrl": "https://...",
    "category": { "id": 9, "name": "Notebooks", "slug": "notebooks" },
    "brand": { "id": 5, "name": "Lenovo", "slug": "lenovo" },
    "metadata": { "ram": "8GB", "storage": "256GB SSD", "processor": "AMD Ryzen 5" },
    "createdAt": "2026-03-01T00:00:00Z"
  },
  "prices": [
    {
      "storeId": 6,
      "storeName": "PCFactory",
      "storeSlug": "pcfactory",
      "storeLogo": "https://...",
      "url": "https://www.pcfactory.cl/producto/...",
      "price": 529990,
      "offerPrice": 429990,
      "isAvailable": true,
      "lastScrapedAt": "2026-03-23T08:00:00Z"
    },
    {
      "storeId": 1,
      "storeName": "Falabella",
      "storeSlug": "falabella",
      "storeLogo": "https://...",
      "url": "https://www.falabella.com/...",
      "price": 549990,
      "offerPrice": null,
      "isAvailable": true,
      "lastScrapedAt": "2026-03-23T08:15:00Z"
    }
  ]
}
```

### GET /api/products/[slug]/history

Historial de precios para graficos.

**Parametros:**
| Param | Tipo | Default | Descripcion |
|-------|------|---------|-------------|
| store | string | all | Slug de tienda, o "all" para todas |
| days | int | 90 | Cantidad de dias hacia atras |

**Respuesta:**
```json
{
  "productSlug": "notebook-lenovo-ideapad-3-15-amd-ryzen-5-8gb-256gb",
  "productName": "Notebook Lenovo IdeaPad 3...",
  "history": [
    {
      "store": { "slug": "pcfactory", "name": "PCFactory" },
      "prices": [
        { "date": "2026-01-15", "price": 549990, "offerPrice": null },
        { "date": "2026-01-20", "price": 549990, "offerPrice": 499990 },
        { "date": "2026-02-01", "price": 529990, "offerPrice": 429990 }
      ]
    },
    {
      "store": { "slug": "falabella", "name": "Falabella" },
      "prices": [
        { "date": "2026-01-15", "price": 579990, "offerPrice": null },
        { "date": "2026-02-10", "price": 549990, "offerPrice": null }
      ]
    }
  ]
}
```

> **Cambio vs version anterior**: La respuesta usa un array de objetos (no object-keyed-by-slug)
> para mejor tipado en TypeScript y consistencia. Incluye nombre de tienda para evitar lookups extra.

### GET /api/deals

Mejores ofertas actuales.

**Parametros:**
| Param | Tipo | Default | Descripcion |
|-------|------|---------|-------------|
| limit | int | 50 | Cantidad de resultados |
| category | string | - | Filtro por categoria slug |
| min_discount | int | 10 | Descuento minimo % |

**Respuesta:**
```json
{
  "data": [
    {
      "productId": 42,
      "productName": "Smart TV Samsung 55\" 4K...",
      "productSlug": "smart-tv-samsung-55-4k",
      "imageUrl": "https://...",
      "storeName": "Ripley",
      "storeSlug": "ripley",
      "normalPrice": 499990,
      "offerPrice": 349990,
      "discountPct": 30,
      "url": "https://simple.ripley.cl/..."
    }
  ]
}
```

### POST /api/admin/scrape

Trigger manual de scraping (protegido con API key simple).

```typescript
// Proteccion basica con API key en header
const ADMIN_API_KEY = process.env.ADMIN_API_KEY;

export async function POST(request: NextRequest) {
  const apiKey = request.headers.get('x-api-key');
  // Usar timingSafeEqual para prevenir timing attacks
  if (!apiKey || !crypto.timingSafeEqual(
    Buffer.from(apiKey),
    Buffer.from(ADMIN_API_KEY!)
  )) {
    return NextResponse.json({ error: { code: 'UNAUTHORIZED', message: 'Invalid API key' } }, { status: 401 });
  }

  const body = await request.json();
  // body: { storeSlug?: string }  -- null = todas las tiendas

  // Encolar job en BullMQ
  // ...

  return NextResponse.json({ jobId: '...', status: 'queued' });
}
```

### GET /api/admin/jobs

Estado de jobs de scraping.

**Respuesta:**
```json
{
  "data": [
    {
      "id": 150,
      "storeName": "Falabella",
      "status": "completed",
      "totalUrls": 1200,
      "successCount": 1185,
      "errorCount": 15,
      "triggeredBy": "cron",
      "startedAt": "2026-03-23T06:00:00Z",
      "finishedAt": "2026-03-23T06:45:00Z",
      "durationMs": 2700000
    }
  ]
}
```

### GET /api/products/suggest

Autocomplete ligero para el SearchBar. Usa trigram search con LIMIT 8.
Respuesta minima para velocidad (debounce 250ms en frontend).

```typescript
export async function GET(request: NextRequest) {
  const q = request.nextUrl.searchParams.get('q') || '';
  if (q.length < 2) return NextResponse.json({ data: [] });

  // Trigram search con limite bajo para velocidad
  const results = await db.select({
    name: products.name,
    slug: products.slug,
    imageUrl: products.imageUrl,
  })
  .from(products)
  .where(sql`similarity(${products.name}, ${q}) > 0.2`)
  .orderBy(sql`similarity(${products.name}, ${q}) DESC`)
  .limit(8);

  return NextResponse.json({ data: results });
}
```

### GET /api/brands

Lista de marcas con conteo de productos. Necesario para filtros de busqueda.

**Respuesta:**
```json
{
  "data": [
    { "name": "Samsung", "slug": "samsung", "productCount": 342 },
    { "name": "Lenovo", "slug": "lenovo", "productCount": 189 }
  ]
}
```

## Service Layer

```typescript
// src/lib/services/product-service.ts

class ProductService {
  /** Buscar productos con full-text search */
  async search(params: SearchParams): Promise<PaginatedResult<ProductSummary>>

  /** Obtener producto por slug con precios actuales en todas las tiendas */
  async getBySlug(slug: string): Promise<ProductDetail | null>

  /** Autocomplete (trigram, max 8 resultados) */
  async suggest(q: string): Promise<ProductSuggestion[]>

  /** Historial de precios */
  async getPriceHistory(productSlug: string, options: HistoryOptions): Promise<PriceHistory>

  /** Mejores ofertas */
  async getDeals(params: DealParams): Promise<ProductDeal[]>

  /** Productos por categoria */
  async getByCategory(slug: string, pagination: Pagination): Promise<PaginatedResult<ProductSummary>>

  /** Comparar 2-4 productos side-by-side */
  async compare(slugs: string[]): Promise<ProductDetail[]>
}

// src/lib/services/store-service.ts

class StoreService {
  /** Listar tiendas activas */
  async getAll(): Promise<Store[]>

  /** Detalle de tienda con stats */
  async getBySlug(slug: string): Promise<StoreDetail | null>
}

// src/lib/services/scraping-service.ts

class ScrapingService {
  /** Encolar scraping para una o todas las tiendas */
  async enqueueJob(storeSlug?: string): Promise<string>

  /** Estado de un job */
  async getJobStatus(jobId: number): Promise<ScrapeJob>

  /** Listar jobs recientes */
  async getRecentJobs(limit: number): Promise<ScrapeJob[]>
}
```

## Caching con Redis

```typescript
// src/lib/cache.ts

import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

interface CacheOptions {
  ttl: number; // segundos
}

async function cached<T>(key: string, fn: () => Promise<T>, options: CacheOptions): Promise<T> {
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);

  const result = await fn();
  await redis.setex(key, options.ttl, JSON.stringify(result));
  return result;
}
```

**Estrategia de cache:**
| Dato | TTL | Key pattern | Invalidacion |
|------|-----|-------------|--------------|
| Busqueda de productos | 5 min | `search:{hash(params)}` | TTL natural |
| Detalle de producto | 10 min | `product:{id}` | Despues de scraping |
| Historial de precios | 1 hora | `history:{productId}:{store}:{days}` | Despues de scraping |
| Lista de categorias | 1 hora | `categories` | Manual |
| Lista de tiendas | 1 hora | `stores` | Manual |
| Deals | 15 min | `deals:{hash(params)}` | TTL natural |

## Invalidacion de Cache Post-Scraping

Despues de que un ciclo de scraping completa y las vistas materializadas se refrescan:

```typescript
// src/lib/cache.ts

/** Invalidar caches de productos scrapeados */
async function invalidateAfterScrape(productIds: number[]): Promise<void> {
  const pipeline = redis.pipeline();

  for (const id of productIds) {
    pipeline.del(`product:${id}`);
    // Invalidar todos los periodos de historial
    const historyKeys = await redis.keys(`history:${id}:*`);
    for (const key of historyKeys) pipeline.del(key);
  }

  // Deals y search se invalidan por TTL natural (5-15 min)
  pipeline.del('deals:*');  // o usar SCAN para borrado por patron

  await pipeline.exec();
}
```

## Data Fetching Strategy

### Server Components (RSC)
Paginas publicas llaman al service layer **directamente** (sin HTTP):
```typescript
// src/app/producto/[slug]/page.tsx (RSC)
const product = await productService.getBySlug(params.slug);
// NO hacer: const res = await fetch('/api/products/...')
```

### Client Components
Componentes interactivos (busqueda, filtros) usan **SWR** contra API routes:
```typescript
// src/components/search/SearchResults.tsx ("use client")
const { data } = useSWR(`/api/products/search?${params}`, fetcher);
```

### API Routes
Existen para: client-side fetching, futura API publica, admin endpoints.
Los RSC **nunca** llaman a sus propias API routes — seria un round-trip innecesario.

## Seguridad

- **Admin endpoints**: Protegidos con `x-api-key` header + `crypto.timingSafeEqual()`
- **Rate limiting**: `@upstash/ratelimit` con Redis, sliding window:
  - Publicos: 100 req/min por IP
  - Search: 30 req/min por IP (mas costoso)
  - Admin: 10 req/min por API key
  - Respuesta: `429 Too Many Requests` con headers `Retry-After` y `X-RateLimit-Remaining`
- **Input validation**: Zod schemas en todos los endpoints
- **SQL injection**: Imposible con Drizzle ORM (queries parametrizadas)
- **CORS**: Configurar solo para dominio propio en produccion

## Estructura de Archivos

```
src/
├── app/
│   └── api/           # Route Handlers (endpoints HTTP)
│       ├── products/
│       ├── categories/
│       ├── stores/
│       ├── deals/
│       └── admin/
├── lib/
│   ├── db/
│   │   ├── index.ts   # Conexion Drizzle
│   │   ├── schema.ts  # Schema de tablas
│   │   └── queries/   # Queries reutilizables
│   ├── services/      # Business logic
│   │   ├── product-service.ts
│   │   ├── store-service.ts
│   │   └── scraping-service.ts
│   ├── cache.ts       # Redis cache helper
│   └── validators/    # Zod schemas para request validation
│       ├── search.ts
│       └── admin.ts
```
