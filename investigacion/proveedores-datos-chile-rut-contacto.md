---
title: "Proveedores de datos en Chile: de RUT a datos de contacto"
tipo: "investigación interna"
estado: "no publicar — documento de trabajo"
fecha: "2026-09-16"
---

# Proveedores de datos en Chile: de RUT a datos de contacto

Investigación de mercado sobre servicios que, recibiendo un **RUT**, devuelvan
datos personales y de contacto (nombre, mail, celular, teléfono fijo) de una
persona natural en Chile. Incluye documentación pública, paquetes y precios,
el ecosistema open source alrededor del problema, y el marco legal aplicable.

---

## 0. Resumen ejecutivo — leé esto primero

Hay **tres hallazgos** que ordenan todo el resto del documento.

### Hallazgo 1: el mercado está partido en dos, y no se cruzan

| | Documentación pública y precios públicos | Entrega mail + celular de persona natural |
|---|---|---|
| **APIs formales tipo SII / identidad** (BaseAPI, SimpleAPI, API Gateway, Verifik, Floid, Didit) | ✅ Sí, excelente | ❌ No |
| **Bureaus y data hubs** (Equifax, TransUnion, Datamart, TuSI) | ⚠️ Parcial — docs sí, precios no | ✅ Sí, bajo contrato |
| **Rutificadores comerciales** (Findatos, Rutify, BasesChile, DataMarket) | ⚠️ Precios sí, docs técnicas pobres | ✅ Sí, pero origen del dato dudoso |

**No existe hoy un proveedor que cumpla simultáneamente las tres condiciones que
pediste** (documentación pública + precios públicos + RUT → mail y celular de
persona natural). Los que publican todo hacen datos de *empresa* desde el SII.
Los que entregan contacto de *persona* venden por contrato, con onboarding,
declaración de finalidad y precio negociado.

### Hallazgo 2: el padrón electoral no tiene ni mail ni celular

Este es el punto técnico más importante y el que explica todo el mercado gris.

La fuente que hace posible el "RUT → nombre, apellido, dirección, comuna" en
Chile es el **padrón electoral del Servel**, que históricamente se publicó como
PDF y contiene nombre, RUT, domicilio electoral, comuna y sexo. Eso es todo.
**El padrón no contiene correos ni números de teléfono.**

Por lo tanto, cualquier servicio que te prometa "entregame un RUT y te devuelvo
el celular y el mail" **no está usando una fuente pública**. Ese dato viene de
otro lado: cesiones comerciales (retail, telco, cajas de compensación,
aseguradoras), o de filtraciones. Cuando evalúes un proveedor, la pregunta
técnica que separa lo legítimo de lo que no lo es es una sola:

> ¿De qué fuente específica sale el celular, y con qué base de licitud fue
> obtenido y cedido?

Si la respuesta es vaga ("fuentes públicas", "datos cruzados de fuentes
chilenas"), la respuesta real casi siempre es filtración.

### Hallazgo 3: el reloj legal corre — quedan ~2,5 meses

La **Ley 21.719** entra en plena vigencia el **1 de diciembre de 2026**. Multas
de hasta **20.000 UTM** (~$1.400 millones CLP) y hasta **4% de ingresos anuales**
en reincidencia, aplicadas por la nueva Agencia de Protección de Datos
Personales. Lo relevante para este caso: la ley **endurece el estándar para
"fuentes de acceso público"** y exige un test de balanceo documentado para
ampararse en interés legítimo, con vara más exigente cuando el fin es comercial.

Si vas a montar un proceso de enriquecimiento de leads en Chile, conviene
diseñarlo directamente contra la Ley 21.719, no contra la 19.628 que está por
morir.

---

## 1. Categoría A — Bureaus y data hubs (los que sí tienen el dato de contacto)

Esta es la categoría a la que realmente hay que llamar si el objetivo es
contactabilidad de persona natural. Documentación existe; los precios son
por cotización y el onboarding incluye declaración de finalidad.

### A.1 — Equifax Chile (DICOM)

El actor histórico y el más grande. Opera el portal de desarrolladores LATAM.

- **Docs:** [developer.latam.equifax.com/documentation](https://developer.latam.equifax.com/documentation)
  y [catálogo de API products](https://developer.latam.equifax.com/products/apiproducts).
  Modelo clásico: te registrás, te aprueban el producto, te mandan credenciales
  de test por mail, después pasás a producción desde la pestaña *Live*.
- **Producto de referencia:** el [Informe DICOM Platinum 360](https://soluciones.equifax.cl/personas/productos/informe-dicom-platinum/)
  incluye datos personales, fecha de nacimiento, estado civil y **direcciones
  históricas** (dónde ha vivido la persona según registros de Equifax), además
  del perfil financiero.
- **Precio de referencia (retail, no API):** ~**$17.990 CLP** IVA incluido por
  informe unitario; plan de monitoreo 6 meses ~**$40.990 CLP**
  ([ficha de producto](https://sec.equifax.cl/compraonline/ficha-producto/1/informe-oficial-dicom-platinum-360)).
  El precio API es por contrato y volumen.
- **Limitación real:** varios productos son *Partner products* y requieren un
  código de acceso que te da tu ejecutivo de cuenta. El catálogo público no
  muestra endpoints Chile hasta que estás logueado.
- ⚠️ **Nota importante:** el Platinum 360 es un producto pensado para que *el
  titular* consulte sus propios datos o para evaluación crediticia con base de
  licitud. Usarlo como fuente de prospección inmobiliaria muy probablemente
  viole los [términos y condiciones](https://soluciones.equifax.cl/legal/terminos-y-condiciones/)
  y la finalidad declarada.

### A.2 — TransUnion Chile

- **Sitio de producto:** [Verificación de Identidad](https://chile.transunion.com/producto/verificacion-de-identidad)
  y [Soluciones de Identidad](https://chile.transunion.com/solucion/soluciones-de-identidad).
- **Qué entrega:** validación de vigencia de cédula, nombre, fecha de nacimiento,
  edad, nacionalidad, estado civil, fecha de matrimonio, datos del cónyuge,
  historial laboral e **información sociodemográfica**. Cruza contra las bases de
  **Sinacofi**.
- **Integración:** API en tiempo real, aplicación web o proceso batch —
  las tres modalidades están documentadas comercialmente.
- **Producto complementario:** [Informe Comercial Titanium](https://chile.transunion.com/producto/informe-comercial-titanium)
  y [Preguntas de Verificación](https://chile.transunion.com/producto/preguntas-de-verificacion) (KBA).
- **Precio:** no público. Contrato B2B.

### A.3 — Datamart SpA — el más interesante técnicamente

Es probablemente el mejor candidato de esta categoría para un equipo de producto.

- **Docs:** [docs.datamart.cl](https://docs.datamart.cl/) y
  [datamart.co/documentacion](https://datamart.co/documentacion/).
- **Arquitectura:** filosofía **API First**, API completa bajo estándar
  **GraphQL**, más webhooks para notificación asincrónica. SDK oficial mantenido
  (solo .NET Framework 4.5+, que es su punto débil si tu stack es JS/Python).
- **Qué hace:** [acceso a data privada](https://datamart.co/accede-a-data-privada/) —
  verificar antecedentes de prospectos en línea, fuentes de información de
  personas y empresas, datos prácticamente en tiempo real. Permite solicitar,
  procesar y almacenar documentos y antecedentes con información personal,
  comercial, tributaria o financiera.
- **Empresa:** Datamart SpA, RUT 76749144-1, Alonso de Monroy 2869 of. 201,
  Vitacura, Santiago.
- **Precio:** no público.

### A.4 — TuSI SpA (P-Box / BigData) — el más directo para tu caso de uso

Es el único de la categoría que describe **exactamente** lo que pediste.

- **Producto:** [Contactabilidad: correos y teléfonos certificados relacionados a un RUT](https://bigdata.tusi.cl/index.php/portfolio-item/contactabilidad-correos-y-telefonos-certificados-relacionados-a-un-rut-bigdatav3/).
  Entrega celulares, fijos y correos **directamente asociados al RUT y nombre**,
  con una clasificación que "certifica" la relación entre el RUT y el medio de
  contacto.
- **Producto adicional:** [contactabilidad en redes sociales asociada a un RUT](https://bigdata.tusi.cl/index.php/portfolio-item/contactabilidad-redes-sociales-relacionados-a-un-rut-bigdatav9/).
- **Integración:** ofrecen un API gateway para integrar la información en tus
  propias herramientas. Ver también [P-Box Suite](https://tusi.cl/pbox).
- **Empresa:** fundada 2016, Maipú, RM. ~11-50 empleados, industria marketing y
  publicidad. Contacto: contacto@tusi.cl / +562 2632 5310.
- **Precio:** no público. Es el primer llamado que haría.
- ⚠️ La palabra "certificados" en su marketing se refiere a su propio scoring de
  confianza del match RUT↔contacto, **no** a una certificación regulatoria ni a
  consentimiento del titular. Hay que pedir por escrito la cadena de origen
  del dato antes de firmar.

---

## 2. Categoría B — APIs formales con documentación y precios públicos

Documentación impecable y precios en la web, pero **solo datos de contribuyente
/ empresa desde el SII**. Sirven para validar y enriquecer el lado B2B, no para
sacar el celular de una persona.

### B.1 — BaseAPI.cl ⭐ la mejor documentada del grupo

- **Docs:** [baseapi.cl/docs](https://baseapi.cl/docs) — **53 endpoints REST** en
  JSON, OpenAPI completo, SDK TypeScript en npm, ejemplos en curl, auth por API Key.
- **Precios (públicos):** [baseapi.cl/precios](https://baseapi.cl/precios) —
  desde **$14.990/mes IVA incluido**. **Capa gratuita permanente**: 50 consultas
  + 20 emisiones al mes. Dos paquetes independientes (Consulta y Emisión).
  Planes enterprise para alto volumen.
- **Cobertura:** [servicios](https://baseapi.cl/servicios) — SII (RCV, boletas,
  DTEs, cesiones, F22/F29, carpeta tributaria), **Previred**, Superintendencia
  de Pensiones, Dirección del Trabajo, TGR, Boletín Concursal, tasación de
  vehículos.
- **Para tu caso:** validación de RUT, situación tributaria, actividades
  económicas. Útil para calificar un lead B2B o un inversor con giro, no para
  contacto de persona natural.
  Ver su [guía de validación de RUT](https://baseapi.cl/blog/validar-rut-chile-api-guia-desarrolladores).

### B.2 — SimpleAPI.cl

- **Docs:** colección **Postman** con ejemplos + serie de videos explicativos.
  [simpleapi.cl/Productos](https://www.simpleapi.cl/Productos).
- **Precios (públicos):** [simpleapi.cl/Precios](https://www.simpleapi.cl/Precios) —
  **gratis hasta 500 consultas/mes**; planes de pago hasta **150.000
  consultas/mes**; paquetes de respaldo desde **2 UF**.
- **Producto "Simple API RUT":** obtiene lo mismo que el SII para un
  contribuyente — razón social, direcciones, correo de intercambio — solo con
  el RUT.
- ⚠️ **Restricción explícita en sus propios términos:** no entrega datos de
  menores ni de personas sin información en el SII, **no almacena los datos
  entregados**, y su uso es **exclusivamente para facturación electrónica**.
  Usarlo para prospección viola su contrato. Ver [FAQ](https://www.simpleapi.cl/FAQ).

### B.3 — API Gateway (apigateway.cl)

- **Docs:** [apigateway.cl/docs/api](https://www.apigateway.cl/docs/api) (API v2, REST).
  Operando desde 2019.
- **Precios:** modelo **pay-as-you-use**, sin contratos anuales ni compromisos.
  Créditos de bienvenida al crear la cuenta. Los planes detallados se ven
  logueado. Precios publicados son **netos, sin IVA**.
- **Cobertura:** contribuyentes, documentos tributarios, registro de compras y
  ventas, boletas de honorarios, Portal MIPYME, eBoleta, **bienes raíces**,
  vehículos, e indicadores previsionales directo de **Previred**.
- **Para tu caso:** el endpoint de **bienes raíces** es genuinamente interesante
  para inmobiliaria — no por contacto, sino por propiedad. Ver
  [términos](https://www.apigateway.cl/legal).

### B.4 — Verifik

- **Docs:** [docs.verifik.co](https://docs.verifik.co/verifik-es/identidad/chile-taxpayer/) —
  documentación pública, multi-país LatAm.
- **Precios (públicos):** **Pay as you go**, promedio ~**USD 0,20 por
  resultado**. Planes de suscripción mensual/anual que bajan el costo por
  consulta. **USD 10 de crédito** al crear cuenta nueva.
- **Endpoint Chile:** consulta de contribuyente por RUT — nombre, categoría de
  negocio, subcategoría, actividades y servicios autorizados. Orientado a KYB,
  facturación y validación de proveedores.
- **Limitación:** datos nominales de registro tributario. Sin contacto personal.

### B.5 — Otros de verificación de identidad (mismo patrón)

| Proveedor | Docs | Qué hace en Chile |
|---|---|---|
| **Floid** | [readme.floid.io](https://readme.floid.io/docs/somos-floid) | [API Registro Civil](https://www.floid.io/servicios/api-registro-civil): cruza RUT contra Registro Civil, verificación en <6 s. Open banking. ISO 27001:2022. Precio por cotización. |
| **Didit** | [docs.didit.me](https://docs.didit.me/api-reference/database-validation/chile/rut) | Valida RUT contra Registro Civil. Pay-per-call. |
| **Metamap** | [docs.metamap.com](https://docs.metamap.com/reference/govchecks-chile-registro-civil) | GovChecks: verifica RUN contra base del Registro Civil. |
| **Truora** | [dev.truora.com/docs](https://dev.truora.com/docs/) | [TruChecks](https://www.truora.com/en/truchecks): RUT, Registro Civil, antecedentes judiciales y penales de la Corte Suprema. |

**Patrón común y crítico:** todos son **verificación**, no **descubrimiento**.
Vos aportás RUT *y* nombre, ellos te dicen sí/no. Ninguno te devuelve un celular
que no tenías.

---

## 3. Categoría C — Rutificadores comerciales (zona gris)

Aquí sí aparece el "RUT → mail + celular", con precios visibles. También aquí
está el riesgo legal y reputacional concentrado.

| Servicio | Qué ofrece | Precio / modelo |
|---|---|---|
| [Findatos](https://findatos.com/personas) | Búsqueda por RUT: nombre completo, dirección, comuna, región, **email, teléfono y celular "verificados"** | Freemium + descarga Excel |
| [Rutify / rutificador.live](https://www.rutificador.live/search) | Personas, empresas y vehículos por RUT, nombre, email, teléfono o patente. "Datos cruzados de fuentes públicas chilenas, en tiempo real". Sin registro | API "limpia" ofrecida, precio no publicado |
| [BasesChile](https://baseschile.com/base-datos/base-de-datos-completa-personas-de-chile-con-rut/) | Base de personas con RUT — **650.000 contactos** | Venta de base completa |
| [BaseDatosChile](https://basededatoschile.cl/blogs/base-datos/rutificador-empresas-chile-2026-encuentra-el-rut-y-contactos-de-cualquier-empresa-en-segundos) | **388.000 empresas** activas: RUT, razón social, rubro, dirección, teléfono, correo corporativo, nombre del gerente / representante legal | Venta de base |
| [DataMarket Chile](https://datamarketchile.cl/bbdd/base-de-datos-telefonica-personas-con-alta-contactabilidad/) | **50.000 registros** con nombre, email, dirección, RUT y **hasta 3 teléfonos** por contacto. Para call center, telemarketing, cobranza, encuestas | Venta de base |
| [Dateas.com](https://www.dateas.com/en-us/services/person/CL) | Consulta de personas en Chile por nombre o cédula, sobre padrón electoral. Regional LatAm | Suscripción, precio no público |
| [Apify — scraperschile/rutificador](https://apify.com/scraperschile/rutificador) | Actor que estructura el flujo de `nombrerutyfirma.com` → Dataset, JSON, CSV, Excel, API. Hasta **1.000 términos por run** en cuentas pagas | Precio en panel Apify, por uso |
| [Apify — conceai/rutificador-2026](https://apify.com/conceai/rutificador-2026) | Idem, "API Rutificador Oficial 2026" | Por uso |

### Por qué esta categoría es un problema

1. **Origen del dato.** Ninguno documenta de dónde salen los celulares. El
   padrón no los tiene (ver Hallazgo 2). La explicación más simple es que
   provienen de las filtraciones masivas listadas en la sección 5.
2. **Los scrapers de Apify son intermediarios de intermediarios.** Scrapean
   `nombrerutyfirma.com`, que a su vez no declara su fuente. Cada capa diluye la
   trazabilidad, que es justo lo que la Ley 21.719 te va a exigir demostrar.
3. **Fragilidad técnica.** Dependen de un sitio de terceros que puede cambiar
   HTML, meter CAPTCHA o caer. No es infraestructura sobre la que construir un
   CRM.
4. **Señal de alarma concreta:** existe un actor de amenazas que opera bajo el
   alias **"rutify"** publicando filtraciones de telcos y servicios públicos
   chilenos ([reporte](https://x.com/VECERTRadar/status/2050013074686648353)),
   y existe un rutificador comercial llamado **Rutify**. Puede ser coincidencia.
   Es exactamente el tipo de coincidencia que no querés tener que explicarle a
   la Agencia de Protección de Datos.

---

## 4. Ecosistema open source y conversaciones de comunidad

Esto es lo que pediste específicamente. Revisé GitHub, directorios de APIs
chilenas, CRAN y foros. La conversación existe y es abundante, pero está
**partida en dos familias que conviene no confundir**.

### 4.1 — Familia 1: validación de RUT (legítima, madura, útil)

Nada que ver con obtener datos de nadie: validan el dígito verificador y
normalizan formato. Son buenas librerías y las vas a necesitar igual.

- **[cortega26/rutificador](https://github.com/cortega26/rutificador)** — el más
  sólido. Librería Python + CLI para validar y formatear RUTs. Sin dependencias
  externas, tipado estático completo, procesamiento batch, streaming, e
  integraciones opcionales con **Pydantic v2, FastAPI, pandas y polars**. Este
  es el que usarías en producción.
- **[rutifier](https://repo.miserver.it.umich.edu/cran/web/packages/rutifier/index.html)** — paquete R en CRAN.
- **[GitHub topic: chilean-rut](https://github.com/topics/chilean-rut)** y
  [chilean-rut-utils](https://github.com/topics/chilean-rut-utils) — decenas de
  implementaciones en todos los lenguajes.

### 4.2 — Familia 2: scrapers de rutificadores (frágiles y mayormente abandonados)

Estos son los que "van de RUT a nombre". Todos comparten el mismo patrón: no
tienen datos propios, le pegan a un sitio rutificador de terceros y parsean el
HTML.

- **[lopezjurip/rutificador](https://github.com/lopezjurip/rutificador)** — "Get chilean RUT from people's name".
- **[sebaiturravaldes/rutificador](https://github.com/sebaiturravaldes/rutificador)** —
  RUT↔nombre vía `chile.rutificador.com` y `datos.24x7.cl`.
- **[obordeu/rutificador](https://github.com/obordeu/rutificador)** — busca nombres en `chile.rutificador.com`, devuelve RUT y nombre "según Padrón".
- **[deividxyz/rutificador_nombrerutfirma](https://github.com/deividxyz/rutificador_nombrerutfirma)** —
  scraping de `nombrerutyfirma.cl` con **Selenium y PhantomJS**. PhantomJS está
  discontinuado desde 2018: el repo es efectivamente arqueología.
- **[Theblood/Rutificador](https://github.com/Theblood/Rutificador)** — script bash.

**Observación central:** ninguno de estos repos devuelve **email ni celular**.
Todos devuelven nombre, RUT, dirección, comuna y sexo. Confirma el Hallazgo 2
desde el lado de la implementación: *nadie en open source puede producir el
celular, porque la fuente pública no lo contiene.*

### 4.3 — Familia 3: extracción del padrón electoral del Servel

Esta es la vertiente más activa y técnicamente interesante de la comunidad. El
Servel publica el padrón en PDF; estos proyectos lo convierten a CSV.

- **[chivke/serveliza](https://github.com/chivke/serveliza)** — el más completo.
  "Application to extract data of the Chilean Electoral Service (SERVEL) from
  different open sources". Combina OSINT con pandas y Python 3.
- **[Dokeh-404/X-Check_Chile-2025](https://github.com/Dokeh-404/X-Check_Chile-2025)** —
  el más reciente. Automatiza descarga, desbloqueo y extracción del Padrón
  Electoral Definitivo 2025, consolidando **15,6 millones de registros**.
- **[gaboflowers/servel-padron](https://github.com/gaboflowers/servel-padron)** — PDF→CSV vía `pdftotext`. Python 2.x.
- **[Eitol/servel_scraper](https://github.com/Eitol/servel_scraper)**, **[Theblood/servel](https://github.com/Theblood/servel)**, **[DiegoIdeas/Servel](https://github.com/DiegoIdeas/Servel)**.
- [GitHub topic: servel](https://github.com/topics/servel).

**Dinámica documentada en estos repos:** el Servel fue endureciendo el formato.
Los PDF recientes incluyen **texto de fondo diseñado para romper la extracción
por OCR y rasterizadores**, y los proyectos tuvieron que adaptar sus métodos.
Es decir: el propio Estado chileno está activamente tratando de impedir esta
extracción masiva. Esa es una señal de dirección regulatoria bastante clara.

### 4.4 — Familia 4: herramientas OSINT explícitas

- **[diegoespindola/osintChile](https://github.com/diegoespindola/osintChile)** —
  "Búsqueda automática de información de personas en Chile. Basado en web
  scraping". Acepta **RUT** (`11111111-1`), **patente** (`aabb11`) y **teléfono**
  (`56999999999`). Ver [/sources](https://github.com/diegoespindola/osintChile/tree/main/sources).
- **[0xSS3K/OSINT-CHILE](https://github.com/0xSS3K/OSINT-CHILE)** — guía markdown
  de OSINT con recursos y dorks *específicamente* chilenos. Explícitamente
  excluye herramientas genéricas ("No truecaller. No pymeyes. No google reverse
  image").

Estas herramientas son de uso dual: investigación periodística y de seguridad
por un lado, doxing por el otro. Su existencia responde tu pregunta sobre si
"se está hablando del tema" — sí, activamente. Pero **ninguna es base para un
proceso comercial**: sin SLA, sin contrato, sin garantía de licitud, y usarlas
para prospección te deja sin ninguna defensa frente a un reclamo del titular.

### 4.5 — Directorios de referencia (guardar estos dos)

- **[Alplox/awesome-chilean-apis](https://github.com/Alplox/awesome-chilean-apis)** —
  **89 APIs y 182 endpoints** chilenos categorizados y mantenidos activamente.
  El mejor punto de partida para cualquier integración chilena.
- **[juanbrujo/listado-apis-publicas-en-chile](https://github.com/juanbrujo/listado-apis-publicas-en-chile)** —
  listado de APIs públicas de servicios digitales nacionales.
- **[ChileDataAPI](https://packages.oit.ncsu.edu/cran/web/packages/ChileDataAPI/index.html)** (CRAN) y
  **[censo2017](https://archive.linux.duke.edu/cran/web/packages/censo2017/index.html)** — datos agregados, útiles para análisis de mercado por comuna.

---

## 5. Contexto de filtraciones — por qué el dato de contacto "existe"

Importa porque explica de dónde sale el celular que te van a vender, y porque
comprarlo puede implicar recibir datos de origen ilícito.

| Caso | Escala | Detalle |
|---|---|---|
| **Caja Los Andes** | **10 millones** de chilenos | Base Apache Cassandra **sin autenticación** expuesta a internet. Incluía nombres, fechas de nacimiento, **direcciones, números de teléfono**, montos de crédito y lugares de pago. [Cybernews](https://cybernews.com/security/caja-los-andes-chile-data-leak/) · [SC Media](https://www.scworld.com/brief/database-misconfiguration-exposes-over-half-of-chilean-populations-data) |
| **WOM** | +1 millón de contratos | Bucket **AWS S3 público** con contratos de prepago escaneados, incluyendo RUT. [Cybernews](https://cybernews.com/security/wom-mobile-operator-data-leak/) |
| **Registro Civil (alegado)** | ~10 millones | Reportado dic-2025 por el actor "BreachLaboratory". **No confirmado**. [Reporte](https://x.com/VECERTRadar/status/2000534772842401947) |
| **"BlindBackdoor"** | +70 millones LatAm | Abril 2026. Datos personales, fiscales y **de contacto** de Perú, Argentina, Chile, Colombia, Ecuador, Uruguay y Venezuela, en venta. [Reporte](https://x.com/VECERTRadar/status/2048435871343358413) |
| **"hxleyys"** | 6 millones | Mayo 2026, residentes en Chile. Muestras sin confirmar. [Reporte](https://x.com/VECERTRadar/status/2056722069459484842?lang=en) |
| **Padrón Servel** | 13,7 millones | Ofrecido como "padrón electoral completo". [DiarioBitcoin](https://www.diariobitcoin.com/chile/datos-de-137-millones-de-chilenos-son-ofrecidos-como-supuesto-padron-electoral-completo/) |
| **Publicación oficial Servel** | — | El propio Servel difundió datos sensibles de electores (militancia, domicilio, género, si votó). [CIPER](https://www.ciperchile.cl/2022/04/28/servel-difunde-datos-sensibles-de-electores-habilitados-para-las-municipales-2021-como-militancia-domicilio-genero-y-si-voto-o-no/) · [La Tercera](https://www.latercera.com/la-tercera-pm/noticia/servel/991444/) |

Chile tiene ~20 millones de habitantes. Entre Caja Los Andes y el padrón, **la
mayoría de la población chilena tiene su combinación RUT + dirección + teléfono
circulando**. Eso es lo que alimenta el mercado gris, y es un dato de origen
ilícito independientemente de quién te lo revenda.

---

## 6. Marco legal — Ley 21.719

### Lo esencial

- Publicada en el Diario Oficial el **13 de diciembre de 2024**. Vigencia plena
  el **1 de diciembre de 2026** (24 meses de vacancia). Reemplaza a la Ley 19.628.
- Crea la **Agencia de Protección de Datos Personales** con facultades de
  fiscalización y sanción.
- **Multas hasta 20.000 UTM** (~$1.400 millones CLP). Reincidencia: hasta **4%
  de los ingresos anuales**.

### Los tres puntos que impactan directamente este proyecto

**1. "Público" ≠ "libremente procesable".**
La 19.628 permitía tratar datos de "fuentes accesibles al público" con bastante
holgura — esa es la base jurídica sobre la que se construyó toda la industria de
rutificadores. La 21.719 endurece el estándar. Como resume el abogado Pablo
Viollier: *"Que un dato sea público no necesariamente significa que se puede
procesar libremente"*
([DOE Actualidad Jurídica](https://actualidadjuridica.doe.cl/pablo-viollier-sobre-fuentes-de-acceso-publico-que-un-dato-sea-publico-no-necesariamente-significa-que-se-puede-procesar-libremente/)).
El padrón es público para su finalidad electoral; reutilizarlo para prospección
comercial es un cambio de finalidad que ahora hay que justificar.

**2. El interés legítimo exige un test de balanceo documentado.**
Es una base de licitud válida, pero **no es un cheque en blanco**. Sirve
naturalmente para recomendar productos a *tus propios clientes* dentro de una
relación comercial existente. Los desarrollos comerciales enfrentan un estándar
**más exigente** en proporcionalidad y finalidad que las iniciativas de interés
público. Comprar una base fría de 650.000 personas que nunca oyeron hablar de
vos y llamarlas no pasa ese test.
([Confidata](https://confidata.cl/blog/interes-legitimo-ley-21719-test-balanceo-ejemplos-limites))

**3. Derecho de oposición al marketing directo — Art. 8 letra b).**
El titular puede oponerse al tratamiento realizado exclusivamente con fines de
marketing directo, incluyendo el perfilamiento asociado. Operativamente: **tenés
que poder apagar el tratamiento para una persona específica cuando lo pide**, y
poder demostrarlo. Eso implica tener supresión persistente en el CRM, no un
"lo saco de la lista" manual.

Además aplican los derechos **ARCO+** (acceso, rectificación, cancelación,
oposición y portabilidad), con plazos de respuesta.
([Ciberlex](https://ciberlex.cl/derechos-arco-ley-21719-guia-empresas-chile/) ·
[Hackmetrix](https://blog.hackmetrix.com/derechos-del-titular-ley-21719-como-responder/))

### Riesgo contractual adicional

Independiente de la ley: **SimpleAPI restringe explícitamente su uso a
facturación electrónica**, y los términos de Equifax acotan la finalidad a
evaluación crediticia. Usar cualquiera de los dos para prospección inmobiliaria
es incumplimiento de contrato aunque la ley no te alcance. Vale la pena leer los
términos antes de integrar, no después.

---

## 7. Comparativa final

| # | Proveedor | Docs públicas | Precio público | RUT → contacto persona | Apto producción |
|---|---|---|---|---|---|
| 1 | **TuSI SpA** | ⚠️ Comercial | ❌ | ✅ Celular + mail + RRSS | ✅ (previa due diligence de origen) |
| 2 | **Datamart** | ✅ GraphQL | ❌ | ✅ Antecedentes persona | ✅ (SDK solo .NET) |
| 3 | **Equifax Chile** | ✅ Portal dev | ⚠️ Solo retail | ✅ Direcciones históricas | ⚠️ Finalidad restringida |
| 4 | **TransUnion Chile** | ⚠️ Comercial | ❌ | ⚠️ Sociodemográfico, no contacto | ✅ |
| 5 | **BaseAPI.cl** | ✅ Excelente | ✅ Desde $14.990/mes | ❌ Solo SII/empresa | ✅ |
| 6 | **SimpleAPI.cl** | ✅ Postman | ✅ Free 500/mes | ❌ Solo SII | ⚠️ Solo facturación |
| 7 | **API Gateway** | ✅ | ✅ Pay-as-you-use | ❌ Solo SII/Previred | ✅ |
| 8 | **Verifik** | ✅ | ✅ ~USD 0,20/consulta | ❌ Solo contribuyente | ✅ |
| 9 | **Floid** | ✅ readme.floid.io | ❌ Cotización | ❌ Solo verificación | ✅ ISO 27001 |
| 10 | **Findatos / Rutify** | ❌ | ⚠️ Freemium | ✅ Declarado | ❌ Origen no trazable |
| 11 | **Apify actors** | ✅ | ✅ Por uso | ❌ Sin mail/celular | ❌ Frágil |

---

## 8. Recomendación

### Los 5 llamados que haría, en este orden

1. **TuSI SpA** (contacto@tusi.cl) — es el único que describe exactamente el
   producto que buscás. Pedir por escrito: origen de cada campo de contacto,
   base de licitud, y si tienen DPA adaptado a Ley 21.719.
2. **Datamart SpA** — mejor arquitectura (GraphQL + webhooks). Preguntar por
   SDK no-.NET o si el GraphQL crudo alcanza.
3. **Equifax Chile** — pedir acceso al portal LATAM y el catálogo Chile
   completo, declarando finalidad inmobiliaria desde el inicio para saber
   rápido si te habilitan o no.
4. **TransUnion Chile** — comparar contra Equifax; su cruce con Sinacofi es
   distinto.
5. **BaseAPI.cl** — contratar ya, con plan gratis. No resuelve contacto, pero
   resuelve validación de RUT, situación tributaria y bienes raíces a un costo
   trivial. Es infraestructura que vas a necesitar igual.

A los cuatro primeros, la misma pregunta filtro: **"¿de qué fuente específica
sale el celular y con qué base de licitud fue cedido?"** La calidad de esa
respuesta te va a ordenar el ranking mejor que cualquier demo.

### La observación estratégica

El modelo "compro una base y llamo en frío" tiene fecha de vencimiento: **1 de
diciembre de 2026**. Faltan ~2,5 meses. El dato que la ley sí te deja usar
cómodamente es el que **el propio prospecto te entrega**, y ahí este repo ya
tiene la mitad del trabajo hecho: los ~60 artículos de `contenido/` son
exactamente el activo que genera leads con consentimiento. Replicar ese motor
de contenido para el mercado chileno (precio del m² por comuna, crédito
hipotecario en UF, normativa de arriendo) y capturar con formulario y opt-in
explícito produce leads **más calificados, más baratos y sin exposición
regulatoria** que cualquier base comprada.

El enriquecimiento por RUT tiene un lugar legítimo y defendible en ese esquema:
**validar y completar un lead que ya levantó la mano**, no descubrir uno que
nunca te conoció. Para eso, la combinación **BaseAPI (validación + situación
tributaria) + Datamart o TuSI (completar contacto del lead que ya consintió)**
es sólida y sostiene una auditoría.

---

## Nota metodológica

La política de egress de la sesión de investigación bloqueó el acceso directo a
varios dominios (`docs.datamart.cl`, `apigateway.cl`, `docs.verifik.co`,
`findatos.com`, `baseapi.cl`, `simpleapi.cl`, `floid.io`), por lo que los datos
de esos proveedores provienen de búsqueda web y no de lectura directa de sus
páginas. **Los precios citados deben reconfirmarse en la fuente antes de
cualquier decisión de compra.** Todo lo demás está enlazado a su fuente.
