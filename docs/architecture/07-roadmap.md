# 07 - Roadmap de Implementacion

## Vision General de Fases

```
Fase 1 (MVP)          Fase 2 (Core)         Fase 3 (Polish)       Fase 4 (Scale)
2-3 semanas           2-3 semanas           2-3 semanas           Continuo
─────────────         ─────────────         ─────────────         ─────────────
DB + 3 scrapers       +10 scrapers          Alertas               Mas tiendas
API basica            Busqueda full         SEO avanzado          Cache agresivo
Frontend minimo       Historial precios     Performance           Analytics
Cron basico           Filtros/paginacion    Admin dashboard       API publica
```

---

## Fase 1: MVP (Minimum Viable Product)

**Objetivo**: Scraping funcional de 3 tiendas + frontend basico para buscar y comparar.

### Tareas

#### 1.1 Setup del proyecto
- [ ] Inicializar proyecto Next.js 15 con App Router
- [ ] Configurar TypeScript, Tailwind CSS, ESLint
- [ ] Configurar Docker Compose dev (PostgreSQL + Redis)
- [ ] Configurar Drizzle ORM + schema inicial
- [ ] Ejecutar migraciones: crear tablas, indices, extensiones (pg_trgm)
- [ ] Seed data: insertar stores, categorias iniciales

#### 1.2 Sistema de scraping (core)
- [ ] Implementar `BaseScraper` (clase base abstracta)
- [ ] Implementar `BrowserPool` (pool de Playwright)
- [ ] Implementar `ScraperRegistry`
- [ ] Implementar `scrapeProductUrl()` (pipeline principal)
- [ ] Implementar validadores de datos
- [ ] Primer scraper: **PCFactory** (HTML mas simple, ideal para empezar)
- [ ] Segundo scraper: **Falabella** (JS-heavy, validar Playwright)
- [ ] Tercer scraper: **Ripley**
- [ ] Tests manuales: scrapear 10-20 productos por tienda

#### 1.3 Scheduling basico
- [ ] Configurar BullMQ queue + worker
- [ ] Implementar `processStoreScrape()`
- [ ] Configurar node-cron (1 ejecucion diaria para empezar)
- [ ] Implementar registro en `scrape_jobs`
- [ ] Implementar refresh de vistas materializadas post-scraping

#### 1.4 API basica
- [ ] `GET /api/products/search?q=...` (full-text search simple)
- [ ] `GET /api/products/[id]` (detalle + precios por tienda)
- [ ] `GET /api/stores` (lista de tiendas)
- [ ] `GET /api/health` (health check)

#### 1.5 Frontend minimo
- [ ] Layout basico (Header con logo + SearchBar, Footer)
- [ ] Home: barra de busqueda + categorias
- [ ] Pagina de busqueda: grid de productos con precio
- [ ] Pagina de producto: tabla de precios por tienda (sin graficos aun)
- [ ] `PriceDisplay` component (formateo CLP)
- [ ] `ProductCard` component

#### 1.6 Poblado inicial de datos
- [ ] Script para agregar product_urls manualmente (CSV o script)
- [ ] Poblar 50-100 productos de prueba (tecnologia: notebooks, celulares)
- [ ] Ejecutar primer ciclo de scraping completo
- [ ] Verificar datos en DB

**Entregable**: App funcional localmente. Buscar "notebook lenovo" → ver precios en 3 tiendas.

---

## Fase 2: Core Features

**Objetivo**: Scraping de 10+ tiendas, busqueda avanzada, historial de precios.

### Tareas

#### 2.1 Mas scrapers
- [ ] Paris
- [ ] Lider
- [ ] Sodimac
- [ ] Hites
- [ ] AbcDin
- [ ] Easy
- [ ] Mercado Libre
- [ ] SP Digital
- [ ] Microplay

#### 2.2 Busqueda avanzada
- [ ] Filtros: categoria, marca, rango de precio, tienda
- [ ] Ordenamiento: relevancia, precio asc/desc, mayor descuento
- [ ] Paginacion
- [ ] Busqueda fuzzy (typos) con pg_trgm como fallback
- [ ] URL params persistentes (compartir busquedas via URL)

#### 2.3 Historial de precios
- [ ] `GET /api/products/[id]/history` endpoint
- [ ] Integrar Recharts
- [ ] Componente `PriceChart`: linea por tienda, selector de periodo
- [ ] Almacenar historial correctamente (no duplicar entries mismo dia)

#### 2.4 Paginas de categoria y tienda
- [ ] `/categoria/[slug]` - productos por categoria con filtros
- [ ] `/tienda/[slug]` - productos por tienda
- [ ] Breadcrumbs de navegacion

#### 2.5 Ofertas
- [ ] `GET /api/deals` endpoint
- [ ] `/ofertas` pagina con grid de mayores descuentos
- [ ] Badge de descuento en ProductCard

#### 2.6 Caching
- [ ] Implementar cache Redis para queries frecuentes
- [ ] Cache de busqueda (5 min TTL)
- [ ] Cache de detalle de producto (10 min TTL)
- [ ] Invalidacion post-scraping

#### 2.7 Poblado masivo de datos
- [ ] Script/herramienta para descubrir productos en tiendas (crawl de categorias)
- [ ] Poblar 1000-5000 productos
- [ ] Ejecutar scraping diario durante 1+ semana para acumular historial

**Entregable**: Plataforma usable con 10+ tiendas, miles de productos, historial de precios.

---

## Fase 3: Polish

**Objetivo**: Calidad de produccion. SEO, performance, admin, errores.

### Tareas

#### 3.1 SEO
- [ ] `generateMetadata()` dinamico por producto/categoria
- [ ] Open Graph tags (imagen, titulo, descripcion)
- [ ] `sitemap.ts` dinamico
- [ ] Schema.org JSON-LD (`Product` + `Offer`)
- [ ] Canonical URLs
- [ ] robots.txt

#### 3.2 Performance
- [ ] Optimizar queries lentas (EXPLAIN ANALYZE)
- [ ] Lazy loading de imagenes
- [ ] Skeleton loaders para UX percibida
- [ ] Streaming SSR donde aplique
- [ ] Comprimir assets (Next.js lo hace, verificar)

#### 3.3 Admin Dashboard
- [ ] `/admin` pagina protegida
- [ ] Stats: productos totales, tiendas activas, ultimo scraping
- [ ] Lista de jobs de scraping con estado y errores
- [ ] Boton de trigger manual de scraping
- [ ] Log de errores de scraping recientes
- [ ] BullMQ dashboard (bull-board)

#### 3.4 Manejo de errores robusto
- [ ] Detectar cambios de HTML en tiendas (selectores rotos)
- [ ] Alertas cuando una tienda falla consistentemente
- [ ] Paginas de error personalizadas (404, 500)
- [ ] Rate limiting en API publica
- [ ] Input validation con Zod en todos los endpoints

#### 3.5 Deployment produccion
- [ ] Docker Compose produccion
- [ ] Deploy a DigitalOcean
- [ ] Configurar dominio (opcional)
- [ ] Configurar backups automaticos de DB
- [ ] Health check + monitoreo basico
- [ ] HTTPS (Let's Encrypt / Caddy reverse proxy)

**Entregable**: Plataforma lista para usuarios reales. SEO indexable. Admin funcional.

---

## Fase 4: Scale (Continuo)

**Objetivo**: Crecer en datos, features, y usuarios.

### Ideas (priorizar segun uso real)

#### Datos
- [ ] Mas tiendas (Zmart, Corona, Jumbo, Santa Isabel, etc.)
- [ ] Mas categorias (electrohogar, ropa, supermercado)
- [ ] Descubrimiento automatico de productos (crawl de catalogo)
- [ ] Matching automatico de productos cross-tienda (por EAN/SKU/nombre)

#### Features
- [ ] Alertas de precio: "Avisame cuando baje de $X"
- [ ] Favoritos / Watchlist (requiere auth de usuario)
- [ ] Comparador side-by-side de 2-3 productos
- [ ] Extension de browser ("ver precio en PrecioChile")
- [ ] API publica (para que otros devs consuman datos)
- [ ] App movil (React Native o PWA)

#### Tecnico
- [ ] Cache mas agresivo (ISR de Next.js para paginas estaticas)
- [ ] CDN para assets
- [ ] Particionamiento de tabla `prices` por mes
- [ ] Job de agregacion de historial (reducir granularidad datos viejos)
- [ ] Metricas y dashboards (Grafana?)
- [ ] CI/CD pipeline

---

## Estructura de Directorios Final (estimada)

```
preciochile/
├── docker-compose.yml          # Produccion
├── docker-compose.dev.yml      # Desarrollo
├── Dockerfile
├── package.json
├── drizzle.config.ts
├── next.config.ts
├── tailwind.config.ts
├── .env.example
├── docs/
│   └── architecture/           # Estos documentos
├── drizzle/
│   └── migrations/             # Migraciones SQL
├── public/
│   └── stores/                 # Logos de tiendas
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx            # Home
│   │   ├── buscar/
│   │   ├── producto/[slug]/
│   │   ├── categoria/[slug]/
│   │   ├── tienda/[slug]/
│   │   ├── ofertas/
│   │   ├── admin/
│   │   ├── api/
│   │   │   ├── products/
│   │   │   ├── categories/
│   │   │   ├── stores/
│   │   │   ├── deals/
│   │   │   ├── admin/
│   │   │   └── health/
│   │   └── sitemap.ts
│   ├── components/
│   │   ├── layout/
│   │   ├── search/
│   │   ├── product/
│   │   ├── deals/
│   │   ├── ui/
│   │   └── admin/
│   └── lib/
│       ├── db/
│       │   ├── index.ts        # Conexion Drizzle
│       │   ├── schema.ts       # Schema completo
│       │   └── queries/        # Queries reutilizables
│       ├── scraping/
│       │   ├── types.ts
│       │   ├── base-scraper.ts
│       │   ├── registry.ts
│       │   ├── browser-pool.ts
│       │   ├── scrape-product.ts
│       │   ├── validators.ts
│       │   └── scrapers/
│       │       ├── falabella.ts
│       │       ├── ripley.ts
│       │       ├── paris.ts
│       │       └── ...
│       ├── jobs/
│       │   ├── queue.ts
│       │   ├── worker.ts
│       │   └── scheduler.ts
│       ├── services/
│       │   ├── product-service.ts
│       │   ├── store-service.ts
│       │   └── scraping-service.ts
│       ├── cache.ts
│       ├── format.ts
│       └── validators/
└── scripts/
    ├── seed.ts                 # Seed data inicial
    └── discover-products.ts    # Descubrir URLs de productos
```
