# 🔒 SSL con Let's Encrypt

## Certificados Automáticos

Traefik puede obtener y renovar certificados SSL de Let's Encrypt automáticamente.

```
┌─────────────────────────────────────────────────────────────┐
│                       Traefik                               │
│                                                             │
│   1. Request llega                                          │
│   2. Let's Encrypt verifica dominio (HTTP Challenge)        │
│   3. Traefik obtiene certificado                            │
│   4. Request servida con HTTPS                              │
│                                                             │
│   📜 Certificados guardados en acme.json                    │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Configuración

### traefik.yml

```yaml
# Entrypoints
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ":443"

# Certificate Resolvers
certificatesResolvers:
  letsencrypt:
    acme:
      # Email para notificaciones de Let's Encrypt
      email: tu@email.com
      # Archivo donde guardar certificados
      storage: /acme.json
      # Usar servidor de staging para pruebas
      # caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      # HTTP Challenge (el más común)
      httpChallenge:
        entryPoint: web
```

### docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json  # Certificados
    networks:
      - web

  app:
    image: mi-app
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.midominio.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls=true"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
    networks:
      - web

networks:
  web:
```

---

## 📁 Preparar acme.json

```bash
# Crear archivo
touch acme.json

# Permisos correctos (MUY IMPORTANTE)
chmod 600 acme.json

# Verificar
ls -la acme.json
# -rw------- 1 user user 0 ... acme.json
```

> ⚠️ **Si los permisos no son 600, Traefik no funcionará**

---

## 🧪 Testing con Staging

Para pruebas, usa el servidor staging de Let's Encrypt (sin límites de rate):

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: tu@email.com
      storage: /acme.json
      # Staging server (para pruebas)
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      httpChallenge:
        entryPoint: web
```

> 💡 Los certificados de staging no son confiables para navegadores, pero sirven para probar la configuración.

---

## 🌐 Tipos de Challenge

### HTTP Challenge (Recomendado)

Let's Encrypt verifica que controlas el dominio haciendo una request HTTP.

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      httpChallenge:
        entryPoint: web  # Puerto 80 debe estar abierto
```

**Requisitos:**
- Puerto 80 accesible desde internet
- DNS apuntando al servidor

### DNS Challenge

Para wildcards o cuando el puerto 80 no está disponible.

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

---

## 📦 Ejemplo Completo: Múltiples Dominios

```yaml
services:
  traefik:
    image: traefik:v3.0
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
      # Dashboard con SSL
      - "traefik.http.routers.traefik.rule=Host(`traefik.midominio.com`)"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik.service=api@internal"

  # Blog
  blog:
    image: wordpress
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.blog.rule=Host(`blog.midominio.com`)"
      - "traefik.http.routers.blog.entrypoints=websecure"
      - "traefik.http.routers.blog.tls.certresolver=letsencrypt"
    networks:
      - web

  # API
  api:
    image: mi-api
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.midominio.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
    networks:
      - web

  # App principal (con www redirect)
  app:
    image: mi-app
    labels:
      - "traefik.enable=true"
      # Ruta principal
      - "traefik.http.routers.app.rule=Host(`midominio.com`) || Host(`www.midominio.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      # Redirect www a non-www
      - "traefik.http.middlewares.www-redirect.redirectregex.regex=^https://www\\.(.+)"
      - "traefik.http.middlewares.www-redirect.redirectregex.replacement=https://$${1}"
      - "traefik.http.routers.app.middlewares=www-redirect"
    networks:
      - web

networks:
  web:
```

---

## 🔧 Troubleshooting

### Ver logs de ACME

```bash
docker compose logs traefik | grep -i acme
```

### Verificar certificado

```bash
# Ver certificado actual
echo | openssl s_client -servername app.midominio.com -connect app.midominio.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Problemas comunes

| Problema | Solución |
|----------|----------|
| `acme.json` sin permisos | `chmod 600 acme.json` |
| Puerto 80 bloqueado | Abrir en firewall |
| DNS no apunta | Verificar registros A |
| Rate limit | Usar staging para pruebas |

---

**[← Anterior: Labels](03-labels-routing.md)** | **[Siguiente: Dashboard →](05-dashboard.md)**
