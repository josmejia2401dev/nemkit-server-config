# Guía Definitiva: Tailscale Funnel y Serve en Docker Compose (Entornos Productivos)

Esta guía detalla cómo desplegar Tailscale como un **Gateway (Reverse Proxy)** declarativo para múltiples microservicios utilizando Docker Compose, garantizando tolerancia a fallos, seguridad y persistencia.

## Arquitectura del Despliegue

Utilizaremos una **red interna de Docker**. Tailscale será el único servicio con salida a Internet (a través de Funnel). Los microservicios (Frontend, Backend, etc.) no tendrán puertos expuestos al servidor físico, comunicándose con Tailscale de forma aislada mediante el DNS interno de Docker.

---

## 1. Estructura del Proyecto

Crea una carpeta en tu servidor Ubuntu (por ejemplo, `/opt/mi-proyecto`) con la siguiente estructura:

```text
mi-proyecto/
├── docker-compose.yml
├── serve.json
├── .env
└── tailscale-data/      # (Se creará automáticamente)

```

### Configuración del `.gitignore` (Si usas Git)

Nunca subas tus secretos ni el estado del nodo al repositorio. Crea un archivo `.gitignore`:

```text
.env
tailscale-data/

```

---

## 2. Archivo de Variables de Entorno (`.env`)

Ve a tu panel de Tailscale, genera una **Auth Key** asegurándote de que sea **Reusable** y **NO Ephemeral** (no efímera). Guárdala en el archivo `.env`.

**Archivo `.env**`:

```env
# Clave de autenticación de Tailscale
TS_AUTHKEY=tskey-auth-tuClaveSecretaAqui

# Nombre fijo para tu servidor en Tailscale
TS_HOSTNAME=servidor-prod

```

---

## 3. Configuración Declarativa de Rutas (`serve.json`)

Este archivo le dice a Tailscale cómo enrutar el tráfico que entra por el puerto 443 (HTTPS) hacia los contenedores internos. Usaremos los nombres de los contenedores (`frontend`, `backend`) en lugar de `127.0.0.1`.

**Archivo `serve.json**`:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://localhost:4200"
        },
        "/api": {
          "Proxy": "http://localhost:4000"
        }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": true
  }
}
```

> **Nota:** La variable `${TS_CERT_DOMAIN}` será reemplazada automáticamente por tu dominio de Tailscale (ej. `servidor-prod.alias.ts.net`).

---

## 4. Orquestación (`docker-compose.yml`)

Aquí definimos la red interna y los servicios. Observa que el `backend` y `frontend` **no tienen la directiva `ports**`, lo que los hace 100% seguros frente a escaneos de red externos.

**Archivo `docker-compose.yml**`:

```yaml
networks:
  tailnet_internal:
    driver: bridge

services:
  # 1. Gateway: Tailscale
  tailscale:
    image: tailscale/tailscale:v1.64.0 # Buena práctica: Usar una versión específica en lugar de 'latest'
    container_name: tailscale-gateway
    hostname: ${TS_HOSTNAME}
    env_file:
      - .env
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_USERSPACE=false
    volumes:
      - ./tailscale-data:/var/lib/tailscale # Estado y criptografía persistente
      - ./serve.json:/config/serve.json # Archivo de rutas
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      - tailnet_internal
    restart: unless-stopped

  # 2. Microservicio: Backend (API)
  backend:
    image: tu-usuario/mi-backend:v1.0.0 # Cambia por tu imagen o usa `build: ./ruta`
    container_name: backend
    environment:
      - NODE_ENV=production
    networks:
      - tailnet_internal
    depends_on:
      - tailscale
    restart: unless-stopped

  # 3. Microservicio: Frontend (Web)
  frontend:
    image: tu-usuario/mi-frontend:v1.0.0 # Cambia por tu imagen o usa `build: ./ruta`
    container_name: frontend
    networks:
      - tailnet_internal
    depends_on:
      - tailscale
    restart: unless-stopped
```

---

## 5. Buenas Prácticas Productivas Aplicadas

1. **Aislamiento Total de Red (Zero Trust):** Al usar una red `bridge` de Docker (`tailnet_internal`) y no usar la directiva `ports:`, es físicamente imposible acceder a tus microservicios escaneando la IP pública de tu servidor o tu IP local. La única puerta de entrada es a través del túnel cifrado de Tailscale.
2. **Infraestructura como Código (IaC):** Con `serve.json`, las rutas de tu Reverse Proxy quedan documentadas y versionables. Si el servidor se destruye, puedes levantarlo en otro lugar en 1 minuto ejecutando `docker compose up -d`.
3. **Gestión de Secretos Segura:** Extraer la `TS_AUTHKEY` a un archivo `.env` evita que las claves de acceso queden expuestas en el código fuente de tu repositorio.
4. **Fijación de Versiones (Pinning):** Se recomienda usar una versión fija de Tailscale (ej. `v1.64.0`) en lugar de `latest`. Esto evita que una actualización automática con "breaking changes" tumbe tus servicios en producción tras un reinicio.
5. **Persistencia Total:** El volumen `./tailscale-data` garantiza que tu dominio asignado (`servidor-prod.alias.ts.net`) no cambie jamás, incluso si el servidor pierde energía o se reinicia.

---

## 6. Despliegue y Comprobación

1. **Levantar todo el stack:**

```bash
  docker compose up -d
```

2. **Verificar el estado del túnel y las rutas:**

```bash
  docker compose exec tailscale tailscale funnel status
```

3. **Verificar los logs en caso de error:**

```bash
  docker compose logs -f tailscale
```
