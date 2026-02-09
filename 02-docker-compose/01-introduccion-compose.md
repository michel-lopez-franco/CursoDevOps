# 🐙 Introducción a Docker Compose

## ¿Qué es Docker Compose?

**Docker Compose** es una herramienta para definir y ejecutar aplicaciones Docker multi-contenedor. Con un solo archivo YAML (`docker-compose.yml`) puedes configurar todos los servicios de tu aplicación y levantarlos con un solo comando.

## 🤔 ¿Por qué usar Docker Compose?

### Sin Docker Compose (manera manual)

```bash
# Crear red
docker network create mi-red

# Crear volumen
docker volume create mysql-data

# Ejecutar MySQL
docker run -d \
  --name mysql-db \
  --network mi-red \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=app \
  mysql:8.0

# Ejecutar aplicación
docker run -d \
  --name mi-app \
  --network mi-red \
  -p 8080:80 \
  -e DB_HOST=mysql-db \
  mi-app:latest

# Para detener
docker stop mi-app mysql-db
docker rm mi-app mysql-db
```

### Con Docker Compose ✨

```yaml
# docker-compose.yml
services:
  mysql-db:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: app

  mi-app:
    image: mi-app:latest
    ports:
      - "8080:80"
    environment:
      DB_HOST: mysql-db
    depends_on:
      - mysql-db

volumes:
  mysql-data:
```

```bash
# Levantar todo
docker compose up -d

# Detener todo
docker compose down
```

---

## ⚡ Beneficios de Docker Compose

| Beneficio | Descripción |
|-----------|-------------|
| **Declarativo** | Define tu infraestructura como código |
| **Reproducible** | Mismo resultado en cualquier máquina |
| **Versionable** | El archivo YAML va en control de versiones |
| **Simplificado** | Un comando para levantar/detener todo |
| **Networking automático** | Los servicios se pueden comunicar por nombre |

---

## 🔨 Comandos Esenciales

```bash
# Levantar servicios (en primer plano)
docker compose up

# Levantar en segundo plano (detached)
docker compose up -d

# Ver logs
docker compose logs

# Seguir logs en tiempo real
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs mi-app

# Ver estado de servicios
docker compose ps

# Detener servicios
docker compose stop

# Detener y eliminar contenedores, redes
docker compose down

# Detener, eliminar contenedores, redes Y volúmenes
docker compose down -v

# Reconstruir imágenes
docker compose build

# Levantar reconstruyendo
docker compose up -d --build

# Escalar un servicio
docker compose up -d --scale web=3

# Ejecutar comando en un servicio
docker compose exec mi-app bash

# Ver configuración procesada
docker compose config
```

---

## 📁 Estructura de un Proyecto

```
mi-proyecto/
├── docker-compose.yml      # Definición de servicios
├── docker-compose.override.yml  # Overrides locales (opcional)
├── docker-compose.prod.yml # Config de producción (opcional)
├── .env                    # Variables de entorno
├── app/
│   ├── Dockerfile
│   └── src/
└── nginx/
    ├── Dockerfile
    └── nginx.conf
```

---

## 🚀 Ejemplo: WordPress + MySQL

```yaml
# docker-compose.yml
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: secret
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress-data:/var/www/html
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: rootsecret
    volumes:
      - db-data:/var/lib/mysql
    restart: unless-stopped

volumes:
  wordpress-data:
  db-data:
```

```bash
# Levantar
docker compose up -d

# Acceder en http://localhost:8080

# Ver logs
docker compose logs -f wordpress

# Detener
docker compose down

# Detener y eliminar datos
docker compose down -v
```

---

## 🔄 Ciclo de Desarrollo con Compose

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│    Editar código ──► docker compose up --build               │
│         │                      │                             │
│         │                      ▼                             │
│         │              Ver cambios ──► Iterar                │
│         │                      │                             │
│         │                      ▼                             │
│         └────────────── docker compose down                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📋 Versiones de Compose

| Versión | Características |
|---------|-----------------|
| v1 (legacy) | `docker-compose` (con guión) |
| v2+ (actual) | `docker compose` (sin guión, plugin integrado) |

> 💡 **Nota**: Ya no es necesario especificar `version: "3"` en los archivos compose modernos.

---

**[← Volver al módulo](README.md)** | **[Siguiente: Sintaxis YAML →](02-sintaxis-yaml.md)**
