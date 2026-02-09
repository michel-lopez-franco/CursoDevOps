# 🐳 Introducción a Docker

## ¿Qué es Docker?

**Docker** es una plataforma de código abierto que permite **automatizar el despliegue de aplicaciones dentro de contenedores**. Un contenedor es una unidad ligera y portátil que incluye todo lo necesario para ejecutar una aplicación: código, runtime, bibliotecas y configuraciones.

## 🤔 ¿Por qué usar Docker?

### Problemas que Docker resuelve

```
"¡En mi máquina funciona!" 
     - Todo desarrollador alguna vez
```

| Problema | Solución con Docker |
|----------|---------------------|
| Inconsistencia entre entornos | El contenedor es idéntico en desarrollo, staging y producción |
| Conflictos de dependencias | Cada contenedor tiene sus propias dependencias aisladas |
| Configuración manual compleja | Todo está definido en código (Dockerfile) |
| Despliegues lentos | Los contenedores inician en segundos |
| Uso ineficiente de recursos | Los contenedores comparten el kernel del host |

## 📦 Contenedores vs Máquinas Virtuales

```
┌─────────────────────────────────────────────────────────────────────────┐
│          MÁQUINAS VIRTUALES              │         CONTENEDORES        │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     │  ┌─────────┐ ┌─────────┐    │
│  │  App A  │ │  App B  │ │  App C  │     │  │  App A  │ │  App B  │    │
│  ├─────────┤ ├─────────┤ ├─────────┤     │  ├─────────┤ ├─────────┤    │
│  │  Libs   │ │  Libs   │ │  Libs   │     │  │  Libs   │ │  Libs   │    │
│  ├─────────┤ ├─────────┤ ├─────────┤     │  └────┬────┘ └────┬────┘    │
│  │Guest OS │ │Guest OS │ │Guest OS │     │       └─────┬─────┘         │
│  └────┬────┘ └────┬────┘ └────┬────┘     │     ┌───────┴────────┐      │
│       └──────────┬┴──────────┘           │     │  Docker Engine │      │
│           ┌──────┴───────┐               │     └───────┬────────┘      │
│           │  Hypervisor  │               │             │               │
│           └──────┬───────┘               │             │               │
│                  │                       │             │               │
│  ┌───────────────┴───────────────┐       │  ┌──────────┴──────────┐   │
│  │         Host OS               │       │  │      Host OS        │   │
│  └───────────────────────────────┘       │  └─────────────────────┘   │
│  ┌───────────────────────────────┐       │  ┌─────────────────────┐   │
│  │       Infrastructure          │       │  │   Infrastructure    │   │
│  └───────────────────────────────┘       │  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Comparación Rápida

| Característica | Máquina Virtual | Contenedor Docker |
|----------------|-----------------|-------------------|
| **Tiempo de inicio** | Minutos | Segundos |
| **Tamaño** | GB | MB |
| **Rendimiento** | Overhead significativo | Casi nativo |
| **Aislamiento** | Completo (kernel propio) | A nivel de proceso |
| **Portabilidad** | Limitada | Excelente |
| **Uso de recursos** | Alto | Bajo |

## 🏗️ Arquitectura de Docker

Docker utiliza una arquitectura **cliente-servidor**:

```
┌────────────────┐          ┌─────────────────────────────────────┐
│  Docker CLI    │          │          Docker Host                │
│  (Cliente)     │   REST   │  ┌─────────────────────────────┐   │
│                │◄────────►│  │      Docker Daemon          │   │
│  $ docker run  │   API    │  │        (dockerd)            │   │
│  $ docker ps   │          │  └──────────────┬──────────────┘   │
│  $ docker build│          │                 │                   │
└────────────────┘          │    ┌────────────┼────────────┐     │
                            │    ▼            ▼            ▼     │
                            │ ┌──────┐   ┌──────┐   ┌──────┐    │
                            │ │ Cont │   │ Cont │   │ Cont │    │
                            │ └──────┘   └──────┘   └──────┘    │
                            │                                     │
                            │ ┌──────┐   ┌──────┐   ┌──────┐    │
                            │ │Image │   │Image │   │Image │    │
                            │ └──────┘   └──────┘   └──────┘    │
                            └─────────────────────────────────────┘
                                              │
                                              ▼
                                     ┌───────────────┐
                                     │  Docker Hub   │
                                     │  (Registry)   │
                                     └───────────────┘
```

### Componentes Principales

| Componente | Descripción |
|------------|-------------|
| **Docker Client** | Interfaz de línea de comandos para interactuar con Docker |
| **Docker Daemon** | Servicio que gestiona imágenes, contenedores, redes y volúmenes |
| **Docker Registry** | Repositorio para almacenar y distribuir imágenes (ej: Docker Hub) |
| **Images** | Plantillas de solo lectura para crear contenedores |
| **Containers** | Instancias ejecutables de imágenes |

## 🔑 Conceptos Clave

### Imagen (Image)
Una **imagen** es una plantilla inmutable que contiene el código de la aplicación, las bibliotecas, dependencias y configuraciones necesarias. Las imágenes se construyen en capas.

```
┌───────────────────────────┐
│     Tu aplicación         │  Layer 4 (RW cuando es contenedor)
├───────────────────────────┤
│     Dependencias (npm)    │  Layer 3
├───────────────────────────┤
│     Node.js runtime       │  Layer 2
├───────────────────────────┤
│     Sistema base (Alpine) │  Layer 1
└───────────────────────────┘
```

### Contenedor (Container)
Un **contenedor** es una instancia ejecutable de una imagen. Puedes tener múltiples contenedores basados en la misma imagen.

### Dockerfile
Un **Dockerfile** es un archivo de texto que contiene las instrucciones para construir una imagen Docker.

### Docker Compose
**Docker Compose** es una herramienta para definir y ejecutar aplicaciones multi-contenedor (lo veremos en el Módulo 2).

---

## 📝 Resumen

- Docker permite empaquetar aplicaciones con todas sus dependencias
- Los contenedores son más ligeros y rápidos que las VMs
- Las imágenes son plantillas inmutables, los contenedores son instancias ejecutables
- Docker usa una arquitectura cliente-servidor

---

**[← Volver al módulo](README.md)** | **[Siguiente: Instalación →](02-instalacion.md)**
