# Backend Infrastructure Kit

Plataforma de servicios compartidos para desarrollo local y despliegue en VPS.

Evita levantar múltiples instancias de PostgreSQL, Redis, Ollama, Qdrant y otros servicios en cada proyecto. Un solo `docker compose up -d` y tienes todo tu backend listo, con **Traefik** como reverse proxy y **Uptime Kuma** para monitoreo.

---

## Arquitectura

Todos los servicios se ejecutan dentro de una red Docker llamada `shared-services`. Los contenedores se descubren entre sí por su **hostname**. Traefik actúa como entrypoint HTTP/HTTPS y enruta las peticiones según el dominio.

```
                    ┌──────────────┐
                    │  Traefik     │  ← puertos 80 / 443
                    │  (reverse    │
                    │   proxy)     │
                    └──────┬───────┘
                           │
               ┌───────────┴───────────────┐
               │       shared-services      │
               │                            │
               │  postgres:5432             │
               │  mysql:3306                │
               │  mongodb:27017             │
               │  redis:6379                │
               │  qdrant:6333               │
               │  ollama:11434              │
               │  n8n:5678                  │
               │  minio:9000 / 9001         │
               │  openwebui:8080            │
               │  uptime-kuma:3001          │
               │                            │
               └────────────────────────────┘
                        ▲        ▲
                   ┌────┴──┐ ┌──┴────────┐
                   │ App A │ │ App B     │
                   │(cont.)│ │(cont.)    │
                   └───────┘ └───────────┘
```

- **Aplicaciones externas** se unen a la red `shared-services` y acceden a los servicios por hostname.
- **Traefik** permite acceder vía rutas amigables (`n8n.localhost`, `openwebui.localhost`, etc.).
- **Uptime Kuma** monitorea la disponibilidad de todos los servicios.

---

## Servicios incluidos

| Servicio     | Versión          | Hostname      | Puerto      | Descripción                            |
|-------------|------------------|---------------|-------------|----------------------------------------|
| PostgreSQL  | 16-alpine        | postgres      | 5432        | Base de datos relacional               |
| MySQL       | 8.4              | mysql         | 3306        | Base de datos relacional               |
| MongoDB     | 7                | mongodb       | 27017       | Base de datos documental               |
| Redis       | 7-alpine         | redis         | 6379        | Cache / message broker                 |
| Qdrant      | latest           | qdrant        | 6333 / 6334 | Vector database (REST + gRPC)          |
| Ollama      | latest           | ollama        | 11434       | Modelos de lenguaje local              |
| n8n         | latest           | n8n           | 5678        | Automatización low-code / AI workflows |
| MinIO       | latest           | minio         | 9000 / 9001 | Object storage S3-compatible           |
| OpenWebUI   | main             | openwebui     | 3000        | Interfaz web para LLMs                 |
| Traefik     | v3.1             | traefik       | 80 / 443    | Reverse proxy con Docker provider      |
| Uptime Kuma | latest           | uptime-kuma   | 3001        | Monitoreo de disponibilidad            |

---

## Puertos expuestos al host

| Puerto          | Servicio      | Uso                          |
|-----------------|---------------|------------------------------|
| 5432            | PostgreSQL    | Conexión de base de datos    |
| 3307            | MySQL         | Conexión de base de datos    |
| 27017           | MongoDB       | Conexión de base de datos    |
| 6379            | Redis         | Conexión de caché            |
| 6333            | Qdrant        | API REST                     |
| 6334            | Qdrant        | API gRPC                     |
| 11434           | Ollama        | API de modelos locales       |
| 5678            | n8n           | Interfaz web + API           |
| 9000            | MinIO         | API S3                       |
| 9001            | MinIO         | Consola web                  |
| 3000            | OpenWebUI     | Interfaz web                 |
| 80              | Traefik       | HTTP                         |
| 443             | Traefik       | HTTPS                        |
| 127.0.0.1:8080  | Traefik       | Dashboard (solo localhost)   |
| 3001            | Uptime Kuma   | Interfaz web de monitoreo    |

Todos los puertos son configurables vía variables de entorno en el archivo `.env`.

---

## Requisitos

- Docker Engine 24+ y Docker Compose v2
- 8 GB de RAM mínimo (16 GB recomendado si usas Ollama con modelos grandes)
- Espacio en disco según los modelos de IA que descargues

---

## Inicio rápido

```bash
# 1. Clona el repositorio
git clone https://github.com/tu-usuario/backend-infrastructure-kit.git
cd backend-infrastructure-kit

# 2. Configura las variables de entorno
cp .env.example .env
# Edita .env con tus valores

# 3. Levanta todos los servicios
docker compose up -d

# 4. Verifica el estado
docker compose ps
```

Los servicios estarán disponibles de inmediato.

---

## Acceso por dominio con Traefik

Traefik rutea automáticamente los servicios HTTP por dominio. Después de levantar el stack, agregá estas líneas a tu archivo **hosts** (`/etc/hosts` en Linux/Mac, `C:\Windows\System32\drivers\etc\hosts` en Windows):

```
127.0.0.1  traefik.localhost
127.0.0.1  n8n.localhost
127.0.0.1  openwebui.localhost
127.0.0.1  qdrant.localhost
127.0.0.1  minio.localhost
127.0.0.1  minio-console.localhost
127.0.0.1  uptime.localhost
```

O usá un resolver DNS local como **dnsmasq** o **pihole** con resolución wildcard `*.localhost → 127.0.0.1`.

Luego accedé desde el navegador:

| URL                              | Servicio      |
|----------------------------------|---------------|
| http://traefik.localhost         | Dashboard Traefik |
| http://n8n.localhost             | n8n           |
| http://openwebui.localhost       | OpenWebUI     |
| http://qdrant.localhost          | Qdrant API    |
| http://minio.localhost           | MinIO API S3  |
| http://minio-console.localhost   | MinIO Console |
| http://uptime.localhost          | Uptime Kuma   |

El dashboard de Traefik también está disponible en http://127.0.0.1:8080 (solo localhost, sin necesidad de hosts).

> **Nota:** Las bases de datos (PostgreSQL, MySQL, MongoDB, Redis) y Ollama no usan HTTP, por lo que no tienen ruteo por Traefik. Se accede directo por hostname y puerto desde la red compartida.

---

## Uptime Kuma — Monitoreo de servicios

Uptime Kuma permite verificar la disponibilidad de todos los servicios desde una interfaz web.

1. Accedé a http://localhost:3001 o http://uptime.localhost
2. Creá una cuenta de administrador
3. Agregá monitores para cada servicio:

| Tipo        | Objetivo                         |
|-------------|----------------------------------|
| HTTP        | http://n8n:5678/healthz          |
| HTTP        | http://openwebui:8080/health     |
| HTTP        | http://qdrant:6333/healthz       |
| HTTP        | http://minio:9001                |
| HTTP        | http://ollama:11434              |
| TCP         | postgres:5432                    |
| TCP         | mysql:3306                       |
| TCP         | mongodb:27017                    |
| TCP         | redis:6379                       |
| TCP         | minio:9000                       |

Uptime Kuma está en la red `shared-services`, así que puede alcanzar todos los servicios por su hostname interno.

---

## Cómo conectarse desde otro contenedor Docker

### 1. Une tu contenedor a la red compartida

```bash
docker run --network shared-services --name mi-app -d mi-imagen
```

O en un `docker-compose.yml` de tu aplicación:

```yaml
services:
  mi-app:
    build: .
    networks:
      - shared-services

networks:
  shared-services:
    external: true
    name: shared-services
```

### 2. Usa los hostnames como dirección de conexión

Desde tu contenedor, los servicios son accesibles por su hostname:

- `postgres:5432`
- `mysql:3306`
- `mongodb:27017`
- `redis:6379`
- `qdrant:6333`
- `ollama:11434`
- `n8n:5678`
- `minio:9000`
- `openwebui:3000`

---

## Cadenas de conexión

### PostgreSQL

```
postgresql://appuser:apppassword@postgres:5432/appdb
```

### MySQL

```
mysql://appuser:apppassword@mysql:3306/appdb
```

### MongoDB

```
mongodb://root:rootpassword@mongodb:27017/appdb?authSource=admin
```

### Redis

```
redis://default:redispassword@redis:6379
```

### Qdrant

```
# REST
http://qdrant:6333

# gRPC
http://qdrant:6334
```

### Ollama

```
http://ollama:11434
```

### MinIO (S3)

```
Endpoint:  http://minio:9000
Access Key: minioadmin
Secret Key: minioadmin
Region:    us-east-1
```

---

## Conexión desde aplicaciones

### Node.js

```javascript
// PostgreSQL
import pg from 'pg';
const pool = new pg.Pool({
  host: 'postgres',
  user: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB,
});

// MongoDB
import { MongoClient } from 'mongodb';
const client = new MongoClient('mongodb://root:rootpassword@mongodb:27017');
await client.connect();

// Redis
import { createClient } from 'redis';
const redis = createClient({ url: 'redis://default:redispassword@redis:6379' });
await redis.connect();

// Qdrant
import { QdrantClient } from '@qdrant/js-client-rest';
const qdrant = new QdrantClient({ url: 'http://qdrant:6333' });

// MinIO
import { Client } from 'minio';
const minio = new Client({
  endPoint: 'minio',
  port: 9000,
  useSSL: false,
  accessKey: 'minioadmin',
  secretKey: 'minioadmin',
});
```

### Python

```python
# PostgreSQL
import psycopg2
conn = psycopg2.connect(
    host='postgres',
    user='appuser',
    password='apppassword',
    dbname='appdb',
)

# MongoDB
from pymongo import MongoClient
client = MongoClient('mongodb://root:rootpassword@mongodb:27017')

# Redis
import redis
r = redis.Redis(host='redis', password='redispassword', decode_responses=True)

# Qdrant
from qdrant_client import QdrantClient
qdrant = QdrantClient(url='http://qdrant:6333')

# MinIO
from minio import Minio
minio = Minio('minio:9000', access_key='minioadmin', secret_key='minioadmin', secure=False)

# Ollama
import requests
response = requests.post('http://ollama:11434/api/generate', json={
    'model': 'llama3.2',
    'prompt': 'Hola, ¿cómo estás?',
})
```

### Java

```java
// PostgreSQL
import java.sql.Connection;
import java.sql.DriverManager;
Connection conn = DriverManager.getConnection(
    "jdbc:postgresql://postgres:5432/appdb", "appuser", "apppassword");

// MongoDB
import com.mongodb.client.MongoClients;
var mongoClient = MongoClients.create("mongodb://root:rootpassword@mongodb:27017");

// Redis
import redis.clients.jedis.Jedis;
var jedis = new Jedis("redis", 6379);
jedis.auth("redispassword");

// Qdrant (REST)
import java.net.http.HttpClient;
import java.net.URI;
var client = HttpClient.newHttpClient();
var request = java.net.http.HttpRequest.newBuilder()
    .uri(URI.create("http://qdrant:6333/collections"))
    .build();
```

---

## Variables de entorno

Todas las configuraciones se centralizan en el archivo `.env`.

| Variable                | Default            | Descripción                           |
|------------------------|--------------------|---------------------------------------|
| `POSTGRES_USER`        | `appuser`          | Usuario de PostgreSQL                 |
| `POSTGRES_PASSWORD`    | `apppassword`      | Contraseña de PostgreSQL              |
| `POSTGRES_DB`          | `appdb`            | Base de datos por defecto             |
| `MYSQL_ROOT_PASSWORD`  | `rootpassword`     | Contraseña root de MySQL              |
| `MONGO_ROOT_PASSWORD`  | `rootpassword`     | Contraseña root de MongoDB            |
| `REDIS_PASSWORD`       | `redispassword`    | Contraseña de Redis                   |
| `MINIO_ROOT_USER`      | `minioadmin`       | Access key de MinIO                   |
| `MINIO_ROOT_PASSWORD`  | `minioadmin`       | Secret key de MinIO                   |
| `WEBUI_SECRET_KEY`     | `...`              | Secreto para JWT de OpenWebUI         |
| `N8N_ENCRYPTION_KEY`   | `...`              | Clave de encriptación de n8n          |
| `N8N_HOST`             | `n8n.localhost`    | Hostname público de n8n               |
| `TRAEFIK_HTTP_PORT`    | `80`               | Puerto HTTP de Traefik                |
| `TRAEFIK_HTTPS_PORT`   | `443`              | Puerto HTTPS de Traefik               |
| `UPTIME_KUMA_PORT`     | `3001`             | Puerto de Uptime Kuma                 |

> **Importante en producción:** cambia todas las contraseñas por defecto y usa valores seguros.

---

## Cómo agregar nuevos servicios al stack

1. Añade el servicio en `docker-compose.yml`:

```yaml
services:
  mi-servicio:
    image: ejemplo/imagen:latest
    container_name: shared-mi-servicio
    hostname: mi-servicio
    networks: [shared-services]
    restart: unless-stopped
    ports:
      - "${MI_SERVICIO_PORT:-9999}:9999"
    volumes:
      - mi_servicio_data:/ruta/datos
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mi-servicio.rule=Host(`mi-servicio.localhost`)"
      - "traefik.http.routers.mi-servicio.entrypoints=web"
      - "traefik.http.services.mi-servicio.loadbalancer.server.port=9999"
    healthcheck:
      test: ["CMD", "comando", "de", "healthcheck"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

volumes:
  mi_servicio_data:
```

Si el servicio no expone HTTP (base de datos, cola, etc.), omití las labels de Traefik.

2. Agrega las variables al `.env.example` y al `.env`.
3. Vuelve a levantar el stack:

```bash
docker compose up -d
```

Tu nuevo servicio será accesible desde cualquier contenedor en la red `shared-services` mediante el hostname `mi-servicio`, y también vía `mi-servicio.localhost` si configuraste las labels de Traefik.

---

## Recomendaciones para producción en VPS

### 1. Seguridad

- Cambia **todas** las contraseñas por defecto en el `.env`.
- Usa `openssl rand -base64 32` para generar claves seguras (`N8N_ENCRYPTION_KEY`, `WEBUI_SECRET_KEY`, `N8N_JWT_SECRET`).
- Configura HTTPS real en Traefik con Let's Encrypt:

```yaml
command:
  - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
  - "--certificatesresolvers.letsencrypt.acme.email=tu@email.com"
  - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
volumes:
  - traefik_letsencrypt:/letsencrypt
```

Luego en cada servicio:

```yaml
labels:
  - "traefik.http.routers.n8n.rule=Host(`n8n.tudominio.com`)"
  - "traefik.http.routers.n8n.entrypoints=websecure"
  - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
```

- Restringe los puertos con un firewall (UFW o iptables):

```bash
ufw default deny incoming
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
# NO expongas bases de datos al exterior
```

- Usa una VPN (WireGuard, Tailscale) para acceder al dashboard de Traefik y Uptime Kuma.

### 2. Persistencia

Todos los servicios usan volúmenes Docker nombrados. Los datos persisten aunque los contenedores se eliminen.

```bash
# Backup de todos los volúmenes
docker run --rm -v postgres_data:/source -v /backup:/dest alpine tar czf /dest/postgres_data.tar.gz -C /source .
```

### 3. Monitoreo con Uptime Kuma

Uptime Kuma ya viene incluido. En producción:

1. Accedé a `http://uptime.localhost` (o el dominio que configures)
2. Configurá monitores para cada servicio interno
3. Activá notificaciones por email, Telegram, Slack, etc.

### 4. Recursos

- **RAM**: 16 GB mínimo si usas Ollama con modelos de 7B+ parámetros.
- **CPU**: 4+ cores recomendados.
- **Disco**: 50 GB + espacio para modelos Ollama (cada modelo 4-8 GB).
- Para VPS con recursos limitados, levanta solo los servicios necesarios:

```bash
# Solo bases de datos
docker compose up -d postgres mysql mongodb redis

# Solo IA
docker compose up -d ollama qdrant openwebui
```

### 5. Logs y rotación

Configura el logging driver para evitar que los logs llenen el disco:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

Agrega esto a cada servicio o globalmente con un ancla YAML (`x-logging: &logging`) al inicio del archivo.

---

## Comandos útiles

```bash
# Estado de los servicios
docker compose ps

# Logs de un servicio específico
docker compose logs -f postgres

# Verificar healthchecks
docker compose ps --format "table {{.Name}}\t{{.Status}}"

# Detener todo
docker compose down

# Detener y eliminar volúmenes (cuidado: borra datos)
docker compose down -v

# Ejecutar un comando dentro de un servicio
docker compose exec postgres psql -U appuser -d appdb

# Descargar un modelo en Ollama
docker compose exec ollama ollama pull llama3.2

# Backup de PostgreSQL
docker compose exec postgres pg_dump -U appuser appdb > backup.sql

# Ver dashboard de Traefik
open http://127.0.0.1:8080
```

---

## Licencia

MIT
# kit-for-dev-backend
