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

  // OVERLAP GUARD: verificar que no haya jobs del ciclo anterior aun corriendo
  const runningJobs = await db.select({ id: scrapeJobs.id })
    .from(scrapeJobs)
    .where(eq(scrapeJobs.status, 'running'))
    .limit(1);

  if (runningJobs.length > 0 && triggeredBy === 'cron') {
    console.warn('[Scheduler] Ciclo anterior aun en ejecucion. Saltando este ciclo.');
    return;
  }

  // ATOMICIDAD: Insertar todos los registros DB en una transaccion,
  // luego encolar a BullMQ. Si falla a mitad, no quedan stores parciales.
  const cycleDate = new Date().toISOString().slice(0, 10).replace(/-/g, ''); // YYYYMMDD
  const cyclePeriod = new Date().getHours() < 12 ? 'AM' : 'PM';

  const jobRecords = await db.transaction(async (tx) => {
    return Promise.all(activeStores.map(store =>
      tx.insert(scrapeJobs).values({
        storeId: store.id,
        status: 'pending',
        triggeredBy,
      }).returning().then(rows => ({ store, dbJob: rows[0] }))
    ));
  });

  // Encolar a BullMQ despues de que la transaccion DB complicó exitosamente
  for (const { store, dbJob } of jobRecords) {
    // DEDUPLICACION: jobId unico por ciclo. Si el ciclo tarda >12h y el
    // siguiente cron dispara, el job existente no se duplica.
    const jobId = `store-${store.slug}-${cycleDate}-${cyclePeriod}`;

    await scrapeQueue.add(
      `store-${store.slug}`,
      {
        type: 'store-scrape',
        storeId: store.id,
        scraperKey: store.scraperKey,
        scrapeJobId: dbJob.id,
      },
      {
        jobId,                                // deduplicacion por ciclo
        priority: getStorePriority(store.slug),
        attempts: 2,
        backoff: { type: 'exponential', delay: 30000 },
        removeOnComplete: { age: 86400 },     // limpiar despues de 24h
        removeOnFail: { age: 604800 },        // mantener fallos 7 dias
      },
    );
  }

  console.log(`[Scheduler] ${jobRecords.length} tiendas encoladas (ciclo ${cycleDate}-${cyclePeriod})`);
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
      // NOTA: No usar BullMQ limiter global con 10-20 tiendas — no aporta nada
      // con concurrency:3. El rate limiting real es delayBetweenRequests por URL.
    },
  );

  worker.on('completed', (job) => {
    console.log(`[Worker] Job ${job.id} completado`);
  });

  worker.on('failed', (job, err) => {
    console.error(`[Worker] Job ${job?.id} fallo:`, err.message);
  });

  // Graceful shutdown — cerrar worker, queue Y conexion Redis
  const workerRedis = (worker as any).connection; // acceso a la conexion interna
  process.on('SIGTERM', async () => {
    console.log('[Worker] Cerrando gracefully...');
    await worker.close();          // esperar jobs en vuelo
    await scrapeQueue.close();     // cerrar queue (producer)
    await browserPool.shutdown();  // cerrar browsers Playwright
    workerRedis?.disconnect();     // cerrar conexion Redis del worker
    process.exit(0);
  });

  return worker;
}

// Timeout en ms para una URL individual (evita que un hang bloquee todo el job)
const URL_SCRAPE_TIMEOUT_MS = 45_000; // 45 segundos

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
  // Umbral de progreso: reportar cada 10% del total (funciona para cualquier tamano de tienda)
  const progressStep = Math.max(1, Math.ceil(urls.length / 10));

  // Procesar URLs secuencialmente por tienda (respetar rate limiting)
  for (const pu of urls) {
    try {
      // TIMEOUT POR URL: si scrapeProductUrl se cuelga, no bloquea el job entero
      const result = await Promise.race([
        scrapeProductUrl({
          productUrlId: pu.id,
          url: pu.url,
          scraperKey,
          cssOverrides: pu.cssSelectors,
        }),
        new Promise<{ success: false; error: string }>((_, reject) =>
          setTimeout(() => reject(new Error(`Timeout: URL no respondio en ${URL_SCRAPE_TIMEOUT_MS / 1000}s`)),
            URL_SCRAPE_TIMEOUT_MS)
        ),
      ]);

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

    // Reportar progreso cada 10% del total (funciona con 10 o 2000 URLs)
    const processed = successCount + errorCount;
    if (processed % progressStep === 0) {
      await job.updateProgress(Math.round((processed / urls.length) * 100));
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

  // REFRESH DEBOUNCED: solo refrescar vistas materializadas cuando NO quedan
  // mas jobs 'running' del mismo ciclo. Evita 3 refreshes concurrentes con
  // lock contention en PG de 512MB cuando varias tiendas terminan juntas.
  // NOTA: REFRESH CONCURRENTLY requiere un UNIQUE index en la vista.
  const stillRunning = await db.select({ id: scrapeJobs.id })
    .from(scrapeJobs)
    .where(eq(scrapeJobs.status, 'running'))
    .limit(1);

  if (stillRunning.length === 0) {
    console.log('[Worker] Ultimo job del ciclo. Refrescando vistas materializadas...');
    await db.execute(sql`REFRESH MATERIALIZED VIEW CONCURRENTLY mv_current_prices`);
    await db.execute(sql`REFRESH MATERIALIZED VIEW CONCURRENTLY mv_best_prices`);
    console.log('[Worker] Vistas materializadas actualizadas.');
  }
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
         ├─ OVERLAP GUARD: si hay jobs 'running' → saltar ciclo
         ├─ DB transaction: INSERT scrape_jobs (status: 'pending') por cada tienda
         └─ BullMQ: scrapeQueue.add() con jobId unico (store-{slug}-YYYYMMDD-AM/PM)
              └─ Si el jobId ya existe en BullMQ → se ignora (deduplicacion)
    │
3.  └─▶ Worker picks up job (concurrency: 3)
         │
4.       └─▶ processStoreScrape()
              ├─ SELECT product_urls WHERE store_id = X AND is_active = true
              ├─ UPDATE scrape_jobs SET status = 'running'
              │
5.            └─▶ FOR EACH url (secuencial, con delays):
                   ├─ Promise.race([scrapeProductUrl(), timeout(45s)])
                   │   ├─ browserPool.acquirePage()
                   │   ├─ page.goto(url)
                   │   ├─ scraper.checkPageHealth(page)
                   │   ├─ scraper.scrapeProduct(page) → ScrapedPrice
                   │   ├─ validateScrapedPrice(data)
                   │   ├─ db.transaction: INSERT prices + UPDATE product_urls
                   │   └─ release page
                   ├─ delay(delayBetweenRequests)
                   └─ updateProgress cada 10% del total de URLs
              │
6.            └─▶ UPDATE scrape_jobs SET status = 'completed'
              └─▶ SI no hay mas jobs 'running':
                   ├─ REFRESH MATERIALIZED VIEW CONCURRENTLY mv_current_prices
                   └─ REFRESH MATERIALIZED VIEW CONCURRENTLY mv_best_prices
    │
7. ~7:00 AM → Todos los jobs completados, vistas materializadas actualizadas
              (estimado ~45-60 min para 10-20 tiendas con 200-500 URLs cada una)
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
