# 🔗 Comunicación entre Contenedores

## Service Discovery Automático

Cuando usas Docker Compose o redes bridge personalizadas, los contenedores pueden encontrarse entre sí usando el nombre del servicio como hostname.

```yaml
services:
  api:
    image: mi-api
    environment:
      # Usar nombre del servicio como hostname
      - DATABASE_HOST=db
      - CACHE_HOST=redis

  db:
    image: postgres:15
    
  redis:
    image: redis:7
```

```python
# En el código de la API
import psycopg2

# 'db' se resuelve automáticamente a la IP del contenedor
conn = psycopg2.connect(host='db', port=5432, ...)
```

---

## 🔌 Métodos de Comunicación

### 1. Por Nombre de Servicio (Compose)

```yaml
# docker-compose.yml
services:
  frontend:
    build: ./frontend
    environment:
      - API_URL=http://backend:3000  # Nombre del servicio

  backend:
    build: ./backend
```

### 2. Por Nombre de Contenedor

```bash
# Crear red
docker network create mi-red

# Ejecutar contenedores con nombres
docker run -d --name database --network mi-red postgres:15
docker run -d --name api --network mi-red -e DB_HOST=database mi-api
```

### 3. Por IP (No Recomendado)

```bash
# Obtener IP
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' database
# 172.18.0.2

# Usar IP (puede cambiar!)
docker run -d --network mi-red -e DB_HOST=172.18.0.2 mi-api
```

> ⚠️ Las IPs pueden cambiar al reiniciar contenedores. Siempre usa nombres.

---

## 🌐 Exponer vs Publicar Puertos

### `expose` - Solo interno

```yaml
services:
  api:
    image: mi-api
    expose:
      - "3000"  # Accesible solo dentro de la red Docker
```

### `ports` - Público

```yaml
services:
  api:
    image: mi-api
    ports:
      - "8080:3000"  # host:contenedor - Accesible desde fuera
```

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│                                                             │
│  ┌──────────────┐        ┌──────────────┐                  │
│  │   frontend   │        │     api      │                  │
│  │              │───────►│   :3000      │                  │
│  │   :80        │        │ (expose)     │                  │
│  │ (ports)      │        └──────────────┘                  │
│  └──────────────┘                                          │
│        │                                                    │
│        │ ports: 80:80                                       │
└────────┼────────────────────────────────────────────────────┘
         │
         ▼
      Internet (solo accede a frontend)
```

---

## 🔒 Aislamiento con Múltiples Redes

```yaml
services:
  # Frontend - red pública
  web:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend

  # API - conecta ambas redes
  api:
    image: mi-api
    networks:
      - frontend
      - backend

  # DB - solo red privada
  db:
    image: postgres
    networks:
      - backend

  # Cache - solo red privada
  redis:
    image: redis
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true  # Sin acceso a internet
```

### Resultado

| Servicio | Puede acceder a | Accesible desde |
|----------|-----------------|-----------------|
| web | api | Internet |
| api | web, db, redis | web |
| db | api, redis | api, redis |
| redis | api, db | api, db |

---

## 📡 Ejemplo Práctico: Comunicación API-DB

### docker-compose.yml

```yaml
services:
  api:
    build: ./api
    ports:
      - "8080:3000"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=user
      - DB_PASS=secret
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  db-data:
```

### Verificar Conectividad

```bash
# Levantar servicios
docker compose up -d

# Entrar al contenedor de la API
docker compose exec api sh

# Dentro del contenedor, probar DNS
ping db
# PING db (172.20.0.2): 56 data bytes

# Probar conexión a PostgreSQL
nc -zv db 5432
# Connection to db 5432 port [tcp/postgresql] succeeded!
```

---

## 🔧 Troubleshooting de Red

```bash
# Ver redes
docker network ls

# Inspeccionar red (ver contenedores conectados)
docker network inspect mi-red

# Ver IP de un contenedor
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER

# Ejecutar comandos de red dentro del contenedor
docker exec -it CONTAINER ping otro-contenedor
docker exec -it CONTAINER nslookup db
docker exec -it CONTAINER nc -zv db 5432

# Ver puertos en uso
docker port CONTAINER
```

---

**[← Anterior: Tipos de Redes](01-tipos-redes.md)** | **[Siguiente: DNS Docker →](03-dns-docker.md)**
