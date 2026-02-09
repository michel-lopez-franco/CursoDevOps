# 📝 Dockerfile

## ¿Qué es un Dockerfile?

Un **Dockerfile** es un archivo de texto que contiene todas las instrucciones necesarias para construir una imagen Docker. Es como una "receta" que Docker sigue para crear tu imagen.

```dockerfile
# Ejemplo básico de Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

---

## 📋 Instrucciones Principales

### FROM - Imagen Base

Define la imagen base sobre la cual construir.

```dockerfile
# Usar imagen oficial
FROM python:3.11-slim

# Usar versión específica
FROM node:18-alpine

# Imagen mínima (scratch)
FROM scratch
```

### WORKDIR - Directorio de Trabajo

Establece el directorio de trabajo para instrucciones posteriores.

```dockerfile
WORKDIR /app
WORKDIR /home/user/proyecto

# Si no existe, se crea automáticamente
```

### COPY y ADD - Copiar Archivos

```dockerfile
# COPY: copiar archivos locales
COPY archivo.txt /app/
COPY src/ /app/src/
COPY ["archivo con espacios.txt", "/app/"]

# ADD: igual que COPY pero con extras
# - Puede extraer archivos .tar
# - Puede descargar URLs (no recomendado)
ADD archivo.tar.gz /app/
```

> 💡 **Recomendación**: Usar `COPY` a menos que necesites las features extra de `ADD`.

### RUN - Ejecutar Comandos

Ejecuta comandos durante la construcción de la imagen.

```dockerfile
# Forma shell
RUN apt-get update && apt-get install -y curl

# Forma exec (recomendada)
RUN ["apt-get", "install", "-y", "curl"]

# Múltiples comandos en una capa (mejor práctica)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        vim \
    && rm -rf /var/lib/apt/lists/*
```

### CMD - Comando por Defecto

Define el comando por defecto al ejecutar el contenedor.

```dockerfile
# Forma exec (recomendada)
CMD ["python", "app.py"]

# Forma shell
CMD python app.py

# Solo puede haber UN CMD (el último gana)
```

### ENTRYPOINT - Punto de Entrada

Similar a CMD pero diseñado para hacer el contenedor "ejecutable".

```dockerfile
# El contenedor siempre ejecutará python
ENTRYPOINT ["python"]

# CMD proporciona argumentos por defecto
CMD ["app.py"]

# Al ejecutar: docker run mi-imagen otro.py
# Ejecutará: python otro.py
```

### ENV - Variables de Entorno

```dockerfile
# Definir variables
ENV APP_HOME=/app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Múltiples variables
ENV NODE_ENV=production \
    PORT=3000
```

### EXPOSE - Documentar Puertos

Documenta qué puertos usa el contenedor (no los publica automáticamente).

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8000/tcp
EXPOSE 8000/udp
```

### ARG - Argumentos de Build

Variables disponibles solo durante la construcción.

```dockerfile
ARG VERSION=1.0
ARG NODE_ENV=development

# Usar en instrucciones
RUN echo "Versión: $VERSION"
```

```bash
# Pasar valor al construir
docker build --build-arg VERSION=2.0 .
```

### USER - Usuario de Ejecución

```dockerfile
# Crear y cambiar a usuario no-root
RUN useradd -m appuser
USER appuser
```

### VOLUME - Puntos de Montaje

```dockerfile
VOLUME /data
VOLUME ["/var/log", "/app/uploads"]
```

---

## 🏗️ Ejemplo Completo: Aplicación Python

### Estructura del Proyecto

```
mi-app/
├── Dockerfile
├── requirements.txt
├── app.py
└── static/
    └── style.css
```

### Dockerfile

```dockerfile
# Imagen base
FROM python:3.11-slim

# Metadata
LABEL maintainer="tu@email.com"
LABEL version="1.0"
LABEL description="Mi aplicación Flask"

# Variables de entorno
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV FLASK_ENV=production

# Directorio de trabajo
WORKDIR /app

# Instalar dependencias primero (mejor caché)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código fuente
COPY . .

# Crear usuario no-root
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Puerto
EXPOSE 5000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Comando por defecto
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

### requirements.txt

```
flask==2.3.0
gunicorn==21.2.0
```

### app.py

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hola desde Docker!'

@app.route('/health')
def health():
    return 'OK', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 🔨 Construir la Imagen

```bash
# Construcción básica
docker build -t mi-app .

# Con tag específico
docker build -t mi-app:v1.0 .

# Con contexto diferente
docker build -t mi-app -f Dockerfile.prod .

# Sin caché
docker build --no-cache -t mi-app .

# Ver progreso detallado
docker build --progress=plain -t mi-app .
```

---

## ⚡ Multi-stage Builds

Permite crear imágenes más pequeñas separando el build del runtime.

```dockerfile
# ===== STAGE 1: Build =====
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ===== STAGE 2: Production =====
FROM nginx:alpine AS production

# Copiar solo los archivos construidos
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Beneficios de Multi-stage

| Sin Multi-stage | Con Multi-stage |
|-----------------|-----------------|
| ~1 GB (Node + fuentes) | ~25 MB (Nginx + estáticos) |
| Incluye herramientas de dev | Solo runtime necesario |
| Más superficie de ataque | Más seguro |

---

## 📁 .dockerignore

Excluye archivos del contexto de build (similar a .gitignore).

```dockerignore
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
*.md
README.md
Dockerfile*
docker-compose*
.dockerignore
__pycache__
*.pyc
*.pyo
.pytest_cache
.coverage
htmlcov
.vscode
.idea
```

---

## ✅ Buenas Prácticas

### 1. Usar imágenes base específicas

```dockerfile
# ❌ Malo
FROM python:latest

# ✅ Bueno
FROM python:3.11-slim
```

### 2. Minimizar el número de capas

```dockerfile
# ❌ Malo: múltiples capas
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim

# ✅ Bueno: una sola capa
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        vim \
    && rm -rf /var/lib/apt/lists/*
```

### 3. Ordenar instrucciones por frecuencia de cambio

```dockerfile
# ✅ Las dependencias cambian menos que el código
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 4. No ejecutar como root

```dockerfile
RUN useradd -m appuser
USER appuser
```

### 5. Usar .dockerignore

### 6. Limpiar caché en la misma capa

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*  # Limpiar aquí
```

---

## 📋 Referencia Rápida

| Instrucción | Propósito |
|-------------|-----------|
| `FROM` | Imagen base |
| `WORKDIR` | Directorio de trabajo |
| `COPY` | Copiar archivos |
| `ADD` | Copiar + extraer + URLs |
| `RUN` | Ejecutar comandos (build time) |
| `CMD` | Comando por defecto (runtime) |
| `ENTRYPOINT` | Punto de entrada fijo |
| `ENV` | Variables de entorno |
| `ARG` | Variables de build |
| `EXPOSE` | Documentar puertos |
| `VOLUME` | Puntos de montaje |
| `USER` | Usuario de ejecución |
| `HEALTHCHECK` | Verificación de salud |
| `LABEL` | Metadata |

---

**[← Anterior: Imágenes y Contenedores](04-imagenes-contenedores.md)** | **[Volver al módulo](README.md)**
