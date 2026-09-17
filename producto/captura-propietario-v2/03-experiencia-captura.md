# 03 · La experiencia del dueño

Principio rector: **el dueño hace una sola cosa, caminar su casa filmando, y el teléfono le dice todo lo demás.** No elige fotos, no sabe qué es un plano, no instala nada, no crea cuenta. Cinco minutos como máximo. Si abandona a la mitad, lo que filmó ya está en el servidor.

Hay un prototipo funcional en [`prototipo/captura.html`](prototipo/captura.html): abrilo en el celular para sentir el flujo (usa la cámara real, no sube nada).

## Flujo Nivel A (dueño, desde un link de WhatsApp)

### 0 · El mensaje de WhatsApp (ya existe, cambia el copy)

> Hola {nombre}! Te paso el link para armar el plano de {dirección}. Vos caminás la casa filmando con el celular, nosotros medimos. Son 3 minutos y en menos de una hora te mando el plano con las medidas: {url}

La placa OG (ya existe) pasa a mostrar un plano de ejemplo con cotas, no un texto. La promesa tiene que ser verdadera: si el pipeline tarda 20 minutos, decir "en menos de una hora".

### 1 · Landing (5 segundos)

- Título: **Vamos a armar el plano de tu casa.**
- Subtítulo: *Caminás filmando, nosotros medimos. 3 minutos. No hace falta instalar nada.*
- Tres pasos ilustrados: (1) Prendé todas las luces, (2) Parate en la puerta de entrada, (3) Recorré ambiente por ambiente.
- Botón único: **Empezar**.
- Abajo, chico: "¿Preferís mandar fotos? Tocá acá" → cae al flujo v1 (fotos sueltas) como degradación para quien no puede o no quiere filmar.

### 2 · Pre-check (10 segundos, automático)

- Pide permiso de cámara (y en Android, de sensores de movimiento / AR si está disponible; en iOS pide permiso de orientación para el medidor de giro).
- Verifica: teléfono en vertical, luz suficiente (mide brillo promedio del preview), batería no crítica, conexión (muestra "estás en datos móviles, la subida va a usar ~30 MB").
- Mensajes cortos y accionables: "Está oscuro: prendé la luz o abrí las cortinas" / "Girá el teléfono a vertical".

### 3 · Calibración (15 segundos)

> **Parate frente a la puerta de entrada, a un paso de distancia, y filmá la puerta completa de arriba a abajo.**

Esto da un objeto de tamaño conocido (2,00-2,05 m de alto en Argentina) visto de frente en la primera toma, y si hay AR, el primer anclaje al piso. Pantalla con silueta de puerta como guía. Si el dueño tiene un metro a mano, un campo opcional: "¿Cuánto mide de ancho la puerta? (opcional, mejora la precisión)".

### 4 · Recorrido guiado, ambiente por ambiente (2 a 4 minutos)

Ciclo por ambiente:

1. **"¿En qué ambiente estás?"** Chips grandes: Living · Comedor · Cocina · Dormitorio · Baño · Toilette · Balcón · Terraza · Patio · Lavadero · Pasillo · Cochera · Otro. Un toque. (Los dormitorios y baños se numeran solos: "Dormitorio 2".)
2. **Grabación con guía en pantalla**, en dos movimientos:
   - *"Desde el centro del ambiente, girá despacio una vuelta completa apuntando a la mitad de la pared."* Un anillo de progreso se va completando con el giro (usa el giroscopio; si no hay, un temporizador de 20 s).
   - *"Ahora caminá hacia el siguiente ambiente sin dejar de filmar."* Esto captura la puerta y la conexión entre ambientes, que es lo que arma el plano.
3. **Control de calidad en vivo** (en el teléfono, sin servidor):
   - Velocidad: si el frame cambia demasiado entre muestras → "Más despacio".
   - Nitidez: varianza del laplaciano sobre el preview → "Está borroso, frená un segundo".
   - Luz: brillo promedio → "Muy oscuro".
   - Encuadre: si el horizonte está muy arriba (mucho techo) → "Apuntá un poco más abajo".
   - Duración mínima por ambiente (12 s) antes de habilitar "Siguiente".
4. **"Listo este ambiente"** → resumen de 1 segundo ("Cocina · 24 s · bien") y vuelve al paso 1 para el siguiente. Lista lateral con los ambientes ya hechos.
5. Botón **"Terminé el recorrido"** siempre visible desde el segundo ambiente.

Mientras el dueño filma el ambiente N, el segmento N-1 ya se está subiendo en segundo plano.

### 5 · Extras opcionales (30 segundos, se pueden saltar)

- **Fachada**: "Salí y sacale una foto al frente del edificio" (para la ficha).
- **Plano de la escritura**: "¿Tenés el plano en la escritura o el reglamento? Sacale una foto y el plano sale exacto." (Esto convierte el Nivel A en publicable.)
- **Datos**: superficie que figura en la escritura, expensas, piso y orientación. Todo opcional.

### 6 · Cierre inmediato

> **¡Listo! Recibimos 6 ambientes:** Living-comedor, Cocina, Dormitorio 1, Dormitorio 2, Baño, Balcón.
> Te mandamos el plano por WhatsApp en menos de una hora. Si querés agregar un ambiente, este link sigue abierto 7 días.

Y una cuenta regresiva real: el backend sabe cuánto tarda el pipeline y el dueño ve un estado ("procesando" / "en revisión del asesor" / "listo").

### 7 · Revisión del plano (mismo link, modo `review`)

- **Plano 2D** con nombres, superficies por ambiente, superficie total, cotas principales, norte si se pudo inferir. Rótulo de confianza honesto: *"Medidas aproximadas ±6 %. Para el plano exacto, tu asesor lo cierra con el plano de mensura."*
- **3D girable** (dollhouse) con cada ambiente coloreado y etiquetado; toggle 2D/3D; se puede compartir como imagen y como link.
- **Correcciones estructuradas** (no un textarea): tocar un ambiente → "Cambiar nombre" / "Este ambiente no existe" / "Esta medida está mal: son __ m" / "Falta un ambiente: filmarlo ahora". Cada corrección crea una versión nueva sin volver a correr todo desde cero.
- **Validar**: "Sí, es mi casa" → estado `validated`, notifica al asesor.
- **CTA comercial** (ya existe): "¿Querés saber cuánto vale? Pedí tasación." Se muestra después de validar, no antes.

## Flujo Nivel B (asesor con iPhone Pro + RoomPlan)

En la visita de tasación, el asesor abre la app interna, escanea ambiente por ambiente (RoomPlan pide lo mismo: recorrido lento), y al terminar la app sube el JSON paramétrico y el USDZ al mismo endpoint que el pipeline del dueño. Misma ficha, mismo editor, mismo output, pero con confianza ±1-2 % y sin gate de escala. Si el dueño ya hizo el Nivel A, el escaneo del asesor reemplaza la geometría y conserva las etiquetas, fotos y estado de conservación.

Alternativa sin desarrollo iOS: el asesor usa Polycam o magicplan y exporta el plano (JSON/DXF); un importador lo mapea al modelo paramétrico. Sirve para arrancar la semana 1.

## Casos borde y cómo se manejan

| Caso | Manejo |
|---|---|
| Sin permiso de cámara o navegador viejo | Cae al flujo v1 (fotos sueltas + video suelto) con un aviso; el pipeline hace lo que puede y marca confianza baja |
| iPhone: MediaRecorder graba MP4/H.264 y a veces corta a los 30-60 s | Segmentar por ambiente (< 60 s por segmento) y grabar en chunks de 10 s; probar en Safari 16+ |
| Se corta la conexión | Los chunks quedan en IndexedDB y se reintentan; el link sigue válido 7 días |
| Ambientes en dos plantas | Chip "Cambiar de planta" antes del ambiente; el plano sale por planta |
| Ambiente sin puerta clara (living-comedor integrado) | Chip "Living-comedor" y el pipeline lo trata como un polígono |
| Casa amueblada y desordenada | La reconstrucción usa piso y paredes visibles; el estado de conservación ya ignora desorden por prompt |
| Dueño de edad o poca destreza | Video tutorial de 30 segundos (Nakala) en la landing; la opción de que un familiar lo haga; y siempre el Nivel B en la visita |
| Espejos y ventanales grandes | Conocido en fotogrametría; se detectan por inconsistencia de profundidad y se enmascaran; el aviso "apuntá a la pared, no al espejo" ayuda |
| Video de WhatsApp (el dueño manda un video por el chat en vez de usar el link) | El CRM lo recibe (WABA, máx. 16 MB) y lo mete al mismo pipeline con confianza baja; el asesor puede pedir la captura guiada después |

## Copy: reglas

- Rioplatense, voseo, frases de una línea. Nada de "IA", "modelo", "pipeline".
- Cada instrucción dice qué hacer con el cuerpo: "parate", "girá", "caminá", "apuntá".
- Cada error dice cómo se arregla.
- Nunca prometer exactitud que no se tiene: "medidas aproximadas" hasta que haya mensura o Nivel B.

## Métricas de la experiencia (eventos)

`link_open` → `capture_start` → `permission_granted` → `calibration_done` → `room_start{tipo}` → `room_done{tipo, seg, calidad}` → `capture_done{ambientes, duración, MB}` → `upload_done` → `plan_ready{minutos}` → `plan_view` → `plan_correction{tipo}` → `plan_validated` → `valuation_requested`. Con `token` y `propertyId` como dimensiones. Se mandan al backend (no dependen de GA4).
