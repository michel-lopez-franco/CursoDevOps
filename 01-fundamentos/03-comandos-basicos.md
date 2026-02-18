# ⌨️ Comandos Básicos de Docker

## Estructura de los Comandos Docker

```bash
docker [OPCIONES] COMANDO [ARG...]

# Ejemplos:
docker run nginx
docker container ls -a
docker image pull python:3.11
```

---

## 🚀 Comandos Esenciales

### Ejecutar Contenedores

```bash
# Ejecutar un contenedor
docker run [IMAGEN]

# Ejecutar en segundo plano (detached)
docker run -d nginx

# Ejecutar con nombre personalizado
docker run -d --name mi-nginx nginx

# Ejecutar con mapeo de puertos (host:contenedor)
docker run -d -p 8080:80 nginx

# Ejecutar y entrar al contenedor interactivamente
docker run -it ubuntu bash

# Ejecutar y eliminar al terminar
docker run --rm alpine echo "Hola mundo"
```

### Ver Contenedores

```bash
# Listar contenedores en ejecución
docker ps

# Listar todos los contenedores (incluyendo detenidos)
docker ps -a

# Ver solo los IDs
docker ps -q

# Formato personalizado
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"
```

### Gestionar Contenedores

```bash
# Detener un contenedor
docker stop [NOMBRE|ID]

# Iniciar un contenedor detenido
docker start [NOMBRE|ID]

# Reiniciar un contenedor
docker restart [NOMBRE|ID]

# Eliminar un contenedor (debe estar detenido)
docker rm [NOMBRE|ID]

# Forzar eliminación
docker rm -f [NOMBRE|ID]

# Eliminar todos los contenedores detenidos
docker container prune
```

### Interactuar con Contenedores

```bash
# Ejecutar comando dentro de un contenedor en ejecución
docker exec [NOMBRE|ID] [COMANDO]

# Entrar al shell de un contenedor
docker exec -it [NOMBRE|ID] bash
docker exec -it [NOMBRE|ID] sh  # Si no tiene bash

# Ver logs
docker logs [NOMBRE|ID]

# Seguir logs en tiempo real
docker logs -f [NOMBRE|ID]

# Ver últimas N líneas
docker logs --tail 100 [NOMBRE|ID]

# Inspeccionar un contenedor (detalles completos)
docker inspect [NOMBRE|ID]
```

---

## 🖼️ Comandos de Imágenes

```bash
# Descargar una imagen
docker pull [IMAGEN]:[TAG]
docker pull nginx:latest
docker pull python:3.11-slim

# Listar imágenes locales
docker images
docker image ls

# Eliminar una imagen
docker rmi [IMAGEN]
docker image rm [IMAGEN]

# Eliminar imágenes no utilizadas
docker image prune

# Buscar imágenes en Docker Hub
docker search nginx

# Ver historial de capas de una imagen
docker history [IMAGEN]

# Inspeccionar una imagen
docker image inspect [IMAGEN]
```

---

## 📊 Comandos de Información

```bash
# Información del sistema Docker
docker info

# Versión de Docker
docker version

# Uso de espacio en disco
docker system df

# Limpieza completa del sistema
docker system prune -a
```

---

## 🔧 Flags Comunes

| Flag | Descripción | Ejemplo |
|------|-------------|---------|
| `-d, --detach` | Ejecutar en segundo plano | `docker run -d nginx` |
| `-p, --publish` | Mapear puertos host:contenedor | `docker run -p 8080:80 nginx` |
| `-v, --volume` | Montar volumen | `docker run -v /local:/container nginx` |
| `-e, --env` | Variable de entorno | `docker run -e MY_VAR=value nginx` |
| `--name` | Asignar nombre al contenedor | `docker run --name web nginx` |
| `-it` | Modo interactivo con terminal | `docker run -it ubuntu bash` |
| `--rm` | Eliminar al detenerse | `docker run --rm alpine echo hi` |
| `--network` | Conectar a una red | `docker run --network mi-red nginx` |

---

## 💡 Ejemplos Prácticos

### Ejemplo 1: Servidor Web Nginx

```bash
# Crear y ejecutar servidor web
docker run -d --name web -p 8080:80 nginx

# Verificar que está corriendo
docker ps

# Ver en el navegador: http://localhost:8080

# Ver logs
docker logs web

# Detener y eliminar
docker stop web
docker rm web

# Es el uso más básico. Puedes montar tus archivos HTML, CSS y JS en el directorio 
/usr/share/nginx/html 
#del contenedor para servir una página web o una aplicación frontend 
```

### Ejemplo 2: Base de Datos MySQL

```bash
# Ejecutar MySQL con variables de entorno
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=mi_contraseña \
  -e MYSQL_DATABASE=mi_base \
  -p 3306:3306 \
  mysql:8.0

# Conectarse a MySQL dentro del contenedor
docker exec -it mysql-db mysql -uroot -p
```

### Ejemplo 3: Sesión Interactiva de Python

```bash
# Ejecutar Python interactivo
docker run -it --rm python:3.11 python

>>> print("Hola desde Docker!")
>>> exit()

# El contenedor se elimina automáticamente (--rm)
```

### Ejemplo 4: Ejecutar Script Local

```bash
# Montar directorio actual y ejecutar script
docker run --rm -v $(pwd):/app -w /app python:3.11 python mi_script.py
```

---

## 📋 Cheatsheet Rápido

```bash
# CICLO DE VIDA BÁSICO
docker run -d --name app nginx    # Crear y ejecutar
docker ps                          # Ver corriendo
docker stop app                    # Detener
docker start app                   # Iniciar
docker rm app                      # Eliminar

# DEBUGGING
docker logs app                    # Ver logs
docker logs -f app                 # Seguir logs
docker exec -it app bash           # Entrar al contenedor
docker inspect app                 # Ver detalles

# LIMPIEZA
docker container prune             # Eliminar contenedores detenidos
docker image prune -a              # Eliminar imágenes no usadas
docker system prune -a             # Limpieza total
```

---

**[← Anterior: Instalación](02-instalacion.md)** | **[Siguiente: Imágenes y Contenedores →](04-imagenes-contenedores.md)**
