# 04 - Frontend (BORRADOR - Modificar antes de implementar)

> **NOTA**: Este documento es un borrador inicial. El usuario revisara y ajustara
> el diseno del frontend antes de la implementacion.

## Overview

Frontend con Next.js 15 App Router. Server-Side Rendering para SEO.
Tailwind CSS para estilos. Recharts para graficos de historial de precios.

## Paginas

### 1. Home (`/`)
- Hero con barra de busqueda prominente
- Categorias populares (grid de iconos)
- Ofertas destacadas del dia (carousel o grid)
- Productos mas buscados
- Footer con links a categorias y tiendas

### 2. Busqueda (`/buscar?q=...`)
- Barra de busqueda (sticky top)
- Filtros laterales: categoria, marca, rango de precio, tienda
- Grid de productos con:
  - Imagen, nombre, mejor precio, tienda del mejor precio
  - Badge de descuento (si aplica)
  - Cantidad de tiendas donde esta disponible
- Ordenar por: relevancia, precio menor, precio mayor, mayor descuento
- Paginacion

### 3. Producto (`/producto/[slug]`)
- Imagen del producto
- Nombre, marca, categoria (breadcrumb)
- **Tabla comparativa de precios** por tienda:
  - Logo tienda | Precio normal | Precio oferta | Disponibilidad | Link "Ir a tienda"
  - Ordenada por precio menor
- **Grafico de historial de precios** (Recharts):
  - Linea por tienda (colores distintos)
  - Selector de periodo: 30d, 90d, 180d, 1 ano
  - Tooltip con precio y fecha
- Especificaciones/metadata del producto
- Productos similares

### 4. Categoria (`/categoria/[slug]`)
- Breadcrumb de categoria padre/hijo
- Grid de productos de esa categoria
- Filtros: marca, rango de precio, tienda
- Paginacion

### 5. Tienda (`/tienda/[slug]`)
- Info de la tienda (logo, nombre, URL)
- Stats: productos monitoreados, ultima actualizacion
- Grid de productos de esa tienda
- Filtros y paginacion

### 6. Ofertas (`/ofertas`)
- Grid de productos con mayor descuento
- Filtros: categoria, porcentaje minimo de descuento
- Ordenar por: mayor descuento, menor precio

### 7. Admin (`/admin`) - Protegido
- Dashboard: stats generales (productos, tiendas, ultimo scraping)
- Lista de jobs de scraping con estado
- Boton "Ejecutar scraping" (por tienda o todas)
- Log de errores recientes

## Componentes Clave

```
src/components/
├── layout/
│   ├── Header.tsx          # Logo + SearchBar + nav
│   ├── Footer.tsx          # Links, categorias, about
│   └── Sidebar.tsx         # Filtros laterales (busqueda)
├── search/
│   ├── SearchBar.tsx       # Input con autocomplete
│   ├── SearchFilters.tsx   # Filtros: categoria, marca, precio
│   └── SearchResults.tsx   # Grid de resultados
├── product/
│   ├── ProductCard.tsx     # Card en grid (imagen, nombre, precio)
│   ├── PriceTable.tsx      # Tabla comparativa de precios por tienda
│   ├── PriceChart.tsx      # Grafico historial (Recharts)
│   ├── ProductSpecs.tsx    # Tabla de especificaciones
│   └── SimilarProducts.tsx # Grid de productos similares
├── deals/
│   ├── DealCard.tsx        # Card con badge de descuento
│   └── DealGrid.tsx        # Grid de ofertas
├── ui/
│   ├── Badge.tsx           # Badge de descuento (-30%)
│   ├── Pagination.tsx      # Controles de paginacion
│   ├── PriceDisplay.tsx    # Formateo de precios CLP ($299.990)
│   ├── StoreLogo.tsx       # Logo de tienda con fallback
│   └── Skeleton.tsx        # Loading skeletons
└── admin/
    ├── JobsList.tsx         # Tabla de jobs de scraping
    ├── StatsCards.tsx       # Cards con metricas
    └── ScrapeButton.tsx     # Trigger manual
```

## Librerias UI

| Libreria | Uso |
|----------|-----|
| Tailwind CSS 4 | Estilos utility-first |
| Recharts | Graficos de historial de precios |
| Lucide React | Iconos |
| clsx / tailwind-merge | Clases condicionales |
| nuqs | URL search params type-safe (filtros/paginacion) |

## SEO

- Server-Side Rendering en todas las paginas publicas
- `generateMetadata()` dinamico por producto/categoria
- Open Graph tags para compartir en redes sociales
- Sitemap generado con `sitemap.ts`
- Schema.org markup para productos (JSON-LD `Product` con `Offer`)
- URLs limpias con slugs (`/producto/notebook-lenovo-ideapad-3`)

## Formateo de Precios

```typescript
// src/lib/format.ts

/** Formatea precio CLP: 299990 → "$299.990" */
function formatPrice(price: number): string {
  return '$' + price.toLocaleString('es-CL');
}

/** Calcula porcentaje de descuento */
function discountPct(normalPrice: number, offerPrice: number): number {
  return Math.round(100 * (normalPrice - offerPrice) / normalPrice);
}
```

## Responsive Design

- Mobile-first approach
- Breakpoints: sm (640px), md (768px), lg (1024px), xl (1280px)
- Grid de productos: 1 col (mobile), 2 col (sm), 3 col (md), 4 col (lg)
- Filtros: drawer lateral en mobile, sidebar en desktop
- Tabla de precios: scroll horizontal en mobile

## Estructura de Archivos

```
src/app/
├── layout.tsx              # Root layout (Header + Footer)
├── page.tsx                # Home
├── buscar/
│   └── page.tsx            # Busqueda con filtros
├── producto/
│   └── [slug]/
│       └── page.tsx        # Detalle de producto
├── categoria/
│   └── [slug]/
│       └── page.tsx        # Productos por categoria
├── tienda/
│   └── [slug]/
│       └── page.tsx        # Productos por tienda
├── ofertas/
│   └── page.tsx            # Mejores ofertas
├── admin/
│   ├── layout.tsx          # Admin layout (protegido)
│   └── page.tsx            # Dashboard admin
└── sitemap.ts              # Sitemap dinamico
```
