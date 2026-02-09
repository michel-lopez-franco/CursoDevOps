# 💾 Tipos de Volúmenes en Docker

## ¿Por qué Persistencia?

Los contenedores son **efímeros** - cuando se eliminan, todos los datos dentro desaparecen. Los volúmenes permiten que los datos persistan más allá del ciclo de vida del contenedor.

```
┌─────────────────────────────────────────────────────────────┐
│                      Sin Volúmenes                          │
│                                                             │
│   Contenedor creado ──► Datos escritos ──► Contenedor      │
│                                             eliminado       │
│                                                  │          │
│                                                  ▼          │
│                                            💀 Datos         │
│                                              perdidos       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Con Volúmenes                          │
│                                                             │
│   Contenedor 1 ──► Escribe ──► [ VOLUMEN ] ◄── Lee ◄── Contenedor 2
│        │                           │                        │
│        ▼                           │                        │
│   Eliminado                        │                        │
│                                    ▼                        │
│                              ✅ Datos                       │
│                               persisten                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 Tipos de Almacenamiento

| Tipo | Descripción | Uso Principal |
|------|-------------|---------------|
| **Named Volume** | Gestionado por Docker | Producción, bases de datos |
| **Bind Mount** | Directorio del host | Desarrollo, código fuente |
| **tmpfs** | Solo en memoria RAM | Datos temporales, secretos |

---

## 🏷️ Named Volumes

Volúmenes gestionados completamente por Docker. Se almacenan en `/var/lib/docker/volumes/`.

```bash
# Crear volumen
docker volume create mi-datos

# Listar volúmenes
docker volume ls

# Inspeccionar
docker volume inspect mi-datos

# Usar en contenedor
docker run -d -v mi-datos:/app/data nginx

# Eliminar
docker volume rm mi-datos

# Eliminar todos los volúmenes no usados
docker volume prune
```

### En Docker Compose

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:  # Declarar el volumen
```

### Ventajas

- ✅ Gestionados por Docker
- ✅ Portables entre hosts
- ✅ Fácil backup
- ✅ Drivers para almacenamiento remoto (NFS, cloud)

---

## 📂 Bind Mounts

Montan un directorio específico del host en el contenedor.

```bash
# Montar directorio actual
docker run -d -v $(pwd):/app nginx

# Ruta absoluta
docker run -d -v /home/user/proyecto:/app nginx

# Solo lectura
docker run -d -v $(pwd)/config:/etc/app/config:ro nginx
```

### En Docker Compose

```yaml
services:
  app:
    build: .
    volumes:
      # Bind mount para desarrollo (hot reload)
      - ./src:/app/src
      # Solo lectura
      - ./config:/app/config:ro
```

### Ventajas

- ✅ Cambios inmediatos (sin rebuild)
- ✅ Ideal para desarrollo
- ✅ Acceso directo desde el host

### Desventajas

- ❌ Dependiente del path del host
- ❌ Problemas de permisos posibles
- ❌ No portable

---

## 🧠 tmpfs Mounts

Solo en memoria RAM. Los datos desaparecen al detener el contenedor.

```bash
docker run -d --tmpfs /app/tmp:rw,size=100m nginx
```

### En Docker Compose

```yaml
services:
  app:
    image: mi-app
    tmpfs:
      - /tmp
      - /run
```

### Uso

- Datos temporales que no deben persistir
- Secretos que no deben escribirse a disco
- Caché de alta velocidad

---

## 📋 Comparación

```
┌────────────────────────────────────────────────────────────────────┐
│                             HOST                                   │
│                                                                    │
│    /home/user/proyecto/        /var/lib/docker/volumes/           │
│           │                           │                            │
│           │ (bind mount)              │ (named volume)             │
│           ▼                           ▼                            │
│    ┌──────────────────────────────────────────────────────┐       │
│    │                    CONTENEDOR                         │       │
│    │                                                       │       │
│    │   /app/src ◄──────────  /app/data ◄──────────        │       │
│    │   (código)              (base datos)                  │       │
│    │                                                       │       │
│    │           /tmp ◄── tmpfs (memoria RAM)               │       │
│    └──────────────────────────────────────────────────────┘       │
└────────────────────────────────────────────────────────────────────┘
```

| Característica | Named Volume | Bind Mount | tmpfs |
|---------------|--------------|------------|-------|
| Ubicación | Docker gestiona | Host específico | RAM |
| Persistencia | ✅ Sí | ✅ Sí | ❌ No |
| Portabilidad | ✅ Alta | ❌ Baja | N/A |
| Rendimiento | Bueno | Muy bueno | Excelente |
| Backup | Fácil | Manual | N/A |
| Desarrollo | ❌ | ✅ | ❌ |
| Producción | ✅ | ⚠️ | ⚠️ |

---

## 🔧 Permisos

### Problema Común

```bash
# Error: Permission denied
docker run -v $(pwd):/app alpine touch /app/test.txt
```

### Soluciones

```bash
# 1. Ejecutar como usuario actual
docker run -v $(pwd):/app -u $(id -u):$(id -g) alpine touch /app/test.txt

# 2. En Dockerfile, crear usuario con mismo UID
FROM alpine
RUN adduser -D -u 1000 appuser
USER appuser
```

---

## 💾 Backup de Volúmenes

```bash
# Backup: crear tar desde volumen
docker run --rm \
  -v mi-volumen:/data:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mi-volumen-backup.tar.gz -C /data .

# Restore: extraer tar a volumen
docker run --rm \
  -v mi-volumen:/data \
  -v $(pwd):/backup:ro \
  alpine tar xzf /backup/mi-volumen-backup.tar.gz -C /data
```

---

**[← Volver al módulo](README.md)** | **[Siguiente: Bind vs Named →](02-bind-vs-named.md)**
