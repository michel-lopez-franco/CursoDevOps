# 🌐 Tipos de Redes en Docker

Docker proporciona varios drivers de red para diferentes casos de uso.

## Drivers de Red

| Driver | Descripción | Caso de Uso |
|--------|-------------|-------------|
| **bridge** | Red privada interna (default) | Contenedores en un solo host |
| **host** | Comparte red del host | Máximo rendimiento de red |
| **overlay** | Red distribuida multi-host | Docker Swarm / Kubernetes |
| **none** | Sin networking | Contenedores aislados |
| **macvlan** | Asigna MAC address | Integración con red física |

---

## 🌉 Bridge Network (Por Defecto)

La red más común. Docker crea una interfaz `docker0` que actúa como puente entre contenedores y el host.

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Contenedor A│  │ Contenedor B│  │ Contenedor C│         │
│  │ 172.17.0.2  │  │ 172.17.0.3  │  │ 172.17.0.4  │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│  ───────┴────────────────┴────────────────┴─────────────   │
│                      docker0 (bridge)                       │
│                       172.17.0.1                            │
│                           │                                 │
│  ─────────────────────────┴─────────────────────────────   │
│                        eth0 (host)                          │
│                      192.168.1.100                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                        Internet
```

### Características

- ✅ Aislamiento entre contenedores y el host
- ✅ Contenedores pueden comunicarse entre sí
- ✅ NAT para acceso a internet
- ❌ Requiere `-p` para exponer puertos externamente

### Comandos

```bash
# Red bridge por defecto
docker run -d --name web nginx

# Ver redes
docker network ls

# Inspeccionar red bridge
docker network inspect bridge

# Ver IP del contenedor
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
```

### Red Bridge Personalizada (Recomendado)

```bash
# Crear red bridge personalizada
docker network create mi-red

# Ejecutar contenedor en la red
docker run -d --name web --network mi-red nginx

# Conectar contenedor existente a red
docker network connect mi-red otro-contenedor

# Desconectar
docker network disconnect mi-red otro-contenedor
```

**Ventajas de bridge personalizado:**
- DNS automático (los contenedores se encuentran por nombre)
- Mejor aislamiento
- Posibilidad de conectar/desconectar en tiempo real

---

## 🏠 Host Network

El contenedor comparte directamente la pila de red del host. No hay aislamiento de red.

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     Contenedor                       │   │
│  │              (Usa IP y puertos del host)             │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│  ─────────────────────────┴─────────────────────────────   │
│                        eth0 (host)                          │
│                      192.168.1.100                          │
└─────────────────────────────────────────────────────────────┘
```

### Características

- ✅ Máximo rendimiento de red
- ✅ No necesita mapeo de puertos
- ❌ Sin aislamiento de red
- ❌ Conflictos de puertos posibles
- ❌ Solo funciona en Linux

### Uso

```bash
# Ejecutar con red del host
docker run -d --network host nginx

# El puerto 80 del contenedor es directamente el puerto 80 del host
curl http://localhost:80
```

### Cuándo Usar

- Aplicaciones que requieren máximo rendimiento de red
- Servicios que necesitan acceso directo a interfaces de red
- Debugging de network

---

## 🌍 Overlay Network

Permite que contenedores en diferentes hosts se comuniquen como si estuvieran en la misma red.

```
┌─────────────────────┐           ┌─────────────────────┐
│      Host 1         │           │      Host 2         │
│  ┌──────────────┐   │           │   ┌──────────────┐  │
│  │ Contenedor A │   │           │   │ Contenedor B │  │
│  │  10.0.0.2    │   │           │   │  10.0.0.3    │  │
│  └──────┬───────┘   │           │   └──────┬───────┘  │
│         │           │           │          │          │
│  ───────┴───────────┴───────────┴──────────┴────────  │
│                   Overlay Network                      │
│                      10.0.0.0/24                       │
└────────────────────────────────────────────────────────┘
```

### Características

- ✅ Comunicación multi-host
- ✅ Encriptación opcional
- ✅ Service discovery automático
- ❌ Requiere Docker Swarm o Kubernetes

### Crear Overlay Network (Swarm)

```bash
# Inicializar Swarm (si no está)
docker swarm init

# Crear red overlay
docker network create --driver overlay mi-overlay

# Usar en un stack
docker stack deploy -c docker-compose.yml mi-stack
```

---

## 🚫 None Network

Sin networking. El contenedor está completamente aislado.

```bash
docker run --network none alpine ip addr
# Solo muestra interfaz loopback
```

### Cuándo Usar

- Procesos batch que no necesitan red
- Máximo aislamiento de seguridad
- Generación de datos offline

---

## 🔌 Macvlan Network

Asigna una dirección MAC real al contenedor, haciéndolo aparecer como un dispositivo físico en la red.

```bash
# Crear red macvlan
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  mi-macvlan

# Ejecutar contenedor
docker run -d --network mi-macvlan --ip 192.168.1.50 nginx
```

### Cuándo Usar

- Integración con redes legacy
- Aplicaciones que esperan IP dedicada
- Cuando necesitas bypass del NAT

---

## 📋 Comandos de Gestión de Redes

```bash
# Listar redes
docker network ls

# Crear red
docker network create [OPCIONES] NOMBRE

# Inspeccionar red
docker network inspect NOMBRE

# Eliminar red
docker network rm NOMBRE

# Eliminar redes no usadas
docker network prune

# Conectar contenedor a red
docker network connect RED CONTENEDOR

# Desconectar
docker network disconnect RED CONTENEDOR
```

---

## 💡 Ejemplo: Múltiples Redes

```yaml
# docker-compose.yml
services:
  # Proxy público - acceso frontend y backend
  proxy:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend
      - backend

  # App web - solo frontend
  web:
    image: mi-frontend
    networks:
      - frontend

  # API - conecta frontend con backend
  api:
    image: mi-api
    networks:
      - frontend
      - backend

  # DB - solo backend (aislada)
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Sin acceso a internet
```

```
                    Internet
                        │
                        ▼
                  ┌─────────┐
                  │  Proxy  │
                  └────┬────┘
                       │
        ┌──────────────┼──────────────┐
        │   frontend   │              │
        │              │              │
   ┌────┴────┐    ┌────┴────┐         │
   │   Web   │    │   API   │         │
   └─────────┘    └────┬────┘         │
                       │              │
        ┌──────────────┼──────────────┤
        │   backend    │   (internal) │
        │              │              │
                  ┌────┴────┐         │
                  │   DB    │         │
                  └─────────┘         │
                                      │
                       ✗ Sin acceso a Internet
```

---

**[← Volver al módulo](README.md)** | **[Siguiente: Comunicación →](02-comunicacion-contenedores.md)**
