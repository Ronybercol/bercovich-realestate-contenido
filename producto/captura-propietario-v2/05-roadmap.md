# 05 · Roadmap, equipo, métricas y riesgos

## Fases

### Semana 0-1 · Spike técnico y instrumentación (gate: seguir o no)

| # | Tarea | Quién | Entregable |
|---|---|---|---|
| 0.1 | Filmar 5 propiedades reales con el guion del 03 (un departamento chico, uno grande, un PH con patio, una casa de 2 plantas, una propiedad vacía) usando el prototipo, con teléfonos Android y iPhone; conseguir el plano de mensura de cada una | Sabri + Ariel + Rony | 5 videos + 5 planos verdad de terreno |
| 0.2 | Correr DA3 (Nested/Streaming) y MapAnything sobre los 5 videos en una GPU on-demand; extraer polígonos por ambiente con el heurístico; medir error de superficie por ambiente y total contra la mensura | Rony | Tabla de errores; decisión de modelo |
| 0.3 | Benchmark opcional con SpatialLM 1.1 (uso interno) y con un plano CubiCasa de una misma propiedad | Rony | Techo de calidad conocido |
| 0.4 | Eventos del funnel en el link actual (v1) para tener línea de base | Nico | Dashboard simple |
| 0.5 | Activar S3 + CloudFront (Fase 2 de `media_pipeline_v2.md`) | Ale | Storage listo para video |

**Gate:** error mediano de superficie total < 10 % y por ambiente < 15 % en al menos 4 de 5 propiedades, con escala por puerta + modelo métrico. Si no se alcanza: plan B = CubiCasa GoToScan para el Nivel A mientras se sigue investigando.

### Sprint 1 (semanas 2-3) · Captura guiada v2

- PWA de captura en `/p/<token>` (landing, pre-check, calibración, recorrido por ambiente, QA en vivo, extras, cierre) — Nico, a partir del prototipo.
- Endpoints de captura chunked + `PropertyCapture` / `CaptureSegment` + eventos — Ale.
- Video tutorial de 30 s para la landing — Nakala.
- Flujo v1 como degradación.

### Sprint 2 (semanas 4-5) · Pipeline geométrico y plano 2D

- Worker GPU (Modal/Replicate) con las etapas 1-5 y 7 (2D) — Rony.
- `PropertyFloorplan` + integración con `process_owner_analyses` — Ale.
- Etapa 6 (semántica) reusando el pipeline Gemini actual + lectura de plano de mensura — Rony.
- Editor paramétrico mínimo en la ficha (renombrar, unir, vértices, cota exacta, aprobar) — Nico.

### Sprint 3 (semanas 6-7) · 3D, revisión del dueño v2 y piloto

- GLB 3D por extrusión + visor three.js en el modo `review` — Nico.
- Correcciones estructuradas del dueño + versionado — Ale.
- Entrega por WhatsApp (mensaje `review`, ya existe) con SLA real.
- **Piloto: 20 propiedades reales** de Usados (Sabri) + 5 de Locales (Tommy) para ver si el flujo sirve también en comercial.

### Sprint 4 (semanas 8-10) · Nivel B y cierre contra mensura

- App iOS RoomPlan interna (o importador Polycam/magicplan como puente) — Rony o externo.
- Compra de 1-2 iPhone Pro para captadores.
- Vectorización del plano de mensura compartida con Emprendimientos (tarea del roadmap 16/09).
- Plano "publicable": se exporta a la ficha pública y a portales.

## Equipo y roles

| Persona | Rol en este producto |
|---|---|
| Rony | Arquitectura, pipeline geométrico, decisiones de modelo, spike |
| Nico Zalcman | Frontend de captura (PWA), editor paramétrico, visor 3D |
| Ale Debard | Backend Django (endpoints chunked, modelos, cola, S3), infra GPU, calidad |
| Ariel Sztajn | Validación arquitectónica de planos, criterio de aprobación, priors del Código de Edificación |
| Sabri | Piloto con dueños reales, feedback de campo, guion de WhatsApp |
| Tommy / Locales | 5 propiedades comerciales en el piloto |
| Nakala | Video tutorial de 30 s y placa OG |
| Steven | Revisión del disclaimer legal del plano ("aproximado", uso comercial) |

## Métricas de éxito (piloto)

| Métrica | Hoy (v1) | Objetivo piloto |
|---|---|---|
| Links abiertos que completan la captura | desconocido (no se mide) | > 60 % |
| Tiempo de captura mediano | — | < 5 min |
| Segmentos con calidad "bien" | — | > 80 % |
| Error mediano de superficie total vs. mensura | ±30 % (estimado) | < 8 % (Nivel A), < 2 % (Nivel B) |
| Plano en `draft` desde `capture_done` | horas/días | < 15 min |
| Plano aprobado y enviado al dueño | — | < 24 h |
| Dueños que validan el plano | — | > 70 % |
| Dueños que piden tasación | — | > 30 % |
| Costo por propiedad | ~USD 0,10 (LLM) | < USD 0,50 |

## Riesgos y mitigaciones

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| La reconstrucción métrica no llega al umbral en departamentos con paredes lisas y poca luz | Media | Calibración con puerta + medida del dueño + cierre contra superficie declarada; plan B CubiCasa; Nivel B para publicar |
| MediaRecorder en iOS Safari corta o cambia de códec | Media | Chunks de 10 s, segmentos < 60 s, pruebas en iOS 16/17/18/26 en la semana 1; fallback a fotos |
| Dueños mayores no completan | Media | Tutorial de 30 s, opción "que lo haga un familiar", y la visita del asesor (Nivel B) siempre existe |
| Licencias NC en modelos | Baja si se respeta la tabla del 04 | Producción sólo Apache-2.0; SpatialLM sólo benchmark |
| Costo GPU se dispara con videos largos | Baja | Cap de 40 segmentos / 8 min; muestreo a 3 fps; GPU spot |
| Exposición legal por un plano con medidas mal publicadas | Media | Gate humano obligatorio; rótulo de confianza; "publicable" sólo con mensura o Nivel B; texto legal revisado por Steven |
| Privacidad del video de la casa | Baja | Retención 90 días, aviso, URLs firmadas |
| El equipo tech está cargado (Terrenos.ai, Backoffice, OBRAS) | Alta | Spike de 1 semana antes de comprometer sprints; el prototipo ya está hecho; considerar un freelance para la app RoomPlan |

## Decisiones pendientes (para Rony)

1. Aprobar el spike de la semana 0-1 y quién filma las 5 propiedades.
2. Elegir proveedor GPU on-demand (Modal vs. Replicate vs. RunPod).
3. Comprar iPhone Pro para captadores: sí/no, cuántos.
4. ¿El Nivel A se ofrece a cualquier dueño en captación fría (lead magnet "tu plano gratis") o sólo a propiedades ya en pipeline? Cambia el volumen y el costo.
5. ¿Se carga este plan al roadmap de Notion (DB Tasks) como épica con las tareas de arriba? Se puede hacer en cuanto lo confirmes.

## Relación con el resto del roadmap

- **"Productizar visita usado (Foto Self Serve)"** (CEO Board, Sabri/Rony/Ariel): queda cubierto por Nivel A + Nivel B.
- **"Piloto planta 2D equipada y prompteable (2525)"** (16/09): comparte el modelo paramétrico, el render SVG por código y el vectorizador de planos. Conviene que `PropertyFloorplan.model_json` sea el mismo formato en los dos productos.
- **"Auto-asignación de planos por unidad"** (16/09): el OCR de rótulos sirve también para leer el plano de mensura del dueño.
- **Tasador con comparables**: el plano con superficie cubierta/descubierta real alimenta la tasación con un dato que hoy se pide a mano.
- **Informe al dueño (M16)**: el plano 2D/3D entra al informe.
