# 04 · Arquitectura técnica

Regla de oro (la misma del 16/09 para Emprendimientos): **la geometría se calcula, el dibujo se genera por código, nunca es imagen generativa.** El LLM etiqueta, describe y valida; no mide ni dibuja.

## Vista general

```
 Dueño (navegador, cualquier teléfono)                         Asesor (iPhone Pro)
 ┌──────────────────────────────────┐                          ┌──────────────────┐
 │ PWA de captura guiada /p/<token> │                          │ App RoomPlan     │
 │  · MediaRecorder por ambiente    │                          │  · JSON + USDZ   │
 │  · tags de ambiente + timestamps │                          └────────┬─────────┘
 │  · QA en vivo (blur/luz/velocid.)│                                   │
 │  · WebXR hit-test (Android, opc.)│                                   │
 └───────────────┬──────────────────┘                                   │
                 │ chunks (≤10 s) + metadata                            │
                 ▼                                                      ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ Backoffice Django (EC2)  — API pública /api/public/owner/<token>/capture/*    │
 │  · PropertyCapture / CaptureSegment (nuevo)  · PropertyFloorplan (nuevo)      │
 │  · cola: PropertyAIAnalysis(status=queued)   · storage: S3 (activar Fase 2)   │
 └───────────────┬──────────────────────────────────────────────────────────────┘
                 │ job                                                  ▲
                 ▼                                                      │ JSON paramétrico
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ Worker GPU on-demand (Modal / Replicate / RunPod)                             │
 │  1 muestreo frames  2 reconstrucción métrica (DA3 / MapAnything)              │
 │  3 piso + segmentación por ambiente (tags)  4 polígonos + muros + aberturas   │
 │  5 fusión de escala  6 LLM: etiquetas, aberturas, estado, coherencia          │
 │  7 render: SVG 2D con cotas · GLB 3D · PNG/PDF con marca                      │
 └──────────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼
 Editor paramétrico en el backoffice (gate humano) → returned al dueño → validated
```

## 1. Captura (frontend, en `templates/owner_upload/`)

- **Misma URL y misma cáscara** (`/p/<token>`, ctx JSON, fallback sin JS). Se reemplaza el cuerpo del modo `upload` por la PWA de captura; el flujo v1 queda como "¿Preferís mandar fotos?".
- **Stack**: HTML + JS sin framework (como hoy) o Preact si el estado crece. `getUserMedia` (cámara trasera, 1280×720 a 30 fps es suficiente y pesa 4-6 MB/min con H.264/VP9), `MediaRecorder` con `timeslice=10000` para chunks de 10 s.
- **Segmentación por ambiente**: cada vez que el dueño toca un chip, se cierra el `MediaRecorder` anterior y se abre uno nuevo. Metadata por segmento: `{room_type, room_index, floor, started_at, duration, gyro_yaw_range, quality:{blur, light, speed}, device:{ua, w, h, fps}}`.
- **QA en vivo**: cada 400 ms se dibuja el frame en un canvas de 160 px, se calcula brillo medio, varianza del laplaciano y diferencia con el frame anterior. Umbrales calibrados con el piloto.
- **Escala en el cliente (mejora progresiva)**: si `navigator.xr` soporta `immersive-ar` con `hit-test` (Chrome Android sobre ARCore), en la calibración se hace un hit-test al piso y se registra la altura de la cámara y la trayectoria; se manda como `scale_hint{source:'arcore', camera_height_m, poses[]}`. En iOS no hay WebXR: se salta.
- **Subida resiliente**: cada chunk va a `POST /capture/segments/<seg>/chunks/<n>/` con `Content-Range`; se guarda en IndexedDB hasta recibir 201; cola con reintentos exponenciales; `POST /capture/complete/` al final. Reutiliza `owner_upload_guards` (sniff, caps, rate limit) con caps nuevos (p. ej. 40 segmentos / 400 MB).
- **Eventos**: `POST /capture/events/` con la lista del 03.

## 2. Modelo de datos (aditivo, sin tocar tablas existentes)

```
PropertyCapture          # una sesión de captura por link
  id, property, link, tier ('owner_web'|'advisor_roomplan'|'import'), status,
  device_json, scale_hints_json, started_at, completed_at

CaptureSegment           # un ambiente filmado
  id, capture, room_type, room_index, floor, order,
  video_url (ensamblado), duration_s, quality_json, external_id (idempotencia)

PropertyFloorplan        # el modelo paramétrico versionado (la fuente de verdad del plano)
  id, property, analysis (FK a PropertyAIAnalysis, 1:1 por versión), version,
  source_tier, scale_confidence (0-1), scale_sources_json,
  model_json = {
    units: 'm', floors: [{ level, ceiling_h,
      rooms: [{ id, type, label, polygon: [[x,y],...], area_m2, perimeter_m, condition, photo_ids, segment_id }],
      walls: [{ id, a:[x,y], b:[x,y], thickness, rooms:[id,id] }],
      openings: [{ id, kind:'door'|'window'|'opening', wall_id, offset, width, height, connects:[id,id] }]
    }],
    north_deg?, total_area_m2, covered_m2, uncovered_m2
  },
  svg_2d (text), glb_3d_url, png_url, pdf_url,
  approved_by, approved_at
```

`PropertyAIAnalysis` conserva el ciclo de estados y `rooms` (para compatibilidad con la UI actual); `PropertyFloorplan` es la geometría. Migraciones sólo aditivas, como manda el CLAUDE.md del backoffice.

## 3. Pipeline geométrico (worker GPU)

| Etapa | Qué hace | Con qué | Tiempo/GPU (estimado) |
|---|---|---|---|
| 1. Muestreo | 3-4 fps por segmento, descarta blur (ya existe `video_frames._is_blurry`), deduplica | ffmpeg + Pillow/numpy | CPU, segundos |
| 2. Reconstrucción métrica | Poses, intrínsecos, depth y nube de puntos de toda la secuencia (los segmentos van en orden, el "caminá al siguiente ambiente" los conecta) | **Depth Anything 3** Nested / Streaming (Apache-2.0 en Base/Small + Metric) o **MapAnything-apache**; correr los dos en el spike y quedarse con el mejor | 1-3 min en una GPU de 24 GB (A10G/L4) |
| 3. Piso y segmentación | Plano de piso por RANSAC; eje vertical; cada punto se asigna al ambiente por el `segment_id` del frame que lo generó (los frames del tramo "caminando" se asignan al más cercano) | numpy / open3d | segundos |
| 4. Polígonos, muros, aberturas | Por ambiente: proyección 2D → mapa de ocupación → contorno → simplificación Manhattan (ángulos 90°, tolerancia 5°) → polígono; muros como bordes compartidos entre polígonos vecinos; aberturas donde la trayectoria de cámara cruza un muro (una puerta es, por definición, donde pasó el dueño) | propio; opcionalmente **RoomFormer/PolyRoom** como segunda opinión | segundos |
| 5. Fusión de escala | Combina: `arcore` (si hay), escala del modelo métrico, puerta de calibración (prior 2,02 ± 0,05 m), medida del dueño, superficie declarada / mensura. Mínimos cuadrados ponderados por confianza; devuelve factor y `scale_confidence` | propio | ms |
| 6. Semántica (LLM) | Sobre 2-3 frames por ambiente: confirma tipo, detecta ventanas y su muro, estado de conservación con evidencia (reusar `_stage2_group` y `_stage1_classify`), lee el plano de mensura si lo hay (cotas, rótulos) y lo alinea con el polígono; chequea coherencia (ambientes declarados vs. detectados) | Gemini 2.5 Flash (ya integrado) o Claude Sonnet 5 (USD 2/10 por millón) | USD 0,02-0,10 |
| 7. Render | SVG 2D con cotas (largo × ancho por ambiente, superficie, total), norte, leyenda de confianza y marca (reusar estilo de `floorplan_render.py`); GLB 3D por extrusión (trimesh) con un material por ambiente y etiquetas; PNG y PDF | propio | segundos |

Salida del worker: `model_json` + archivos. El backoffice crea `PropertyFloorplan` y pasa `PropertyAIAnalysis` a `draft`.

## 4. Editor paramétrico (backoffice, gate humano)

En la sección "Análisis del dueño" de la ficha: un canvas SVG editable (arrastrar vértices, unir/dividir ambientes, renombrar, mover aberturas, ingresar una cota exacta que re-escala el ambiente o el plano, marcar "cerrado contra mensura"). Guarda una versión nueva de `PropertyFloorplan`. Botón **Aprobar** → `approved` → **Enviar al dueño** (ya existe). Nadie ve un plano que un humano no aprobó.

Alcance mínimo para el piloto: renombrar, borrar/unir ambiente, arrastrar vértices, cota exacta. Lo demás después.

## 5. Nivel B: RoomPlan

- App iOS interna mínima (SwiftUI + `RoomCaptureView`), 2-3 semanas: login con el backoffice, elegir propiedad, escanear ambiente por ambiente (`RoomCaptureSession` con `structure` multi-room de iOS 17+), exportar `CapturedStructure` a JSON (muros, puertas, ventanas, objetos, en metros) y USDZ, `POST /api/properties/<id>/floorplan/import/` con `tier='advisor_roomplan'`.
- Conversión RoomPlan → `model_json`: directa (los muros ya vienen como segmentos con espesor; los ambientes como polígonos).
- Hasta que exista la app: importar exportaciones de Polycam/magicplan por el mismo endpoint.

## 6. Infra y costos

- **GPU on-demand**: Modal o Replicate (ya usan Replicate para `PropertyImageEdit`). Contenedor con DA3 + MapAnything + open3d; frío 30-60 s, caliente 1-3 min por propiedad. A10G/L4 ≈ USD 1-1,5/hora → **USD 0,05-0,10 por propiedad** en GPU. Alternativa RunPod spot para bajar más.
- **LLM**: USD 0,02-0,10 por propiedad.
- **Storage**: video 20-40 MB por propiedad; activar la Fase 2 de `media_pipeline_v2.md` (S3 + CloudFront) antes del piloto, la t3.micro no aguanta video en disco.
- **Cola**: el cron de 10 min (`process_owner_analyses`) pasa a disparar el job remoto y a hacer polling; o webhook de vuelta. Objetivo: plano en `draft` en < 15 min desde `capture_done`.
- **Total por propiedad**: < USD 0,50 todo incluido. Contra USD 15-30 de CubiCasa.

## 7. Licencias (decisión explícita)

| Componente | Licencia | Uso comercial |
|---|---|---|
| Depth Anything 3 Base/Small + Metric/Mono | Apache-2.0 | Sí |
| Depth Anything 3 Large/Giant | CC-BY-NC-4.0 | **No** (sólo spike interno) |
| MapAnything `map-anything-apache` | Apache-2.0 | Sí |
| MapAnything (default) | CC-BY-NC-4.0 | No |
| SpatialLM 1.1 (encoder Sonata) | CC-BY-NC-4.0 | No sin licencia de Manycore; sólo benchmark |
| MASt3R-SLAM | CC-BY-NC | No |
| RoomFormer / PolyRoom | revisar cada repo antes de integrar | — |
| RoomPlan | Apple developer | Sí |

El spike se corre con todo (incluidas las variantes NC) para saber cuál es el techo de calidad; producción sólo con Apache-2.0 o licencia negociada.

## 8. Privacidad y seguridad

- El video de una casa es dato sensible: retención 90 días para los segmentos crudos (borrado automático), se conservan frames seleccionados, el modelo paramétrico y los renders. Aviso en la landing.
- Todo lo que ya hace `owner_upload_guards` (sniff, caps, rate limit, host guard) se mantiene; los chunks se validan al ensamblar.
- El worker GPU recibe URLs firmadas de corta duración, nunca credenciales del bucket.

## 9. Qué se reutiliza del código actual

`upload_links.py` (token), `views_owner_public.py` (cáscara + guards + validate/valuation), `owner_upload_og.py` (OG), `video_frames.py` (muestreo + blur), `_stage0_gate`, `_stage1_classify`, `_stage2_group` (semántica), `floorplan_render.py` (estilo de render, watermark), `process_owner_analyses.py` (cola y notificaciones), `OwnerUploadDialog.tsx`, `OwnerAIAnalysisSection.tsx` (se le agrega el editor). Se retiran `_stage3_measure`, `_stage4_layout`, `_validate_layout`, `_fallback_grid_layout` y `_scale_areas_to_total`.
