# 01 · Diagnóstico del producto actual

Fuente: `Ronybercol/bercobackoffice` en `main` al 2026-09-17. Piezas: `backend/properties/views_owner_public.py`, `templates/owner_upload/page.html`, `services/owner_ai_pipeline.py`, `services/floorplan_render.py`, `services/video_frames.py`, `management/commands/process_owner_analyses.py`, `src/components/properties/OwnerUploadDialog.tsx`, `OwnerAIAnalysisSection.tsx`. Todo entró el 2026-07-09 (WP1-WP6) con un fix el 07-10 (WP7).

## Lo que está bien y hay que conservar

- **El link firmado sin login** (`upload_links.py`): token `django.core.signing` + estado en DB, revocable, con vencimiento, OG dinámico para WhatsApp, UTM que resuelve a `difusion_whatsapp`. Sólido. Se reutiliza tal cual.
- **La superficie pública en `re.bercovich.com/p/<token>`** con guard de host, sniff de MIME, caps (60 archivos / 300 MB), rate limit y fallback sin JS. Se reutiliza como cáscara.
- **El modelo de estados** `PropertyAIAnalysis` (queued → running → draft → approved → returned → validated / failed) con versionado. Es exactamente el ciclo que necesita un plano con gate humano. Se reutiliza.
- **La idempotencia** de `PropertyImage(source, external_id)` y la extracción de frames con PySceneDetect + filtro de blur (`video_frames.py`). Reutilizable como etapa de muestreo.
- **La devolución al dueño** (validar / "hay algo distinto" / pedir tasación) y las notificaciones al asesor. Se conserva el circuito; cambia el contenido.
- **La sección "Análisis del dueño"** en la ficha: tabla editable de ambientes, estado de conservación con evidencia. Se conserva y se le agrega el editor del plano.

## Por qué no alcanzó

### 1. La captura no guía: el dueño decide qué filmar y decide mal

`page.html` muestra tres bullets ("fotos con buena luz, una por ambiente", "si tenés un plano viejo sirve", "un video corto ayuda") y un botón "Subir fotos y videos". No hay lista de ambientes, no hay progreso, no hay control de calidad en el teléfono, no hay orden. El resultado típico es un set incompleto (faltan baños, pasillos, balcón), fotos de detalle (una canilla, un mueble) y video acelerado. Todo lo que falta aguas arriba, el pipeline lo inventa aguas abajo.

### 2. El pipeline no mide: estima con un modelo de lenguaje

- **Etapa 3** (`_stage3_measure`): le pide a Gemini `width_m`, `length_m`, `area_m2` por ambiente "usando anclas de referencia" (puerta 2,00 × 0,75, cerámico 0,30-0,60, mesada 0,60). Es una heurística de razonamiento visual, no una medición. Los benchmarks 2026 sobre lectura de planos por VLMs (Blueprint-Bench, AECV-bench) dan ~50 % de acierto en conteos simples; en metros sobre fotos de interiores el error típico es de ±20-40 %.
- **`_scale_areas_to_total`**: fuerza la suma de ambientes a `Property.superficie` (±5 %). Si el campo trae superficie total con muros, balcón y partes comunes, distorsiona todo; si no está, no corrige nada.
- **Etapa 4** (`_stage4_layout`): le pide a Gemini un `bbox {x,y,w,h}` en escala 0-100 por ambiente "coherente con una distribución real". El modelo no vio la casa: adivina. La validación sólo chequea solapamiento (<30 %) y proporción de áreas (±20 %). Si falla dos veces, `_fallback_grid_layout` pone **todas las cajas en una fila**.
- **`floorplan_render.py`**: dibuja el bbox de cada ambiente como rectángulo, un rectángulo exterior envolvente como "muro", una abertura por adyacencia y la leyenda *"CROQUIS APROXIMADO — NO A ESCALA"*. No hay muros reales, ni forma del ambiente, ni cotas, ni orientación, ni ventanas. Es un diagrama de bloques, no un plano.

Conclusión: aunque cada etapa funcione "bien", el techo del producto es un diagrama con superficies de ±30 %. Ningún dueño reconoce su casa ahí, y por eso el circuito de validación y tasación no tracciona.

### 3. El video, que es la mejor fuente de geometría, es ciudadano de segunda

`run_pipeline` sólo extrae frames del video si hay menos de 6 fotos. Y cuando los extrae, los trata como fotos sueltas (clasificar, agrupar, estimar). Se pierde lo único que un video tiene y las fotos no: **continuidad espacial**. Una secuencia continua permite reconstruir poses de cámara y un modelo 3D métrico; fotos sueltas, no.

Además el video se sube entero (hasta 200 MB) sin compresión ni segmentación en el cliente. En 4G real de CABA eso falla, y el reintento vuelve a subir todo.

### 4. La devolución al dueño no permite corregir

En modo `review` el dueño ve el SVG y tiene dos botones y 280 caracteres. No puede renombrar un ambiente, marcar uno faltante ni corregir una medida. Cada "regenerar" en el backoffice vuelve a correr las 5 etapas desde cero (nuevo costo, nuevo resultado aleatorio) sin incorporar lo que el dueño dijo.

### 5. No hay instrumentación

`page.html` deja un `TODO GA4` sin measurement ID: no sabemos cuántos links se abrieron, cuántos empezaron, cuántos terminaron, cuánto tardaron. No se puede mejorar lo que no se mide. Hoy la única traza es `uploads_count` en el link.

### 6. Promesa vs. realidad

El WhatsApp dice "te toma 90 segundos" y la página de gracias dice "en unos días tu asesor te comparte el plano". El cron corre cada 10 minutos y el pipeline tarda minutos, así que "unos días" es una expectativa inventada que enfría al dueño en el momento de máxima intención.

### 7. Infra

Todo corre en la EC2 t3.micro (1 GB RAM) que ya está al límite con Pillow (`docs/media_pipeline_v2.md`). Cualquier reconstrucción 3D necesita GPU y no puede convivir ahí.

## Qué se aprende para la v2

| Lección | Implicancia de diseño |
|---|---|
| El material define el techo del resultado | Invertir en la captura guiada, no en más etapas de IA |
| Un LLM no mide | La geometría sale de reconstrucción 3D; el LLM etiqueta, describe y valida coherencia |
| El dueño sabe en qué ambiente está | Pedírselo (un toque) elimina la agrupación probabilística y da segmentación perfecta |
| La escala es el problema central | Tres fuentes de escala en cascada + cierre contra escritura o plano de mensura |
| El dueño quiere reconocer su casa | Forma real de los ambientes, muros, puertas, ventanas y cotas; y un 3D que se pueda girar |
| Sin gate humano no se publica | Editor paramétrico en el backoffice; el asesor o Ariel aprueban |
| Medir el funnel | Eventos en cada paso desde el día 1 |
