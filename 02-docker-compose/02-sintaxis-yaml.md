# 📝 Sintaxis YAML de Docker Compose

## Estructura Básica

```yaml
# Servicios (contenedores)
services:
  nombre-servicio:
    # configuración...

# Volúmenes persistentes
volumes:
  nombre-volumen:

# Redes personalizadas
networks:
  nombre-red:

# Secrets (para datos sensibles)
secrets:
  nombre-secret:

# Configs (para archivos de configuración)
configs:
  nombre-config:
```

---

## 🔧 Configuración de Servicios

### Imagen vs Build

```yaml
services:
  # Usar imagen existente
  nginx:
    image: nginx:alpine

  # Construir desde Dockerfile
  app:
    build: ./app

  # Build con más opciones
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.prod
      args:
        - VERSION=1.0
```

### Puertos

```yaml
services:
  web:
    image: nginx
    ports:
      # host:contenedor
      - "8080:80"
      - "443:443"
      
      # Solo puerto del contenedor (puerto aleatorio en host)
      - "80"
      
      # Especificar IP
      - "127.0.0.1:8080:80"
      
      # Rango de puertos
      - "3000-3005:3000-3005"
```

### Variables de Entorno

```yaml
services:
  app:
    image: mi-app
    environment:
      # Forma lista
      - NODE_ENV=production
      - DB_HOST=database
      - DEBUG=false
      
    # O forma diccionario
    environment:
      NODE_ENV: production
      DB_HOST: database
      DEBUG: "false"  # Strings entre comillas

    # Desde archivo
    env_file:
      - .env
      - .env.local
```

### Volúmenes

```yaml
services:
  db:
    image: mysql:8.0
    volumes:
      # Named volume
      - mysql-data:/var/lib/mysql
      
      # Bind mount (desarrollo)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      
      # Solo lectura
      - ./config:/etc/mysql/conf.d:ro
      
      # Volumen anónimo
      - /var/lib/mysql

volumes:
  mysql-data:  # Declarar named volumes
```

### Redes

```yaml
services:
  web:
    image: nginx
    networks:
      - frontend
      - backend

  api:
    image: mi-api
    networks:
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
    driver: bridge
```

### Dependencias

```yaml
services:
  web:
    image: nginx
    depends_on:
      - api
      - db
    # El servicio web esperará a que api y db INICIEN
    # (no garantiza que estén listos)

  # Con condiciones (más control)
  api:
    build: ./api
    depends_on:
      db:
        condition: service_healthy
        
  db:
    image: postgres
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Restart Policy

```yaml
services:
  app:
    image: mi-app
    restart: unless-stopped  # Recomendado para producción
    
# Opciones:
# - "no"            : Nunca reiniciar
# - always          : Siempre reiniciar
# - on-failure      : Solo si falla
# - unless-stopped  : Siempre, excepto si se detuvo manualmente
```

### Healthcheck

```yaml
services:
  api:
    image: mi-api
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Límites de Recursos

```yaml
services:
  app:
    image: mi-app
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## 📦 Ejemplo Completo

```yaml
# docker-compose.yml
services:
  # Frontend
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8080
    volumes:
      - ./frontend/src:/app/src  # Hot reload
    depends_on:
      - api
    networks:
      - frontend-net

  # API Backend
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - frontend-net
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Base de datos
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Cache
  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend-net
    restart: unless-stopped

# Volúmenes
volumes:
  postgres-data:
  redis-data:

# Redes
networks:
  frontend-net:
  backend-net:
```

---

## 🔀 Archivos Override

Docker Compose automáticamente combina `docker-compose.yml` con `docker-compose.override.yml`:

**docker-compose.yml** (base):
```yaml
services:
  api:
    build: ./api
    environment:
      - NODE_ENV=production
```

**docker-compose.override.yml** (desarrollo):
```yaml
services:
  api:
    volumes:
      - ./api/src:/app/src  # Hot reload
    environment:
      - NODE_ENV=development
      - DEBUG=true
    ports:
      - "9229:9229"  # Debug port
```

```bash
# Usa ambos archivos automáticamente
docker compose up

# Especificar archivos manualmente
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## 📋 Referencia Rápida

| Propiedad | Descripción |
|-----------|-------------|
| `image` | Imagen a usar |
| `build` | Path al Dockerfile |
| `ports` | Mapeo de puertos |
| `volumes` | Volúmenes y bind mounts |
| `environment` | Variables de entorno |
| `env_file` | Archivos con variables |
| `networks` | Redes a conectar |
| `depends_on` | Dependencias de inicio |
| `restart` | Política de reinicio |
| `healthcheck` | Verificación de salud |
| `command` | Override del CMD |
| `entrypoint` | Override del ENTRYPOINT |
| `working_dir` | Directorio de trabajo |
| `user` | Usuario de ejecución |
| `deploy` | Configuración de despliegue |

---

**[← Anterior: Introducción](01-introduccion-compose.md)** | **[Siguiente: Multi-container →](03-multi-container.md)**
