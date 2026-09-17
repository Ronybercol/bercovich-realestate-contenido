# 02 · Estado del arte (septiembre 2026)

Pregunta que guió la investigación: **¿cómo se obtiene hoy un plano 2D con medidas y un 3D por ambientes a partir de lo que una persona sin conocimientos puede capturar con su celular?**

## 1. Cómo lo resuelven los productos que ya lo hacen

| Producto | Captura | Cómo mide | Fidelidad | Requisito | Modelo comercial |
|---|---|---|---|---|---|
| **Apple RoomPlan** (framework de iOS) | Recorrido de 1-2 min por ambiente | LiDAR + ARKit: geometría paramétrica en vivo (muros, puertas, ventanas, muebles) | ±1-2 % | iPhone 12 Pro o superior (LiDAR), app nativa | Gratis para desarrolladores; exporta USDZ y JSON paramétrico |
| **CubiCasa** | Video walkthrough de 1-5 min dentro de su app (iOS y Android, sin LiDAR) | ARCore/ARKit para trayectoria + procesamiento en nube + control humano | ±2-3 %; es el estándar de tasadores en EE. UU. (ANSI Z765) | Instalar la app CubiCasa (o SDK embebido en app propia, o "GoToScan" por link) | USD 23-30 por plano en retail; SDK/API enterprise con precio a negociar; entrega en horas |
| **Zillow 3D Home** | Panorámicas 360° por ambiente con el celular, enlazadas caminando | Layout desde cada panorámica + altura de cámara como escala + unión por puertas | Aproximada ("AI-predicted room dimensions") | App Zillow, sólo mercado EE. UU. | Gratis para captar listings |
| **Matterport** | App (iPhone LiDAR o cámara Pro3) | Fotogrametría + LiDAR | Alta | App + suscripción | Plan Pro ~USD 69/mes + USD 15-50 por plano |
| **Polycam / magicplan / Canvas** | App, LiDAR o AR | RoomPlan/ARKit/ARCore | Alta con LiDAR, media sin | App | USD 8-20/mes |
| **SkyeBrowse / Manifold Orbit** | Video común subido a la web | Videogrametría en nube; el usuario fija la escala con una medida conocida | Media | Ninguno (web) | Suscripción |

Tres patrones se repiten en todos los que funcionan:

1. **La captura está guiada y es continua** (video o secuencia de panorámicas), nunca fotos sueltas.
2. **La escala viene de un sensor o de un dato conocido** (LiDAR, trayectoria AR, altura de cámara, una medida ingresada). Nadie estima metros "a ojo" con IA.
3. **Hay un modelo paramétrico intermedio** (muros, aberturas, polígonos por ambiente) y el dibujo final se genera por código a partir de él.

## 2. Qué se puede hacer sin LiDAR y sin app (el caso del dueño en Argentina)

Argentina: ~89 % Android / 11 % iOS (StatCounter, 2026). Los iPhone con LiDAR (Pro) son una minoría del 11 %. Y WebXR `immersive-ar` **no existe en Safari de iPhone** (sólo Vision Pro, y sólo VR): en iOS no hay AR desde el navegador salvo con App Clip (Variant Launch, con costo por proyecto). En Android sí: Chrome expone WebXR con hit-test, planos y depth-sensing sobre ARCore en la mayoría de los equipos de gama media para arriba.

Conclusión práctica: **la captura del dueño tiene que ser video común desde el navegador**, con AR como mejora progresiva en Android, y la geometría se recupera en el servidor.

### 2.1 Reconstrucción 3D métrica a partir de video: el salto de 2025-2026

Hasta 2024 reconstruir una casa desde video exigía SLAM clásico (COLMAP, ORB-SLAM) frágil ante paredes lisas y sin escala. Desde 2025 hay modelos **feed-forward** que en un solo paso devuelven poses de cámara, intrínsecos, mapas de profundidad y nube de puntos, y algunos **en escala métrica** sin sensores:

| Modelo | Quién | Qué da | Escala métrica | Licencia | Notas |
|---|---|---|---|---|---|
| **Depth Anything 3** (nov-dic 2025) | ByteDance Seed | Poses + profundidad consistente desde N vistas sin poses conocidas; variante "Nested" combina any-view con estimador métrico; DA3-Streaming para video largo con < 12 GB VRAM | Sí (Nested / Metric) | Base/Small y Metric/Mono: **Apache-2.0**; Large/Giant: CC-BY-NC | Candidato principal para el MVP |
| **MapAnything** (sep 2025, 3DV 2026) | Meta FAIR | Reconstrucción métrica universal; acepta imágenes solas o con intrínsecos/poses/depth; hasta 2.000 vistas | Sí, desde imágenes solas | Hay variante **Apache-2.0** (`map-anything-apache`) y una CC-BY-NC | Alternativa / segunda opinión de escala |
| **VGGT** (CVPR 2025) y sucesores (VGGT-Ω, Pi3) | Meta / comunidad | Poses + depth + point maps en un paso | Relativa (hasta escala) | Variada | Base de MapAnything |
| **MASt3R-SLAM** | Naver Labs | SLAM denso a partir de video | Relativa | CC-BY-NC | Es lo que usa SpatialLM para video; licencia no comercial |

### 2.2 De la nube de puntos al plano estructurado

| Método | Entrada | Salida | Licencia | Notas |
|---|---|---|---|---|
| **SpatialLM 1.1** (NeurIPS 2025; Manycore) | Nube de puntos (de video vía MASt3R-SLAM, RGBD o LiDAR) | Muros, puertas, ventanas y cajas de objetos; F1 94 % en Structured3D | Modelo Llama-1B (licencia Llama 3.2) o Qwen-0.5B (Apache) **pero el encoder Sonata/SceneScript es CC-BY-NC-4.0** | Excelente para benchmark interno; para producción hay que negociar licencia con Manycore o reemplazar el encoder |
| **RoomFormer** (CVPR 2023) / **PolyRoom** (ECCV 2024) | Mapa de densidad 2D de la nube proyectada | Polígonos por ambiente | Abiertos (revisar cada repo) | Especialistas, livianos, entrenables con Structured3D / CubiCasa5K |
| **Heurístico geométrico propio** | Nube de puntos **ya segmentada por ambiente** (porque el dueño tocó "estoy en la cocina") | Plano de piso (RANSAC) → proyección 2D por ambiente → polígono Manhattan-fit → muros compartidos por adyacencia | Propio | El más simple y robusto para el MVP; la segmentación por ambiente elimina el 70 % del problema |

La observación clave para nuestro caso: **si el dueño marca en qué ambiente está antes de filmarlo, el problema de "qué puntos pertenecen a qué ambiente" desaparece**. Lo que queda (ajustar un polígono a una nube de puntos de un ambiente) es geometría básica y no necesita un modelo de layout entrenado.

### 2.3 Escala: el problema que hay que resolver tres veces

Un video común no tiene escala absoluta. Fuentes disponibles, de más a menos confiable:

1. **Sensor AR** (ARCore en Android vía WebXR; ARKit sólo en app nativa): trayectoria métrica real. Cubre quizás la mitad de los dueños.
2. **Modelo métrico monocular** (DA3-Metric, MapAnything): ±5-10 % en interiores según los papers, mejor cuando hay piso y paredes visibles.
3. **Priors argentinos**: puerta interior 2,00-2,05 m de alto y 0,70-0,80 de ancho (Código de Edificación CABA exige 2,00 m mínimo); altura de techo 2,60 m en edificios post-1970, 3,00-3,40 en "antiguos"; cerámicos de 30, 45 o 60 cm. Un paso de calibración de 10 segundos ("filmá la puerta de entrada completa") convierte el prior en una medida.
4. **Una medida del dueño**: "medí el ancho de la puerta de entrada con un metro" (todo el mundo tiene un metro o una app de regla) o una hoja A4 apoyada en el piso (21,0 × 29,7 cm).
5. **Cierre contra documento**: superficie de la escritura, reglamento de copropiedad o **plano de mensura** (en CABA toda propiedad lo tiene; muchos dueños lo tienen en la escritura). Una foto del plano de mensura, vectorizada, es la fuente más exacta de todas y comparte pipeline con la vectorización de planos de Emprendimientos que ya está en el roadmap (tarea "Piloto planta 2D equipada", 16/09).

La v2 fusiona todas las disponibles y reporta una **confianza de escala** (±%). Ese número decide si el plano sale como "croquis con medidas aproximadas" (Nivel A) o como plano publicable (Nivel B o Nivel A cerrado contra mensura).

### 2.4 Qué puede y qué no puede hacer un modelo de lenguaje multimodal (Gemini / Claude / GPT)

- **Sirve para**: filtrar material inservible (ya lo hace), clasificar ambiente por foto, describir estado de conservación con evidencia (ya lo hace y está bien), detectar puertas/ventanas en frames, leer un plano de mensura (OCR de cotas y rótulos), validar coherencia ("el dueño dijo 3 ambientes y salieron 5"), redactar la descripción.
- **No sirve para**: medir. Benchmarks 2026 (Blueprint-Bench, AECV-bench, pruebas de RoomSketcher) muestran ~50 % de acierto en conteos simples sobre planos y errores grandes en medidas. Google mismo posiciona la comprensión espacial de Gemini 3 para razonamiento, no para metrología.

### 2.5 El 3D "con ambientes especificados"

No hace falta fotogrametría ni Gaussian splatting para el 3D que pediste. Con el modelo paramétrico (polígono por ambiente + altura de techo + aberturas) el 3D se **extruye por código** (three.js en la web, trimesh en Python para exportar GLB/USDZ) y cada ambiente queda etiquetado y coloreado. Es la "casa de muñecas" que muestran RoomPlan y CubiCasa. Un splat fotorrealista (Polycam, Luma, KIRI: USD 8-30/mes) es un extra de marketing para más adelante, no parte del núcleo.

## 3. Comprar vs. construir

| Opción | Pros | Contras | Costo estimado |
|---|---|---|---|
| **A. CubiCasa (GoToScan o SDK)** | Fidelidad probada, entrega en horas, cero I+D geométrico, Android y iOS | El dueño tiene que instalar la app CubiCasa (fricción alta en captación fría); sin control sobre UX ni datos; precio por plano en USD; sin mercado ni soporte local; no da la nube de puntos propia | ~USD 15-30 por plano (retail); enterprise a negociar |
| **B. RoomPlan (app propia iOS)** | Máxima fidelidad, gratis, 2-3 semanas de dev | Sólo iPhone Pro: sirve para el asesor, no para el dueño | 1-2 iPhone Pro (~USD 1.500 c/u en AR) + dev |
| **C. Pipeline propio con modelos abiertos** | Funciona desde el navegador en cualquier teléfono; datos y UX propios; costo marginal bajo; se integra con el backoffice y con el vectorizador de planos de Emprendimientos | I+D: 6-8 semanas hasta piloto; fidelidad ±5-10 % hasta cerrar contra mensura; necesita GPU on-demand | GPU ~USD 0,05-0,30 por propiedad + dev |
| **D. SpatialLM comercial** | Estado del arte en layout desde nube de puntos | Licencia CC-BY-NC en encoders; hay que negociar con Manycore; sigue necesitando la reconstrucción previa | A negociar |

**Recomendación:** C como núcleo (Nivel A, dueño) + B como complemento (Nivel B, asesor). Mantener A (CubiCasa) como plan de contingencia si el spike técnico de la semana 1 no alcanza el umbral de error, y D como benchmark de calidad para el pipeline propio.

## Fuentes

- SpatialLM: [GitHub](https://github.com/manycore-research/SpatialLM) · [paper NeurIPS 2025](https://arxiv.org/abs/2506.07491) · [SpatialLM 1.5 / SpatialGen, Manycore](https://www.prnewswire.com/news-releases/manycore-tech-unveils-next-gen-spatial-ai-models-spatiallm-1-5-and-spatialgen-accelerating-open-source-ecosystem-for-3d-scene-understanding-and-generation-302540091.html)
- Depth Anything 3: [GitHub ByteDance-Seed](https://github.com/ByteDance-Seed/Depth-Anything-3)
- MapAnything: [GitHub facebookresearch](https://github.com/facebookresearch/map-anything) · [paper](https://www.alphaxiv.org/abs/2509.13414)
- VGGT: [explicación LearnOpenCV](https://learnopencv.com/vggt-visual-geometry-grounded-transformer-3d-reconstruction/)
- RoomFormer: [GitHub](https://github.com/ywyue/RoomFormer) · PolyRoom: [GitHub](https://github.com/3dv-casia/PolyRoom/)
- Apple RoomPlan: [developer.apple.com](https://developer.apple.com/augmented-reality/roomplan) · [Apple ML Research](https://machinelearning.apple.com/research/roomplan) · [guía 2026 con y sin LiDAR](https://www.scanmanifold.com/blog-posts/floor-plan-app-iphone-guide)
- CubiCasa: [developers](https://www.cubi.casa/developers/) · [Conversion API](https://www.cubi.casa/developers/conversion-api-documentation/) · [GoToScan](https://www.cubi.casa/developers/gotoscan-implementation-tutorial/) · [SDK Android](https://github.com/CubiCasa/cubicasa-android-sdk-example-project) · [precios 2026 (G2)](https://www.g2.com/products/cubicasa/pricing)
- Zillow 3D Home: [cómo funciona](https://www.zillow.com/3d-home/) · [algoritmos de backend](https://www.zillow.com/news/behind-zillow-3d-home-backend-algorithms/)
- Matterport: [planes](https://matterport.com/plans) · [precios 2026](https://www.thefuture3d.com/blog/matterport-pricing-guide-2026/)
- WebXR en iOS: [Variant Launch](https://launch.variant3d.com/) · [estado WebXR 2026](https://www.testmuai.com/learning-hub/webxr-compatible-browsers/) · [Apple Developer Forums: immersive-ar no soportado](https://developer.apple.com/forums/thread/743655)
- ARCore / WebXR en Chrome Android: [WebXR vs ARCore](https://developers.google.com/ar/develop/webxr/arcore-comparison) · [Depth API](https://developers.google.com/ar/develop/depth)
- Gemini y planos: [Gemini 3 Pro vision](https://blog.google/innovation-and-ai/technology/developers-tools/gemini-3-pro-vision/) · [Blueprint-Bench](https://arxiv.org/pdf/2509.25229) · [AECV-bench](https://www.aecfoundry.com/blog/can-ai-really-read-your-building-plans-aecv-bench-gets-a-major-upgrade) · [prueba RoomSketcher](https://www.roomsketcher.com/blog/can-gemini-create-floor-plans/)
- Mercado móvil Argentina: [StatCounter](https://gs.statcounter.com/os-market-share/mobile/argentina)
- WhatsApp Cloud API límites de media (video 16 MB): [Meta for Developers](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/media/)
- Gaussian splatting móvil: [comparativa 2026](https://www.polyvia3d.com/guides/gaussian-splatting-tools-comparison)
