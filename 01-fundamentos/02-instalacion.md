# 🛠️ Instalación de Docker

## Requisitos del Sistema

### Linux
- Kernel 3.10 o superior
- 64-bit
- iptables versión 1.4 o superior

### Windows
- Windows 10/11 64-bit: Pro, Enterprise, o Education (Build 18362+)
- WSL 2 habilitado
- Virtualización habilitada en BIOS

### macOS
- macOS 10.15 (Catalina) o superior
- Procesador Intel o Apple Silicon

---

## 🐧 Instalación en Linux (Ubuntu/Debian)

### Paso 1: Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade -y
```

### Paso 2: Instalar dependencias

```bash
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Paso 3: Agregar la clave GPG oficial de Docker

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Paso 4: Configurar el repositorio

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Paso 5: Instalar Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Paso 6: Agregar tu usuario al grupo docker (evitar usar sudo)

```bash
sudo usermod -aG docker $USER
```

> ⚠️ **Importante**: Cierra sesión y vuelve a iniciarla para que los cambios surtan efecto.

### Paso 7: Verificar la instalación

```bash
docker --version
docker compose version
docker run hello-world
```

---

## 🪟 Instalación en Windows

### Opción 1: Docker Desktop (Recomendado)

1. Descargar [Docker Desktop para Windows](https://www.docker.com/products/docker-desktop/)
2. Ejecutar el instalador
3. Seguir el asistente de instalación
4. Reiniciar el sistema cuando se solicite
5. Abrir Docker Desktop

### Opción 2: WSL 2 + Docker

```powershell
# Habilitar WSL
wsl --install

# Instalar Ubuntu desde Microsoft Store
# Luego seguir las instrucciones de Linux dentro de WSL
```

### Verificar instalación (PowerShell o CMD)

```powershell
docker --version
docker compose version
docker run hello-world
```

---

## 🍎 Instalación en macOS

### Opción 1: Docker Desktop (Recomendado)

1. Descargar [Docker Desktop para Mac](https://www.docker.com/products/docker-desktop/)
   - Apple Silicon (M1/M2/M3): Seleccionar "Mac with Apple chip"
   - Intel: Seleccionar "Mac with Intel chip"
2. Arrastrar Docker.app a la carpeta Aplicaciones
3. Abrir Docker Desktop
4. Aceptar los términos de servicio

### Opción 2: Homebrew

```bash
brew install --cask docker
```

### Verificar instalación

```bash
docker --version
docker compose version
docker run hello-world
```

---

## ✅ Verificación de la Instalación

Ejecuta los siguientes comandos para verificar que todo está funcionando:

```bash
# Verificar versiones
docker --version
# Salida esperada: Docker version 24.x.x, build xxxxxxx

docker compose version
# Salida esperada: Docker Compose version v2.x.x

# Verificar que el daemon está corriendo
docker info

# Ejecutar el contenedor de prueba
docker run hello-world
```

Si ves el mensaje "Hello from Docker!", ¡la instalación fue exitosa! 🎉

---

## 🔧 Post-instalación (Linux)

### Iniciar Docker al arranque

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Verificar estado del servicio

```bash
sudo systemctl status docker
```

---

## 🐞 Solución de Problemas Comunes

### Error: "Cannot connect to the Docker daemon"

```bash
# Verificar que el servicio esté corriendo
sudo systemctl status docker

# Reiniciar el servicio
sudo systemctl restart docker
```

### Error: "Permission denied while trying to connect to the Docker daemon socket"

```bash
# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Cerrar sesión y volver a iniciar, o ejecutar:
newgrp docker
```

### Error en Windows: "WSL 2 installation is incomplete"

1. Abrir PowerShell como administrador
2. Ejecutar: `wsl --update`
3. Reiniciar el sistema

---

**[← Anterior: Introducción](01-introduccion.md)** | **[Siguiente: Comandos Básicos →](03-comandos-basicos.md)**
