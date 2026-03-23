# 02 - Sistema de Scraping

## Overview

Sistema de scraping basado en Playwright con scrapers especificos por tienda.
Cada tienda tiene su propia clase que implementa la interfaz `StoreScraper`,
con CSS selectors especificos para extraer precios, nombres, y disponibilidad.

## Arquitectura

```
┌──────────────────────────────────────────────────────┐
│                   ScrapingService                     │
│                                                       │
│  ┌─────────────┐    ┌─────────────────────────────┐  │
│  │ BrowserPool  │    │     ScraperRegistry          │  │
│  │              │    │                              │  │
│  │ max: 3       │    │  'falabella' → FalabellaSc.  │  │
│  │ browsers     │    │  'ripley'    → RipleySc.     │  │
│  │              │    │  'paris'     → ParisSc.      │  │
│  │ reuse across │    │  'lider'     → LiderSc.      │  │
│  │ scrapes      │    │  'pcfactory' → PCFactorySc.  │  │
│  └──────┬───────┘    │  ...                         │  │
│         │            └──────────────┬───────────────┘  │
│         │                           │                   │
│         └───────────┬───────────────┘                   │
│                     ▼                                   │
│  ┌──────────────────────────────────────────────────┐  │
│  │              scrapeProductUrl()                    │  │
│  │                                                    │  │
│  │  1. Get browser from pool                         │  │
│  │  2. Get scraper by store.scraper_key              │  │
│  │  3. Navigate to URL                               │  │
│  │  4. Wait for page load / selectors                │  │
│  │  5. Extract data with CSS selectors               │  │
│  │  6. Validate extracted data                       │  │
│  │  7. Return ScrapedPrice or throw error            │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Interfaces TypeScript

```typescript
// src/lib/scraping/types.ts

interface ScrapedPrice {
  price: number;            // precio normal en CLP (entero)
  offerPrice: number | null; // precio oferta, null si no hay
  isAvailable: boolean;
  productName: string;      // nombre del producto en la tienda
  imageUrl: string | null;
  currency: string;         // default 'CLP'
  metadata?: Record<string, unknown>; // datos extras (SKU tienda, rating, etc.)
}

interface ScrapeResult {
  success: boolean;
  data: ScrapedPrice | null;
  error: string | null;
  durationMs: number;
  url: string;
}

interface StoreScraper {
  /** Identificador unico de la tienda */
  readonly key: string;

  /** Selectores CSS por defecto para esta tienda */
  readonly selectors: StoreSelectors;

  /** Tiempo max de espera para carga de pagina (ms) */
  readonly pageTimeout: number;

  /** Delay entre scrapes a esta tienda (ms) - rate limiting */
  readonly delayBetweenRequests: number;

  /**
   * Scrapea un producto individual.
   * Recibe la pagina de Playwright ya navegada a la URL.
   */
  scrapeProduct(page: Page, url: string): Promise<ScrapedPrice>;

  /**
   * Verifica si la pagina cargo correctamente (no captcha, no 404).
   * Retorna null si OK, string con error si fallo.
   */
  checkPageHealth(page: Page): Promise<string | null>;
}

interface StoreSelectors {
  price: string;              // selector del precio normal
  offerPrice: string;         // selector del precio oferta
  productName: string;        // selector del nombre
  availability: string;       // selector de disponibilidad
  image: string;              // selector de imagen principal
  outOfStock?: string;        // selector que indica agotado (si existe = no disponible)
  priceAttribute?: string;    // atributo del cual leer el precio (default: textContent)
}
```

## Implementacion de Scrapers por Tienda

### Clase Base

```typescript
// src/lib/scraping/base-scraper.ts

abstract class BaseScraper implements StoreScraper {
  abstract readonly key: string;
  abstract readonly selectors: StoreSelectors;
  readonly pageTimeout = 15000;      // 15 segundos default
  readonly delayBetweenRequests = 2000; // 2 segundos entre requests

  async scrapeProduct(page: Page, url: string): Promise<ScrapedPrice> {
    // Esperar a que el selector de precio este visible
    await page.waitForSelector(this.selectors.price, {
      timeout: this.pageTimeout,
      state: 'visible',
    });

    // Extraer datos en paralelo
    const [priceText, offerText, name, imageUrl, isOutOfStock] = await Promise.all([
      this.extractText(page, this.selectors.price),
      this.extractText(page, this.selectors.offerPrice).catch(() => null),
      this.extractText(page, this.selectors.productName),
      this.extractAttribute(page, this.selectors.image, 'src').catch(() => null),
      this.selectors.outOfStock
        ? page.$(this.selectors.outOfStock).then(el => el !== null)
        : Promise.resolve(false),
    ]);

    // Parsear precio
    const price = this.parsePrice(priceText);
    const offerPrice = offerText ? this.parsePrice(offerText) : null;

    if (!price || price <= 0) {
      throw new Error(`Precio invalido extraido: "${priceText}" → ${price}`);
    }

    return {
      price,
      offerPrice: offerPrice && offerPrice < price ? offerPrice : null,
      isAvailable: !isOutOfStock,
      productName: name?.trim() || '',
      imageUrl: imageUrl || null,
      currency: 'CLP',
    };
  }

  async checkPageHealth(page: Page): Promise<string | null> {
    // Verificar captcha comun
    const hasCaptcha = await page.$('[class*="captcha"], #captcha, .g-recaptcha');
    if (hasCaptcha) return 'CAPTCHA detectado';

    // Verificar 404
    const title = await page.title();
    if (title.toLowerCase().includes('404') || title.toLowerCase().includes('no encontr')) {
      return 'Pagina no encontrada (404)';
    }

    return null;
  }

  /** Parsea texto de precio chileno: "$299.990" → 299990 */
  protected parsePrice(text: string | null): number | null {
    if (!text) return null;
    // Remover todo excepto digitos
    const cleaned = text.replace(/[^0-9]/g, '');
    const value = parseInt(cleaned, 10);
    return isNaN(value) || value <= 0 ? null : value;
  }

  protected async extractText(page: Page, selector: string): Promise<string | null> {
    const el = await page.$(selector);
    if (!el) return null;
    return el.textContent();
  }

  protected async extractAttribute(
    page: Page,
    selector: string,
    attribute: string,
  ): Promise<string | null> {
    const el = await page.$(selector);
    if (!el) return null;
    return el.getAttribute(attribute);
  }
}
```

### Scrapers Especificos

```typescript
// src/lib/scraping/scrapers/falabella.ts

class FalabellaScraper extends BaseScraper {
  readonly key = 'falabella';
  readonly pageTimeout = 20000; // Falabella es lenta
  readonly delayBetweenRequests = 3000;

  readonly selectors: StoreSelectors = {
    price: '[data-testid="price-0"] span, .prices-0 .copy10',
    offerPrice: '[data-testid="price-event-0"] span, .prices-0 .copy1',
    productName: '.product-name h1, [data-testid="product-title"]',
    availability: '[data-testid="add-to-cart-button"]',
    image: '.jsx-image img, [data-testid="product-image"] img',
    outOfStock: '[data-testid="out-of-stock"], .out-of-stock-message',
  };

  async checkPageHealth(page: Page): Promise<string | null> {
    const baseCheck = await super.checkPageHealth(page);
    if (baseCheck) return baseCheck;

    // Falabella redirige a home cuando el producto no existe
    const currentUrl = page.url();
    if (!currentUrl.includes('/product/') && !currentUrl.includes('/producto/')) {
      return 'Redirigido fuera de pagina de producto';
    }
    return null;
  }
}
```

```typescript
// src/lib/scraping/scrapers/ripley.ts

class RipleyScraper extends BaseScraper {
  readonly key = 'ripley';
  readonly delayBetweenRequests = 2500;

  readonly selectors: StoreSelectors = {
    price: '.product-price .normal-price, [data-testid="normal-price"]',
    offerPrice: '.product-price .offer-price, [data-testid="offer-price"]',
    productName: '.product-header h1, [data-testid="product-title"]',
    availability: '.add-to-cart-btn',
    image: '.product-gallery img.primary, [data-testid="product-image"]',
    outOfStock: '.out-of-stock, .product-unavailable',
  };
}
```

```typescript
// src/lib/scraping/scrapers/pcfactory.ts

class PCFactoryScraper extends BaseScraper {
  readonly key = 'pcfactory';
  readonly pageTimeout = 10000; // PCFactory es mas rapida (menos JS)
  readonly delayBetweenRequests = 1500;

  readonly selectors: StoreSelectors = {
    price: '#normal_price, .price-normal',
    offerPrice: '#offer_price, .price-offer',
    productName: 'h1.product-title, h1[itemprop="name"]',
    availability: '.btn-add-to-cart',
    image: '#main-image img, .product-image img',
    outOfStock: '.out-of-stock-label, .producto-agotado',
  };
}
```

> **NOTA IMPORTANTE**: Los CSS selectors de arriba son ejemplos/aproximaciones.
> Los selectores reales deben ser verificados manualmente inspeccionando cada sitio
> antes de la implementacion. Los sitios cambian su HTML periodicamente.

### Registry de Scrapers

```typescript
// src/lib/scraping/registry.ts

import { FalabellaScraper } from './scrapers/falabella';
import { RipleyScraper } from './scrapers/ripley';
import { PCFactoryScraper } from './scrapers/pcfactory';
// ... mas scrapers

const scrapers: Map<string, StoreScraper> = new Map();

function registerScraper(scraper: StoreScraper) {
  scrapers.set(scraper.key, scraper);
}

// Registrar todos los scrapers
registerScraper(new FalabellaScraper());
registerScraper(new RipleyScraper());
registerScraper(new PCFactoryScraper());
// registerScraper(new ParisScraper());
// registerScraper(new LiderScraper());
// ... etc

export function getScraper(key: string): StoreScraper {
  const scraper = scrapers.get(key);
  if (!scraper) throw new Error(`Scraper no registrado: ${key}`);
  return scraper;
}

export function getAllScraperKeys(): string[] {
  return Array.from(scrapers.keys());
}
```

## Browser Pool

```typescript
// src/lib/scraping/browser-pool.ts

import { chromium, Browser, BrowserContext, Page } from 'playwright';

interface BrowserPoolConfig {
  maxBrowsers: number;       // default: 3
  maxPagesPerBrowser: number; // default: 5
  browserArgs: string[];
}

const defaultConfig: BrowserPoolConfig = {
  maxBrowsers: 3,
  maxPagesPerBrowser: 5,
  browserArgs: [
    '--no-sandbox',
    '--disable-setuid-sandbox',
    '--disable-dev-shm-usage',       // Importante para Docker/VPS con poca RAM
    '--disable-gpu',
    '--disable-extensions',
    '--disable-background-networking',
    '--disable-default-apps',
    '--disable-sync',
    '--no-first-run',
  ],
};

/**
 * Pool de browsers Playwright para reutilizar instancias.
 * Limita la cantidad de browsers abiertos simultaneamente
 * para controlar el uso de RAM (~150-300MB por browser).
 */
class BrowserPool {
  private browsers: Browser[] = [];
  private config: BrowserPoolConfig;

  constructor(config: Partial<BrowserPoolConfig> = {}) {
    this.config = { ...defaultConfig, ...config };
  }

  async acquirePage(): Promise<{ page: Page; context: BrowserContext; release: () => Promise<void> }> {
    // Reutilizar browser existente o crear nuevo si hay espacio
    let browser: Browser;

    if (this.browsers.length < this.config.maxBrowsers) {
      browser = await chromium.launch({
        headless: true,
        args: this.config.browserArgs,
      });
      this.browsers.push(browser);
    } else {
      // Round-robin entre browsers existentes
      browser = this.browsers[Math.floor(Math.random() * this.browsers.length)];
    }

    const context = await browser.newContext({
      userAgent: this.getRandomUserAgent(),
      viewport: { width: 1280, height: 720 },
      locale: 'es-CL',
      timezoneId: 'America/Santiago',
    });

    const page = await context.newPage();

    // Bloquear recursos innecesarios para ahorrar RAM y tiempo
    await page.route('**/*', (route) => {
      const type = route.request().resourceType();
      if (['image', 'media', 'font', 'stylesheet'].includes(type)) {
        route.abort();
      } else {
        route.continue();
      }
    });

    return {
      page,
      context,
      release: async () => {
        await context.close();
      },
    };
  }

  async shutdown(): Promise<void> {
    await Promise.all(this.browsers.map(b => b.close()));
    this.browsers = [];
  }

  private getRandomUserAgent(): string {
    const agents = [
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/131.0.0.0 Safari/537.36',
      'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 Chrome/131.0.0.0 Safari/537.36',
      'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/131.0.0.0 Safari/537.36',
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:133.0) Gecko/20100101 Firefox/133.0',
    ];
    return agents[Math.floor(Math.random() * agents.length)];
  }
}

export const browserPool = new BrowserPool();
```

## Pipeline de Scraping

```typescript
// src/lib/scraping/scrape-product.ts

import { browserPool } from './browser-pool';
import { getScraper } from './registry';
import { db } from '../db';
import { prices, productUrls } from '../db/schema';
import { eq } from 'drizzle-orm';

interface ScrapeProductUrlParams {
  productUrlId: number;
  url: string;
  scraperKey: string;
  cssOverrides?: Record<string, string> | null;
}

async function scrapeProductUrl(params: ScrapeProductUrlParams): Promise<ScrapeResult> {
  const startTime = Date.now();
  const scraper = getScraper(params.scraperKey);

  const { page, release } = await browserPool.acquirePage();

  try {
    // 1. Navegar
    await page.goto(params.url, {
      waitUntil: 'domcontentloaded',
      timeout: scraper.pageTimeout,
    });

    // 2. Verificar salud de la pagina
    const healthError = await scraper.checkPageHealth(page);
    if (healthError) {
      return {
        success: false,
        data: null,
        error: healthError,
        durationMs: Date.now() - startTime,
        url: params.url,
      };
    }

    // 3. Extraer datos
    const data = await scraper.scrapeProduct(page, params.url);

    // 4. Validar datos
    validateScrapedPrice(data);

    // 5. Guardar en DB
    await db.insert(prices).values({
      productUrlId: params.productUrlId,
      price: data.price,
      offerPrice: data.offerPrice,
      isAvailable: data.isAvailable,
      currency: data.currency,
    });

    // 6. Actualizar cache en product_urls
    await db.update(productUrls)
      .set({
        lastScrapedAt: new Date(),
        lastPrice: data.price,
        lastOfferPrice: data.offerPrice,
        updatedAt: new Date(),
      })
      .where(eq(productUrls.id, params.productUrlId));

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
  } finally {
    await release();
  }
}

function validateScrapedPrice(data: ScrapedPrice): void {
  // Precio debe estar en rango razonable para CLP
  if (data.price < 100 || data.price > 100_000_000) {
    throw new Error(`Precio fuera de rango razonable: ${data.price} CLP`);
  }

  if (data.offerPrice !== null) {
    if (data.offerPrice < 100 || data.offerPrice >= data.price) {
      // offerPrice invalido - ignorar silenciosamente
      data.offerPrice = null;
    }
  }

  if (!data.productName || data.productName.length < 3) {
    throw new Error(`Nombre de producto invalido: "${data.productName}"`);
  }
}
```

## Manejo de Errores y Reintentos

```
Error detectado
    │
    ├── CAPTCHA → Marcar URL, reintentar en proximo ciclo con delay mayor
    │
    ├── Timeout → Reintentar 1 vez con timeout extendido
    │
    ├── 404 / Producto no existe → Marcar product_url como is_active=false
    │
    ├── Precio invalido → Loguear warning, no guardar, reintentar
    │
    ├── Selector no encontrado → Loguear error (puede indicar cambio de HTML)
    │
    └── Error de red → Reintentar hasta 2 veces con backoff
```

Los reintentos se manejan a nivel de BullMQ (ver `05-scheduling-jobs.md`):

```typescript
// Configuracion de reintentos del job
{
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 5000, // 5s, 10s, 20s
  },
}
```

## Anti-Deteccion (basico)

Medidas incluidas para evitar bloqueos (sin ser agresivo):

1. **User-Agent rotation**: 4+ user agents reales, rotar por sesion
2. **Delays entre requests**: configurable por tienda (2-5 segundos)
3. **Bloqueo de recursos**: no cargar imagenes/fonts/CSS (reduce footprint)
4. **Locale chileno**: `es-CL`, timezone `America/Santiago`
5. **Viewport realista**: 1280x720
6. **No paralelismo excesivo**: max 3 browsers, scraping secuencial por dominio
7. **Respetar robots.txt**: verificar antes de scrapear (primera vez por dominio)

### Que NO hacer (evitar ban):
- No scrapear mas de 1-2 veces al dia por producto
- No hacer requests en rafaga (respetar delays)
- No scrapear miles de paginas en minutos
- No ignorar robots.txt

## Estructura de Archivos

```
src/lib/scraping/
├── types.ts                 # Interfaces: StoreScraper, ScrapedPrice, etc.
├── base-scraper.ts          # Clase base abstracta
├── registry.ts              # Registro de scrapers por key
├── browser-pool.ts          # Pool de browsers Playwright
├── scrape-product.ts        # Funcion principal de scraping
├── validators.ts            # Validacion de datos extraidos
└── scrapers/
    ├── falabella.ts
    ├── ripley.ts
    ├── paris.ts
    ├── lider.ts
    ├── sodimac.ts
    ├── pcfactory.ts
    ├── hites.ts
    ├── abcdin.ts
    ├── easy.ts
    ├── mercadolibre.ts
    ├── spdigital.ts
    └── microplay.ts
```

## Agregar una Tienda Nueva

Para agregar soporte a una nueva tienda:

1. Crear `src/lib/scraping/scrapers/nueva-tienda.ts`
2. Extender `BaseScraper` con los CSS selectors de esa tienda
3. Registrar en `registry.ts`
4. Agregar entrada en tabla `stores` de la DB
5. Agregar `product_urls` para los productos de esa tienda

Tiempo estimado por tienda nueva: 1-3 horas (inspeccion HTML + implementacion + testing).
