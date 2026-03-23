# 05 - Scheduling y Jobs

## Overview

Sistema de scheduling basado en BullMQ (Redis) para gestionar jobs de scraping.
node-cron dispara los ciclos de scraping 1-2 veces al dia.
Workers procesan los jobs con concurrencia controlada.

## Arquitectura

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  node-cron   │────▶│   BullMQ Queue   │────▶│   Workers (N)    │
│              │     │   "scraping"     │     │                  │
│  6:00 AM     │     │                  │     │  concurrency: 3  │
│  6:00 PM     │     │  Jobs:           │     │  (max browsers)  │
│              │     │  - store scrapes  │     │                  │
│  (triggers)  │     │  - single URLs   │     │  Playwright      │
└──────────────┘     └──────────────────┘     │  BrowserPool     │
                                               └──────────────────┘
      │
      │ manual trigger
      │
┌─────▼────────┐
│ POST /api/   │
│ admin/scrape │
└──────────────┘
```

## Job Types

### 1. Store Scrape Job
Scrapea todos los `product_urls` activos de una tienda.

```typescript
interface StoreScrapeJobData {
  type: 'store-scrape';
  storeId: number;
  scraperKey: string;
  scrapeJobId: number; // ID del registro en tabla scrape_jobs
}
```

### 2. Single URL Scrape Job
Scrapea una URL individual (para reintentos o scraping on-demand).

```typescript
interface SingleScrapeJobData {
  type: 'single-scrape';
  productUrlId: number;
  url: string;
  scraperKey: string;
}
```

## Implementacion

### Cron Scheduler

```typescript
// src/lib/jobs/scheduler.ts

import cron from 'node-cron';
import { scrapeQueue } from './queue';
import { db } from '../db';
import { stores, scrapeJobs } from '../db/schema';
import { eq } from 'drizzle-orm';

export function startScheduler() {
  // Scraping a las 6:00 AM y 6:00 PM (hora Chile)
  cron.schedule('0 6,18 * * *', async () => {
    console.log('[Scheduler] Iniciando ciclo de scraping...');
    await enqueueAllStores('cron');
  }, {
    timezone: 'America/Santiago',
  });

  console.log('[Scheduler] Cron programado: 6:00 AM y 6:00 PM (Chile)');
}

export async function enqueueAllStores(triggeredBy: 'cron' | 'manual') {
  const activeStores = await db.select()
    .from(stores)
    .where(eq(stores.isActive, true));

  for (const store of activeStores) {
    // Crear registro en scrape_jobs
    const [job] = await db.insert(scrapeJobs).values({
      storeId: store.id,
      status: 'pending',
      triggeredBy,
    }).returning();

    // Encolar job en BullMQ
    await scrapeQueue.add(
      `store-${store.slug}`,
      {
        type: 'store-scrape',
        storeId: store.id,
        scraperKey: store.scraperKey,
        scrapeJobId: job.id,
      },
      {
        priority: getStorePriority(store.slug),
        attempts: 2,
        backoff: { type: 'exponential', delay: 30000 },
        removeOnComplete: { age: 86400 },   // limpiar despues de 24h
        removeOnFail: { age: 604800 },       // mantener fallos 7 dias
      },
    );
  }
}

/** Prioridad: menor numero = mayor prioridad */
function getStorePriority(slug: string): number {
  const priorities: Record<string, number> = {
    'falabella': 1,
    'ripley': 1,
    'paris': 1,
    'pcfactory': 2,
    'lider': 2,
    'sodimac': 3,
    'mercadolibre': 3,
  };
  return priorities[slug] ?? 5;
}
```

### Queue

```typescript
// src/lib/jobs/queue.ts

import { Queue, Worker, Job } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
  maxRetriesPerRequest: null,
});

export const scrapeQueue = new Queue('scraping', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 5000,
    },
  },
});
```

### Worker

```typescript
// src/lib/jobs/worker.ts

import { Worker, Job } from 'bullmq';
import { scrapeProductUrl } from '../scraping/scrape-product';
import { browserPool } from '../scraping/browser-pool';
import { getScraper } from '../scraping/registry';
import { db } from '../db';
import { productUrls, scrapeJobs } from '../db/schema';
import { eq, and } from 'drizzle-orm';

export function startWorker() {
  const worker = new Worker(
    'scraping',
    async (job: Job) => {
      if (job.data.type === 'store-scrape') {
        await processStoreScrape(job);
      } else if (job.data.type === 'single-scrape') {
        await processSingleScrape(job);
      }
    },
    {
      connection: new Redis(process.env.REDIS_URL || 'redis://localhost:6379', {
        maxRetriesPerRequest: null,
      }),
      concurrency: 3,  // max 3 jobs simultaneos (= max 3 browsers)
      limiter: {
        max: 10,        // max 10 jobs por minuto (rate limit global)
        duration: 60000,
      },
    },
  );

  worker.on('completed', (job) => {
    console.log(`[Worker] Job ${job.id} completado`);
  });

  worker.on('failed', (job, err) => {
    console.error(`[Worker] Job ${job?.id} fallo:`, err.message);
  });

  // Graceful shutdown
  process.on('SIGTERM', async () => {
    console.log('[Worker] Cerrando...');
    await worker.close();
    await browserPool.shutdown();
  });

  return worker;
}

async function processStoreScrape(job: Job) {
  const { storeId, scraperKey, scrapeJobId } = job.data;
  const scraper = getScraper(scraperKey);

  // Obtener todas las URLs activas de esta tienda
  const urls = await db.select()
    .from(productUrls)
    .where(and(
      eq(productUrls.storeId, storeId),
      eq(productUrls.isActive, true),
    ));

  // Actualizar job status a 'running'
  await db.update(scrapeJobs)
    .set({ status: 'running', startedAt: new Date(), totalUrls: urls.length })
    .where(eq(scrapeJobs.id, scrapeJobId));

  let successCount = 0;
  let errorCount = 0;
  const errors: Array<{ url: string; error: string }> = [];

  // Procesar URLs secuencialmente por tienda (respetar rate limiting)
  for (const pu of urls) {
    try {
      const result = await scrapeProductUrl({
        productUrlId: pu.id,
        url: pu.url,
        scraperKey,
        cssOverrides: pu.cssSelectors,
      });

      if (result.success) {
        successCount++;
      } else {
        errorCount++;
        errors.push({ url: pu.url, error: result.error || 'Unknown error' });
      }
    } catch (err) {
      errorCount++;
      errors.push({
        url: pu.url,
        error: err instanceof Error ? err.message : String(err),
      });
    }

    // Delay entre requests (respetar rate limiting de la tienda)
    await delay(scraper.delayBetweenRequests);

    // Actualizar progreso periodicamente
    if ((successCount + errorCount) % 50 === 0) {
      await job.updateProgress(
        Math.round(((successCount + errorCount) / urls.length) * 100)
      );
    }
  }

  // Actualizar job status a 'completed'
  const finishedAt = new Date();
  await db.update(scrapeJobs)
    .set({
      status: errorCount > urls.length * 0.5 ? 'failed' : 'completed',
      successCount,
      errorCount,
      errors: errors.slice(0, 100), // max 100 errores guardados
      finishedAt,
      durationMs: finishedAt.getTime() - job.timestamp,
    })
    .where(eq(scrapeJobs.id, scrapeJobId));

  // Refrescar vistas materializadas
  await db.execute(sql`REFRESH MATERIALIZED VIEW CONCURRENTLY mv_current_prices`);
  await db.execute(sql`REFRESH MATERIALIZED VIEW CONCURRENTLY mv_best_prices`);
}

async function processSingleScrape(job: Job) {
  const { productUrlId, url, scraperKey } = job.data;
  await scrapeProductUrl({ productUrlId, url, scraperKey });
}

function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

## Flujo Completo de un Ciclo de Scraping

```
1. 6:00 AM → node-cron trigger
    │
2.  └─▶ enqueueAllStores('cron')
         ├─ INSERT scrape_jobs (status: 'pending') por cada tienda
         └─ scrapeQueue.add() por cada tienda (con prioridad)
    │
3.  └─▶ Worker picks up job (concurrency: 3)
         │
4.       └─▶ processStoreScrape()
              ├─ SELECT product_urls WHERE store_id = X AND is_active = true
              ├─ UPDATE scrape_jobs SET status = 'running'
              │
5.            └─▶ FOR EACH url (secuencial, con delays):
                   ├─ browserPool.acquirePage()
                   ├─ page.goto(url)
                   ├─ scraper.checkPageHealth(page)
                   ├─ scraper.scrapeProduct(page) → ScrapedPrice
                   ├─ validateScrapedPrice(data)
                   ├─ INSERT prices (historial)
                   ├─ UPDATE product_urls (cache)
                   ├─ release page
                   └─ delay(delayBetweenRequests)
              │
6.            └─▶ UPDATE scrape_jobs SET status = 'completed'
              └─▶ REFRESH MATERIALIZED VIEW mv_current_prices
              └─▶ REFRESH MATERIALIZED VIEW mv_best_prices
    │
7. ~7:00 AM → Todos los jobs completados (estimado ~45-60 min para 10-20 tiendas)
```

## Estimacion de Tiempos

| Metrica | Valor |
|---------|-------|
| Tiempo por producto | ~3-5 segundos (navegacion + extraccion + delay) |
| Productos por tienda | 200-2000 |
| Tiempo por tienda | 10-60 minutos |
| Jobs concurrentes | 3 (limitado por RAM) |
| Tiempo total (10 tiendas) | ~30-90 minutos |
| Tiempo total (20 tiendas) | ~60-180 minutos |

## Monitoreo

### BullMQ Dashboard (Bull Board)

```typescript
// src/app/api/admin/bull-board/route.ts (opcional)
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { ExpressAdapter } from '@bull-board/express';

// Accesible en /api/admin/bull-board
```

### Logs Estructurados

Cada operacion de scraping loguea:
- `[Scheduler]` - triggers de cron
- `[Worker]` - inicio/fin de jobs
- `[Scraper:{store}]` - resultado por URL (exito/error)

### Alertas (futuro)

- Job fallo con >50% errores → notificacion
- Tienda con 100% errores → posible cambio de HTML → revisar selectores
- Scraping no ejecutado en >24h → alerta
