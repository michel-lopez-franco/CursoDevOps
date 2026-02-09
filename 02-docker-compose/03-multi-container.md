# 🔗 Aplicaciones Multi-Contenedor

## Concepto

Las aplicaciones modernas suelen estar compuestas por múltiples servicios que trabajan juntos. Docker Compose facilita la orquestación de estos servicios.

```
┌──────────────────────────────────────────────────────────────┐
│                    Aplicación Multi-tier                     │
│                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │ Frontend│───►│   API   │───►│   DB    │    │  Cache  │   │
│  │ (React) │    │ (Node)  │    │(Postgres)│◄──│ (Redis) │   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│       │              │              │              │         │
│       └──────────────┴──────────────┴──────────────┘         │
│                    Docker Network                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 🌐 Comunicación entre Servicios

### DNS Interno de Docker

Docker Compose crea automáticamente una red donde cada servicio es accesible **por su nombre**.

```yaml
services:
  api:
    image: mi-api
    environment:
      # Usar nombre del servicio como hostname
      - DB_HOST=db
      - REDIS_HOST=cache

  db:
    image: postgres:15

  cache:
    image: redis:7
```

```javascript
// Dentro del servicio "api", puedes conectar a:
// - db:5432     (PostgreSQL)
// - cache:6379  (Redis)
```

---

## 📦 Ejemplo: Stack Flask + Redis + PostgreSQL

### Estructura del Proyecto

```
flask-app/
├── docker-compose.yml
├── .env
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
└── db/
    └── init.sql
```

### docker-compose.yml

```yaml
services:
  # API Flask
  api:
    build: ./api
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
      - DATABASE_URL=postgresql://user:password@db:5432/appdb
      - REDIS_URL=redis://cache:6379/0
    volumes:
      - ./api:/app
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  # PostgreSQL
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: appdb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis Cache
  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    restart: unless-stopped

  # Adminer (gestión de DB - desarrollo)
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  postgres-data:
  redis-data:
```

### api/Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0"]
```

### api/requirements.txt

```
flask==2.3.0
psycopg2-binary==2.9.9
redis==5.0.0
```

### api/app.py

```python
import os
from flask import Flask, jsonify
import redis
import psycopg2

app = Flask(__name__)

# Conexión a Redis
redis_client = redis.from_url(os.environ.get('REDIS_URL', 'redis://localhost:6379'))

# Conexión a PostgreSQL
def get_db_connection():
    return psycopg2.connect(os.environ.get('DATABASE_URL'))

@app.route('/')
def index():
    # Incrementar contador en Redis
    visits = redis_client.incr('visits')
    return jsonify({
        'message': 'Hola desde Flask + Docker!',
        'visits': int(visits)
    })

@app.route('/users')
def users():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT * FROM users;')
    users = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(users)

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### db/init.sql

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');
```

---

## 🚀 Ejecutar la Aplicación

```bash
# Levantar todos los servicios
docker compose up -d

# Ver estado
docker compose ps

# Ver logs de la API
docker compose logs -f api

# Probar endpoints
curl http://localhost:5000/
curl http://localhost:5000/users

# Acceder a Adminer
# http://localhost:8080
# Server: db, User: user, Password: password, Database: appdb

# Detener
docker compose down
```

---

## 📊 Ejemplo: Escalado Horizontal

```yaml
services:
  web:
    image: nginx
    # No exponer puerto fijo para escalar
    expose:
      - "80"
    deploy:
      replicas: 3

  loadbalancer:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
```

```bash
# Escalar el servicio web
docker compose up -d --scale web=5

# Ver instancias
docker compose ps
```

---

## 🔄 Patrones Comunes

### 1. Frontend + Backend + DB

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

### 2. Microservicios con API Gateway

```yaml
services:
  gateway:
    image: nginx
    ports:
      - "80:80"
    depends_on:
      - users-api
      - products-api
      - orders-api

  users-api:
    build: ./services/users
    
  products-api:
    build: ./services/products
    
  orders-api:
    build: ./services/orders
```

### 3. Workers y Colas

```yaml
services:
  web:
    build: ./web
    ports:
      - "8080:8080"

  worker:
    build: ./worker
    deploy:
      replicas: 3

  queue:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
```

---

**[← Anterior: Sintaxis YAML](02-sintaxis-yaml.md)** | **[Siguiente: Variables y Secrets →](04-variables-secrets.md)**
