# 06 - Deployment

## Overview

El proyecto se despliega con Docker Compose tanto en desarrollo local como en produccion.
3 servicios: app (Next.js + workers), PostgreSQL, Redis.

## Docker Compose - Desarrollo Local

```yaml
# docker-compose.dev.yml

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: preciochile
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: preciochile
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U preciochile"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

En desarrollo, Next.js corre fuera de Docker (hot reload):

```bash
# Terminal 1: Levantar servicios
docker compose -f docker-compose.dev.yml up -d

# Terminal 2: App Next.js
pnpm dev

# Terminal 3: Worker (opcional, se puede integrar en el mismo proceso)
pnpm worker
```

## Docker Compose - Produccion

```yaml
# docker-compose.yml

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://preciochile:${POSTGRES_PASSWORD}@postgres:5432/preciochile
      REDIS_URL: redis://redis:6379
      ADMIN_API_KEY: ${ADMIN_API_KEY}
      PORT: 3000
    ports:
      - "${PORT:-3000}:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    # Limites de recursos
    deploy:
      resources:
        limits:
          memory: 1.5G
          cpus: '2.0'

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: preciochile
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: preciochile
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U preciochile"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru --save 60 1000
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

volumes:
  postgres_data:
  redis_data:
```

## Dockerfile

```dockerfile
# Dockerfile

FROM node:22-slim AS base
RUN apt-get update && apt-get install -y \
    # Dependencias de Playwright Chromium
    libnss3 libnspr4 libdbus-1-3 libatk1.0-0 libatk-bridge2.0-0 \
    libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
    libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 libcairo2 \
    libasound2 libatspi2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Instalar pnpm
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app

# Copiar package files
COPY package.json pnpm-lock.yaml ./

# Instalar dependencias
FROM base AS deps
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile

# Instalar Playwright browser (solo Chromium)
RUN npx playwright install chromium

# Build
FROM deps AS build
COPY . .
RUN pnpm build

# Production
FROM base AS runner
ENV NODE_ENV=production

COPY --from=deps /root/.cache/ms-playwright /root/.cache/ms-playwright
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/.next ./.next
COPY --from=build /app/public ./public
COPY --from=build /app/package.json ./
COPY --from=build /app/drizzle ./drizzle

EXPOSE 3000

# Ejecutar Next.js + Worker en el mismo proceso
CMD ["node", "server.js"]
```

## Variables de Entorno

### Produccion (.env)

```bash
# === Base de datos ===
DATABASE_URL=postgresql://preciochile:CHANGE_ME_STRONG_PASSWORD@postgres:5432/preciochile
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD

# === Redis ===
REDIS_URL=redis://redis:6379

# === App ===
NODE_ENV=production
PORT=3000
NEXT_PUBLIC_APP_URL=https://tu-dominio.cl

# === Admin ===
ADMIN_API_KEY=CHANGE_ME_RANDOM_STRING

# === Scraping ===
MAX_BROWSERS=3              # max browsers simultaneos
SCRAPE_CRON="0 6,18 * * *"  # horario de scraping (default: 6AM y 6PM)
REQUEST_DELAY_MS=2000        # delay entre requests (ms)

# === Opcional ===
# PROXY_URL=http://user:pass@proxy:port   # proxy para scraping
# SENTRY_DSN=https://...                   # error tracking
```

## Uso de RAM Estimado

| Servicio | RAM |
|----------|-----|
| Next.js (SSR + API) | ~300-500 MB |
| Playwright (3 browsers) | ~450-900 MB |
| PostgreSQL | ~200-400 MB |
| Redis | ~128-256 MB |
| **Total** | **~1.1-2.0 GB** |
| OS + overhead | ~500 MB |
| **Total con OS** | **~1.6-2.5 GB** |

**VPS recomendado**: 4 GB RAM, 2 vCPU, 40 GB SSD

## Plan de Migracion

```
Fase 1: Desarrollo Local
├── Docker Compose dev (postgres + redis)
├── Next.js en host (hot reload)
├── Playwright local
└── Datos de prueba

Fase 2: DigitalOcean (3 meses gratis)
├── Droplet 4GB RAM / 2 vCPU (~$24/mes)
├── Docker Compose produccion
├── Dominio (opcional, puede ser IP directa)
└── Backups manuales de DB

Fase 3: VPS Economico (post-DO)
├── Hetzner CX22: 4GB RAM / 2 vCPU ($4.5 EUR/mes)
│   o Contabo VPS S: 8GB RAM / 4 vCPU (~$6 EUR/mes)
│   o DigitalOcean Basic: 4GB RAM ($24/mes)
├── Transferir datos: pg_dump + pg_restore
├── DNS apuntando al nuevo servidor
└── Backups automatizados
```

## Backups

```bash
# Backup diario de PostgreSQL (agregar a crontab del host)
# Corre a las 4 AM, antes del scraping de las 6 AM

0 4 * * * docker compose exec -T postgres pg_dump -U preciochile preciochile | gzip > /backups/preciochile_$(date +\%Y\%m\%d).sql.gz

# Limpiar backups mayores a 30 dias
0 5 * * * find /backups -name "preciochile_*.sql.gz" -mtime +30 -delete
```

## Actualizaciones

```bash
# Pull cambios
git pull origin main

# Rebuild y restart
docker compose build app
docker compose up -d app

# Ejecutar migraciones si hay cambios de schema
docker compose exec app npx drizzle-kit migrate
```

## Monitoreo Basico

- **Uptime**: `curl -f http://localhost:3000/api/health` en cron cada 5 min
- **Disk**: Alerta si disco > 80% (`df -h`)
- **Docker**: `docker compose ps` para verificar servicios corriendo
- **Logs**: `docker compose logs -f app --tail=100`

### Health Check Endpoint

```typescript
// src/app/api/health/route.ts

export async function GET() {
  const checks = {
    db: await checkDatabase(),
    redis: await checkRedis(),
    uptime: process.uptime(),
  };

  const healthy = checks.db && checks.redis;

  return Response.json(checks, {
    status: healthy ? 200 : 503,
  });
}
```
