# Product Requirements Document (PRD) — CIP (Captador Inteligente de Propiedades)

> Documento canónico de requisitos y especificación de producto para **CIP (Captador Inteligente de Propiedades)**.

---

## 1. Visión y Propósito del Producto

**CIP** es una solución tecnológica diseñada para empoderar a asesores y agencias inmobiliarias durante la captación de inmuebles en campo. 
Su propósito es reducir radicalmente el tiempo de captura de una nueva propiedad (de más de 20 minutos a menos de 2 minutos) combinando:
- **Captura manual rápida y asistida**.
- **Dictado de voz inteligente con extracción de entidades tipadas (LLM en la nube)**.
- **Procesamiento y optimización automática de imágenes**.
- **Persistencia relacional segura en PostgreSQL 18 con trazabilidad y auditoría inmutable**.

---

## 2. Antecedentes y Separación Arquitectónica

CIP fue concebido e implementado inicialmente sobre la infraestructura arquitectónica de **Golden Starter V3**:
- Utiliza la arquitectura desacoplada **Backend for Frontend (BFF)** con Next.js 16, React 19 y Tailwind CSS 4.
- Integra un microservicio satélite de IA en Python 3.12 (Flask + Celery + Redis) aislado en red interna Docker.
- Mantiene contratos de datos estrictos compartidos con Zod (`@starter/contracts`).

Para garantizar un ciclo de vida limpio y enfocado, se formalizó la separación:
- `GoldenStarterV3` continúa como esqueleto base reutilizable para cualquier industria.
- **`BR-CIP`** (este repositorio) asume la titularidad completa del producto inmobiliario, su modelo de negocio, sus reglas específicas y su evolución hacia producción.

---

## 3. Requerimientos Funcionales Implementados (MVP / V1)

### RF-01: Autenticación y Control de Acceso
- El asesor inicia sesión de forma segura a través de **Better Auth**.
- Soporte para roles y permisos por organización multi-tenant.

### RF-02: Formulario Híbrido de Alta (`/propiedades/nueva`)
- **Campos estructurados:** Título, tipo de propiedad (`casa`, `departamento`, `terreno`, `comercial`), precio, moneda (`MXN`/`USD`), dirección, coordenadas GPS, superficie de terreno ($m^2$), superficie de construcción ($m^2$), recámaras, baños, estacionamientos, descripción y acabados.
- **Módulo de Dictado por Voz:**
  - Grabación directa desde el navegador o subida de archivo `.m4a`, `.mp3`, `.wav`.
  - Envío seguro a S3 y delegación asíncrona a Celery para transcripción y extracción JSON.
  - Autocompletado reactivo de los campos del formulario.
- **Resolución de Conflictos (Smart Merge):**
  - Si un campo ya fue capturado manualmente, la IA no lo sobrescribe a menos que el usuario lo confirme explícitamente.

### RF-03: Carga y Optimización de Fotos
- Selección múltiple de fotos del inmueble.
- Carga hacia S3 con validación de tipo MIME y tamaño.
- Llamada síncrona al microservicio de Python para extraer dimensiones (`width`, `height`) y validar integridad.

### RF-04: Persistencia y Auditoría
- Validación final con esquema Zod `propertyCreateSchema`.
- Persistencia en tabla `properties` de PostgreSQL 18 con Drizzle ORM.
- Creación de registro inmutable en `audit_logs`.

### RF-05: Cartera de Inmuebles (`/propiedades`)
- Visualización de propiedades capturadas mediante `GenericCrudDataGrid`.
- Filtros por tipo, rangos de precio, paginación y ordenamiento.

---

## 4. Requerimientos No Funcionales

1. **Aislamiento de IA:** El microservicio de Python (`backend-ai/`) no tiene acceso directo a la base de datos PostgreSQL; la comunicación se realiza exclusivamente a través del BFF Next.js enviando `s3_keys`.
2. **Restricción de Modelos:** Queda prohibido el uso de modelos locales autohospedados (ej. Ollama). Se usan exclusivamente SDKs oficiales en la nube (OpenAI, Gemini).
3. **Resiliencia y Concurrencia:** Las tareas de transcripción y extracción pesada se delegan a Celery con broker Redis para evitar bloquear el hilo del servidor web.
4. **Seguridad y Redacción de Secretos:** Los logs de la aplicación pasan por un logger redactado que enmascara automáticamente tokens, contraseñas y claves privadas.

---

## 5. Roadmap de Evolución hacia Producción

- **V1.1 (WhatsApp Ingestion):** Ingesta de notas de voz y fotos enviadas por asesores a través de WhatsApp Cloud API.
- **V1.2 (Geolocalización & Mapas):** Búsqueda geográfica interactiva con Mapbox / OpenStreetMap y análisis de radio.
- **V1.3 (Valuación AVM):** Estimación de valor por $m^2$ y generación de fichas técnicas en PDF.
- **V1.4 (Sindicación MLS):** Exportación automática a portales inmobiliarios (EasyBroker, Inmuebles24, etc.).
- **V2.0 (Producción Cloud):** Despliegue contenerizado con Docker Compose y Caddy con SSL automático en VPS Linux.
