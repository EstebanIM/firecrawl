# 07 - Roadmap de Implementacion

## Vision General de Fases

> **Nota de timeline**: Las estimaciones asumen 1 desarrollador, dedicacion parcial.
> Las animaciones GSAP y el polish visual son **Phase 3** — Phase 1 entrega frontend
> funcional con Tailwind + shadcn/ui sin animaciones avanzadas.

```
Fase 1 (MVP)          Fase 2a (Core-A)      Fase 2b (Core-B)      Fase 3 (Polish)       Fase 4 (Scale)
4-5 semanas           2-3 semanas           2-3 semanas           2-3 semanas           Continuo
─────────────         ─────────────         ─────────────         ─────────────         ─────────────
DB + 3 scrapers       +5 scrapers           +4 scrapers           Animaciones GSAP      Mas tiendas
API basica            Busqueda avanzada     Historial precios     SEO avanzado          Cache agresivo
Frontend plano        Filtros/paginacion    Caching Redis         Admin dashboard       Analytics
Cron + BullMQ         Paginas cat/tienda    Ofertas page          Performance audit     API publica
CI/CD basico          Tests de busqueda     Tests de historial    Lighthouse
Logging + Sentry
Deploy VPS basico
```

---

## Fase 1: MVP — 4-5 semanas

**Objetivo**: Scraping funcional de 3 tiendas + frontend funcional (sin animaciones) + primer deploy a VPS.

> **Por que 4-5 semanas**: El scope incluye DB particionada, BrowserPool, BullMQ, 3 scrapers,
> API con cache, frontend con 4+ paginas, CI/CD y deploy. Para 1 dev a tiempo parcial,
> 2-3 semanas es irreal. Mejor estimar correctamente que atrasarse.

### Tareas

#### 1.1 Setup del proyecto
- [ ] Inicializar proyecto Next.js 15 con App Router + `output: 'standalone'`
- [ ] Configurar TypeScript strict, Tailwind CSS 4, ESLint
- [ ] Configurar Docker Compose dev (PostgreSQL 16 + Redis 7)
- [ ] Configurar Drizzle ORM + schema inicial (con particionamiento mensual de `prices`)
- [ ] Ejecutar migraciones: crear tablas, indices, extensiones (`pg_trgm`, `unaccent`)
- [ ] Seed data: insertar stores, categorias iniciales
- [ ] **CI/CD**: GitHub Actions con lint + typecheck + test (configurar desde dia 1)
- [ ] **Logging**: instalar `pino` + `pino-pretty` para desarrollo
- [ ] **Error tracking**: configurar Sentry (DSN en `.env.example`)

#### 1.2 Sistema de scraping (core, via Firecrawl)
- [ ] Configurar `FirecrawlApp` singleton (`FIRECRAWL_API_URL`, `FIRECRAWL_API_KEY`)
- [ ] Implementar `StoreScraper` interface (objeto plano: selectors + parsePrice + delayMs)
- [ ] Implementar helper `parsePriceCLP()` compartido
- [ ] Implementar `ScraperRegistry` (Map scraperKey → StoreScraper)
- [ ] Implementar `scrapeProductUrl()` — Firecrawl call → cheerio → INSERT + UPDATE en transaccion
- [ ] Primer scraper: **PCFactory** (server-rendered, ideal para empezar)
- [ ] Segundo scraper: **Falabella** (SPA, `waitMs: 2000`)
- [ ] Tercer scraper: **Ripley**
- [ ] **Tests**: fixtures con HTML guardado para cada scraper (cheerio parse, sin llamadas a Firecrawl en CI)

#### 1.3 Scheduling basico
- [ ] Configurar BullMQ queue + worker con `concurrency: 3`
- [ ] Implementar `processStoreScrape()` con timeout por URL (45s) y progress tracking
- [ ] Configurar node-cron (2 ejecuciones diarias: 6AM y 6PM Chile)
- [ ] Implementar registro en `scrape_jobs` con overlap guard y deduplicacion por jobId
- [ ] Implementar refresh debounced de vistas materializadas (solo al terminar ultimo job)
- [ ] Worker startup via `src/instrumentation.ts`

#### 1.4 API basica
- [ ] `GET /api/products/search?q=...` (full-text search simple, cache Redis 5min)
- [ ] `GET /api/products/[slug]` (detalle + precios por tienda — usar slug, no id)
- [ ] `GET /api/products/suggest?q=` (autocomplete, trigram, LIMIT 8)
- [ ] `GET /api/stores` (lista de tiendas activas)
- [ ] `GET /api/health` (health check con DB + Redis status)
- [ ] **Tests**: happy path + failure para cada endpoint (no mocks de DB)

#### 1.5 Frontend funcional (sin animaciones GSAP)
- [ ] Configurar shadcn/ui + paleta de colores en `globals.css`
- [ ] Fuentes: Satoshi + General Sans (local, `next/font`)
- [ ] Layout: Header (con `SearchBar` variante header) + Footer
- [ ] Home: `SearchBar` variante hero + categorias estaticas
- [ ] `/buscar`: `ProductGrid` + filtros basicos (por precio, tienda)
- [ ] `/producto/[slug]`: `PriceTable` + `PriceDisplay` (sin grafico aun)
- [ ] `error.tsx` por ruta + `loading.tsx` skeletons
- [ ] `MeshBackground` (gradiente estatico, sin animacion CSS en Phase 1)
- [ ] **Accesibilidad**: focus indicators, semantica HTML, `alt` en imagenes
- [ ] **NOTA**: Las animaciones GSAP y el hero timeline completo son Phase 3

#### 1.6 Poblado inicial de datos
- [ ] Script para agregar `product_urls` manualmente (CSV o script TypeScript)
- [ ] Poblar 50-100 productos de prueba (tecnologia: notebooks, celulares)
- [ ] Ejecutar primer ciclo de scraping completo
- [ ] Verificar datos en DB

#### 1.7 Deploy a VPS (fin de Phase 1)
- [ ] Configurar VPS (Hetzner CX22 o DigitalOcean 4GB)
- [ ] Docker Compose produccion con Caddy (HTTPS automatico)
- [ ] Variables de entorno en produccion (incluido Redis password)
- [ ] Backups automaticos de DB (crontab host, 4 AM)
- [ ] Health check + monitoreo basico (`curl /api/health` cada 5 min)

**Entregable**: App en produccion (VPS). Buscar "notebook lenovo" → ver precios en 3 tiendas. CI/CD verde.

---

## Fase 2a: Core — Scrapers + Busqueda — 2-3 semanas

**Objetivo**: 8 tiendas totales, busqueda avanzada con filtros, paginas de categoria/tienda.

### Tareas

#### 2a.1 Mas scrapers (5 nuevos, total 8)
- [ ] **Paris** (similar a Falabella, JS-heavy)
- [ ] **Lider** (Walmart Chile)
- [ ] **Sodimac** (home improvement, diferente estructura)
- [ ] **Hites** (cadena retail chilena)
- [ ] **AbcDin** (electrodomesticos + tecnologia)
- [ ] Tests de fixture para cada scraper nuevo

#### 2a.2 API: endpoints faltantes
- [ ] `GET /api/brands` (lista de marcas con conteo)
- [ ] `GET /api/categories` (arbol de categorias)
- [ ] `GET /api/products/[slug]/history` (historial de precios por tienda)
- [ ] Tests para cada endpoint nuevo

#### 2a.3 Busqueda avanzada
- [ ] Filtros: categoria, marca, rango de precio, tienda
- [ ] Ordenamiento: relevancia, precio asc/desc, mayor descuento
- [ ] Paginacion con `nuqs` (URL params persistentes)
- [ ] Busqueda fuzzy (pg_trgm como fallback cuando full-text no retorna resultados)
- [ ] Tests de busqueda con multiples combinaciones de filtros

#### 2a.4 Paginas de categoria y tienda
- [ ] `/categoria/[slug]` con filtros y paginacion
- [ ] `/tienda/[slug]` con filtros y paginacion
- [ ] Breadcrumbs de navegacion (RSC)

#### 2a.5 Poblado inicial escalado
- [ ] Script para descubrir `product_urls` en tiendas (crawl de categorias)
- [ ] Poblar 500-1000 productos (tecnologia: notebooks, celulares, tablets)
- [ ] Ejecutar scraping diario

**Entregable**: 8 tiendas activas, busqueda con filtros funcional, 500+ productos.

---

## Fase 2b: Core — Historial + Caching + Ofertas — 2-3 semanas

**Objetivo**: Historial de precios visual, caching Redis, pagina de ofertas, 12 tiendas.

### Tareas

#### 2b.1 Mas scrapers (4 nuevos, total 12)
- [ ] **Easy** (home improvement)
- [ ] **SP Digital** (tecnologia, precio competitivo)
- [ ] **Microplay** (gaming + tecnologia)
- [ ] **Mercado Libre Chile** (marketplace — estructura diferente, requiere estrategia especial)
- [ ] **NOTA**: Linio.cl cerro operaciones en Chile — NO incluir en scrapers

#### 2b.2 Historial de precios visual
- [ ] Integrar Recharts en `PriceChart`
- [ ] Componente `PriceChart`: linea por tienda, selector de periodo (30d/90d/180d/1a)
- [ ] Logica: no duplicar entries si precio no cambia (solo guardar cambios)
- [ ] Tests de historial (que los periodos filtren correctamente)

#### 2b.3 Caching Redis completo
- [ ] Cache de busqueda (5 min TTL)
- [ ] Cache de detalle de producto (10 min TTL)
- [ ] Invalidacion post-scraping (borrar keys afectadas)
- [ ] Cache de `suggest` (1 min TTL, per-query key)

#### 2b.4 Ofertas
- [ ] `GET /api/deals` endpoint (mayores descuentos del dia)
- [ ] `/ofertas` pagina con `DealGrid`
- [ ] Badge de descuento en `ProductCard`

#### 2b.5 Poblado masivo
- [ ] Escalar a 1000-5000 productos en todas las categorias
- [ ] Ejecutar scraping diario durante 1+ semana para acumular historial real

**Entregable**: 12 tiendas activas, historial de precios visual, caching Redis operativo.

---

## Fase 3: Polish — Animaciones + SEO + Admin — 2-3 semanas

**Objetivo**: Calidad de produccion. Animaciones GSAP, SEO completo, admin dashboard, performance.

> Esta fase agrega el polish visual que hace que la plataforma se sienta premium.
> Las animaciones son lo ultimo porque dependen de que el contenido real ya este estable.

### Tareas

#### 3.1 Animaciones GSAP (ver doc 04 para specs completas)
- [ ] Instalar GSAP + `@gsap/react` + ScrollTrigger
- [ ] `gsap-setup.ts` + `presets.ts` (configuracion centralizada)
- [ ] Animation primitives: `ScrollReveal`, `TextReveal`, `StaggerContainer`, `CountUp`, `FadeIn`
- [ ] `HeroSection` con timeline completa (mesh fade-in, nav, titulo, search, logos)
- [ ] `MeshBackground`: activar animacion CSS desktop (estatico en mobile)
- [ ] `ProductCard`: ScrollTrigger.batch() entrance + hover CSS
- [ ] `PriceTable`: row stagger entrance + glow en mejor precio
- [ ] Secciones landing: HowItWorks, FeaturedDeals, CategoryGrid, StatsCounter (CountUp)
- [ ] Audit `prefers-reduced-motion` en todos los componentes animados
- [ ] Test visual en mobile (verificar que mesh estatico no tiene jank)
- [ ] Lighthouse audit: Performance > 85, Accessibility > 95

#### 3.2 SEO
- [ ] `generateMetadata()` dinamico por producto/categoria/tienda
- [ ] Open Graph tags (imagen, titulo, descripcion)
- [ ] `sitemap.ts` dinamico (con todos los productos/categorias)
- [ ] Schema.org JSON-LD (`Product` + `Offer`)
- [ ] Canonical URLs
- [ ] `robots.txt`

#### 3.3 Admin Dashboard
- [ ] `/admin` pagina protegida (API key en header)
- [ ] Stats: productos totales, tiendas activas, ultimo scraping
- [ ] Lista de jobs con estado, duracion, conteo de errores
- [ ] Boton de trigger manual de scraping por tienda
- [ ] Log de errores de scraping recientes (con URL y mensaje)
- [ ] BullMQ dashboard (`@bull-board/nextjs`)

#### 3.4 Manejo de errores + validacion
- [ ] Detectar cambios de HTML en tiendas (tasa de error > 80% → alerta en admin)
- [ ] Input validation con Zod en todos los endpoints (query params, body)
- [ ] Rate limiting en API publica (`@upstash/ratelimit` o middleware custom)
- [ ] Paginas 404 + error personalizadas con branding

**Entregable**: Plataforma con animaciones fluidas, SEO indexable, admin funcional. Lista para usuarios reales.

---

## Fase 4: Scale (Continuo)

**Objetivo**: Crecer en datos, features, y usuarios basado en uso real.

### Ideas de datos
- [ ] Mas tiendas (Zmart, Corona, Jumbo, Santa Isabel, etc.)
- [ ] Mas categorias (electrohogar, ropa, supermercado)
- [ ] Descubrimiento automatico de productos (crawl de catalogo)
- [ ] Matching automatico de productos cross-tienda (por EAN/SKU/nombre similar)

### Ideas de features
- [ ] Alertas de precio: "Avisame cuando baje de $X" (requiere email o push)
- [ ] Favoritos / Watchlist (requiere auth de usuario)
- [ ] Comparador side-by-side de 2-3 productos (`GET /api/products/compare`)
- [ ] Extension de browser ("ver precio en PrecioChile")
- [ ] API publica (para que otros devs consuman datos)
- [ ] Dark mode (next-themes, ver doc 04 — diferido de Phase 1)
- [ ] App movil (PWA primero, React Native si hay demanda)

### Ideas tecnicas
- [ ] Cache mas agresivo (ISR de Next.js para paginas de categoria)
- [ ] CDN para assets estaticos (Cloudflare o similar)
- [ ] Job de agregacion de historial (reducir granularidad de datos >6 meses)
- [ ] Metricas y dashboards (Grafana + Prometheus o similar)
- [ ] Upgrade VPS si el trafico lo justifica
- [ ] **NOTA**: Particionamiento de `prices` es OBLIGATORIO desde Phase 1 (doc 01), no es tarea de Phase 4

---

## Estructura de Directorios Final (estimada)

```
preciochile/
├── docker-compose.yml          # Produccion (con Caddy)
├── docker-compose.dev.yml      # Desarrollo (solo PG + Redis)
├── Dockerfile
├── Caddyfile                   # Reverse proxy + HTTPS automatico
├── package.json
├── drizzle.config.ts
├── next.config.ts              # output: 'standalone' (requerido)
├── tailwind.config.ts
├── .env.example
├── docs/
│   └── architecture/           # Estos documentos
├── drizzle/
│   └── migrations/             # Migraciones SQL generadas por Drizzle
├── public/
│   ├── stores/                 # Logos de tiendas (SVG/PNG)
│   └── noise.png               # Textura de ruido 200x200 ~5KB
├── src/
│   ├── instrumentation.ts      # Startup hook: startWorker() + startScheduler()
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx            # Home (RSC)
│   │   ├── error.tsx           # Global error boundary
│   │   ├── not-found.tsx
│   │   ├── buscar/             # page + loading + error
│   │   ├── producto/[slug]/    # page + loading + error
│   │   ├── categoria/[slug]/   # page + error
│   │   ├── tienda/[slug]/      # page + error
│   │   ├── ofertas/            # page + error
│   │   ├── admin/              # layout + page + error
│   │   ├── api/
│   │   │   ├── products/       # search, [slug], [slug]/history, suggest, compare
│   │   │   ├── brands/
│   │   │   ├── categories/
│   │   │   ├── stores/
│   │   │   ├── deals/
│   │   │   ├── admin/          # scrape trigger (API key protected)
│   │   │   └── health/
│   │   └── sitemap.ts
│   ├── components/
│   │   ├── layout/             # Header, HeaderClient, Footer, Sidebar
│   │   │                       # MeshBackground, NoiseOverlay (NO PageTransition)
│   │   ├── landing/            # HeroSection, HowItWorks, FeaturedDeals, etc.
│   │   ├── search/             # SearchBar (variant hero|header), SearchFilters
│   │   ├── product/            # ProductCard, ProductGrid, PriceTable, PriceChart
│   │   ├── animation/          # ScrollReveal, TextReveal, StaggerContainer (Phase 3)
│   │   ├── ui/                 # shadcn/ui generados
│   │   └── admin/
│   └── lib/
│       ├── db/
│       │   ├── index.ts        # Conexion Drizzle + connection pool
│       │   ├── schema.ts       # Schema completo (prices particionado)
│       │   └── queries/        # ProductService, StoreService, etc.
│       ├── scraping/
│       │   ├── types.ts
│       │   ├── base-scraper.ts
│       │   ├── registry.ts
│       │   ├── browser-pool.ts # Semaforo real, crash recovery
│       │   ├── scrape-product.ts
│       │   ├── validators.ts
│       │   └── scrapers/
│       │       ├── pcfactory.ts
│       │       ├── falabella.ts
│       │       ├── ripley.ts
│       │       └── ...
│       ├── jobs/
│       │   ├── queue.ts
│       │   ├── worker.ts       # processStoreScrape con timeout + debounced refresh
│       │   └── scheduler.ts    # overlap guard + deduplicacion por jobId
│       ├── animation/
│       │   ├── gsap-setup.ts   # registerPlugin, defaults (Phase 3)
│       │   └── presets.ts      # ANIM presets (Phase 3)
│       ├── cache.ts            # Redis helpers con TTL
│       ├── format.ts           # formatPrice(), discountPct()
│       └── cn.ts               # cn() = clsx + twMerge
└── scripts/
    ├── start.sh                # Startup: migraciones → node server.js
    ├── seed.ts                 # Seed data inicial (stores, categorias)
    └── discover-products.ts    # Crawl de categorias para descubrir URLs
```
