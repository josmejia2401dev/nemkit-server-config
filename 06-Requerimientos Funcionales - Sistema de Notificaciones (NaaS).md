# Requerimientos Funcionales - Sistema de Notificaciones (NaaS)

**Stack Tecnológico:** Node.js, Express, MongoDB, Angular, TailwindCSS, PrimeNG.
**Canales Soportados:** Email, SMS, WebPush.

---

## 1. Portal del Super Administrador (WebApp Frontend)

Módulo de uso interno para la gestión global de la plataforma.

- **Autenticación y Autorización:**
  - Iniciar y cerrar sesión con credenciales de Super Admin. (users)
  - Recuperación de contraseña.
- **Gestión de Empresas (Clientes/Tenants):**
  - Perfil de la empresa (users)
  - Crear, leer, actualizar y eliminar (CRUD) cuentas de empresas.
  - Activar, suspender o bloquear el acceso a una empresa.
  - Ver detalles de consumo de cada empresa (cantidad de mensajes enviados por tipo: Email, SMS, WebPush).
- **Dashboard Global:**
  - Visualizar métricas generales: total de empresas registradas, total de notificaciones enviadas por canal, y tasa de errores general.

---

## 2. Portal de la Empresa / Cliente (WebApp Frontend)

Plataforma de autogestión para los clientes que integrarán el servicio.

- **Registro y Autenticación:**
  - Registro de nueva empresa (Sign up) e inicio de sesión (Sign in).
- **Gestión de API Keys:**
  - Generar nuevas API Keys (para identificar el origen de las peticiones REST).
  - Revocar/Eliminar API Keys comprometidas.
  - Visualizar la API Key de forma segura (oculta por defecto, botón de copiar).
- **Gestión de Tópicos (Topics):**
  - Crear, editar y eliminar tópicos (ej. `alertas`, `marketing`).
  - Visualizar la cantidad de usuarios suscritos a cada tópico.
- **Historial y Logs de Notificaciones:**
  - Tabla de historial de notificaciones enviadas (PrimeNG Table).
  - Filtros por: fecha, canal (Email, SMS, WebPush), estado (Enviado, Fallido) y destinatario.
  - Visualización de detalle de errores en caso de fallo de envío.
- **Dashboard del Cliente:**
  - Gráficas (PrimeNG Charts) con estadísticas de envíos exitosos vs fallidos en los últimos 30 días, segmentados por canal.

---

## 3. API REST para Integración (Backend Node.js/Express)

Endpoints protegidos (requieren `API Key` en los Headers) para ser consumidos por los sistemas de los clientes.

- **Módulo de Suscriptores:**
  - `POST /api/v1/subscribers`: Registrar un nuevo usuario. Recibe un ID externo, email, teléfono y/o la suscripción (endpoint/keys) de WebPush.
  - `PUT /api/v1/subscribers/:id`: Actualizar datos de contacto de un suscriptor.
  - `DELETE /api/v1/subscribers/:id`: Eliminar un suscriptor.
- **Módulo de Suscripción a Tópicos:**
  - `POST /api/v1/subscribers/:id/topics`: Suscribir un usuario a uno o varios tópicos.
  - `DELETE /api/v1/subscribers/:id/topics/:topicId`: Desuscribir a un usuario de un tópico.
- **Módulo de Envío de Notificaciones:**
  - `POST /api/v1/notifications/send`: Envío directo a un usuario. Permite especificar los canales (ej. `channels: ['email', 'webpush']`).
  - `POST /api/v1/notifications/broadcast`: Envío masivo a todos los usuarios suscritos a un `topic` específico.

---

## 4. Motor de Notificaciones (Backend Core)

Lógica interna del servidor para procesar los mensajes.

- **Enrutamiento Multicanal:**
  - Interpretación del payload para derivar el mensaje al servicio de Email, SMS o WebPush según se solicite.
- **Procesamiento Asíncrono / Colas:**
  - Encolamiento de las peticiones de envío masivo (Broadcast) para evitar bloqueos en el servidor y manejar reintentos en caso de fallo.

---

## 5. Notas Arquitectónicas y de Diseño

### Implementación del Patrón Strategy

Para el Motor de Notificaciones, el **Patrón Strategy** es la opción ideal. Permite aislar la lógica de cada proveedor sin llenar el código de condicionales `if/else` o `switch`.

- **Concepto:** Creas una interfaz base (ej. `NotificationStrategy` con un método `send(payload)`).
- **Implementaciones:** Creas clases concretas: `EmailStrategy`, `SmsStrategy`, y `WebPushStrategy`.
- **Fábrica/Contexto:** Un manejador evalúa el evento o array de `channels` recibido en el request y ejecuta dinámicamente el `send()` de las estrategias correspondientes. Si a futuro agregas notificaciones Push Móvil, solo creas una nueva estrategia sin tocar el código existente.

### WebPush vs WebSockets

- **No requieres WebSockets para implementar WebPush.**
- **WebPush (Push API + VAPID):** Es el estándar de los navegadores. Funciona a través de un **Service Worker** en el frontend del cliente. El usuario debe recibir la notificación solo si tiene una sesión activa y esta la pestaña del navegador abierta.
