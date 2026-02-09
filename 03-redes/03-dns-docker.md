# 🌐 DNS Interno de Docker

## ¿Cómo Funciona?

Docker tiene un servidor DNS interno (127.0.0.11) que resuelve nombres de contenedores y servicios automáticamente dentro de redes bridge personalizadas.

```
┌─────────────────────────────────────────────────────────────┐
│                      Red Docker                             │
│                                                             │
│   ┌───────────┐     DNS Query: "db"     ┌───────────────┐  │
│   │           │ ──────────────────────► │  Docker DNS   │  │
│   │    api    │                         │  127.0.0.11   │  │
│   │           │ ◄────────────────────── │               │  │
│   └───────────┘    Response: 172.18.0.3 └───────────────┘  │
│         │                                       │           │
│         │        connect to 172.18.0.3:5432     │           │
│         ▼                                       │           │
│   ┌───────────┐                                 │           │
│   │    db     │ ◄───────────────────────────────┘           │
│   │172.18.0.3 │                                             │
│   └───────────┘                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 Reglas del DNS de Docker

### ✅ Funciona en:
- Redes bridge personalizadas (`docker network create`)
- Docker Compose (crea red automáticamente)

### ❌ NO funciona en:
- Red bridge por defecto (`docker0`)
- Red host
- Red none

---

## 📝 Resolución de Nombres

### En Docker Compose

```yaml
services:
  web:
    image: nginx
    # Puede resolver: api, database, cache
    
  api:
    image: mi-api
    # Puede resolver: web, database, cache
    
  database:
    image: postgres
    
  cache:
    image: redis
```

Docker Compose resuelve:
- **Nombre del servicio**: `database` → IP del contenedor
- **Nombre del contenedor**: `proyecto_database_1` → IP del contenedor

### En Docker CLI

```bash
# Crear red personalizada
docker network create mi-red

# Los contenedores con --name son resolvibles
docker run -d --name db --network mi-red postgres
docker run -d --name api --network mi-red mi-api

# 'db' es resolvible desde 'api'
docker exec api ping db
```

---

## 🔧 Aliases de Red

Puedes dar múltiples nombres (aliases) a un servicio:

```yaml
services:
  database:
    image: postgres
    networks:
      default:
        aliases:
          - db
          - postgres-master
          - primary-db
```

```bash
# Todos estos nombres resuelven al mismo contenedor
ping database
ping db
ping postgres-master
ping primary-db
```

---

## 🏷️ Hostname Personalizado

```yaml
services:
  api:
    image: mi-api
    hostname: api-server  # Nombre interno del host
    domainname: example.local
    # FQDN: api-server.example.local
```

---

## 📊 Ejemplo: Balanceo DNS Round-Robin

Docker Compose con servicios escalados automáticamente balancea por DNS:

```yaml
services:
  lb:
    image: nginx
    ports:
      - "80:80"
    depends_on:
      - web

  web:
    image: mi-app
    deploy:
      replicas: 3
```

```bash
# Escalar el servicio
docker compose up -d --scale web=5

# El DNS de 'web' retorna IPs de todas las instancias (round-robin)
docker compose exec lb nslookup web

# Respuesta:
# Name: web
# Address: 172.20.0.3
# Address: 172.20.0.4
# Address: 172.20.0.5
# Address: 172.20.0.6
# Address: 172.20.0.7
```

---

## 🔍 Debugging DNS

```bash
# Verificar que el DNS funciona
docker compose exec api nslookup database

# Salida esperada:
# Server:    127.0.0.11
# Address:   127.0.0.11#53
# 
# Non-authoritative answer:
# Name: database
# Address: 172.20.0.2

# Verificar resolución con dig (si está instalado)
docker compose exec api dig database +short

# Verificar conectividad
docker compose exec api ping -c 3 database

# Ver configuración DNS del contenedor
docker compose exec api cat /etc/resolv.conf
# nameserver 127.0.0.11
```

---

## ⚠️ Problemas Comunes

### 1. "Could not resolve host"

```bash
# Verificar que están en la misma red
docker network inspect mi-red
```

### 2. Red bridge por defecto no resuelve

```bash
# ❌ No funciona
docker run -d --name db postgres
docker run -it alpine ping db  # Error

# ✅ Funciona con red personalizada
docker network create mi-red
docker run -d --name db --network mi-red postgres
docker run -it --network mi-red alpine ping db  # OK
```

### 3. Conflictos de nombres

```bash
# Si hay conflicto, usar nombre completo del contenedor
docker compose exec api ping proyecto_database_1
```

---

## 📋 Resumen

| Contexto | DNS Automático | Método de Acceso |
|----------|----------------|------------------|
| Docker Compose | ✅ Sí | Nombre del servicio |
| Red bridge personalizada | ✅ Sí | Nombre del contenedor (--name) |
| Red bridge default | ❌ No | IP del contenedor |
| Red host | ❌ No | localhost |

---

**[← Anterior: Comunicación](02-comunicacion-contenedores.md)** | **[Volver al módulo](README.md)**
