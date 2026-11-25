# Exposición de Backend Local a Internet con Tailscale Funnel y Docker

Esta guía documenta la configuración utilizada para exponer una API local que corre en el puerto `4000` hacia internet público mediante **Tailscale Funnel** dentro de un contenedor Docker, con certificados TLS/HTTPS automáticos.

---

## 1. Requisitos previos en el Panel de Tailscale

Antes de levantar el contenedor, es obligatorio habilitar dos opciones en la consola web de Tailscale (`https://tailscale.com`):

1. **Habilitar HTTPS:**
   - Ir a **DNS -> HTTPS Certificates** y hacer clic en **Enable HTTPS**.
2. **Permisos de Funnel (ACL):**
   - Asegurarse de que la cuenta/nodo tenga habilitada la capacidad `funnel` (se aprueba la primera vez desde el enlace que genera el comando `tailscale funnel`).

---

## 2. Estructura de Archivos

```text
.tailscale/
├── .env
├── docker-compose.yml
├── serve.json
└── tailscale-data/   # Carpeta generada automáticamente para persistir el estado
```

### Archivo `.env`
El `TS_HOSTNAME` debe ser un nombre corto (sin incluir `.tailnet.ts.net`), ya que Tailscale concatena automáticamente el dominio de tu red.

```env
TS_HOSTNAME=gateway
TS_AUTHKEY=tskey-auth-tu-clave-aqui
```

### Archivo `docker-compose.yml`
Se utiliza `network_mode: host` para que el contenedor de Tailscale comparta la red de la máquina anfitriona y pueda comunicarse directamente con `http://localhost:4000` (donde corren los demás servicios).

```yaml
services:
  tailscale-gateway:
    image: tailscale/tailscale:latest
    container_name: tailscale-gateway
    network_mode: host
    env_file:
      - .env
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_USERSPACE=false
    volumes:
      - ./tailscale-data:/var/lib/tailscale
      - ./serve.json:/config/serve.json
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
      - SYS_MODULE
    restart: unless-stopped
```

> **Nota:** Al usar `network_mode: host`, no se debe definir la propiedad `hostname:` en el `docker-compose.yml` para evitar sobrescribir el hostname del sistema operativo anfitrión; Tailscale toma el nombre desde la variable `TS_HOSTNAME` del `.env`.

### Archivo `serve.json`
Configura el proxy inverso en el puerto `443`, enruta cada microservicio por su
prefijo y habilita la exposición pública (`AllowFunnel`).

Los handlers se resuelven por el prefijo más largo; se declaran con y sin slash
final para cubrir la ruta exacta y sus descendientes. El API de inventario vive
en `/api/v1/inventory` (su `API_PREFIX`), así que el `Proxy` apunta al puerto
`4000` sin reescribir el path. Cada microservicio nuevo agrega su propio handler.

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "\${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/api/v1/inventory": {
          "Proxy": "http://localhost:4000"
        },
        "/api/v1/inventory/": {
          "Proxy": "http://localhost:4000"
        }
      }
    }
  },
  "AllowFunnel": {
    "\${TS_CERT_DOMAIN}:443": true
  }
}
```

> `/health` y `/ready` no se publican en el gateway: el frontend los consulta
> bajo el prefijo (`/api/v1/inventory/ready`) y Docker Compose los sigue llamando
> en raíz dentro del contenedor.

---

## 3. Despliegue y Activación

### 1. Levantar el contenedor
```bash
docker compose up -d
```

### 2. Generar certificado y activar Funnel (Primera vez)
En instalaciones nuevas, es necesario forzar la emisión del certificado TLS y activar Funnel mediante el socket interno de Tailscale (`/tmp/tailscaled.sock`):

```bash
# Solicitar certificado SSL
docker exec tailscale-gateway tailscale --socket=/tmp/tailscaled.sock cert gateway.tail224652.ts.net

# Activar Funnel en segundo plano apuntando al puerto local 4000
docker exec tailscale-gateway tailscale --socket=/tmp/tailscaled.sock funnel --bg 4000
```

---

## 4. Comandos Útiles de Verificación y Mantenimiento

- **Verificar estado de rutas y Funnel activo:**
  ```bash
  docker exec tailscale-gateway tailscale --socket=/tmp/tailscaled.sock serve status
  ```

- **Verificar resolución DNS pública (saltando caché local):**
  ```bash
  dig +short @1.1.1.1 gateway.tail224652.ts.net
  ```

- **Probar endpoint público:**
  ```bash
  curl -i https://ts.net
  ```

- **Reiniciar tras cambios en `serve.json` o `.env`:**
  ```bash
  docker compose up -d --force-recreate
  ```
