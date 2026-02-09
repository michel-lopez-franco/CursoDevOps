# ⚙️ Configuración Básica de Traefik

## Estructura de Archivos

```
proyecto/
├── docker-compose.yml
├── traefik/
│   ├── traefik.yml          # Configuración estática
│   ├── dynamic/
│   │   └── config.yml       # Configuración dinámica (opcional)
│   └── acme.json            # Certificados SSL (creado automáticamente)
└── servicios/
    └── ...
```

---

## 🚀 Configuración Básica con Docker Compose

### docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # Socket de Docker (para detectar contenedores)
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Configuración estática
      - ./traefik/traefik.yml:/traefik.yml:ro
      # Configuración dinámica (opcional)
      - ./traefik/dynamic:/dynamic:ro
      # Certificados SSL
      - ./traefik/acme.json:/acme.json
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      # Dashboard (opcional, para producción usar autenticación)
      - "traefik.http.routers.dashboard.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=web"

networks:
  traefik-public:
    external: true
```

### traefik/traefik.yml

```yaml
# Configuración global
global:
  checkNewVersion: true
  sendAnonymousUsage: false

# Logging
log:
  level: INFO
  # level: DEBUG  # Para desarrollo

# API y Dashboard
api:
  dashboard: true
  insecure: true  # Solo desarrollo, usar autenticación en producción

# Entrypoints
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

# Providers
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false  # Solo servicios con traefik.enable=true
    network: traefik-public
  file:
    directory: /dynamic
    watch: true

# Certificados SSL (Let's Encrypt)
certificatesResolvers:
  letsencrypt:
    acme:
      email: tu@email.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

---

## 🌐 Crear la Red

```bash
# Crear red externa para Traefik
docker network create traefik-public

# Verificar
docker network ls
```

---

## 📦 Agregar Servicios

### Servicio de Ejemplo

```yaml
# docker-compose.yml (otro proyecto)
services:
  web:
    image: nginx
    labels:
      - "traefik.enable=true"
      # Router
      - "traefik.http.routers.mi-web.rule=Host(`mi-app.localhost`)"
      - "traefik.http.routers.mi-web.entrypoints=web"
      # Service (puerto del contenedor)
      - "traefik.http.services.mi-web.loadbalancer.server.port=80"
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

---

## 🏷️ Labels Comunes

```yaml
labels:
  # Habilitar Traefik para este contenedor
  - "traefik.enable=true"
  
  # Router: nombre único para este router
  - "traefik.http.routers.NOMBRE.rule=Host(`dominio.com`)"
  
  # Entrypoint a usar
  - "traefik.http.routers.NOMBRE.entrypoints=websecure"
  
  # Habilitar TLS
  - "traefik.http.routers.NOMBRE.tls=true"
  
  # Resolver de certificados
  - "traefik.http.routers.NOMBRE.tls.certresolver=letsencrypt"
  
  # Puerto del servicio (si no es 80)
  - "traefik.http.services.NOMBRE.loadbalancer.server.port=3000"
  
  # Middlewares
  - "traefik.http.routers.NOMBRE.middlewares=mi-middleware"
```

---

## 💻 Ejemplo Completo: Múltiples Servicios

### docker-compose.yml

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
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.traefik.service=api@internal"

  # Servicio 1: Blog
  blog:
    image: wordpress
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: secret
      WORDPRESS_DB_NAME: wordpress
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.blog.rule=Host(`blog.localhost`)"
      - "traefik.http.routers.blog.entrypoints=web"
    networks:
      - web
      - internal
    depends_on:
      - db

  # Servicio 2: API
  api:
    image: mi-api:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.localhost`)"
      - "traefik.http.routers.api.entrypoints=web"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
    networks:
      - web
      - internal

  # Servicio 3: Documentación
  docs:
    image: squidfunk/mkdocs-material
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.docs.rule=Host(`docs.localhost`)"
      - "traefik.http.routers.docs.entrypoints=web"
      - "traefik.http.services.docs.loadbalancer.server.port=8000"
    networks:
      - web
    volumes:
      - ./docs:/docs

  # Base de datos (sin Traefik, solo interno)
  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: rootsecret
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - internal  # Solo red interna, no accesible desde Traefik

networks:
  web:
  internal:

volumes:
  db-data:
```

---

## 🔧 Comandos Útiles

```bash
# Crear red
docker network create traefik-public

# Crear archivo para certificados (con permisos correctos)
touch traefik/acme.json
chmod 600 traefik/acme.json

# Levantar Traefik
docker compose up -d

# Ver logs de Traefik
docker compose logs -f traefik

# Verificar configuración
docker exec traefik traefik healthcheck
```

---

## 🐛 Debugging

```bash
# Ver routers registrados
curl http://localhost:8080/api/http/routers | jq

# Ver servicios registrados
curl http://localhost:8080/api/http/services | jq

# Ver middlewares
curl http://localhost:8080/api/http/middlewares | jq

# Logs con más detalle
# En traefik.yml, cambiar: log.level: DEBUG
```

---

**[← Anterior: Introducción](01-introduccion-traefik.md)** | **[Siguiente: Labels y Routing →](03-labels-routing.md)**
