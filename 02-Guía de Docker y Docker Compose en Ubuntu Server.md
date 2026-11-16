# Guía de Docker y Docker Compose en Ubuntu Server

Esta guía detalla la instalación oficial de Docker, la configuración para proteger el SSD de tu servidor, y los comandos esenciales para administrar contenedores, imágenes y volúmenes.

---

## 1. Instalación de Docker y Docker Compose

Utilizaremos el repositorio oficial de Docker para obtener siempre la última versión estable.

### Paso 1.1: Instalar dependencias previas

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release -y
```

### Paso 1.2: Agregar la clave GPG oficial de Docker

```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Paso 1.3: Agregar el repositorio de Docker a Ubuntu

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Paso 1.4: Instalar Docker y el plugin de Compose

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### Paso 1.5: Permisos de usuario (Opcional pero recomendado)

Para ejecutar comandos de Docker sin escribir `sudo` cada vez, añade tu usuario al grupo `docker`:

```bash
sudo usermod -aG docker $USER
```

_(Para que este cambio haga efecto, debes cerrar la sesión SSH y volver a entrar)._

---

## 2. Optimización Obligatoria para SSD (Rotación de Logs)

Por defecto, Docker guarda logs ilimitados, lo que puede llenar y desgastar rápidamente un disco SSD.

### Paso 2.1: Crear archivo de configuración global

```bash
sudo nano /etc/docker/daemon.json
```

### Paso 2.2: Limitar tamaño de logs

Pega el siguiente contenido:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

_(Guarda con `Ctrl + O`, `Enter` y sal con `Ctrl + X`)._

### Paso 2.3: Reiniciar Docker para aplicar

```bash
sudo systemctl restart docker
```

---

## 3. Comandos de Gestión (Cheat Sheet)

### Contenedores

- **Listar contenedores activos:** `docker ps`
- **Listar TODOS los contenedores (activos y apagados):** `docker ps -a`
- **Detener un contenedor:** `docker stop <ID_o_Nombre>`
- **Iniciar un contenedor:** `docker start <ID_o_Nombre>`
- **Ver logs de un contenedor:** `docker logs -f <ID_o_Nombre>`
- **Eliminar un contenedor:** `docker rm <ID_o_Nombre>` (Debe estar detenido)
- **Forzar eliminación de un contenedor activo:** `docker rm -f <ID_o_Nombre>`

### Imágenes

- **Listar imágenes descargadas:** `docker images`
- **Eliminar una imagen:** `docker rmi <ID_o_Nombre>`
- **Forzar eliminación de imagen (si está en uso por un contenedor apagado):** `docker rmi -f <ID_o_Nombre>`

### Volúmenes (Donde persisten los datos, ej. Bases de Datos)

- **Listar volúmenes:** `docker volume ls`
- **Inspeccionar un volumen (ver dónde está en el disco):** `docker volume inspect <Nombre>`
- **Eliminar un volumen específico:** `docker volume rm <Nombre>`

### Redes

- **Listar redes internas de Docker:** `docker network ls`

---

## 4. Docker Compose

Docker Compose te permite levantar aplicaciones complejas (ej. Node.js + MongoDB + Nginx) usando un archivo `docker-compose.yml`.

- **Levantar servicios en segundo plano:** `docker compose up -d`
- **Ver logs de los servicios levantados:** `docker compose logs -f`
- **Apagar y eliminar los contenedores del compose:** `docker compose down`
- **Apagar contenedores y también eliminar sus volúmenes (¡Borra datos!):** `docker compose down -v`

---

## 5. Limpieza y Purga (Liberar espacio en disco)

Cuando actualizas y creas muchos contenedores, Docker acumula imágenes antiguas, redes y caché que consumen gigabytes de almacenamiento. Usa estos comandos según el nivel de limpieza que necesites.

### Purgas Específicas

- **Borrar contenedores detenidos:** `docker container prune`
- **Borrar imágenes "colgantes" (sin etiqueta/huérfanas):** `docker image prune`
- **Borrar redes sin uso:** `docker network prune`
- **Borrar volúmenes sin uso (¡Cuidado, borra datos persistentes!):** `docker volume prune`
- **Borrar caché de compilación (BuildKit):** `docker builder prune -a`

### Purga General del Sistema (Mantenimiento seguro)

Elimina todo lo que **no esté en uso** actualmente (contenedores apagados, redes huérfanas e imágenes sin etiqueta), respetando tus servicios activos:

```bash
docker system prune

```

### Purga Total / Factory Reset (Dejar Docker en cero) 🚨

El comando `prune` por sí solo ignora los contenedores que estén corriendo. Si quieres **borrar absolutamente todo** (contenedores activos y detenidos, todas las imágenes, redes, volúmenes con datos y caché de compilación) para dejar Docker como recién instalado, ejecuta este bloque en orden:

```bash
# 1. Detener y eliminar TODOS los contenedores (activos y detenidos)
docker ps -aq | xargs -r docker rm -f

# 2. Eliminar todas las imágenes, redes y volúmenes (incluidos los con nombre)
docker system prune -a --volumes -f

# 3. Limpiar toda la caché de construcción de imágenes (BuildKit)
docker builder prune -a -f

```

_(La bandera `-f` omite la confirmación manual; `xargs -r` evita errores si no hay contenedores corriendo)._

### Verificar que quedó limpio

Comprueba que no quede ningún rastro en el sistema:

```bash
docker system df

```

_(Todas las columnas en `TOTAL` y `SIZE` deben aparecer en `0` / `0B`)._
