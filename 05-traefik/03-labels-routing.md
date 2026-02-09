# 🏷️ Labels y Routing en Traefik

## Estructura de Labels

Las labels de Traefik siguen un patrón específico:

```
traefik.<protocolo>.<tipo>.<nombre>.<propiedad>=<valor>
```

- **protocolo**: `http`, `tcp`, `udp`
- **tipo**: `routers`, `services`, `middlewares`
- **nombre**: Identificador único
- **propiedad**: Configuración específica

---

## 🔀 Routers

### Reglas de Matching

```yaml
labels:
  # Por Host (dominio)
  - "traefik.http.routers.app.rule=Host(`app.ejemplo.com`)"
  
  # Por Path exacto
  - "traefik.http.routers.api.rule=Path(`/api/health`)"
  
  # Por PathPrefix
  - "traefik.http.routers.api.rule=PathPrefix(`/api`)"
  
  # Combinación con AND
  - "traefik.http.routers.app.rule=Host(`app.com`) && PathPrefix(`/v1`)"
  
  # Combinación con OR
  - "traefik.http.routers.app.rule=Host(`app.com`) || Host(`www.app.com`)"
  
  # Por Headers
  - "traefik.http.routers.app.rule=Headers(`X-Api-Key`, `secret`)"
  
  # Por Query
  - "traefik.http.routers.app.rule=Query(`debug=true`)"
  
  # Por Método
  - "traefik.http.routers.app.rule=Method(`POST`)"
```

### Prioridad

Cuando múltiples routers coinciden, se usa la prioridad (mayor = primero):

```yaml
labels:
  # Ruta específica con mayor prioridad
  - "traefik.http.routers.api-health.rule=PathPrefix(`/api/health`)"
  - "traefik.http.routers.api-health.priority=100"
  
  # Ruta general con menor prioridad
  - "traefik.http.routers.api.rule=PathPrefix(`/api`)"
  - "traefik.http.routers.api.priority=50"
```

---

## 🔧 Middlewares Comunes

### Redirect HTTP → HTTPS

```yaml
labels:
  - "traefik.http.middlewares.redirect-https.redirectscheme.scheme=https"
  - "traefik.http.middlewares.redirect-https.redirectscheme.permanent=true"
  - "traefik.http.routers.app.middlewares=redirect-https"
```

### Autenticación Básica

```bash
# Generar password hash
echo $(htpasswd -nb admin password123) | sed -e s/\\$/\\$\\$/g
# admin:$$apr1$$...
```

```yaml
labels:
  - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$xyz..."
  - "traefik.http.routers.app.middlewares=auth"
```

### Rate Limiting

```yaml
labels:
  - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
  - "traefik.http.routers.api.middlewares=ratelimit"
```

### Headers de Seguridad

```yaml
labels:
  - "traefik.http.middlewares.security.headers.framedeny=true"
  - "traefik.http.middlewares.security.headers.sslredirect=true"
  - "traefik.http.middlewares.security.headers.browserxssfilter=true"
  - "traefik.http.middlewares.security.headers.contenttypenosniff=true"
  - "traefik.http.middlewares.security.headers.customframeoptionsvalue=SAMEORIGIN"
```

### Strip Prefix

Elimina parte del path antes de enviar al backend:

```yaml
labels:
  # /api/users → /users (al backend)
  - "traefik.http.middlewares.strip-api.stripprefix.prefixes=/api"
  - "traefik.http.routers.api.middlewares=strip-api"
```

### Compress

```yaml
labels:
  - "traefik.http.middlewares.compress.compress=true"
  - "traefik.http.routers.app.middlewares=compress"
```

### Múltiples Middlewares

```yaml
labels:
  - "traefik.http.routers.app.middlewares=auth,ratelimit,security,compress"
```

---

## 📦 Services

### Puerto del Contenedor

```yaml
labels:
  # Si el contenedor expone un puerto diferente a 80
  - "traefik.http.services.app.loadbalancer.server.port=3000"
```

### Health Check

```yaml
labels:
  - "traefik.http.services.app.loadbalancer.healthcheck.path=/health"
  - "traefik.http.services.app.loadbalancer.healthcheck.interval=10s"
  - "traefik.http.services.app.loadbalancer.healthcheck.timeout=3s"
```

### Sticky Sessions

```yaml
labels:
  - "traefik.http.services.app.loadbalancer.sticky.cookie=true"
  - "traefik.http.services.app.loadbalancer.sticky.cookie.name=server_id"
```

---

## 💻 Ejemplo Completo: API con Múltiples Rutas

```yaml
services:
  # API principal
  api:
    image: mi-api
    labels:
      - "traefik.enable=true"
      
      # Router para API v1
      - "traefik.http.routers.api-v1.rule=Host(`api.ejemplo.com`) && PathPrefix(`/v1`)"
      - "traefik.http.routers.api-v1.entrypoints=websecure"
      - "traefik.http.routers.api-v1.tls.certresolver=letsencrypt"
      - "traefik.http.routers.api-v1.middlewares=ratelimit,security"
      
      # Router para API v2
      - "traefik.http.routers.api-v2.rule=Host(`api.ejemplo.com`) && PathPrefix(`/v2`)"
      - "traefik.http.routers.api-v2.entrypoints=websecure"
      - "traefik.http.routers.api-v2.tls.certresolver=letsencrypt"
      - "traefik.http.routers.api-v2.middlewares=ratelimit,security"
      
      # Service
      - "traefik.http.services.api.loadbalancer.server.port=3000"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/health"
      
      # Middlewares
      - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.security.headers.framedeny=true"
    networks:
      - traefik-public

  # Admin Panel (con autenticación)
  admin:
    image: mi-admin
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.admin.rule=Host(`admin.ejemplo.com`)"
      - "traefik.http.routers.admin.entrypoints=websecure"
      - "traefik.http.routers.admin.tls.certresolver=letsencrypt"
      - "traefik.http.routers.admin.middlewares=admin-auth"
      
      # Autenticación básica
      - "traefik.http.middlewares.admin-auth.basicauth.users=admin:$$apr1$$..."
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

---

## 📋 Cheatsheet de Labels

| Propósito | Label |
|-----------|-------|
| Habilitar | `traefik.enable=true` |
| Regla | `traefik.http.routers.X.rule=Host(\`...\`)` |
| Entrypoint | `traefik.http.routers.X.entrypoints=websecure` |
| TLS | `traefik.http.routers.X.tls=true` |
| Cert Resolver | `traefik.http.routers.X.tls.certresolver=le` |
| Puerto | `traefik.http.services.X.loadbalancer.server.port=3000` |
| Middleware | `traefik.http.routers.X.middlewares=auth,rate` |

---

**[← Anterior: Configuración](02-configuracion-basica.md)** | **[Siguiente: SSL Let's Encrypt →](04-ssl-letsencrypt.md)**
