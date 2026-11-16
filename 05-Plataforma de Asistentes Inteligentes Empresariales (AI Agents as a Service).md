# Plataforma de Asistentes Inteligentes Empresariales (AI Agents as a Service)

**Definición Core:** Agentes de IA que consultan conocimiento (vía RAG), consumen APIs, ejecutan acciones, mantienen contexto y siguen reglas de negocio.

## 🛠️ Arquitectura de IA y Entorno de Desarrollo (Hardware Limitado: 8GB RAM / i5 4th Gen)

Para garantizar que el desarrollo en la máquina local sea fluido y sin bloqueos por falta de memoria, el sistema se basará en las siguientes decisiones técnicas:

1. **Gestor de Modelos Locales (Inferencia):** Uso de **Ollama** con modelos ultra ligeros (SLMs).
   - _Recomendación para Dev:_ `qwen2.5:1.5b` (ocupa ~1.2GB de RAM) o `gemma2:2b` (~1.6GB de RAM). Dejarán RAM libre para Node.js, Angular y el SO.
2. **Base de Datos y Vectores en la Nube:** Uso de **MongoDB Atlas (Free Tier)** con _Vector Search_ integrado. Esto evita consumir RAM local levantando una base de datos vectorial adicional.
3. **Estrategia de Conocimiento (RAG vs Fine-Tuning):**
   - El conocimiento de los documentos de las empresas **NO se enseñará reentrenando** el modelo. Se usará estrictamente **RAG (Retrieval-Augmented Generation)** con embeddings ligeros (ej. `nomic-embed-text` o embeddings gratuitos de HuggingFace/Cohere).
   - El reentrenamiento (Fine-Tuning con LoRA) se reservará para fases avanzadas (producción en servidores con GPU) únicamente para afinar el comportamiento o la clasificación de intenciones.

---

## 🚀 Fase 1: MVP Comercializable (Go-to-Market)

_Esta fase incluye únicamente lo necesario para comercializar, validar con clientes y generar valor inmediato._

### 1. Arquitectura Base

- **RF-000 Multiempresa (Tenancy):** Sistema SaaS con aislamiento lógico por cliente en MongoDB.

### 2. Constructor de Asistentes (MVP 1.0)

- **RF-001 Crear asistente:** Definir Nombre, Descripción, Logo, Idioma y Prompt del sistema (Instrucciones base que guían al SLM local).
- **RF-002 Editar asistente:** Modificar comportamiento, personalidad y temperatura.
- **RF-003 Eliminar asistente:** Borrado lógico del agente.

### 3. Base de Conocimiento (RAG Local/Cloud)

- **RF-004 Subir documentos:** Soporte para PDF, TXT, CSV y Markdown (priorizar texto plano para reducir carga de procesamiento local).
- **RF-005 Administración de fuentes:** Listado y gestión de archivos subidos.
- **RF-006 Indexación automática (Pipeline optimizado):**
  - Extracción de texto -> Fragmentación (Chunks pequeños) -> Generación de Embeddings -> Guardado en MongoDB Atlas Vector Search.
- **RF-007 Reindexar conocimiento:** Actualizar documentos sin necesidad de reentrenar ningún modelo.

### 4. Chat y Pruebas

- **RF-008 Chat de pruebas:** Interfaz en el panel de admin para conversar con el SLM de Ollama de forma local.
- **RF-009 Historial de chat:** Registro de Pregunta, Respuesta, Fecha y Usuario.
- **RF-010 Feedback:** Sistema de calificación (👍/👎) en las respuestas.

### 5. Publicación Básica

- **RF-011 Widget Web:** Script embebible (`<script src="chat.js"></script>`).
- **RF-012 Personalización UI:** Ajuste de color principal, logo y nombre.
- **RF-021 Integración WhatsApp:** Conexión vía Meta Cloud API (Webhooks directos a Node.js).

### 6. Conectores y Herramientas (MVP 1.5)

- _Nota Técnica:_ Al usar SLMs de 1.5B/2B, las instrucciones para retornar JSON deben ser muy estrictas (Structured Outputs).
- **RF-013 Registrar API REST:** Configurar URL, Método, Headers.
- **RF-014 Parámetros dinámicos:** Mapeo de variables en rutas (ej. `GET /clientes/{clienteId}`).
- **RF-016 Crear herramienta:** Definir acciones específicas.
- **RF-017 Asociar herramienta:** Vincular herramientas al agente.
- **RF-018 Ejecución automática (Function Calling básico):** El modelo decide cuándo llamar a la API e inyecta la respuesta.

---

## 🗺️ Roadmap de Evolución (Próximas Fases)

### Fase 2: Omnicanalidad y Reglas

- **Reglas de Negocio (RF-019, RF-020):** Condicionales pre-prompteados (ej. "Si deuda > 60 días, rechazar").
- **Nuevos Canales (RF-023 a RF-027):** Messenger, Instagram, Bandeja Única.

### Fase 3: Agentes Inteligentes y Flujos

- **Memoria de Usuario (RF-028, RF-029):** Mantener el contexto histórico.
- **Constructor Visual (RF-030 a RF-032):** Interfaz basada en nodos tipo n8n.
- **Delegación (RF-033, RF-034):** Un modelo "Router" ultra ligero (clasificador) que enruta a diferentes agentes.

### Fase 4: Inteligencia Avanzada y Producción Cloud

- **Gestión Multi-LLM (RF-038, RF-039):** Migración a la nube permitiendo elegir APIs (OpenAI, Claude) o SLMs en VPS dedicados (vLLM).
- **Adaptadores LoRA (RF-041):** Reentrenamiento de adaptadores basados en el feedback (👍/👎) para afinar precisión en llamadas a APIs.
