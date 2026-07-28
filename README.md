# 🕸️ Trawl: Motor de scraping adaptativo que bypasa Cloudflare con Docker

[![GitHub](https://img.shields.io/badge/GitHub-Repositorio-blue)](https://github.com/JLalib/trawl-docker) [![Docker](https://img.shields.io/badge/Docker-Trawl-blue)](https://ghcr.io/germondai/trawl) [![License](https://img.shields.io/badge/Licencia-MIT-green)](https://github.com/JLalib/trawl-docker/blob/main/LICENSE)

## 📋 Descripción general

Trawl es un motor de scraping web self-hosted que resuelve desafíos de protección (Cloudflare, captchas) de forma nativa sin APIs externas, permitiendo raspado de contenido web con total privacidad y cero cuotas o tokens. Es compatible drop-in con el stack *arr (Prowlarr, Jackett, Sonarr, Radarr) como reemplazo directo de FlareSolverr.

Este repositorio contiene la configuración necesaria para desplegar Trawl con Docker Compose, siguiendo el tutorial de Genbyte para bypasear Cloudflare y captchas sin depender de servicios cloud.

## ✨ Características principales

- **Bypass nativo de Cloudflare**: resuelve challenges en 4-15 segundos, sin APIs externas ni cuotas
- **Captchas automáticos**: soporte para Turnstile, reCAPTCHA v2/v3, hCaptcha, GeeTest e Imperva (experimental)
- **Caching agresivo**: resultados repetidos en menos de 500ms gracias a la persistencia en Redis
- **Estrategia adaptativa de 5 tiers**: fetch simple → Cloudflare cached → Cloudflare fresh → contexto de navegador → headers personalizados
- **Compatible con FlareSolverr**: reemplazo directo para Prowlarr, Jackett, Sonarr y Radarr
- **API REST simple**: endpoint v1 con metadata enriquecida (tier usado, timings, cookies) y endpoint `/scrape` simplificado
- **Dos builds según hardware**: `:latest` para hardware moderno (AVX2) y `:baseline` para Synology/kernel 4.4+
- **Multiarch**: compatible con amd64 y arm64 (Raspberry Pi, Synology, servidores x86)
- **Bun runtime**: rendimiento rápido y desarrollo activo
- **MIT open source**: código abierto y auditable

## 📋 Requisitos del sistema

- Docker y Docker Compose v2+
- Entre 2 GB y 8 GB de RAM mínimo (browser pool + Redis)
- 10 GB de espacio en disco (imagen + cache + logs)
- Puerto 8191 disponible (configurable, API de scraping)
- Redis integrado para el caching de sesiones
- CPU moderno con AVX2, o baseline con kernel 4.4+ (según el build elegido)
- Conexión de red estable (tráfico HTTP/HTTPS saliente)
- Opcional: Prowlarr, Jackett, Sonarr o Radarr para integración *arr

💡 Boot time: la primera ejecución tarda 15-30 segundos en calentar el pool de navegadores; las siguientes son rápidas gracias al cache.

## 🐳 Instalación

### Opción 1: Git clone + Docker Compose (recomendado)

```bash
git clone https://github.com/germondai/trawl.git
cd trawl
cp .env.example .env
docker compose up -d

# Verificar health check
curl http://localhost:8191/health
```

### Opción 2: Docker Compose manual

Crea un archivo `docker-compose.yml` con el siguiente contenido:

```yaml
version: '3.8'

services:
  trawl:
    image: ghcr.io/germondai/trawl:latest
    container_name: trawl
    restart: unless-stopped
    ports:
      - "8191:8191"
    environment:
      - TRAWL_PORT=8191
      # Redis connection (local container por defecto)
      - REDIS_URL=redis://redis:6379
      # Logging level: debug, info, warn, error
      - LOG_LEVEL=info
    depends_on:
      - redis
    volumes:
      - trawl_cache:/tmp/trawl

  redis:
    image: redis:7-alpine
    container_name: trawl_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  trawl_cache:
  redis_data:
```

Luego, inicia el servicio:

```bash
docker compose up -d
```

### Opción 3: Hardware antiguo (Synology, kernel 4.4+)

Usa el mismo `docker-compose.yml` de la Opción 2 pero cambiando la imagen a la variante `:baseline`:

```yaml
image: ghcr.io/germondai/trawl:baseline  # en lugar de :latest
```

### Primer test (scraping simple)

```bash
curl -X POST http://localhost:8191/v1 \
  -H 'Content-Type: application/json' \
  -d '{
    "cmd":"request.get",
    "url":"https://example.com",
    "maxTimeout":60000
  }'
```

La respuesta incluye: tier usado, HTML, cookies, timings y si la sesión estaba cacheada.

## ⚙️ Configuración

Antes de iniciar el contenedor, revisa estas variables en tu `.env`:

1. **TRAWL_PORT**: puerto de la API (por defecto: 8191)
2. **REDIS_URL**: cadena de conexión a Redis (por defecto: `redis://redis:6379`)
3. **LOG_LEVEL**: nivel de detalle de los logs (`debug`, `info`, `warn`, `error`)
4. **BROWSER_POOL_SIZE**: tamaño del pool de navegadores para resolver captchas
5. **SESSION_CACHE_TTL**: tiempo de vida (en segundos) de las sesiones cacheadas en Redis

💡 Consejo: si tu hardware es antiguo o corres en un Synology con kernel 4.4+, usa el tag `:baseline` en lugar de `:latest`.

## 🚀 Primeros pasos

1. Verifica que el contenedor está corriendo: `curl http://localhost:8191/health` (respuesta esperada: `{"status":"ok"}`)
2. Haz un test de scraping simple sin protección, apuntando a cualquier URL pública
3. Prueba con un sitio protegido por Cloudflare: la primera petición tardará 4-15 segundos en resolver el challenge, la segunda será casi instantánea (cacheada)
4. Si tu herramienta espera el endpoint simplificado, usa `/scrape` en lugar de `/v1`
5. Integra Trawl con Prowlarr:
   - Settings → Clients → busca "FlareSolverr" (Trawl es compatible)
   - URL: `http://trawl:8191` (en Docker Compose) o `http://localhost:8191`
   - Guarda y Prowlarr empezará a usar Trawl para bypasear Cloudflare
6. Integra Trawl con Jackett:
   - Indexer Proxy Settings → Proxy: "FlareSolverr"
   - Host: `http://trawl:8191`
7. Revisa los logs para ver qué tier se está usando en cada petición: `docker logs -f trawl`

## 💡 Casos de uso

- ***arr stack** (Prowlarr, Jackett, Sonarr, Radarr): bypass nativo de Cloudflare, reemplazo directo de FlareSolverr sin cuotas
- **Web scraping automatizado**: sitios protegidos por Cloudflare, datos públicos, automatización inteligente
- **Data collection**: APIs bloqueadas por CF, scraping escalable con caching eficiente
- **Reverse proxy scraping**: integrable en cualquier stack web mediante su API REST
- **Testing de aplicaciones web**: simula interacción de usuario y resuelve captchas automáticamente

## 🔒 Acceso remoto seguro (opcional)

⚠️ **Importante**: Trawl NO debe exponerse a internet sin autenticación. Limita el acceso solo al stack *arr o protégelo con autenticación básica.

### Configuración Caddyfile (ejemplo con basic auth)

```
trawl.tudominio.com {
    reverse_proxy localhost:8191
    basicauth / {
        usuario contraseña_fuerte_hash
    }
}
```

### Pasos para acceso seguro

1. Instala y configura Caddy, Nginx Proxy Manager o Traefik en tu servidor
2. Añade autenticación básica o restringe el acceso a nivel de firewall/red
3. Configura el proxy inverso para apuntar a `localhost:8191`
4. Accede a Trawl a través de tu dominio seguro (<https://trawl.tudominio.com>)

## 🛠️ Gestión y mantenimiento

### Ver logs detallados

```bash
docker logs -f trawl
```

### Limpiar cache de Redis (borra todo)

```bash
docker exec trawl_redis redis-cli FLUSHALL
```

### Limpieza selectiva por dominio

```bash
docker exec trawl_redis redis-cli KEYS "example.com*" | xargs docker exec trawl_redis redis-cli DEL
```

### Reiniciar contenedores

```bash
docker compose restart
# O solo Trawl
docker compose restart trawl
```

### Actualizar a la última versión

```bash
docker compose pull
docker compose up -d

# O en repo clonado
cd trawl && git pull && docker compose pull && docker compose up -d
```

### Monitorear consumo

```bash
docker stats trawl trawl_redis
```

### Backup del cache de Redis

```bash
docker exec trawl_redis redis-cli BGSAVE
docker cp trawl_redis:/data/dump.rdb ./redis-backup-$(date +%Y%m%d).rdb
```

## 📝 Licencia

Este proyecto se basa en [Trawl](https://github.com/germondai/trawl), licenciado bajo MIT. La configuración y documentación proporcionada aquí está bajo la [MIT License](https://github.com/JLalib/trawl-docker/blob/main/LICENSE).

---

> ✨ **Nota**: Este repositorio contiene la configuración Docker y documentación extraída del tutorial de Genbyte: <a href="https://genbyte.blogspot.com/2026/07/como-instalar-trawl-en-docker-motor.html" target="_blank" rel="noopener noreferrer">Cómo instalar Trawl en Docker - Motor scraping adaptativo que bypasa Cloudflare autohospedado</a>
