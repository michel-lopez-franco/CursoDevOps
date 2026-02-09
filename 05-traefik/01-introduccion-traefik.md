# 🦊 Introducción a Traefik

## ¿Qué es Traefik?

**Traefik** es un edge router y reverse proxy moderno diseñado específicamente para microservicios y contenedores. Se integra nativamente con Docker, Kubernetes, y otros orquestadores.

```
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │    Traefik    │
                    │               │
                    │  • Routing    │
                    │  • SSL/TLS    │
                    │  • Load Bal.  │
                    │  • Middlewares│
                    └───────┬───────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ api.com │        │ app.com │        │admin.com│
   │ Backend │        │Frontend │        │  Admin  │
   └─────────┘        └─────────┘        └─────────┘
```

---

## 🤔 ¿Por qué usar Traefik?

### Comparación con Nginx

| Característica | Nginx | Traefik |
|---------------|-------|---------|
| Configuración | Archivos estáticos | Dinámica (labels Docker) |
| Service Discovery | Manual | Automático |
| SSL Let's Encrypt | Requiere certbot | Integrado |
| Reload config | Requiere reload | Sin downtime |
| Dashboard | Terceros | Incluido |
| Curva aprendizaje | Media | Baja-Media |

### Ventajas Principales

1. **Configuración Dinámica**: Los servicios se registran automáticamente
2. **Service Discovery**: Detecta contenedores nuevos sin reiniciar
3. **SSL Automático**: Obtiene y renueva certificados Let's Encrypt
4. **Middlewares**: Auth, rate limiting, headers, compression, etc.
5. **Dashboard**: Visualización en tiempo real de rutas y servicios
6. **Cloud Native**: Diseñado para Docker, Kubernetes, etc.

---

## 🏗️ Arquitectura de Traefik

```
┌─────────────────────────────────────────────────────────────────┐
│                           TRAEFIK                               │
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │
│  │ Entrypoints │   │   Routers   │   │      Services       │   │
│  │             │   │             │   │                     │   │
│  │  :80 (web)  │──►│  Rules      │──►│  LoadBalancer      │   │
│  │  :443 (wss) │   │  Middleware │   │  Mirrors           │   │
│  │  :8080 (api)│   │  TLS        │   │  Weighted          │   │
│  └─────────────┘   └─────────────┘   └─────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      Providers                           │   │
│  │   Docker  │  Kubernetes  │  File  │ Consul │ etcd       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Componentes Clave

| Componente | Descripción |
|------------|-------------|
| **Entrypoints** | Puertos donde Traefik escucha (80, 443, etc.) |
| **Routers** | Reglas para dirigir el tráfico (por dominio, path, etc.) |
| **Services** | Los backends a los que se envía el tráfico |
| **Middlewares** | Modificadores de request/response |
| **Providers** | Fuentes de configuración (Docker, archivos, etc.) |

---

## 🚀 Ejemplo Mínimo

### docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
```

### Ejecutar

```bash
# Levantar
docker compose up -d

# Probar servicio
curl -H "Host: whoami.localhost" http://localhost
# O añadir a /etc/hosts: 127.0.0.1 whoami.localhost
curl http://whoami.localhost

# Ver dashboard
# http://localhost:8080
```

---

## 🔄 Flujo de una Request

```
1. Request llega a Entrypoint (:80)
        │
        ▼
2. Router evalúa reglas (Host, Path, Headers...)
        │
        ▼
3. Middlewares procesan la request (auth, compress...)
        │
        ▼
4. Service recibe la request (contenedor)
        │
        ▼
5. Response vuelve por el mismo camino
```

---

## 📋 Conceptos Clave

### Entrypoints

Puntos de entrada donde Traefik acepta conexiones:

```yaml
# En línea de comandos o traefik.yml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
```

### Routers

Conectan entrypoints con services basándose en reglas:

```yaml
# Vía labels de Docker
labels:
  - "traefik.http.routers.mi-router.rule=Host(`app.ejemplo.com`)"
  - "traefik.http.routers.mi-router.entrypoints=websecure"
  - "traefik.http.routers.mi-router.tls=true"
```

### Services

Backend services (contenedores Docker):

```yaml
labels:
  - "traefik.http.services.mi-servicio.loadbalancer.server.port=3000"
```

### Middlewares

Modifican requests/responses:

```yaml
labels:
  # Redirección HTTP → HTTPS
  - "traefik.http.middlewares.redirect.redirectscheme.scheme=https"
  
  # Autenticación básica
  - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."
  
  # Headers de seguridad
  - "traefik.http.middlewares.secure.headers.framedeny=true"
```

---

## 🌐 Reglas de Routing

| Regla | Ejemplo | Descripción |
|-------|---------|-------------|
| `Host` | `Host(\`app.com\`)` | Por dominio |
| `Path` | `Path(\`/api\`)` | Por ruta exacta |
| `PathPrefix` | `PathPrefix(\`/api\`)` | Por prefijo de ruta |
| `Headers` | `Headers(\`X-Custom\`, \`value\`)` | Por header |
| `Method` | `Method(\`POST\`)` | Por método HTTP |
| `Query` | `Query(\`foo=bar\`)` | Por query string |

### Combinaciones

```yaml
labels:
  # Host Y PathPrefix
  - "traefik.http.routers.api.rule=Host(`api.ejemplo.com`) && PathPrefix(`/v1`)"
  
  # Host O Host
  - "traefik.http.routers.web.rule=Host(`www.ejemplo.com`) || Host(`ejemplo.com`)"
```

---

## 📚 Próximos Pasos

En los siguientes capítulos veremos:
1. Configuración detallada 
2. Labels y routing avanzado
3. SSL con Let's Encrypt
4. Dashboard y monitoreo

---

**[← Volver al módulo](README.md)** | **[Siguiente: Configuración Básica →](02-configuracion-basica.md)**
