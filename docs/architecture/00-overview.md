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
| Scraping | Playwright | Maneja JS-heavy sites (Falabella, Ripley), headless Chrome |
| ORM | Drizzle ORM | Type-safe, migraciones, lightweight, SQL-first |
| Estilos | Tailwind CSS 4 | Utility-first, rapido de prototipar |
| Charts | Recharts | Graficos de historial de precios, basado en React |
| Runtime | Node.js 22 LTS | Soporte nativo de TypeScript (--experimental-strip-types) |
| Containerizacion | Docker + Docker Compose | Reproducible en desarrollo y produccion |

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
GET /api/products/[id] → Precios por tienda + historial
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
| Linio | Marketplace | linio.cl | Si | Baja |
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
- Concurrencia controlada (max N browsers simultaneos)
- Dashboard visual para monitorear jobs
- Prioridades: productos populares primero
- Rate limiting por dominio integrado

### Por que CSS Selectors y no LLM extraction?
- Precision: CSS selectors son 100% deterministas para campos estructurados (precio)
- Costo: $0 vs tokens de OpenAI por cada scrape
- Velocidad: parseo instantaneo vs llamada API
- Mantenimiento: cuando cambia el HTML, solo actualizas el selector (no re-entrenas)
- LLM queda como fallback futuro para sitios muy dinamicos

## Estimacion de Recursos

| Metrica | Estimacion |
|---------|------------|
| Productos a monitorear | 5,000 - 50,000 |
| Scrapes por dia | 10,000 - 100,000 |
| Tamano DB (1 ano) | ~2-5 GB |
| RAM en produccion | ~2-3 GB total |
| Disco | ~10-20 GB |
| VPS recomendado | 4GB RAM, 2 vCPU, 40GB SSD ($10-15/mes) |
