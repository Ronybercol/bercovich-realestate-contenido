# Captura del propietario v2 — del video al plano 2D con medidas y 3D por ambientes

**Fecha:** 2026-09-17 · **Autor:** investigación nocturna con Claude Code, a pedido de Rony · **Estado:** propuesta para decidir

## Resumen ejecutivo

El producto actual ("Pedir fotos al dueño" → link `/p/<token>` → pipeline Gemini → croquis SVG) no alcanzó el objetivo por una razón de fondo, no de detalle: **nunca mide nada**. Le pide al dueño fotos sueltas sin guía, le pide a un modelo de lenguaje que "adivine" metros por anclas y que "adivine" una distribución en cajas 0-100, y dibuja rectángulos que el propio SVG rotula *"CROQUIS APROXIMADO — NO A ESCALA"*. Con ese insumo es imposible llegar a un plano 2D con medidas y menos a un 3D por ambientes. El detalle está en [01-diagnostico.md](01-diagnostico.md).

La propuesta cambia tres cosas a la vez:

1. **La captura pasa de "subí fotos" a "caminá tu casa filmando, con el celular guiándote"** (2 a 5 minutos, cualquier teléfono con navegador, sin instalar app). El dueño toca en qué ambiente está antes de filmarlo, y eso segmenta el material por ambiente sin que la IA tenga que adivinarlo. Ver [03-experiencia-captura.md](03-experiencia-captura.md) y el [prototipo](prototipo/captura.html).
2. **La geometría se calcula, no se estima**: reconstrucción 3D métrica del video con modelos feed-forward abiertos (Depth Anything 3 / MapAnything), extracción de polígonos por ambiente, escala real por tres fuentes en cascada (sensor AR si hay, modelo métrico + altura de puerta, y una medida del dueño o el plano de la escritura). El plano 2D con cotas y el 3D se dibujan **por código** a partir de un modelo paramétrico, nunca con imagen generativa. Es la misma regla que fijaste el 16/09 para la planta 2D de Emprendimientos. Ver [04-arquitectura.md](04-arquitectura.md).
3. **Dos niveles de fidelidad, un solo producto**: Nivel A *self-serve* del dueño (±5-10 % en superficies, sirve para captación y tasación preliminar) y Nivel B *visita pro* del asesor con iPhone Pro + RoomPlan (±1-2 %, sirve para publicar). Misma ficha, mismo editor, mismo output. Esto absorbe la tarea "Productizar visita usado (Foto Self Serve)" del CEO Board.

Con un piloto de 20 propiedades reales en 6-8 semanas se puede validar si el error de superficie queda por debajo del 8 % mediano y si más del 60 % de los dueños completan la captura. Ver [05-roadmap.md](05-roadmap.md).

## Qué hay en esta carpeta

| Archivo | Qué es |
|---|---|
| [01-diagnostico.md](01-diagnostico.md) | Por qué el producto actual (WP1-WP7, julio 2026) no llegó, con evidencia del código |
| [02-estado-del-arte.md](02-estado-del-arte.md) | Investigación: qué hacen Zillow, CubiCasa, Matterport, Apple RoomPlan; modelos abiertos 2025-2026; comprar vs. construir; fuentes |
| [03-experiencia-captura.md](03-experiencia-captura.md) | La experiencia del dueño paso a paso, copy, controles de calidad en vivo, casos borde, y la revisión del plano |
| [04-arquitectura.md](04-arquitectura.md) | Pipeline técnico, modelo de datos, escala métrica, infra GPU, costos por propiedad, licencias |
| [05-roadmap.md](05-roadmap.md) | Fases, work packages, quién hace qué, métricas de éxito, riesgos, decisiones que hay que tomar |
| [prototipo/captura.html](prototipo/captura.html) | Prototipo funcional de la captura guiada (abrir en el celular; usa la cámara real, no sube nada) |

## Decisiones que necesitan tu OK

1. **Nivel A sobre web (PWA), no app nativa.** Argentina es ~89 % Android / 11 % iOS (StatCounter 2026); iPhone con LiDAR es una fracción del 11 %. La captura del dueño tiene que funcionar en un Samsung A-series con Chrome desde un link de WhatsApp.
2. **Construir el pipeline geométrico con modelos abiertos** (licencias Apache-2.0 donde toque) y dejar SpatialLM y CubiCasa como alternativas evaluadas, no como base. Detalle de licencias en 04.
3. **Comprar 1-2 iPhone Pro para los captadores** y construir una app mínima de RoomPlan (o usar Polycam/magicplan y exportar) para el Nivel B.
4. **Infra GPU on-demand** (Modal / Replicate / RunPod) en vez de correr el pipeline en la EC2 t3.micro. Costo objetivo: < USD 0,50 por propiedad.
5. **Gate humano obligatorio** (Ariel o el asesor) antes de que un plano llegue al dueño, con editor paramétrico en el backoffice. Un plano mal medido publicado es exposición comercial y legal.
