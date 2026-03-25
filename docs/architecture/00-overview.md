# 00 - Vision General del Sistema

## Nombre del Proyecto: PrecioChile (nombre de trabajo)

Plataforma web de monitoreo y comparacion de precios de tiendas de retail en Chile.
Similar a solotodo.cl / knasta.cl.

## Objetivo

Permitir a los usuarios buscar productos y comparar precios entre multiples tiendas chilenas,
ver el historial de precios, y encontrar las mejores ofertas.

## Stack Tecnologico

| Componente | Tecnologia | Justificacion |
|------------|------------|---------------|
| Frontend + Backend | Next.js 15 (App Router) | SSR para SEO, API routes integradas, React Server Components |
| Base de datos | PostgreSQL 16 | Full-text search nativo, JSONB, vistas materializadas, robusto |
| Cache + Colas | Redis 7 | Cache de queries, BullMQ para jobs de scraping |
| Cola de jobs | BullMQ | Integra con Redis, reintentos, prioridades, dashboard |
| Scraping | Firecrawl (`@mendable/firecrawl-js`) | Renderiza JS-heavy sites via self-hosted Firecrawl; devuelve HTML para parsear con CSS selectors |
| HTML parsing | cheerio | Extraccion determinista de precios con CSS selectors (sin LLM, $0 por scrape) |
| ORM | Drizzle ORM | Type-safe, migraciones, lightweight, SQL-first |
| Estilos | Tailwind CSS 4 | Utility-first, rapido de prototipar |
| Charts | Recharts | Graficos de historial de precios, basado en React |
| Logging | pino | Logging estructurado JSON, bajo overhead, compatible con Next.js |
| Error tracking | Sentry | Captura excepciones en produccion, alertas, stack traces |
| Runtime | Node.js 22 LTS | Soporte nativo de TypeScript (--experimental-strip-types) |
| Containerizacion | Docker + Docker Compose | Reproducible en desarrollo y produccion |
| Reverse proxy | Caddy 2 | HTTPS automatico via Let's Encrypt, cero configuracion TLS |

### Connection Pooling

Drizzle ORM usa `pg` (node-postgres) directamente. En produccion, configurar pool
limitado para evitar saturar PostgreSQL en un VPS de 512MB:

```typescript
// src/lib/db/index.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,           // max conexiones simultaneas (default: 10 — ok para 1 VPS)
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 2_000,
});

export const db = drizzle(pool);
```

Para cargas mayores (futuro), considerar PgBouncer como proxy de pooling externo.

## Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────┐
│                     USUARIO / BROWSER                    │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    NEXT.JS APP (SSR)                      │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │   Pages /    │  │  API Routes  │  │Server Actions  │  │
│  │  Components  │  │  /api/*      │  │  (Admin)       │  │
│  └──────────────┘  └──────┬───────┘  └───────┬───────┘  │
│                           │                   │           │
└───────────────────────────┼───────────────────┼──────────┘
                            │                   │
                 ┌──────────▼───────────────────▼──────────┐
                 │              SERVICE LAYER               │
                 │                                          │
                 │  ┌─────────────┐  ┌──────────────────┐  │
                 │  │  Product    │  │  Scraping         │  │
                 │  │  Service    │  │  Service          │  │
                 │  └──────┬──────┘  └────────┬─────────┘  │
                 │         │                  │             │
                 └─────────┼──────────────────┼────────────┘
                           │                  │
              ┌────────────▼──────┐    ┌──────▼──────────┐
              │   PostgreSQL      │    │     Redis        │
              │                   │    │                  │
              │ - products        │    │ - Cache          │
              │ - prices          │    │ - BullMQ queues  │
              │ - stores          │    │ - Rate limiting  │
              │ - categories      │    │                  │
              └───────────────────┘    └────────┬─────────┘
                                                │
                                      ┌─────────▼─────────┐
                                      │   BullMQ Workers   │
                                      │                    │
                                      │  ┌──────────────┐  │
                                      │  │ Scrape Jobs   │  │
                                      │  │ (Playwright)  │  │
                                      │  └──────────────┘  │
                                      │                    │
                                      │  ┌──────────────┐  │
                                      │  │ Store         │  │
                                      │  │ Scrapers      │  │
                                      │  │ - Falabella   │  │
                                      │  │ - Ripley      │  │
                                      │  │ - Paris       │  │
                                      │  │ - Lider       │  │
                                      │  │ - ...         │  │
                                      │  └──────────────┘  │
                                      └────────────────────┘
```

## Flujo de Datos Principal

### Flujo de Scraping (background, 1-2 veces al dia)

```
node-cron (trigger)
    │
    ▼
Crear jobs en BullMQ (1 job por tienda o grupo de productos)
    │
    ▼
Worker toma job → Selecciona StoreScraper de la tienda
    │
    ▼
Playwright abre browser → Navega a URL del producto
    │
    ▼
CSS Selectors extraen: precio, nombre, disponibilidad, imagen
    │
    ▼
Validacion de datos (precio numerico, rango razonable)
    │
    ▼
INSERT en tabla `prices` (historial) + UPDATE vista materializada
    │
    ▼
Log resultado en `scrape_jobs` (exito/error/duracion)
```

### Flujo de Busqueda (usuario en frontend)

```
Usuario escribe "notebook lenovo" en SearchBar
    │
    ▼
GET /api/products/search?q=notebook+lenovo
    │
    ▼
PostgreSQL full-text search (tsvector + pg_trgm)
    │
    ▼
Retorna productos con: mejor precio actual, # tiendas, imagen
    │
    ▼
Usuario hace click en producto
    │
    ▼
GET /api/products/[slug] → Precios por tienda + historial
    │
    ▼
Render: tabla comparativa + grafico de historial (Recharts)
```

## Tiendas Target (iniciales)

| Tienda | Tipo | URL | JS-heavy | Prioridad |
|--------|------|-----|----------|-----------|
| Falabella | Departamental | falabella.com/falabella-cl | Si | Alta |
| Ripley | Departamental | simple.ripley.cl | Si | Alta |
| Paris | Departamental | paris.cl | Si | Alta |
| Lider / Walmart | Supermercado | lider.cl | Si | Alta |
| Sodimac | Home improvement | sodimac.cl | Si | Media |
| PCFactory | Tecnologia | pcfactory.cl | Moderado | Alta |
| Hites | Departamental | hites.com | Moderado | Media |
| AbcDin | Departamental | abcdin.cl | Moderado | Media |
| Easy | Home improvement | easy.cl | Si | Media |
| Mercado Libre | Marketplace | mercadolibre.cl | Si | Media |
| ~~Linio~~ | ~~Marketplace~~ | linio.cl | — | **CERRADO** — Linio.cl ceso operaciones en Chile. No incluir. |
| Microplay | Tecnologia/Gaming | microplay.cl | Moderado | Baja |
| SP Digital | Tecnologia | spdigital.cl | Bajo | Media |
| Knasta referencia | Comparador | knasta.cl | Si | Referencia |

## Decisiones Tecnicas Clave

### Por que Next.js y no un backend separado?
- Para 10-20 tiendas y scraping 1-2x/dia, no necesitamos microservicios
- API Routes de Next.js son suficientes para los endpoints necesarios
- SSR mejora SEO drasticamente (importante para un comparador de precios)
- Un solo deploy simplifica infraestructura en VPS economico

### Por que Drizzle ORM y no Prisma?
- Mas ligero (menos RAM, importante en VPS)
- SQL-first: cuando necesitas queries complejas (vistas materializadas, full-text search), Drizzle no estorba
- Migraciones type-safe sin generar cliente pesado

### Por que BullMQ y no un cron simple?
- Reintentos automaticos con backoff exponencial
- Concurrencia controlada (max N jobs simultaneos)
- Dashboard visual para monitorear jobs
- Prioridades: productos populares primero
- Rate limiting por dominio integrado

### Por que Firecrawl y no Playwright directo?
- Firecrawl ya maneja browser pool, crash recovery, stealth mode, y JS rendering
- El worker PrecioChile hace una llamada HTTP a Firecrawl self-hosted y recibe HTML limpio
- Sin dependencias de Playwright en el contenedor de la app (imagen Docker mucho mas pequena)
- Self-hosted = $0 por scrape, igual que Playwright pero con mucho menos codigo a mantener

### Por que CSS Selectors y no LLM extraction?
- Precision: CSS selectors son 100% deterministas para campos estructurados (precio)
- Costo: $0 vs tokens de LLM por cada scrape (incluso con Firecrawl self-hosted)
- Velocidad: parseo instantaneo con cheerio vs overhead de LLM
- Mantenimiento: cuando cambia el HTML, solo actualizas el selector
- LLM queda como fallback futuro para sitios muy dinamicos o sin patron claro

## Estimacion de Recursos

| Metrica | Estimacion |
|---------|------------|
| Productos a monitorear | 5,000 - 50,000 |
| Scrapes por dia | 10,000 - 100,000 |
| Tamano DB (1 ano) | ~2-5 GB |
| RAM en produccion | ~2-3 GB total |
| Disco | ~10-20 GB |
| VPS recomendado | 4GB RAM, 2 vCPU, 40GB SSD ($10-15/mes) |
