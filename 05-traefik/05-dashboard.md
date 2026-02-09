# 📊 Dashboard de Traefik

## Activar el Dashboard

El dashboard de Traefik proporciona una interfaz visual para ver routers, services, middlewares y el estado general.

### Configuración Básica (Desarrollo)

```yaml
# traefik.yml
api:
  dashboard: true
  insecure: true  # Solo para desarrollo local
```

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
      - "8080:8080"  # Dashboard
    # ...
```

Acceder en: `http://localhost:8080`

---

## 🔒 Dashboard Seguro (Producción)

### Con Autenticación Básica

```bash
# Generar hash de contraseña
# Instalar htpasswd: apt-get install apache2-utils
echo $(htpasswd -nb admin tu_password_seguro) | sed -e s/\\$/\\$\\$/g
# Resultado: admin:$$apr1$$xyz...
```

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
    labels:
      - "traefik.enable=true"
      # Router para dashboard
      - "traefik.http.routers.dashboard.rule=Host(`traefik.midominio.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      # Autenticación
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$xyz..."
```

### Con IP Whitelist

```yaml
labels:
  # Solo permitir IPs específicas
  - "traefik.http.middlewares.dashboard-whitelist.ipwhitelist.sourcerange=192.168.1.0/24,10.0.0.0/8"
  - "traefik.http.routers.dashboard.middlewares=dashboard-auth,dashboard-whitelist"
```

---

## 🖥️ Interfaz del Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│                    Traefik Dashboard                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   HTTP      │  │    TCP      │  │    UDP      │         │
│  │  Routers    │  │   Routers   │  │   Routers   │         │
│  │    12       │  │     3       │  │     0       │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   HTTP      │  │    TCP      │  │    UDP      │         │
│  │  Services   │  │  Services   │  │  Services   │         │
│  │    15       │  │     3       │  │     0       │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Middlewares: 8                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Secciones Principales

| Sección | Muestra |
|---------|---------|
| **Overview** | Resumen de todos los componentes |
| **HTTP** | Routers, Services y Middlewares HTTP |
| **TCP** | Routers y Services TCP |
| **UDP** | Routers y Services UDP |
| **Features** | Providers activos, TLS, etc. |

---

## 📡 API de Traefik

El dashboard usa la API REST de Traefik. Puedes consultarla directamente:

```bash
# Listar routers
curl http://localhost:8080/api/http/routers | jq

# Listar services
curl http://localhost:8080/api/http/services | jq

# Listar middlewares
curl http://localhost:8080/api/http/middlewares | jq

# Información del entrypoint
curl http://localhost:8080/api/entrypoints | jq

# Versión de Traefik
curl http://localhost:8080/api/version | jq
```

---

## 📊 Métricas y Monitoreo

### Prometheus

```yaml
# traefik.yml
metrics:
  prometheus:
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0
```

```yaml
# Exponer métricas
services:
  traefik:
    labels:
      - "traefik.http.routers.metrics.rule=PathPrefix(`/metrics`)"
      - "traefik.http.routers.metrics.service=prometheus@internal"
```

### Integración con Grafana

1. Agregar Prometheus como data source en Grafana
2. Importar dashboard oficial de Traefik (ID: 4475 o 12250)

---

## 💻 Ejemplo Completo con Dashboard Seguro

```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
    networks:
      - web
    labels:
      - "traefik.enable=true"
      
      # Dashboard Router
      - "traefik.http.routers.traefik-dashboard.rule=Host(`traefik.midominio.com`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.service=api@internal"
      - "traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt"
      
      # Middlewares de seguridad
      - "traefik.http.routers.traefik-dashboard.middlewares=dashboard-chain"
      
      # Chain de middlewares
      - "traefik.http.middlewares.dashboard-chain.chain.middlewares=dashboard-auth,dashboard-ratelimit"
      
      # Auth
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$ruca84Hq$$mbjdMZBAG.KWn7vfN/SNK/"
      
      # Rate limit
      - "traefik.http.middlewares.dashboard-ratelimit.ratelimit.average=10"
      - "traefik.http.middlewares.dashboard-ratelimit.ratelimit.burst=20"

networks:
  web:
```

### traefik.yml

```yaml
api:
  dashboard: true
  # Desactivar acceso inseguro
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false

certificatesResolvers:
  letsencrypt:
    acme:
      email: tu@email.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web

log:
  level: INFO
```

---

**[← Anterior: SSL](04-ssl-letsencrypt.md)** | **[Volver al módulo](README.md)**
