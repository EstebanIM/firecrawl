# 04 - Frontend: Diseno y Animaciones

## Direccion Estetica: "Precision Commerce"

Inspirada en la geografia chilena — el contraste de los Andes contra cielos limpios,
el azul profundo del Pacifico, los tonos calidos del Atacama. La plataforma debe sentirse
como un instrumento financiero premium: preciso, confiable, con momentos de sofisticacion
a traves del movimiento. No infantil, no corporativo — **autoritativo y accesible**.

---

## Stack Frontend

| Tecnologia | Rol | Peso |
|------------|-----|------|
| Next.js 15 (App Router) | SSR + RSC + API routes | Framework |
| Tailwind CSS 4 | Estilos utility-first | ~10KB gzip |
| shadcn/ui | Componentes base (Button, Card, Input, etc.) | Solo lo usado |
| GSAP + ScrollTrigger | Animaciones de entrada, scroll, timeline | ~30KB gzip |
| @gsap/react (useGSAP) | Hook para GSAP en React (cleanup automatico) | ~2KB |
| Recharts | Graficos de historial de precios | ~40KB gzip |
| Lucide React | Iconos | Tree-shaken |
| nuqs | URL search params type-safe | ~3KB |

---

## Paleta de Colores

Definida como CSS custom properties mapeadas a tokens semanticos de shadcn/ui.

### Light Mode

```css
--background: #FAFAF8;          /* warm off-white, no clinico */
--foreground: #1A1A1A;          /* near-black, mas suave que negro puro */
--primary: #1D4ED8;             /* sapphire blue — confianza, finanzas */
--primary-foreground: #FFFFFF;
--secondary: #F4F1EC;           /* warm stone */
--secondary-foreground: #44403C;
--muted: #F0EDE8;               /* light stone */
--muted-foreground: #78716C;
--accent: #10B981;              /* emerald — bajadas de precio, positivo */
--destructive: #EF4444;         /* subidas de precio, negativo */
--card: #FFFFFF;
--border: #E7E5E0;              /* warm gray */
```

### Dark Mode

> **DECISION: Dark mode DIFERIDO a Phase 2.**
> Las CSS variables de dark mode estan definidas abajo como referencia,
> pero en Phase 1 NO se implementa toggle de tema, ni se instala `next-themes`.
> La app siempre renderiza en light mode. Quitar el bloque `.dark` de globals.css
> hasta que se decida implementar. Evita complejidad innecesaria en MVP.

```css
/* RESERVADO PARA PHASE 2 — no activar en Phase 1 */
.dark {
  --background: #0C0C0C;
  --foreground: #EAEAEA;
  --primary: #3B82F6;
  --secondary: #1C1C1C;
  --card: #141414;
  --border: #262626;
}
```

### Gradient Accent (uso limitado: hero, highlights)

```css
--gradient-accent: linear-gradient(135deg, #1D4ED8 0%, #7C3AED 50%, #EC4899 100%);
```

### Contraste WCAG AA

| Par | Ratio | Cumple |
|-----|-------|--------|
| foreground / background | 15.3:1 | AA + AAA |
| primary-fg / primary | 7.1:1 | AA + AAA |
| muted-fg / background | 4.8:1 | AA (texto grande) |

---

## Tipografia

Fuentes distintivas, NO genericas (nada de Inter, Roboto, Arial, system-ui).

| Rol | Fuente | Fuente | Pesos |
|-----|--------|--------|-------|
| Headings | **Satoshi** | Fontshare (geometric, modern, free commercial) | 500, 700 |
| Body | **General Sans** | Fontshare (grotesque limpio, legible) | 400, 500 |
| Precios | **JetBrains Mono** | Google Fonts (monospace, sensacion financiera) | 400, 500 |

### Carga de fuentes

```typescript
// src/lib/fonts.ts
import localFont from 'next/font/local';

export const satoshi = localFont({
  src: [
    { path: '../fonts/Satoshi-Medium.woff2', weight: '500' },
    { path: '../fonts/Satoshi-Bold.woff2', weight: '700' },
  ],
  variable: '--font-heading',
  display: 'swap',
});

export const generalSans = localFont({
  src: [
    { path: '../fonts/GeneralSans-Regular.woff2', weight: '400' },
    { path: '../fonts/GeneralSans-Medium.woff2', weight: '500' },
  ],
  variable: '--font-body',
  display: 'swap',
});
```

### Display de precios

Los precios usan monospace + `font-variant-numeric: tabular-nums` para alineacion
vertical en tablas. Sensacion de "ticker financiero":

```css
.price {
  font-family: var(--font-mono); /* JetBrains Mono */
  font-variant-numeric: tabular-nums;
  letter-spacing: -0.02em;
}
```

---

## Background Animado

### Mesh Gradient (CSS puro, zero JS)

3 radial-gradients con blur, animados con keyframes CSS. Costo de CPU: casi cero.

> **MOBILE**: La animacion `mesh-drift` se DESACTIVA en mobile para evitar jank
> en GPUs de gama baja. Solo se muestra el gradiente estatico.

```css
.mesh-gradient {
  position: fixed;
  inset: 0;
  z-index: -1;
  opacity: 0.4;
  background:
    radial-gradient(ellipse at 20% 50%, hsl(var(--primary) / 0.15) 0%, transparent 50%),
    radial-gradient(ellipse at 80% 20%, hsl(270 60% 50% / 0.1) 0%, transparent 50%),
    radial-gradient(ellipse at 50% 80%, hsl(330 70% 50% / 0.08) 0%, transparent 50%);
  filter: blur(80px);
  /* NO se anima en mobile — ver @media abajo */
}

/* Solo animar en desktop donde el GPU puede manejarlo sin jank */
@media (min-width: 1024px) {
  .mesh-gradient {
    will-change: transform;
    animation: mesh-drift 20s ease-in-out infinite alternate;
  }
}

@keyframes mesh-drift {
  0%   { transform: translate(0, 0) scale(1); }
  33%  { transform: translate(2%, -1%) scale(1.02); }
  66%  { transform: translate(-1%, 2%) scale(0.98); }
  100% { transform: translate(1%, -2%) scale(1.01); }
}
```

### Grain/Noise Overlay

Textura de ruido que rompe la sensacion "digital perfecta" y agrega caracter organico.

> **FIX**: z-index DEBE ser bajo (1, no 9999). Con z-index: 9999, el grano se
> renderiza ENCIMA de Dialogs, Sheets, Tooltips, Dropdowns de shadcn/ui (portales Radix).
> Se aplica al content wrapper, no a `body::after`, para evitar conflictos con portales.

```css
/* Aplicar como pseudo-element del main content wrapper, NO de body */
.main-content::after {
  content: '';
  position: fixed;
  inset: 0;
  z-index: 1;  /* BAJO: entre mesh (-1) y contenido. Portales Radix usan z-index mayores */
  pointer-events: none;
  opacity: 0.03;
  background: url('/noise.png') repeat; /* PNG 200x200, ~5KB */
}
```

### Donde aparece

| Pagina | Mesh opacity | Grain |
|--------|-------------|-------|
| Home (hero) | 0.4 | Si |
| Busqueda | 0.15 | Si |
| Producto | 0.2 | Si |
| Admin | 0 (sin mesh) | No |

### Parallax con mouse (opcional, solo hero desktop)

```typescript
// Solo si (min-width: 1024px) y no es touch device
// GSAP quickTo() para seguimiento suave del mouse
const xTo = gsap.quickTo(".mesh-gradient", "x", { duration: 0.8, ease: "power3" });
const yTo = gsap.quickTo(".mesh-gradient", "y", { duration: 0.8, ease: "power3" });

window.addEventListener("mousemove", (e) => {
  xTo((e.clientX / window.innerWidth - 0.5) * 30);
  yTo((e.clientY / window.innerHeight - 0.5) * 30);
});
```

---

## Landing Page: Coreografia de Animaciones

### Hero Section (viewport completo, centrado)

Estado inicial: todos los elementos invisibles (`[data-gsap-reveal] { opacity: 0 }` en CSS).

**Timeline GSAP:**

```
t=0ms     Mesh gradient fade in (opacity 0→0.4, 1.2s ease)
t=200ms   Navegacion slide down (y:-20→0, opacity 0→1, 0.6s)
t=400ms   Titulo linea 1: clip-reveal desde abajo (yPercent:100→0, 0.8s, power3.out)
t=600ms   Titulo linea 2: clip-reveal desde abajo (0.8s, power3.out)
t=900ms   Subtitulo: fade + slide up (y:20→0, opacity 0→1, 0.6s)
t=1100ms  SearchBar: scale-in (scaleX:0.9→1, opacity 0→1, 0.5s, back.out)
t=1300ms  Logos de tiendas: stagger desde abajo (0.05s entre cada uno, 0.4s)
t=1500ms  Indicador de scroll: fade in
```

**Tecnica de clip-reveal para texto:**

```html
<!-- Wrapper con overflow:hidden, span interna animada -->
<div class="overflow-hidden">
  <span data-gsap-reveal class="inline-block">Compara precios</span>
</div>
```

```typescript
// GSAP anima yPercent del span interno
gsap.from("[data-gsap-reveal]", { yPercent: 100, duration: 0.8, ease: "power3.out" });
```

### Secciones con ScrollTrigger (below the fold)

**Seccion 1: "Como funciona" (3 pasos)**
- Trigger: section enters 20% del viewport
- Cards stagger con `ScrollTrigger.batch()`, 150ms delay entre cada una
- Cada card: `opacity:0→1, y:40→0, duration:0.7s`
- Linea conectora SVG se dibuja (strokeDashoffset animado)

**Seccion 2: "Ofertas destacadas"**
- Titulo con clip-reveal (misma tecnica que hero)
- Deal cards con `ScrollTrigger.batch()`, stagger 0.1s
- Cards: `opacity:0→1, y:30→0, scale:0.97→1`

**Seccion 3: "Categorias populares"**
- Grid de iconos/cards con batch stagger
- Hover: CSS gradient shift (sin GSAP)

**Seccion 4: Stats (numeros contadores)**
- Numeros se animan contando (GSAP snap en textContent)
- "12,500+ productos", "14 tiendas", "50,000+ precios"
- `ScrollTrigger: { once: true }` — solo una vez

**Seccion 5: CTA + Footer**
- CTA fade in con leve parallax

### Implementacion del Hero

```typescript
// src/components/landing/HeroSection.tsx
"use client";

import { useRef } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger, useGSAP);

export function HeroSection() {
  const containerRef = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    const mm = gsap.matchMedia();

    mm.add({
      isDesktop: "(min-width: 1024px)",
      isMobile: "(max-width: 1023px)",
      prefersReducedMotion: "(prefers-reduced-motion: reduce)",
    }, (context) => {
      const { prefersReducedMotion, isMobile } = context.conditions!;

      if (prefersReducedMotion) {
        gsap.set("[data-gsap-reveal]", { opacity: 1, y: 0, scale: 1 });
        return;
      }

      // NOTA: Usar data-anim attributes en vez de clases (.nav, .hero-line-1)
      // para evitar colisiones con Tailwind/CSS modules y hacer selectores robustos
      const tl = gsap.timeline({ defaults: { ease: "power2.out" } });
      const distance = isMobile ? 16 : 24;

      tl.to("[data-anim='mesh']", { opacity: 0.4, duration: 1.2 })
        .from("[data-anim='nav']", { y: -20, opacity: 0, duration: 0.6 }, 0.2)
        .from("[data-anim='hero-line-1'] [data-gsap-reveal]", { yPercent: 100, duration: 0.8, ease: "power3.out" }, 0.4)
        .from("[data-anim='hero-line-2'] [data-gsap-reveal]", { yPercent: 100, duration: 0.8, ease: "power3.out" }, 0.6)
        .from("[data-anim='subtitle']", { y: distance, opacity: 0, duration: 0.6 }, 0.9)
        .from("[data-anim='search']", { scaleX: 0.9, opacity: 0, duration: 0.5, ease: "back.out(1.4)" }, 1.1)
        .from("[data-anim='store-logo']", { y: 16, opacity: 0, stagger: 0.05, duration: 0.4 }, 1.3)
        .from("[data-anim='scroll-cta']", { opacity: 0, duration: 0.4 }, 1.5);
    });
  }, { scope: containerRef });

  return <div ref={containerRef}>{/* ... JSX del hero ... */}</div>;
}
```

---

## Especificaciones de Componentes Animados

### ProductCard

**Entrance** (en grids, via ScrollTrigger.batch):
```
opacity:0, y:24 → opacity:1, y:0 | duration: 0.5s | ease: power2.out | stagger: 0.08s
```

**Hover** (CSS transitions, NO GSAP):
```css
.product-card {
  transition: transform 200ms ease, box-shadow 200ms ease;
}
.product-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}
.product-card:hover .product-image {
  transform: scale(1.03); /* overflow-hidden en container */
  transition: transform 400ms ease;
}
.product-card:hover .view-link::after {
  transform: scaleX(1); /* underline expands from left */
}
```

**Discount badge**: En entrance, scale bounce (0→1.1→1) con 200ms delay despues de la card.

### PriceTable

**Row stagger** (en page load, no scroll):
```
Header: opacity:0→1, 0.3s
Rows: x:-16→0, opacity:0→1 | stagger: 0.06s entre filas
Mejor precio: glow pulse en box-shadow despues del stagger (0.4s)
```

### PriceChart (Recharts)

```typescript
<Line
  animationDuration={1500}
  animationEasing="ease-in-out"
  animationBegin={index * 300} // stagger por linea (tienda)
/>
```

Cambio de periodo (30d/90d/180d):
- Chart wrapper: `opacity:0.3` (200ms CSS transition)
- Data se actualiza
- Chart wrapper: `opacity:1` (200ms)

### SearchBar

> **Un solo componente con variantes** (no dos archivos separados).
> `SearchBar.tsx` y `SearchBarHero.tsx` comparten logica de autocomplete,
> keyboard navigation, y llamada a API. Usar `cva()` para diferencias de tamano.

```typescript
// src/components/search/SearchBar.tsx
interface SearchBarProps {
  variant: "hero" | "header";  // controla tamano y animacion expand
}
```

**Focus** (CSS + GSAP menor):
- Border: transition a `primary` (150ms CSS)
- Box-shadow glow (CSS transition)
- Hero variant: width expande (GSAP 0.3s)
- Autocomplete dropdown: `y:-8→0, opacity:0→1` (GSAP 0.2s)
- Resultados autocomplete: stagger 0.03s
- **Debounce: 250ms** en cada keystroke antes de llamar `/api/products/suggest`

| Variante | Tamano | Animacion expand |
|----------|--------|-----------------|
| Hero (landing) | h-14, text-lg | Si |
| Header (sticky) | h-10, text-sm | No |

### Buttons

**Hover** (CSS only, 150ms ease):
- Primary: background darkens, translateY(-1px), shadow appears
- Ghost: background → secondary

**Click** (CSS only, 50ms):
- `transform: scale(0.97)` en `:active`

**CTA buttons** (hero, acciones importantes):
- Pseudo-element con `--gradient-accent` como border
- Hover: gradient background-position shift (CSS transition)

### Transiciones de Pagina

> **ELIMINADO: PageTransition wrapper con usePathname()**.
> Conflicta con App Router streaming/Suspense — genera double-flash
> (contenido viejo al 60% → skeleton al 60% → contenido nuevo).
>
> **Estrategia actual**: Cada pagina maneja su propia animacion de entrada
> usando GSAP (via `FadeIn` o `ScrollReveal` wrappers). Esto se coordina
> naturalmente con Suspense boundaries y loading.tsx skeletons.
>
> El archivo `PageTransition.tsx` **no existe** en la estructura final.

---

## Componentes de Animacion Reutilizables

### ScrollReveal

Wrapper generico que aplica animacion scroll-triggered a cualquier children.

```typescript
// src/components/animation/ScrollReveal.tsx
"use client";

interface ScrollRevealProps {
  children: React.ReactNode;
  direction?: "up" | "left" | "right";   // default: "up"
  delay?: number;                         // seconds
  className?: string;
}

// Usa useGSAP con scope, aplica ScrollTrigger
// Maneja reduced-motion con gsap.set() al estado final
```

### TextReveal

Animacion signature de clip para titulos.

```typescript
// src/components/animation/TextReveal.tsx
"use client";

// Envuelve cada linea en <span overflow-hidden><span data-gsap-reveal>texto</span></span>
// Anima spans internos con yPercent: 100→0
// Usado en hero headings y titulos de seccion
```

### StaggerContainer

Wrapper que aplica ScrollTrigger.batch() a hijos directos.

```typescript
// src/components/animation/StaggerContainer.tsx
"use client";

interface StaggerContainerProps {
  children: React.ReactNode;
  stagger?: number;          // default: 0.08
  className?: string;
}

// Usa ScrollTrigger.batch() en children con [data-stagger-item]
// Componente clave para TODOS los product grids, deal grids, category grids
```

### CountUp

Contador numerico animado.

```typescript
// src/components/animation/CountUp.tsx
"use client";

interface CountUpProps {
  end: number;
  duration?: number;
  suffix?: string;  // "+" para "12,500+"
  prefix?: string;  // "$" para precios
}

// GSAP tween en un objeto { value: 0 } → { value: end }
// onUpdate formatea y escribe en el DOM
// ScrollTrigger { once: true }
```

### FadeIn

Fade + slide simple.

```typescript
// src/components/animation/FadeIn.tsx
"use client";

// El mas simple: opacity:0→1, y:24→0
// Wrapper para elementos individuales que no necesitan batch
```

---

## Utilidades de Animacion

### gsap-setup.ts

```typescript
// src/lib/animation/gsap-setup.ts
"use client";

import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger, useGSAP);

gsap.defaults({
  ease: "power2.out",
  duration: 0.6,
});

export { gsap, ScrollTrigger, useGSAP };
```

### presets.ts

```typescript
// src/lib/animation/presets.ts

export const ANIM = {
  // Presets de animacion
  fadeUp:        { opacity: 0, y: 24, duration: 0.6, ease: "power2.out" },
  fadeUpMobile:  { opacity: 0, y: 16, duration: 0.5, ease: "power2.out" },
  clipReveal:    { yPercent: 100, duration: 0.8, ease: "power3.out" },
  scaleIn:       { opacity: 0, scale: 0.95, duration: 0.5, ease: "back.out(1.4)" },

  // Stagger amounts
  stagger: {
    grid: 0.08,    // product grid items
    list: 0.06,    // table rows
    fast: 0.03,    // autocomplete results
    logos: 0.05,   // store logos
  },

  // ScrollTrigger configs
  trigger: {
    default: { start: "top 85%", once: true },
    eager:   { start: "top 95%", once: true },
  },
} as const;
```

---

## Anti-Flash: CSS pre-hydration

Para evitar que los elementos aparezcan brevemente antes de que GSAP los anime:

```css
/* globals.css */
[data-gsap-reveal] {
  opacity: 0;
}
```

GSAP usa `fromTo()` o `to()` para llevarlos a `opacity: 1` durante la animacion.
Elementos sin animacion (admin, contenido estatico) NO usan `data-gsap-reveal`.

> **IMPORTANTE: Los `Skeleton` components (loading states) JAMAS usan `data-gsap-reveal`.**
> Si un skeleton tuviera `opacity: 0`, seria invisible durante la carga — exactamente
> lo contrario de su proposito. Los skeletons aparecen inmediatamente y se reemplazan
> con el contenido real una vez cargado; ese contenido real SÍ puede usar `data-gsap-reveal`.

---

## Paginas

### 1. Home (`/`)
- Hero: titulo animado + SearchBarHero + logos tiendas
- Seccion "Como funciona" (3 pasos con SVG connector)
- Ofertas destacadas (DealGrid con StaggerContainer)
- Categorias populares (grid con StaggerContainer)
- Stats contadores (CountUp)
- CTA final

### 2. Busqueda (`/buscar?q=...`)
- SearchBar sticky en header
- Filtros laterales: categoria, marca, rango precio, tienda (Sidebar)
- ProductGrid con StaggerContainer (ScrollTrigger.batch)
- Ordenar: relevancia, precio asc/desc, mayor descuento
- Paginacion

### 3. Producto (`/producto/[slug]`)
- Breadcrumb (RSC)
- Imagen + nombre + marca
- PriceTable con row stagger animation
- PriceChart (Recharts con animacion por linea)
- Tabs: periodo 30d/90d/180d/1a
- ProductSpecs (RSC, sin animacion)
- SimilarProducts (ProductGrid reutilizado)

### 4. Categoria (`/categoria/[slug]`)
- Titulo con TextReveal
- ProductGrid con StaggerContainer
- Filtros y paginacion

### 5. Tienda (`/tienda/[slug]`)
- Info tienda (logo, nombre, stats)
- ProductGrid
- Filtros y paginacion

### 6. Ofertas (`/ofertas`)
- Titulo con TextReveal
- DealGrid con StaggerContainer (badge de descuento animado)
- Filtros: categoria, % descuento minimo

### 7. Admin (`/admin`) - Protegido
- Sin animaciones fancy (funcionalidad pura)
- Dashboard: StatsCards, JobsList, ScrapeButton
- shadcn/ui components directos

---

## Componentes: Estructura de Archivos

```
src/
├── app/
│   ├── layout.tsx                    # RootLayout: fonts, theme, MeshBackground
│   ├── page.tsx                      # Home (RSC, pasa data a client sections)
│   ├── loading.tsx                   # Global loading skeleton
│   ├── error.tsx                     # Global error boundary (client component)
│   ├── not-found.tsx                 # 404 personalizado
│   ├── buscar/
│   │   ├── page.tsx
│   │   ├── loading.tsx               # Skeleton de SearchResults
│   │   └── error.tsx                 # Error en busqueda (ej: DB caida)
│   ├── producto/[slug]/
│   │   ├── page.tsx
│   │   ├── loading.tsx               # Skeleton de PriceTable + PriceChart
│   │   └── error.tsx                 # Producto no encontrado / API error
│   ├── categoria/[slug]/
│   │   ├── page.tsx
│   │   └── error.tsx
│   ├── tienda/[slug]/
│   │   ├── page.tsx
│   │   └── error.tsx
│   ├── ofertas/
│   │   ├── page.tsx
│   │   └── error.tsx
│   ├── admin/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── error.tsx
│   └── sitemap.ts
│
├── components/
│   ├── layout/
│   │   ├── Header.tsx                # RSC shell
│   │   ├── HeaderClient.tsx          # "use client" — sticky, mobile menu
│   │   ├── Footer.tsx                # RSC
│   │   ├── Sidebar.tsx               # Filtros (busqueda)
│   │   ├── MeshBackground.tsx        # "use client" — mesh CSS + optional mouse parallax
│   │   └── NoiseOverlay.tsx          # Grain texture (CSS puro)
│   │   # NOTA: PageTransition.tsx ELIMINADO — ver seccion "Transiciones de Pagina"
│   │
│   ├── landing/                      # Secciones del Home (todas "use client")
│   │   ├── HeroSection.tsx           # Timeline GSAP completa
│   │   ├── HowItWorks.tsx            # 3 pasos + SVG connector
│   │   ├── FeaturedDeals.tsx         # Deal cards con ScrollTrigger
│   │   ├── CategoryGrid.tsx          # Category cards con ScrollTrigger
│   │   ├── StatsCounter.tsx          # Numeros contadores
│   │   └── CTASection.tsx            # Call-to-action final
│   │
│   ├── search/
│   │   ├── SearchBar.tsx             # "use client" — variante hero|header, autocomplete, debounce 250ms
│   │   ├── SearchFilters.tsx         # "use client" — filtros interactivos
│   │   └── SearchResults.tsx         # "use client" — grid con batch animations
│   │   # NOTA: SearchBarHero.tsx ELIMINADO — fusionado en SearchBar con variant prop
│   │
│   ├── product/
│   │   ├── ProductCard.tsx           # "use client" — hover CSS effects
│   │   ├── ProductGrid.tsx           # "use client" — StaggerContainer wrapper
│   │   ├── PriceTable.tsx            # "use client" — row stagger animation
│   │   ├── PriceChart.tsx            # "use client" — Recharts wrapper
│   │   ├── PriceDisplay.tsx          # RSC-safe — formateo monospace
│   │   ├── PriceBadge.tsx            # Badge descuento con entrance anim
│   │   ├── ProductSpecs.tsx          # RSC
│   │   └── SimilarProducts.tsx       # "use client" — usa ProductGrid
│   │
│   ├── deals/
│   │   ├── DealCard.tsx              # "use client" — extiende ProductCard
│   │   └── DealGrid.tsx              # "use client" — usa ProductGrid pattern
│   │
│   ├── animation/                    # Primitivos de animacion reutilizables
│   │   ├── ScrollReveal.tsx          # Wrapper scroll-triggered generico
│   │   ├── TextReveal.tsx            # Clip-path text entrance
│   │   ├── StaggerContainer.tsx      # ScrollTrigger.batch() para grids
│   │   ├── CountUp.tsx               # Contador numerico
│   │   └── FadeIn.tsx                # Fade + slide simple
│   │
│   ├── ui/                           # shadcn/ui (generados con CLI)
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── input.tsx
│   │   ├── badge.tsx
│   │   ├── skeleton.tsx
│   │   ├── separator.tsx
│   │   ├── select.tsx
│   │   ├── dialog.tsx
│   │   ├── sheet.tsx
│   │   ├── tabs.tsx
│   │   ├── tooltip.tsx
│   │   └── pagination.tsx
│   │
│   └── admin/
│       ├── JobsList.tsx
│       ├── StatsCards.tsx
│       └── ScrapeButton.tsx
│
├── hooks/
│   ├── useScrollProgress.ts          # Retorna 0-1 progreso de scroll
│   ├── usePrefersReducedMotion.ts    # Boolean hook
│   └── useMediaQuery.ts             # Media query generico
│
├── lib/
│   ├── animation/
│   │   ├── gsap-setup.ts             # registerPlugin, defaults
│   │   ├── presets.ts                 # Configs reutilizables (ANIM)
│   │   └── easings.ts               # Custom easing curves
│   ├── fonts.ts                      # next/font: Satoshi + General Sans
│   ├── format.ts                     # formatPrice(), discountPct()
│   └── cn.ts                         # cn() = clsx + twMerge
│
├── fonts/                            # woff2 files (Satoshi, General Sans)
│   ├── Satoshi-Medium.woff2
│   ├── Satoshi-Bold.woff2
│   ├── GeneralSans-Regular.woff2
│   └── GeneralSans-Medium.woff2
│
└── styles/
    └── globals.css                    # Tailwind, CSS vars, mesh, noise, [data-gsap-reveal]
```

---

## Librerias Completas

| Libreria | Version | Uso |
|----------|---------|-----|
| next | 15.x | Framework SSR + App Router |
| react | 19.x | UI library |
| tailwindcss | 4.x | Estilos utility-first |
| gsap | 3.12+ | Animaciones core |
| @gsap/react | 2.x | useGSAP hook (cleanup automatico) |
| recharts | 2.x | Graficos de historial de precios |
| lucide-react | latest | Iconos (tree-shaken) |
| nuqs | latest | URL search params type-safe |
| clsx | latest | Clases condicionales |
| tailwind-merge | latest | Merge de clases Tailwind |

---

## SEO

- Server-Side Rendering en TODAS las paginas publicas
- `generateMetadata()` dinamico por producto/categoria
- Open Graph tags para redes sociales
- `sitemap.ts` dinamico
- Schema.org JSON-LD (`Product` con `Offer`)
- URLs limpias: `/producto/notebook-lenovo-ideapad-3`
- `[data-gsap-reveal]` no afecta SEO: el HTML esta en el server render, solo la opacidad es 0

---

## Responsive Design

- Mobile-first approach
- Breakpoints: sm (640px), md (768px), lg (1024px), xl (1280px)
- Grid de productos: 1 col (mobile), 2 col (sm), 3 col (md), 4 col (lg)
- Filtros: Sheet (drawer) en mobile, Sidebar fijo en desktop
- PriceTable: scroll horizontal en mobile
- Animaciones: distancias reducidas en mobile (y:16 vs y:24)
- Mouse parallax: solo desktop (min-width: 1024px)

---

## Performance y Accesibilidad

### Performance Budget

| Recurso | Tamano | Costo runtime |
|---------|--------|---------------|
| Mesh gradient | 0 KB (CSS puro) | GPU-accelerated, ~0% CPU |
| Noise texture | ~5 KB (PNG cached) | 0% CPU |
| GSAP + ScrollTrigger | ~30 KB gzip | Solo en client components |
| Fuentes (4 archivos) | ~120 KB total | Preloaded, swap |

### Reglas de performance
- **Zero Canvas/WebGL** — no particles.js, no Three.js, no WebGL
- **CSS para continuas** — mesh gradient con @keyframes, no requestAnimationFrame
- **GSAP para one-shot** — animaciones play-once-on-scroll, luego limpias
- **ScrollTrigger.batch()** — un solo observer en vez de N listeners
- **Max ~20 ScrollTrigger instances por pagina**

### Accesibilidad (prefers-reduced-motion)

**OBLIGATORIO** en cada componente animado:

```typescript
useGSAP(() => {
  const mm = gsap.matchMedia();

  mm.add("(prefers-reduced-motion: reduce)", () => {
    // Saltar animaciones: set estado final inmediatamente
    gsap.set("[data-gsap-reveal]", { opacity: 1, y: 0, scale: 1 });
  });

  mm.add("(prefers-reduced-motion: no-preference)", () => {
    // Animaciones completas aqui
  });
}, { scope: containerRef });
```

### SSR Safety

- Todo GSAP vive en `"use client"` components exclusivamente
- RSC pages importan client components pero nunca referencian `gsap`, `window`, `document`
- Server render muestra HTML con `[data-gsap-reveal]` (opacity:0 via CSS)
- GSAP hydrata y anima en el cliente
- Sin flash of unstyled content (FOUC)

### Focus indicators

Todos los elementos interactivos: `focus-visible:ring-2 focus-visible:ring-primary`

---

## Secuencia de Implementacion

> **Nota de Roadmap**: Esta secuencia corresponde a Phases 1-3 del roadmap global.
> **Las animaciones GSAP son Phase 3 (polish), no Phase 1.**
> Phase 1 implementa el frontend con Tailwind + shadcn/ui puro (sin GSAP).
> Solo en Phase 3 se agregan las animaciones y el hero completo.

| Semana | Fase | Tareas |
|--------|------|--------|
| 1-2 | Phase 1 | Fonts + shadcn/ui init + paleta en globals.css + MeshBackground (sin animacion) + NoiseOverlay + componentes basicos: SearchBar, ProductCard, ProductGrid, PriceTable, PriceDisplay |
| 3-4 | Phase 1 | Paginas con datos reales (busqueda, producto, categoria, ofertas) + error.tsx por ruta + loading.tsx skeletons + SEO basico |
| 5-6 | Phase 2 | PriceChart (Recharts) + SearchFilters + paginacion + dark mode (next-themes) |
| 7-8 | Phase 3 | gsap-setup + presets + animation primitives (ScrollReveal, TextReveal, FadeIn, StaggerContainer, CountUp) + HeroSection timeline + secciones scroll + reduced-motion audit + mobile testing + Lighthouse |

---

## Formateo de Precios

```typescript
// src/lib/format.ts

/** Formatea precio CLP: 299990 → "$299.990" */
export function formatPrice(price: number): string {
  return '$' + price.toLocaleString('es-CL');
}

/** Calcula porcentaje de descuento */
export function discountPct(normalPrice: number, offerPrice: number): number {
  return Math.round(100 * (normalPrice - offerPrice) / normalPrice);
}
```
