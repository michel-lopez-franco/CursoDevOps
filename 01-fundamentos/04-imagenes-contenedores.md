# 🖼️ Imágenes y Contenedores

## Diferencia Fundamental

| Concepto | Descripción | Analogía |
|----------|-------------|----------|
| **Imagen** | Plantilla inmutable (solo lectura) | Clase en programación |
| **Contenedor** | Instancia ejecutable de una imagen | Objeto/instancia de una clase |

```
┌─────────────────────────────────────────────────────────────┐
│                        IMAGEN                               │
│                   (Plantilla inmutable)                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Capa 4: Configuración de aplicación                │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Capa 3: Dependencias (npm, pip, etc.)              │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Capa 2: Runtime (node, python, java)               │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Capa 1: Sistema operativo base (alpine, debian)    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ docker run
                           ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Contenedor 1│  │ Contenedor 2│  │ Contenedor 3│
│  (Instancia)│  │  (Instancia)│  │  (Instancia)│
│             │  │             │  │             │
│ Capa RW     │  │ Capa RW     │  │ Capa RW     │
│ ─────────── │  │ ─────────── │  │ ─────────── │
│ Capas RO    │  │ Capas RO    │  │ Capas RO    │
│ (Imagen)    │  │ (Imagen)    │  │ (Imagen)    │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## 🏷️ Tags de Imágenes

Las imágenes usan el formato: `[registro/]nombre[:tag]`

```bash
# Ejemplos de nombres de imágenes
nginx                         # Imagen oficial, tag "latest" implícito
nginx:1.25                    # Versión específica
nginx:alpine                  # Variante Alpine (más ligera)
python:3.11-slim              # Python 3.11 versión slim
mysql:8.0                     # MySQL versión 8.0

# Imágenes de registros privados
registry.ejemplo.com/mi-app:v1.0
ghcr.io/usuario/imagen:latest
```

### Tags Comunes

| Tag | Significado |
|-----|-------------|
| `latest` | Última versión (⚠️ evitar en producción) |
| `alpine` | Basada en Alpine Linux (~5MB) |
| `slim` | Versión reducida sin extras |
| `bullseye`, `bookworm` | Nombres de versiones de Debian |
| `X.Y.Z` | Versión semántica específica |

---

## 🔍 Sistema de Capas

Las imágenes Docker se construyen en **capas (layers)**. Cada instrucción en un Dockerfile crea una nueva capa.

```dockerfile
FROM python:3.11-slim          # Capa base
WORKDIR /app                   # Capa: crear directorio
COPY requirements.txt .        # Capa: copiar archivo
RUN pip install -r requirements.txt  # Capa: instalar deps
COPY . .                       # Capa: copiar código
CMD ["python", "app.py"]       # Metadata (no crea capa)
```

### Beneficios de las Capas

1. **Caché**: Si una capa no cambia, se reutiliza del caché
2. **Compartir**: Múltiples imágenes pueden compartir capas base
3. **Eficiencia**: Solo se descargan/almacenan las capas que faltan

```bash
# Ver las capas de una imagen
docker history nginx

# Ejemplo de salida:
IMAGE          CREATED       SIZE   COMMAND
d84f98..       2 weeks ago   0B     CMD ["nginx" "-g"...
<missing>      2 weeks ago   0B     EXPOSE 80
<missing>      2 weeks ago   61.2MB /bin/sh -c #(nop)...
<missing>      2 weeks ago   80.4MB /bin/sh -c apt-get...
<missing>      2 weeks ago   0B     /bin/sh -c #(nop)...
```

---

## 📦 Gestión de Imágenes

### Descargar Imágenes

```bash
# Descargar última versión
docker pull nginx

# Descargar versión específica
docker pull nginx:1.25-alpine

# Descargar todas las versiones
docker pull -a nginx
```

### Listar Imágenes

```bash
# Listar todas las imágenes
docker images

# Filtrar por nombre
docker images nginx

# Ver solo IDs
docker images -q

# Ver imágenes "colgantes" (sin tag)
docker images -f dangling=true
```

### Eliminar Imágenes

```bash
# Eliminar imagen específica
docker rmi nginx:latest

# Eliminar por ID
docker rmi abc123

# Forzar eliminación
docker rmi -f nginx

# Eliminar imágenes no utilizadas
docker image prune

# Eliminar TODAS las imágenes no usadas
docker image prune -a
```

---

## 🔄 Ciclo de Vida de un Contenedor

```
                     ┌────────────────┐
                     │    docker rm   │
                     │                │
     ┌───────────────┴────────────────┴───────────────┐
     │                                                 │
     ▼                                                 │
┌─────────┐    docker run    ┌─────────┐              │
│ Created │ ───────────────► │ Running │              │
└────┬────┘                  └────┬────┘              │
     │                            │                    │
     │                            │ docker stop/kill   │
     │                            ▼                    │
     │                       ┌─────────┐              │
     │                       │ Stopped │ ─────────────┘
     │                       └────┬────┘
     │                            │
     │                            │ docker start
     │                            │
     │                            ▼
     │                       ┌─────────┐
     └───────────────────────│ Running │
                             └─────────┘
```

### Estados de un Contenedor

| Estado | Descripción |
|--------|-------------|
| `created` | Contenedor creado pero no iniciado |
| `running` | Contenedor en ejecución |
| `paused` | Contenedor pausado |
| `restarting` | Contenedor reiniciándose |
| `exited` | Contenedor detenido (código de salida) |
| `dead` | Contenedor que falló al detenerse |

```bash
# Ver estado de contenedores
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.State}}"
```

---

## 💾 Crear Imagen desde Contenedor

Puedes crear una nueva imagen desde un contenedor modificado:

```bash
# 1. Ejecutar contenedor y hacer cambios
docker run -it --name mi-ubuntu ubuntu bash
# Dentro del contenedor:
apt update && apt install -y vim
exit

# 2. Commit: crear imagen desde contenedor
docker commit mi-ubuntu mi-ubuntu-con-vim

# 3. Verificar
docker images
# REPOSITORY           TAG       SIZE
# mi-ubuntu-con-vim    latest    150MB
```

> ⚠️ **Nota**: Es mejor usar Dockerfile para crear imágenes (reproducible y documentado).

---

## 🏷️ Etiquetar y Publicar Imágenes

### Etiquetar (Tag)

```bash
# Crear tag adicional
docker tag mi-app:latest mi-app:v1.0

# Preparar para Docker Hub
docker tag mi-app:latest usuario/mi-app:v1.0
```

### Publicar en Docker Hub

```bash
# 1. Iniciar sesión
docker login

# 2. Etiquetar con tu usuario
docker tag mi-app:latest usuario/mi-app:v1.0

# 3. Subir imagen
docker push usuario/mi-app:v1.0
```

---

## 📋 Inspeccionar Contenedores e Imágenes

```bash
# Inspeccionar imagen (detalles completos en JSON)
docker image inspect nginx

# Inspeccionar contenedor
docker inspect mi-contenedor

# Obtener dato específico con formato
docker inspect --format '{{.NetworkSettings.IPAddress}}' mi-contenedor

# Ver uso de recursos en tiempo real
docker stats

# Ver procesos dentro del contenedor
docker top mi-contenedor
```

---

## 💡 Buenas Prácticas

1. **Usar tags específicos** en producción, no `latest`
2. **Imágenes base pequeñas**: preferir `alpine` o `slim`
3. **Menos capas** = imagen más pequeña
4. **No almacenar datos** en contenedores (usar volúmenes)
5. **Un proceso por contenedor** (principio de responsabilidad única)

---

**[← Anterior: Comandos Básicos](03-comandos-basicos.md)** | **[Siguiente: Dockerfile →](05-dockerfile.md)**
