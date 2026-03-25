# 02 - Sistema de Scraping

## Overview

Sistema de scraping basado en **Firecrawl** (self-hosted) para renderizado de paginas JS-heavy,
mas **cheerio** para extraccion determinista con CSS selectors por tienda.

El worker BullMQ llama a Firecrawl via HTTP para obtener el HTML renderizado,
luego extrae precio/nombre/disponibilidad con CSS selectors definidos por tienda.
Firecrawl maneja toda la complejidad del browser (stealth, JS rendering, retries).

## Arquitectura

```
BullMQ Worker
      │
      ▼
scrapeProductUrl(url, scraperKey)
      │
      ├── 1. Obtener config de tienda: getScraper(scraperKey)
      │
      ├── 2. Llamar Firecrawl self-hosted:
      │       POST http://localhost:3002/v1/scrape
      │       { url, formats: ['html'], waitFor: scraper.waitMs }
      │       ← devuelve HTML completamente renderizado
      │
      ├── 3. cheerio.load(html) + CSS selectors por tienda
      │       → precio, offerPrice, nombre, disponibilidad, imagen
      │
      ├── 4. validateScrapedPrice(data)
      │
      └── 5. DB transaction:
              INSERT prices + UPDATE product_urls.last_price
```

Firecrawl corre como servicio separado (docker-compose o VPS).
PrecioChile nunca lanza browsers — solo hace llamadas HTTP a Firecrawl.

## Interfaces TypeScript

```typescript
// src/lib/scraping/types.ts

export interface ScrapedPrice {
  price: number;             // precio normal en CLP (entero)
  offerPrice: number | null; // precio oferta, null si no hay
  isAvailable: boolean;
  productName: string;       // nombre del producto en la tienda
  imageUrl: string | null;
  currency: string;          // default 'CLP'
  metadata?: Record<string, unknown>; // SKU tienda, rating, etc.
}

export interface ScrapeResult {
  success: boolean;
  data: ScrapedPrice | null;
  error: string | null;
  durationMs: number;
  url: string;
}

export interface StoreSelectors {
  price: string;         // selector del precio normal
  offerPrice?: string;   // selector del precio oferta
  productName: string;   // selector del nombre
  availability?: string; // selector del boton add-to-cart (existe = disponible)
  image?: string;        // selector de imagen principal
  outOfStock?: string;   // selector que SI existe = agotado
}

/**
 * Configuracion de un scraper por tienda.
 * Objeto plano — no clase, no herencia.
 */
export interface StoreScraper {
  readonly key: string;
  readonly selectors: StoreSelectors;
  /** Delay entre requests a esta tienda en ms (rate limiting) */
  readonly delayMs: number;
  /** ms a esperar despues de que Firecrawl carga la pagina (para SPAs lentas) */
  readonly waitMs?: number;
  /**
   * Parsea texto de precio chileno → numero entero CLP.
   * "$299.990" → 299990
   */
  parsePrice(text: string): number | null;
}
```

## Cliente Firecrawl

```typescript
// src/lib/scraping/firecrawl-client.ts

import FirecrawlApp from '@mendable/firecrawl-js';

// Singleton — una instancia compartida por todos los workers del proceso
export const firecrawl = new FirecrawlApp({
  apiKey: process.env.FIRECRAWL_API_KEY ?? 'local',
  apiUrl: process.env.FIRECRAWL_API_URL ?? 'http://localhost:3002',
});
```

Variables de entorno requeridas:

```bash
FIRECRAWL_API_URL=http://localhost:3002  # instancia self-hosted
FIRECRAWL_API_KEY=local                  # cualquier string si self-hosted sin auth
```

## Scrapers por Tienda

```typescript
// src/lib/scraping/scrapers/falabella.ts

import type { StoreScraper } from '../types';

export const falabellaScraper: StoreScraper = {
  key: 'falabella',
  delayMs: 3000,
  waitMs: 2000, // SPA React — esperar a que los precios carguen via API
  selectors: {
    price: '[data-testid="price-0"] span, .prices-0 .copy10',
    offerPrice: '[data-testid="price-event-0"] span, .prices-0 .copy1',
    productName: '.product-name h1, [data-testid="product-title"]',
    availability: '[data-testid="add-to-cart-button"]',
    image: '[data-testid="product-image"] img, .jsx-image img',
    outOfStock: '[data-testid="out-of-stock"], .out-of-stock-message',
  },
  parsePrice(text) {
    return parsePriceCLP(text);
  },
};
```

```typescript
// src/lib/scraping/scrapers/ripley.ts

import type { StoreScraper } from '../types';

export const ripleyScraper: StoreScraper = {
  key: 'ripley',
  delayMs: 2500,
  waitMs: 1500,
  selectors: {
    price: '.product-price .normal-price, [data-testid="normal-price"]',
    offerPrice: '.product-price .offer-price, [data-testid="offer-price"]',
    productName: '.product-header h1, [data-testid="product-title"]',
    availability: '.add-to-cart-btn',
    image: '.product-gallery img.primary',
    outOfStock: '.out-of-stock, .product-unavailable',
  },
  parsePrice(text) {
    return parsePriceCLP(text);
  },
};
```

```typescript
// src/lib/scraping/scrapers/pcfactory.ts

import type { StoreScraper } from '../types';

export const pcfactoryScraper: StoreScraper = {
  key: 'pcfactory',
  delayMs: 1500,
  // PCFactory es server-rendered: no necesita waitMs extra
  selectors: {
    price: '#normal_price, .price-normal',
    offerPrice: '#offer_price, .price-offer',
    productName: 'h1.product-title, h1[itemprop="name"]',
    availability: '.btn-add-to-cart',
    image: '#main-image img, .product-image img',
    outOfStock: '.out-of-stock-label, .producto-agotado',
  },
  parsePrice(text) {
    return parsePriceCLP(text);
  },
};
```

> **NOTA**: Los CSS selectors son ejemplos/aproximaciones. Verificar inspeccionando
> cada sitio antes de implementar. Los sitios cambian su HTML periodicamente.

### Helper parsePrice (compartido)

```typescript
// src/lib/scraping/scrapers/_utils.ts

/**
 * Parsea texto de precio chileno → entero CLP.
 * "$299.990" → 299990
 * "3 cuotas de $99.997" → 99997  (extrae primer patron $X)
 *
 * IMPORTANTE: No usar text.replace(/[^0-9]/g, '') en el texto completo.
 * "3 cuotas de $99.997" se convertiria incorrectamente en 399997.
 */
export function parsePriceCLP(text: string | null | undefined): number | null {
  if (!text) return null;
  // Extraer primer patron de precio chileno: $123.456 o 123.456 o 123456
  const match = text.match(/\$?\s*([\d.]+)/);
  if (!match) return null;
  // Remover puntos de miles (en CLP "." es separador de miles, no decimales)
  const cleaned = match[1].replace(/\./g, '');
  const value = parseInt(cleaned, 10);
  return isNaN(value) || value <= 0 ? null : value;
}
```

## Registry de Scrapers

```typescript
// src/lib/scraping/registry.ts

import { falabellaScraper } from './scrapers/falabella';
import { ripleyScraper } from './scrapers/ripley';
import { pcfactoryScraper } from './scrapers/pcfactory';
import type { StoreScraper } from './types';

const registry = new Map<string, StoreScraper>([
  ['falabella', falabellaScraper],
  ['ripley', ripleyScraper],
  ['pcfactory', pcfactoryScraper],
  // Phase 2: paris, lider, sodimac, hites, abcdin, ...
]);

export function getScraper(key: string): StoreScraper {
  const scraper = registry.get(key);
  if (!scraper) throw new Error(`Scraper no registrado: "${key}"`);
  return scraper;
}

export function getAllScraperKeys(): string[] {
  return Array.from(registry.keys());
}
```

## Pipeline de Scraping

```typescript
// src/lib/scraping/scrape-product.ts

import * as cheerio from 'cheerio';
import { firecrawl } from './firecrawl-client';
import { getScraper } from './registry';
import { db } from '../db';
import { prices, productUrls, priceAnomalies } from '../db/schema';
import { eq } from 'drizzle-orm';
import type { ScrapeResult, ScrapedPrice } from './types';

interface ScrapeProductUrlParams {
  productUrlId: number;
  url: string;
  scraperKey: string;
  lastPrice?: number | null; // para deteccion de anomalias
  cssOverrides?: Record<string, string> | null; // overrides por URL especifica
}

export async function scrapeProductUrl(
  params: ScrapeProductUrlParams,
): Promise<ScrapeResult> {
  const startTime = Date.now();
  const scraper = getScraper(params.scraperKey);

  // Merge CSS overrides sobre selectors del scraper (por URL especifica)
  const selectors = params.cssOverrides
    ? { ...scraper.selectors, ...params.cssOverrides }
    : scraper.selectors;

  try {
    // 1. Llamar Firecrawl para obtener HTML renderizado
    const fcResult = await firecrawl.scrapeUrl(params.url, {
      formats: ['html'],
      ...(scraper.waitMs && { waitFor: scraper.waitMs }),
    });

    if (!fcResult.success || !fcResult.html) {
      return {
        success: false,
        data: null,
        error: fcResult.error ?? 'Firecrawl no retorno HTML',
        durationMs: Date.now() - startTime,
        url: params.url,
      };
    }

    // 2. Parsear HTML con cheerio + CSS selectors
    const $ = cheerio.load(fcResult.html);
    const data = extractWithSelectors($, selectors, scraper);

    // 3. Validar datos
    validateScrapedPrice(data);

    // 4. Guardar en DB (transaccion atomica)
    await db.transaction(async (tx) => {
      // Detectar anomalia de precio (>50% variacion)
      if (params.lastPrice && data.price) {
        const changePct =
          (Math.abs(data.price - params.lastPrice) / params.lastPrice) * 100;
        if (changePct > 50) {
          await tx.insert(priceAnomalies).values({
            productUrlId: params.productUrlId,
            previousPrice: params.lastPrice,
            newPrice: data.price,
            changePct,
          });
        }
      }

      // Insertar precio en historial
      await tx.insert(prices).values({
        productUrlId: params.productUrlId,
        price: data.price,
        offerPrice: data.offerPrice,
        isAvailable: data.isAvailable,
        currency: data.currency,
      });

      // Actualizar cache en product_urls
      await tx
        .update(productUrls)
        .set({
          lastScrapedAt: new Date(),
          lastPrice: data.price,
          lastOfferPrice: data.offerPrice,
        })
        .where(eq(productUrls.id, params.productUrlId));
    });

    return {
      success: true,
      data,
      error: null,
      durationMs: Date.now() - startTime,
      url: params.url,
    };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: error instanceof Error ? error.message : String(error),
      durationMs: Date.now() - startTime,
      url: params.url,
    };
  }
}

function extractWithSelectors(
  $: cheerio.CheerioAPI,
  selectors: typeof falabellaScraper.selectors,
  scraper: ReturnType<typeof getScraper>,
): ScrapedPrice {
  const priceText = $(selectors.price).first().text().trim();
  const offerText = selectors.offerPrice
    ? $(selectors.offerPrice).first().text().trim()
    : null;
  const productName = $(selectors.productName).first().text().trim();
  const imageUrl =
    selectors.image
      ? ($(selectors.image).first().attr('src') ?? null)
      : null;

  // outOfStock: si el selector existe en el HTML, el producto esta agotado
  const isOutOfStock = selectors.outOfStock
    ? $(selectors.outOfStock).length > 0
    : false;

  const price = scraper.parsePrice(priceText);
  if (!price) {
    throw new Error(`Precio no extraido. Selector: "${selectors.price}", texto: "${priceText}"`);
  }

  const offerPrice = offerText ? scraper.parsePrice(offerText) : null;

  return {
    price,
    offerPrice: offerPrice && offerPrice < price ? offerPrice : null,
    isAvailable: !isOutOfStock,
    productName: productName || '',
    imageUrl,
    currency: 'CLP',
  };
}

function validateScrapedPrice(data: ScrapedPrice): void {
  if (data.price < 100 || data.price > 100_000_000) {
    throw new Error(`Precio fuera de rango razonable: ${data.price} CLP`);
  }
  if (!data.productName || data.productName.length < 3) {
    throw new Error(`Nombre invalido: "${data.productName}"`);
  }
  // offerPrice invalido: silenciosamente ignorar
  if (data.offerPrice !== null && (data.offerPrice < 100 || data.offerPrice >= data.price)) {
    data.offerPrice = null;
  }
}
```

## Manejo de Errores

### Errores por URL (dentro de processStoreScrape)

```
Error en scrapeProductUrl()
    │
    ├── Firecrawl falla (timeout, 5xx) → result.success=false → incrementar errorCount
    │
    ├── Selector no encontrado → throw → result.success=false → incrementar errorCount
    │
    ├── Precio invalido → throw → result.success=false → incrementar errorCount
    │
    ├── 404 → Firecrawl retorna HTML de error → selector falla → errorCount
    │          (en Phase 2: detectar 404 y marcar product_url.is_active=false)
    │
    └── Error de DB → throw → BullMQ reintenta el JOB completo
```

> **IMPORTANTE**: `scrapeProductUrl` NO lanza excepciones para errores de extraccion.
> Retorna `{ success: false, error: "..." }`. BullMQ reintenta el JOB completo
> solo si hay error critico (DB caida, etc.).

### Reintentos BullMQ (errores criticos del job)

```typescript
{
  attempts: 2,
  backoff: { type: 'exponential', delay: 30000 }, // 30s, 60s
}
```

## Anti-Deteccion

Firecrawl self-hosted maneja la mayor parte del stealth (headers, fingerprinting).
Medidas adicionales en PrecioChile:

1. **Delays entre requests**: configurable por tienda (`delayMs` en StoreScraper)
2. **No paralelismo excesivo**: BullMQ worker `concurrency: 3` (3 jobs simultaneos maximo)
3. **Scraping 2x/dia**: cron 6AM y 6PM, no en rafaga
4. **waitMs por tienda**: dar tiempo a SPAs para cargar precios via API internas

## Estructura de Archivos

```
src/lib/scraping/
├── types.ts                  # Interfaces: StoreScraper, ScrapedPrice, ScrapeResult
├── firecrawl-client.ts       # Singleton FirecrawlApp (FIRECRAWL_API_URL)
├── registry.ts               # Map scraperKey → StoreScraper
├── scrape-product.ts         # Pipeline principal: Firecrawl → cheerio → DB
└── scrapers/
    ├── _utils.ts             # parsePriceCLP() compartido
    ├── falabella.ts          # Phase 1
    ├── ripley.ts             # Phase 1
    ├── pcfactory.ts          # Phase 1
    ├── paris.ts              # Phase 2a
    ├── lider.ts              # Phase 2a
    ├── sodimac.ts            # Phase 2a
    ├── hites.ts              # Phase 2a
    ├── abcdin.ts             # Phase 2a
    ├── easy.ts               # Phase 2b
    ├── spdigital.ts          # Phase 2b
    ├── microplay.ts          # Phase 2b
    └── mercadolibre.ts       # Phase 2b (estructura distinta, requiere estrategia especial)
```

## Agregar una Tienda Nueva

1. Crear `src/lib/scraping/scrapers/nueva-tienda.ts` con el objeto `StoreScraper`
2. Registrar en `registry.ts`
3. Insertar fila en tabla `stores` de la DB
4. Agregar `product_urls` para los productos de esa tienda

Tiempo estimado: 1-3 horas por tienda (inspeccion HTML + selectors + test).

## Tests de Scraper

Los tests de scraper usan **HTML fixtures** guardados, no llamadas reales a Firecrawl.
Esto los hace rapidos y sin dependencias externas en CI.

```typescript
// src/lib/scraping/__tests__/falabella.test.ts

import * as fs from 'fs';
import * as path from 'path';
import * as cheerio from 'cheerio';
import { falabellaScraper } from '../scrapers/falabella';

const fixtureHtml = fs.readFileSync(
  path.join(__dirname, 'fixtures/falabella-product.html'),
  'utf-8',
);

test('extrae precio de pagina de Falabella', () => {
  const $ = cheerio.load(fixtureHtml);
  const priceText = $(falabellaScraper.selectors.price).first().text().trim();
  const price = falabellaScraper.parsePrice(priceText);
  expect(price).toBeGreaterThan(0);
});

test('parsePrice maneja cuotas correctamente', () => {
  expect(falabellaScraper.parsePrice('$299.990')).toBe(299990);
  expect(falabellaScraper.parsePrice('3 cuotas de $99.997')).toBe(99997);
  expect(falabellaScraper.parsePrice('')).toBeNull();
});
```

Fixtures: guardar HTML real de la pagina del producto en `__tests__/fixtures/`.
Actualizar el fixture cuando el sitio cambie su HTML.
