# 🏗️ Arquitectura de Microservicios

## ¿Qué son los Microservicios?

Los **microservicios** son un estilo arquitectónico donde una aplicación se divide en **servicios pequeños, independientes y desplegables por separado**.

```
┌─────────────────────────────────────────────────────────────┐
│                       MONOLITO                              │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   Una sola aplicación               │   │
│   │   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │   │
│   │   │Users │ │Orders│ │Prods │ │Notify│ │ Pay  │    │   │
│   │   └──────┘ └──────┘ └──────┘ └──────┘ └──────┘    │   │
│   │              Todo acoplado                         │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

                           vs

┌─────────────────────────────────────────────────────────────┐
│                     MICROSERVICIOS                          │
│                                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│   │ Users   │  │ Orders  │  │Products │  │Payments │       │
│   │Service  │  │ Service │  │ Service │  │ Service │       │
│   │  🐳     │  │   🐳    │  │   🐳    │  │   🐳    │       │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘       │
│        │            │            │            │             │
│        └────────────┴──────┬─────┴────────────┘             │
│                            │                                │
│                    ┌───────┴───────┐                       │
│                    │  API Gateway  │                       │
│                    │     🐳       │                       │
│                    └───────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚖️ Monolito vs Microservicios

| Aspecto | Monolito | Microservicios |
|---------|----------|----------------|
| **Despliegue** | Todo junto | Independiente |
| **Escalado** | Toda la app | Por servicio |
| **Tecnología** | Una sola | Polyglot |
| **Complejidad** | Simple al inicio | Mayor operacional |
| **Equipos** | Un equipo | Múltiples equipos |
| **Fallo** | Afecta todo | Aislado |

---

## 🎯 Principios Clave

### 1. Responsabilidad Única
Cada servicio hace UNA cosa bien.

```yaml
# ❌ Malo: servicio que hace todo
services:
  monolith:
    image: mi-app-todo-en-uno

# ✅ Bueno: servicios separados
services:
  users:
    image: mi-app/users
  orders:
    image: mi-app/orders
  products:
    image: mi-app/products
```

### 2. Independencia de Datos
Cada servicio tiene su propia base de datos.

```yaml
services:
  users:
    image: users-service
  users-db:
    image: postgres
    
  orders:
    image: orders-service
  orders-db:
    image: mongodb
    
  products:
    image: products-service
  products-db:
    image: mysql
```

### 3. Comunicación por API
Los servicios se comunican vía HTTP/REST, gRPC o mensajes.

### 4. Despliegue Independiente
Cada servicio puede actualizarse sin afectar a otros.

---

## 🏛️ Patrones de Arquitectura

### API Gateway

Punto de entrada único que enruta a los servicios.

```yaml
services:
  gateway:
    image: traefik:v3.0
    ports:
      - "80:80"
    labels:
      - "traefik.enable=true"
      
  users:
    labels:
      - "traefik.http.routers.users.rule=PathPrefix(`/api/users`)"
      
  orders:
    labels:
      - "traefik.http.routers.orders.rule=PathPrefix(`/api/orders`)"
```

### Backend for Frontend (BFF)

APIs específicas para cada frontend.

```yaml
services:
  # BFF para app móvil
  mobile-bff:
    image: mobile-bff
    labels:
      - "traefik.http.routers.mobile.rule=Host(`mobile-api.ejemplo.com`)"

  # BFF para web
  web-bff:
    image: web-bff
    labels:
      - "traefik.http.routers.web.rule=Host(`web-api.ejemplo.com`)"
```

### Sidecar Pattern

Funcionalidad auxiliar junto al servicio principal.

```yaml
services:
  app:
    image: mi-app
    
  # Sidecar para logs
  log-agent:
    image: fluentd
    volumes:
      - app-logs:/var/log/app
```

---

## 💻 Ejemplo: E-commerce con Microservicios

```yaml
services:
  # API Gateway
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - public
      - internal

  # Frontend
  frontend:
    build: ./frontend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`shop.ejemplo.com`)"
    networks:
      - public

  # Servicio de Usuarios
  users:
    build: ./services/users
    environment:
      - DATABASE_URL=postgres://user:pass@users-db:5432/users
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.users.rule=Host(`api.ejemplo.com`) && PathPrefix(`/users`)"
    depends_on:
      - users-db
    networks:
      - public
      - internal

  users-db:
    image: postgres:15-alpine
    volumes:
      - users-data:/var/lib/postgresql/data
    networks:
      - internal

  # Servicio de Productos
  products:
    build: ./services/products
    environment:
      - MONGO_URL=mongodb://products-db:27017/products
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.products.rule=Host(`api.ejemplo.com`) && PathPrefix(`/products`)"
    networks:
      - public
      - internal

  products-db:
    image: mongo:6
    volumes:
      - products-data:/data/db
    networks:
      - internal

  # Servicio de Pedidos
  orders:
    build: ./services/orders
    environment:
      - DATABASE_URL=postgres://user:pass@orders-db:5432/orders
      - USERS_SERVICE_URL=http://users:3000
      - PRODUCTS_SERVICE_URL=http://products:3000
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.orders.rule=Host(`api.ejemplo.com`) && PathPrefix(`/orders`)"
    networks:
      - public
      - internal

  orders-db:
    image: postgres:15-alpine
    volumes:
      - orders-data:/var/lib/postgresql/data
    networks:
      - internal

  # Cache compartida
  redis:
    image: redis:7-alpine
    networks:
      - internal

networks:
  public:
  internal:

volumes:
  users-data:
  products-data:
  orders-data:
```

---

## ✅ Buenas Prácticas

1. **Empezar simple**: No microservicios desde el día 1
2. **Límites claros**: Definir bien qué hace cada servicio
3. **Monitoreo**: Imprescindible con muchos servicios
4. **Circuit breaker**: Para manejar fallos
5. **Logs centralizados**: ELK, Loki, etc.

---

**[← Volver al módulo](README.md)** | **[Siguiente: Patrones →](02-patrones-comunicacion.md)**
