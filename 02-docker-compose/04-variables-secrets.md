# 🔐 Variables de Entorno y Secrets

## Variables de Entorno

### Métodos para Definir Variables

#### 1. En el docker-compose.yml

```yaml
services:
  app:
    image: mi-app
    environment:
      - NODE_ENV=production
      - API_KEY=mi-clave-api
      - DEBUG=false
```

#### 2. Archivo .env (Recomendado)

**.env**
```bash
# Configuración de la aplicación
NODE_ENV=production
API_PORT=3000

# Base de datos
DB_HOST=db
DB_PORT=5432
DB_NAME=miapp
DB_USER=admin
DB_PASSWORD=secreto123

# Redis
REDIS_URL=redis://cache:6379
```

**docker-compose.yml**
```yaml
services:
  app:
    image: mi-app
    ports:
      - "${API_PORT}:3000"
    environment:
      - NODE_ENV=${NODE_ENV}
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}
      - REDIS_URL=${REDIS_URL}
    
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
```

#### 3. Archivo env_file Separado

**database.env**
```bash
POSTGRES_USER=admin
POSTGRES_PASSWORD=secreto123
POSTGRES_DB=miapp
```

**docker-compose.yml**
```yaml
services:
  db:
    image: postgres:15
    env_file:
      - ./database.env
```

---

## 🔒 Buenas Prácticas de Seguridad

### ❌ NO hacer (inseguro)

```yaml
# ¡NO! Contraseñas en texto plano en el compose
services:
  db:
    environment:
      - MYSQL_ROOT_PASSWORD=mi_contraseña_super_secreta
```

### ✅ SÍ hacer (seguro)

```yaml
# Usar variables de entorno
services:
  db:
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
```

### .gitignore

```gitignore
# Nunca subir archivos con secretos
.env
.env.local
.env.production
*.env
secrets/
```

---

## 🗝️ Docker Secrets (Swarm/Producción)

Para entornos de producción, Docker Secrets proporciona almacenamiento seguro.

### Crear Secrets

```bash
# Desde archivo
echo "mi_contraseña_super_segura" | docker secret create db_password -

# Desde archivo
docker secret create db_password ./password.txt

# Listar secrets
docker secret ls
```

### Usar en Compose (modo Swarm)

```yaml
services:
  db:
    image: postgres:15
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true  # Secret creado externamente
    
  # O desde archivo local
  api_key:
    file: ./secrets/api_key.txt
```

### Acceso en el Contenedor

Los secrets se montan en `/run/secrets/`:

```bash
# Dentro del contenedor
cat /run/secrets/db_password
```

```python
# En código Python
with open('/run/secrets/db_password', 'r') as f:
    password = f.read().strip()
```

---

## 📁 Configs (Archivos de Configuración)

Para archivos de configuración que no son secretos pero necesitan ser gestionados.

```yaml
services:
  nginx:
    image: nginx
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf

configs:
  nginx_config:
    file: ./nginx.conf
```

---

## 🔄 Variables por Ambiente

### Estructura Recomendada

```
proyecto/
├── docker-compose.yml          # Base
├── docker-compose.override.yml # Desarrollo (auto-cargado)
├── docker-compose.prod.yml     # Producción
├── .env                        # Variables desarrollo
├── .env.example               # Plantilla (sí commitear)
└── .env.production            # Variables producción (no commitear)
```

### .env.example (para documentación)

```bash
# Copiar a .env y completar valores
NODE_ENV=development

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=

# API Keys
API_SECRET_KEY=
THIRD_PARTY_API_KEY=
```

### Cargar Variables Específicas

```bash
# Desarrollo (usa .env automáticamente)
docker compose up

# Producción con variables específicas
docker compose --env-file .env.production -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 💡 Ejemplo Completo: Ambiente Seguro

### .env

```bash
# Desarrollo local
COMPOSE_PROJECT_NAME=miapp_dev
ENVIRONMENT=development

# Puertos
APP_PORT=3000
DB_PORT=5432

# Database
DB_NAME=miapp
DB_USER=developer
DB_PASSWORD=dev_password_123

# Redis
REDIS_PASSWORD=redis_dev_123

# API
JWT_SECRET=dev_jwt_secret_change_in_prod
API_KEY=dev_api_key
```

### docker-compose.yml

```yaml
services:
  app:
    build: ./app
    ports:
      - "${APP_PORT}:3000"
    environment:
      - NODE_ENV=${ENVIRONMENT}
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@cache:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - db
      - cache

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

---

## 📋 Resumen

| Método | Uso | Seguridad |
|--------|-----|-----------|
| `environment:` | Variables simples, desarrollo | Baja |
| `.env` + `${VAR}` | Configuración por ambiente | Media |
| `env_file:` | Múltiples archivos de config | Media |
| Docker Secrets | Producción, datos sensibles | Alta |
| Configs | Archivos de configuración | Media |

---

**[← Anterior: Multi-container](03-multi-container.md)** | **[Volver al módulo](README.md)**
