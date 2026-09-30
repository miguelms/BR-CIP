# CIP — Captador Inteligente de Propiedades (`BR-CIP`)

> Plataforma especializada de captura en campo, enriquecimiento multimodal por Inteligencia Artificial (Voz + Visión) y gestión integral de cartera inmobiliaria. Construida bajo el patrón **Backend for Frontend (BFF)** con Next.js 16, React 19, PostgreSQL 18 + Drizzle ORM y un microservicio satélite de IA en Python/Flask/Celery.

---

## 🏢 Contexto y Separación Arquitectónica

Este repositorio (**`BR-CIP`**) es el hogar oficial y exclusivo de la aplicación de negocio para **Bienes Raíces CIP**:
- **Origen Arquitectónico:** Nació a partir de los estándares y la infraestructura canónica de **Golden Starter V3** (Next.js 16 App Router, React 19, Tailwind CSS 4, Shadcn UI, Better Auth, PostgreSQL 18, microservicio satélite de IA en Docker y catálogo de 17 agentes autónomos de Antigravity).
- **Separación de Responsabilidades:** Se separó formalmente del starter general para que `GoldenStarterV3` permanezca como plantilla virgen agnóstica a cualquier industria, mientras que **`BR-CIP` concentra toda la lógica de negocio inmobiliario, experiencia del asesor en campo y roadmap hacia una aplicación real de producción**.

---

## ✨ Capacidades Implementadas (MVP / V1 Funcional)

### 1. Formulario Híbrido de Captura (`/propiedades/nueva`)
- **Captura Manual Completa:**
  - Título, tipo de inmueble (`casa`, `departamento`, `terreno`, `comercial`), precio, moneda (`MXN` / `USD`), dirección completa y coordenadas GPS.
  - Superficie de terreno ($m^2$), superficie de construcción ($m^2$), recámaras, baños, cajones de estacionamiento, acabados y notas adicionales.
- **Asistente de Voz Complementario:**
  - Grabador de audio directo en el navegador con interfaz visual de estados (*"Grabando audio..."*, *"Subiendo a S3..."*, *"Extrayendo datos con IA..."*).
  - Soporte para subir archivos de voz pregrabados en campo (`.m4a`, `.mp3`, `.wav`).
- **Enrutamiento Asíncrono BFF:**
  - Next.js sube el binario a AWS S3 (o almacenamiento local seguro en desarrollo) y obtiene el `s3Key`.
  - Notifica a Flask (`POST /api/v1/voice/transcribe`) enviando únicamente la referencia de S3.
  - Flask encola la tarea en Celery + Redis y retorna de inmediato `{ "task_id": "...", "status": "queued" }`.
  - Next.js sondea el progreso y recibe las entidades extraídas por el LLM en la nube en formato JSON estructurado.
- **Fusión Inteligente y Resolución de Conflictos (Smart Merge):**
  - Si el asesor ya escribió manualmente datos (ej. Precio o Recámaras), la IA **no los sobrescribe arbitrariamente**.
  - El sistema detecta discrepancias y permite elegir entre *"Conservar datos manuales y rellenar vacíos"* o *"Aplicar sugerencias de la IA"*.

### 2. Procesamiento y Optimización de Fotos
- Carga múltiple de imágenes de la propiedad con previsualización en miniatura.
- Subida directa a S3 y notificación síncrona a Flask (`POST /api/v1/images/process`) para validación de formato, extracción de dimensiones (`width`, `height`) y metadatos básicos.

### 3. Persistencia Relacional y Auditoría Inmutable
- Validación rigurosa de contratos de entrada con esquemas Zod en `@starter/contracts`.
- Persistencia en base de datos PostgreSQL 18 mediante Drizzle ORM sobre la tabla `properties`.
- Registro de auditoría append-only en `audit_logs` con trazabilidad del usuario, organización y timestamp.

### 4. Cartera de Propiedades (`/propiedades`)
- Dashboard responsivo basado en el componente canónico `GenericCrudDataGrid`.
- Filtrado por tipo de propiedad, ordenamiento por precio o fecha, badges de estado y acceso rápido al detalle.

---

## 🏗️ Arquitectura de la Solución

El sistema opera mediante una topología desacoplada y contenerizada en **Docker Compose**:

```text
                                  ┌──────────────────────────────────────────────┐
                                  │               RED INTERNA DOCKER             │
                                  │                                              │
┌───────────────────────────┐     │  ┌──────────────┐      ┌──────────────────┐  │
│   Cliente Web / Móvil     │     │  │  PostgreSQL  │      │  Redis 7 Alpine  │  │
│   (Next.js 16 + React 19) │────┼──▶    18 DB      │      │  (Broker Celery) │  │
│       [Puerto 3000]       │     │  └──────────────┘      └────────┬─────────┘  │
└─────────────┬─────────────┘     │                                 │            │
              │                   │                                 ▼            │
              │ (BFF / S3 Key)    │  ┌──────────────┐      ┌──────────────────┐  │
              └───────────────────┼─▶│  Flask API   │─────▶│  Celery Worker   │──┼──▶ SDKs Cloud
                                  │  │  (Satélite)  │      │ (Whisper/Gemini) │  │   (OpenAI/Gemini)
                                  │  └──────────────┘      └──────────────────┘  │
                                  └──────────────────────────────────────────────┘
```

- **Aislamiento Seguro:** Solo `next-app` y `db` exponen puertos al host. `flask-api`, `redis` y `celery-worker` permanecen aislados en la red bridge privada `internal-network`.
- **Restricción Estricta de Modelos:** Queda prohibido el uso de modelos locales (como Ollama); la transcripción y el análisis estructurado utilizan exclusivamente SDKs oficiales en la nube (OpenAI Whisper / Gemini).

---

## 📁 Estructura del Repositorio

```text
├── src/
│   ├── app/
│   │   ├── propiedades/
│   │   │   ├── page.tsx             # Cartera de propiedades con DataGrid
│   │   │   └── nueva/page.tsx       # Formulario interactivo híbrido (Manual + Voz)
│   │   └── api/
│   │       ├── properties/          # Route Handler para persistir en PostgreSQL 18
│   │       ├── upload/              # Carga de audios e imágenes a AWS S3
│   │       └── ai/                  # Proxy hacia el microservicio satélite Flask
│   ├── components/                  # Shadcn UI + Galería de fotos + Grabador de voz
│   ├── db/                          # Esquemas Drizzle (properties, users, audit_logs)
│   └── lib/                         # Clientes S3, Better Auth y logger redactado
│
├── backend-ai/                      # Microservicio satélite de IA (Python 3.12)
│   ├── app.py                       # Servidor Flask con CORS interno y fábrica Celery
│   ├── tasks.py                     # Tareas Celery de transcripción y extracción JSON
│   ├── Dockerfile                   # Contenedor aislado de Python
│   └── requirements.txt             # flask, celery, redis, boto3, openai, google-genai
│
├── packages/contracts/              # Esquemas Zod compartidos (properties, voz, fotos)
├── specs/                           # Especificaciones SDD (FEAT-CIP-001)
├── tests/
│   ├── unit/                        # Tests Vitest (validación Zod, normalización de voz)
│   └── e2e/                         # Pruebas Playwright (flujo de captura completo)
└── .agents/                         # Catálogo de 17 agentes especializados Antigravity
```

---

## 🛠️ Comandos de Desarrollo y Pruebas

```bash
# 1. Instalar dependencias
npm install

# 2. Configurar variables de entorno
cp .env.example .env

# 3. Levantar infraestructura de soporte (PostgreSQL + Redis)
docker compose up -d db redis

# 4. Iniciar servidor de desarrollo Next.js
npm run dev

# 5. Ejecutar suite de pruebas unitarias (Vitest)
npm run test:unit

# 6. Comprobación de tipos TypeScript
npm run typecheck

# 7. Validar catálogo de 17 agentes de Antigravity
npm run check:agents

# 8. Ejecutar pruebas de extremo a extremo (Playwright)
npm run test:e2e:web
```

---

## 🗺️ Roadmap de Evolución hacia Producción

Para llevar CIP al nivel de un producto comercial SaaS de alta disponibilidad:

### Fase 2: Ingesta Inmobiliaria por WhatsApp Business
- Conectar webhooks de WhatsApp Cloud API para que el asesor envíe una nota de voz y fotografías directamente desde un chat grupal o individual.
- Tarea Celery procesa el mensaje de WhatsApp, genera el borrador de la propiedad y envía un enlace mágico al asesor para revisar y publicar.

### Fase 3: Geolocalización Avanzada y Mapas Interactivos
- Integración de Mapbox / OpenStreetMap con autocompletado de direcciones por API catastral.
- Vista de mapa interactiva en `/propiedades` con clusters y búsqueda por polígono o radio en kilómetros.

### Fase 4: Modelo de Valuación Automática (AVM) y Comparables
- Módulo en Flask que calcula el valor estimado por $m^2$ comparando con inmuebles similares de la cartera histórica o fuentes externas.
- Generación automática de ficha técnica comercial en PDF con branding de la agencia inmobiliaria.

### Fase 5: Sindicación y Multidifusión a Portales (MLS)
- Conexión con APIs de difusión masiva (EasyBroker, Inmuebles24, Lamudi, Vivanuncios) para sincronizar inventario automáticamente con un clic.

### Fase 6: Infraestructura de Producción y SRE
- Despliegue automatizado con Docker Compose y Caddy como proxy inverso con certificados SSL automáticos de Let's Encrypt.
- Backups automáticos de PostgreSQL 18 hacia bucket privado S3 con rotación de snapshots y monitoreo de salud.

---

## 🤖 Desarrollo Guiado por Especificaciones (SDD)

Todas las nuevas características de CIP se gestionan a través de los agentes de **Antigravity**:
1. Especificación formal en `specs/` (ver [`specs/cip-captador-inteligente.md`](specs/cip-captador-inteligente.md)).
2. Desglose en `TASKS.md` por `change-planner-agent`.
3. Implementación respetando contratos en `@starter/contracts`.
4. Verificación de gates (`typecheck`, `test:unit`, `test:e2e:web`) por `qa-agent`.
