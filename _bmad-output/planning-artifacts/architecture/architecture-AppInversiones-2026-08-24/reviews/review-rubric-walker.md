# Reviewer Gate — Rubric Walker

**Artefacto revisado:** `_bmad-output/planning-artifacts/architecture/architecture-AppInversiones-2026-08-24/ARCHITECTURE-SPINE.md`
**Contexto de origen:** `MVP_MOTOR_FRICCION_BRIEF.md`
**Memlog:** `.memlog.md` del mismo run
**Fecha:** 2026-08-24
**Altitud declarada:** initiative · **Propósito:** build-substrate

---

## Veredicto

**CONCERNS.**

El spine es correctamente aburrido y esa es su virtud principal: un proceso Python local, sin base de datos, sin servicios, sin contenedor, con el estado curado en git y el estado regenerable en Parquet. Eso es la respuesta correcta al brief §11 y no hay una sola línea de sobre-ingeniería que reprocharle. El núcleo del spine —AD-1, AD-3, AD-4, AD-7, AD-8— es de calidad alta: fija los puntos de divergencia caros del dominio (pureza, hogar del estado, aritmética monetaria, vigencias tributarias, trazabilidad) y AD-1 incluso trae su propio mecanismo de ejecución (`tests/test_arquitectura.py`).

Los hallazgos **no son "esto no sigue las mejores prácticas"**. Son de tres tipos, todos legítimos bajo la checklist:

1. Reglas que el propio spine declara exigibles pero que delega a un mecanismo (`"falla el build"`, `"bloqueante para merge"`) que **no existe en ninguna parte del documento**.
2. Un requisito del brief marcado como bloqueante o como criterio de aceptación que **no tiene ningún AD que lo sostenga** (§6/§10.2 tres escenarios; §8 prohibiciones no numéricas).
3. Al menos un punto de divergencia real **diferido** (la comparación entre vehículos), donde dos entradas —Streamlit y CLI— van a resolverlo cada una a su manera.

No hay ninguna dimensión entera en silencio. El sobre operacional/ambiental (despliegue, entornos, infraestructura, costo) sí está cubierto y bien cubierto.

---

## Recorrido de la checklist

### 1. ¿Fija los puntos de divergencia reales del nivel de abajo, sin omitir ninguno?

**Mayormente sí, con tres omisiones.**

Fijados bien, y son los correctos:

| Punto de divergencia | AD | Comentario |
|---|---|---|
| Dirección de dependencia / pureza del núcleo | AD-1 | El AD más importante del documento y el único con mecanismo de ejecución propio |
| Hogar del estado | AD-3 | Dos niveles, ambos justificados, `.gitignore` explícito para el regenerable |
| Representación del dinero | AD-4 | `Money(Decimal, Moneda)` con rechazo en runtime de aritmética cross-currency — para un sistema que mezcla COP y USD en cada paso, este es *el* invariante caro |
| Vigencias tributarias y huecos de config | AD-7 | Falla ruidosa, sin default numérico, id de versión en el resultado |
| Trazabilidad de toda cifra | AD-8 | `LibroDeAsientos` append-only; todo total se deriva sumando, no por vía paralela |
| Evento gravable anual | AD-5 | Corrige un defecto real del brief §5.4 (CDT/TES/FIC generan rendimiento gravable sin dividendo) |
| Convención de TER | AD-6 | Previene el doble descuento; campo obligatorio sin default |
| Frontera del LLM | AD-9 | Guard como código, no como prompt (ver hallazgo F-1) |
| Emisión de disclaimers | AD-10 | Un único `Renderer`; §9 resuelto estructuralmente |
| Reproducibilidad de datos de mercado | AD-11 | Snapshots inmutables direccionados por contenido, id registrado en el libro |

Omisiones (detalladas abajo como hallazgos):

- **La invariante de los tres escenarios** (§6 + AC §10.2) aparece en la tabla de capas (`motor/escenarios/` "ejecuta todo cálculo en A, B y C") y en el árbol de directorios, pero **nunca como un AD con Rule**. → **F-2**
- **La serie de tipo de cambio / TRM por año** no tiene dueño ni forma declarada. → **F-5**
- **La comparación entre vehículos** (el eje del AC §10.1) no tiene módulo dueño y está diferida. → **F-4**

### 2. ¿La Rule de cada AD es exigible y previene de verdad lo que declara?

Auditoría AD por AD:

| AD | Exigibilidad | Nota |
|---|---|---|
| AD-1 | **Mecánica** | Test de arquitectura que recorre imports. Modelo a seguir |
| AD-2 | Revisable | Objetiva: si aparece un `Dockerfile` obligatorio o un endpoint, se ve |
| AD-3 | Cuasi-mecánica | Un import de `sqlite3`/`psycopg` es detectable por el mismo test de AD-1 (no está declarado así, pero es trivial) |
| AD-4 | Runtime | `Money` rechaza cross-currency en ejecución. La parte "ningún `float` cruza un límite" es solo tipado no verificado — no hay gate de `mypy`/`ty` declarado (ver F-3) |
| AD-5 | Mecánica | Campo requerido de pydantic, enum cerrado |
| AD-6 | Mecánica | "obligatorio, sin default" = campo requerido de pydantic |
| AD-7 | Runtime | Excepción tipada. Fuerte |
| AD-8 | **Solo revisable** | "Ningún total se calcula por una vía paralela" no es verificable mecánicamente. Se endurece barato: que la firma del `Renderer` acepte **únicamente** un `LibroDeAsientos` y nada más, con lo cual "el render no recalcula" se vuelve estructural en vez de disciplinario |
| AD-9 | Mecánica pero incompleta | El guard es código con test propio: correcto. Pero cubre 1 de las 4 prohibiciones de §8 (**F-1**) y no declara contrato de normalización (**F-6**) |
| AD-10 | Revisable, mecanizable | "Ningún módulo escribe un archivo de salida sin pasar por el `Renderer`" es exactamente del tipo que el test de AD-1 ya sabe cazar (buscar `open(...,'w')`, `Path.write_*`, `savefig` fuera de `adaptadores/render/`). No está declarado y debería estarlo — es el requisito bloqueante de §9 |
| AD-11 | Mecánica | Contenido direccionado por hash: la inmutabilidad es una propiedad del esquema de nombres |
| AD-12 | Revisable | Trivialmente objetiva |
| AD-13 | Revisable | Trivialmente objetiva |
| AD-14 | Mixta | "`app/` no calcula" se apoya en AD-8 (toda cifra viene del libro). "Streamlit escucha en localhost" es una bandera de arranque verificable |

El problema transversal no está en ningún AD individual: **AD-1 dice "falla el build" y el árbol de tests dice "bloqueante para merge", pero el spine nunca define qué es el build ni qué es el gate de merge.** Ver **F-3**.

### 3. ¿Algo bajo "Deferred" podría dejar que dos unidades diverjan?

Seis de las siete entradas están bien diferidas y el razonamiento es sólido:

- Proveedor de datos de mercado / TRM — correctamente aislado tras `FuenteMercado` (AD-11). Diferir la *elección* está bien; lo que no está fijado es la *forma de la serie de cambio* (F-5), que es otra cosa.
- Docker, multi-tenant, valoración/derivados, cifrado en reposo, respaldo — todos fuera de alcance explícito o justificados para un operador único. Diferir el cifrado en reposo con el Supuesto 7 declarado es una decisión defendible y honesta para este MVP; no es un punto de divergencia entre módulos.

La séptima **sí es un hallazgo**: *"Modo por lotes para comparar los 15 vehículos sin UI — el CLI de `app/` lo cubre; su forma exacta la decide Fase 1."* Ver **F-4**.

### 4. ¿Toda tecnología nombrada está verificada como vigente?

**Sí. Verificado independientemente en esta revisión, no aceptado por confianza.**

| Componente | Spine dice | Verificación de esta revisión | Resultado |
|---|---|---|---|
| `anthropic` | 1.0.0 | PyPI JSON: `info.version = 1.0.0` (anteriores 0.125.0, 0.124.0) | ✅ |
| `streamlit` | 1.62.0 | PyPI JSON: `1.62.0`, `requires_python >=3.10` | ✅ compatible con 3.12 |
| `pyarrow` | 25.0.1 | PyPI JSON: `25.0.1`, `>=3.10` | ✅ |
| `matplotlib` | 3.11.1 | PyPI JSON: `3.11.1`, `>=3.11` | ✅ compatible con 3.12 |
| Modelo LLM | `claude-opus-5` | Confirmado como ID vigente | ✅ |
| Precio LLM (Supuesto 4) | 5 USD/M in, 25 USD/M out | Confirmado: Opus 5 = $5.00 / $25.00 por MTok | ✅ el cuadro de costos se sostiene |
| Python 3.12 | fijado en `.python-version` | Todas las dependencias verificadas lo admiten | ✅ |

No se detectó una sola versión inventada. La afirmación del spine ("verificado en vivo contra la API JSON de PyPI, ninguna versión citada de memoria") resiste el muestreo. Esto es exactamente lo que el brief §12 pide y muy pocos artefactos lo cumplen.

Única observación, menor y de consistencia interna, no de vigencia: **`anthropic 1.0.0` es un `.0` de un mayor recién publicado** (con guía de migración v1 activa). AD-15 descartó Python 3.14 con el argumento de que "en una herramienta financiera la ruleta de dependencias no compensa la novedad". Ese mismo argumento aplica al `.0` de un SDK. Ver **F-9**.

### 5. ¿Cubre las dimensiones de la altitud "initiative"? (con foco en el sobre operacional/ambiental)

**Ninguna dimensión está en silencio.** En particular, el sobre operacional que la checklist marca como el fallo típico está explícitamente cubierto:

| Dimensión | ¿Cubierta? | Dónde |
|---|---|---|
| Despliegue | Sí | AD-2 (proceso local), AD-13 (un proceso), Structural Seed con la máquina del operador dibujada |
| Estrategia de infraestructura/proveedor | Sí | AD-2/AD-3, con alternativas descartadas y costo de reversión en el memlog |
| Costo | Sí, y con supuestos declarados | Cuadro de tres escenarios, cota superior explícita, base del cálculo publicada |
| Entornos | Implícita pero suficiente | Hay deliberadamente **un** entorno: la máquina del operador. Merecería una frase explícita porque `output/` y `data/cache/` los comparten las corridas de prueba y las de cliente |
| Persistencia y ciclo de vida del dato | Sí | AD-3, AD-11, `.gitignore` declarado |
| Seguridad / secretos / superficie de red | Sí | Convención de secretos, AD-14 (localhost), y la observación de que solo dos flechas salen de la máquina |
| Frontera de integración externa | Sí | AD-9, AD-11, ambas tras puerto |
| Operación en fallo | **Parcial** | No hay ruta degradada declarada cuando la API del LLM está caída o el guard de AD-9 dispara. Ver F-6 |
| Testing / gate de calidad | **Parcial** | Los tests están en el árbol y son "bloqueantes", pero el gate no existe. Ver F-3 |
| Rendimiento / latencia | **No** | El AC §10.5 pide PDF en menos de dos minutos y el spine no lo presupuesta. Ver F-7 |
| Aprovisionamiento reproducible | Casi | uv + `.python-version` cubren Python; falta el paso `playwright install chromium`, que es una descarga aparte de ~150 MB y es lo que rompe un README "desde cero" en Windows. Ver F-10 |

No recomiendo, y explícitamente descarto como no aplicables aquí: contenedores, orquestación, colas, observabilidad de plataforma, CI multiplataforma, alta disponibilidad, respaldos gestionados. Nada de eso pertenece a este proyecto.

### 6. ¿El spine cubre lo que el brief exige? (§9 bloqueante y §10 criterios de aceptación)

**§9 — requisitos legales bloqueantes:** cubierto y bien cubierto. AD-10 los convierte en una propiedad estructural del pipeline de render en vez de una instrucción de plantilla, e incluye la fecha de vigencia de la config usada (§9 cuarto viñetazo), que es la parte que casi siempre se olvida. Única grieta menor: §9 exige que sean **"visibles y no descartables"**, incluido en pantalla; en Streamlit "no descartable" significa que no puede vivir detrás de un `expander` colapsado, y el spine no lo dice (F-11, bajo).

**§10 — criterios de aceptación, uno por uno:**

| AC | Estado | Sustento / hueco |
|---|---|---|
| 10.1 — Cargar 15 vehículos y compararlos en un comando o pantalla | ⚠️ **Parcial** | El catálogo está (AD-3, `catalogo/vehiculos.yaml`), pero la **comparación** entre vehículos no tiene módulo dueño y está diferida → **F-4** |
| 10.2 — Cada comparación corre en los tres escenarios | ⚠️ **Sin invariante** | Existe la carpeta y la frase, no existe la Rule → **F-2** |
| 10.3 — Waterfall entendible en 30 s | ✅ Adecuado a la altitud | `adaptadores/render/` + matplotlib. El *camino* de la figura (PNG embebido como data-URI en el HTML de Jinja2, que es el único intermedio por AD-12) no está fijado → F-12, bajo |
| 10.4 — Mostrar qué porción del impuesto viene de devaluación | ✅ **Explícito** | AD-8 lo nombra literalmente como total derivado del libro. Depende, eso sí, de que la serie de cambio sea única → **F-5** |
| 10.5 — Memorando PDF en menos de dos minutos | ❌ **No abordado** | → **F-7** |
| 10.6 — Cobertura de tests con casos a mano en fiscal y fricción | ⚠️ **Declarado, no exigible** | El árbol lo dice y lo marca bloqueante; no hay gate → **F-3** |
| 10.7 — Cambiar una tarifa = editar configuración | ✅ **Fuerte** | AD-7 + AD-3. Es el AC mejor resuelto del documento |

**§7 — módulo de cumplimiento:** el brief exige que cada alerta cite la norma "como texto configurable". El spine no le da hogar a esos textos ni a los umbrales (US$60.000 de *situs*, umbral de Formulario 160, umbrales de patrimonio) → **F-8**.

**§2.4 y §8 — prohibiciones regulatorias:** solo una de las cuatro está guardada → **F-1**.

---

## Hallazgos

| # | Severidad | Hallazgo |
|---|---|---|
| F-1 | **critical** | AD-9 guarda solo 1 de las 4 prohibiciones de §8; "no recomendar" queda como instrucción de prompt |
| F-2 | **high** | La invariante de los tres escenarios (§6, AC §10.2) no es un AD y no es exigible |
| F-3 | **high** | AD-1 y §2.6 delegan su ejecución a un "build"/"merge gate" que el spine nunca define |
| F-4 | **high** | La comparación entre vehículos no tiene módulo dueño y está diferida: Streamlit y CLI divergirán |
| F-5 | **medium** | La serie de tipo de cambio por año no tiene dueño ni forma declarada |
| F-6 | **medium** | El guard numérico de AD-9 no declara contrato de normalización ni ruta degradada |
| F-7 | **medium** | El AC §10.5 (PDF en < 2 min) no tiene presupuesto de latencia; 3 llamadas seriales lo ponen en riesgo |
| F-8 | **medium** | `motor/cumplimiento/` queda fuera de los Binds de AD-7 y sus umbrales/textos no tienen hogar en `config/` |
| F-9 | **low** | `anthropic 1.0.0` (`.0` de un mayor nuevo) contradice el criterio de conservadurismo de AD-15 |
| F-10 | **low** | Falta el paso de aprovisionamiento `playwright install chromium` en la ruta "desde cero" |
| F-11 | **low** | "No descartable" (§9) no está traducido a una restricción concreta en la pantalla de Streamlit |
| F-12 | **low** | Ambigüedad de la convención de Determinismo frente a la prosa del LLM; camino de la figura al HTML sin fijar |

---

### F-1 — critical — El guard de AD-9 cubre solo una de las cuatro prohibiciones de §8

**Qué dice el spine.** AD-9 *Prevents:* "que la prohibición del brief §8 sea solo una instrucción de prompt, que un modelo puede ignorar". La Rule implementa un guard determinístico sobre **tokens numéricos**.

**El problema.** §8 prohíbe cuatro cosas: (a) generar cifras que no vengan del motor, (b) **emitir recomendaciones de compra o venta**, (c) estimar retornos futuros, (d) interpretar normativa tributaria por su cuenta. AD-9 mecaniza (a) y deja (b), (c) y (d) exactamente en el estado que el propio AD declara inaceptable: una instrucción de prompt. Y (b) no es estética: §2.4 la llama "una restricción regulatoria" y §9 obliga a declarar que el emisor **no está inscrito en el RNPMV ni vigilado por la Superintendencia Financiera**. Un memorando que diga "conviene entrar a X" emitido por un no inscrito es precisamente el riesgo que los disclaimers de AD-10 intentan acotar.

Complemento del mismo hueco: §8 dice que la sección de **abogado del diablo es obligatoria en todo memorando**. Ningún AD la convierte en una precondición estructural del `Renderer`; hoy depende de que quien arme el pipeline se acuerde.

**Por qué es de altitud initiative.** Determina si *toda* ruta de salida —Streamlit, CLI, PDF, Markdown— hereda la misma barrera, o si cada una la reimplementa. Es la misma clase de decisión que AD-10 ya resolvió bien para los disclaimers.

**Remedio barato (sin sobre-ingeniería).** Extender AD-9 con dos verificaciones más de la misma naturaleza —código, no prompt— y un requisito estructural:
- un guard léxico sobre verbos imperativos y fórmulas de recomendación (`recomiendo`, `conviene comprar`, `debería vender`, `la mejor opción es`), que falle el render igual que el numérico;
- una precondición del `Renderer`: el documento no se emite si falta la sección de abogado del diablo por cada vehículo analizado;
- ambos con test propio, igual que el guard numérico.

Son dos funciones y dos tests, no una capa nueva.

---

### F-2 — high — La invariante de los tres escenarios nunca se vuelve una Rule

**Qué dice el spine.** `motor/escenarios/` aparece en la tabla de capas descrito como "Ejecuta todo cálculo en A, B y C", y en el árbol de directorios como "corre A, B y C sobre el mismo input". Eso es todo.

**El problema.** §6 lo declara **requisito de honestidad del sistema**: "ninguna comparación puede presentarse en un solo escenario. Si un vehículo gana en A pero pierde en B, el memorando debe decirlo en la primera página." El AC §10.2 lo repite. Es una invariante de presentación que cruza todas las rutas de salida — exactamente la misma forma que los disclaimers de §9, que **sí** recibieron un AD (AD-10). Sin Rule, nada impide que una vista de Streamlit muestre solo el escenario A "para simplificar la pantalla", o que el CLI acepte un `--escenario A` y produzca un memorando de un solo escenario que el `Renderer` firmará felizmente, con disclaimers y todo.

**Remedio.** Un AD-15 nuevo, del mismo molde que AD-10: el `Renderer` rechaza todo artefacto cuyo `LibroDeAsientos` no contenga los tres escenarios, y la detección de inversión de orden entre escenarios (gana en A, pierde en B) es una función del núcleo cuyo resultado el `Renderer` está obligado a colocar en la primera página. Una Rule, con la misma exigibilidad que ya tiene AD-10.

---

### F-3 — high — Dos de las reglas más cargadas delegan a un gate que no existe

**Qué dice el spine.** AD-1: *"Un test de arquitectura recorre los `import` de `motor/` y **falla el build** ante cualquier violación."* Árbol de tests: `fiscal/` y `friccion/` con "casos calculados a mano (§2.6, **bloqueante para merge**)".

**El problema.** El spine nunca define qué es "el build" ni qué es "merge". No hay hook de pre-commit, no hay workflow, no hay comando canónico. En un repo de un solo desarrollador sin gate, "bloqueante para merge" significa "bloqueante si me acuerdo". La checklist es clara: una Rule que no se verifica mecánicamente ni se revisa objetivamente es un hallazgo — y aquí el problema es peor, porque la Rule **sí** es mecánicamente verificable pero el mecanismo está ausente del documento.

Esto degrada de golpe a AD-1, que es la decisión estructural que sostiene todas las demás (el memlog dice explícitamente "costo de revertir: ALTO").

**No es una petición de CI empresarial.** El memlog registra `gh autenticado` y `git 2.53.0` en la máquina. La solución mínima es una línea en el spine que nombre el gate: un `pre-push` hook que corre `uv run pytest` (o un único workflow de GitHub Actions de ~15 líneas). Elegir cuál es trivial; **no nombrarlo deja dos reglas sin dientes.**

Relacionado: AD-4 ("ningún `float` cruza un límite de módulo") tampoco nombra un verificador de tipos, y AD-10 ("ningún módulo escribe un archivo de salida sin pasar por el `Renderer`") es cazable por el mismo test que ya existe para AD-1 pero no lo declara. Ambos se cierran con el mismo gate.

---

### F-4 — high — La comparación entre vehículos no tiene dueño, y está diferida

**Qué dice el spine.** Deferred: *"Modo por lotes para comparar los 15 vehículos sin UI — El CLI de `app/` lo cubre; su forma exacta la decide Fase 1."*

**El problema.** El AC §10.1 es "cargar 15 vehículos y compararlos en un solo comando o pantalla". "Un solo comando **o** pantalla" son dos unidades: el CLI y Streamlit. `motor/escenarios/` corre A/B/C **sobre un input** — un vehículo. Nadie es dueño de la capa de arriba: el barrido sobre el catálogo, la agregación, el ordenamiento, los deltas entre vehículos, el criterio de "gana/pierde".

Y aquí el spine se contradice consigo mismo: AD-14 dice que `app/` **no calcula** y que toda cifra que muestre viene de un `LibroDeAsientos`. Pero un ranking entre vehículos y un delta entre dos vehículos **son cálculos**, y hoy no hay ningún módulo de `motor/` que los produzca. La consecuencia mecánica es que Streamlit y el CLI construirán cada uno su propia comparación —quizá con criterios de ordenamiento distintos, quizá anualizando distinto— y el sistema podrá reportar dos rankings distintos para el mismo catálogo. Ese es el defecto que AD-8 fue creado para prevenir, escapándose por arriba.

**Diferir la *forma de la CLI* está bien. Diferir *quién es dueño de la comparación* no.**

**Remedio.** Una frase: la comparación multi-vehículo vive en `motor/escenarios/` (o en un `motor/comparacion/`), consume `LibroDeAsientos` y emite un `LibroDeAsientos` de comparación; `app/` la invoca y la presenta. Con eso, la ergonomía del CLI puede seguir diferida sin riesgo.

---

### F-5 — medium — La serie de tipo de cambio por año no tiene dueño ni forma declarada

**El problema.** AD-4 exige que toda conversión pase "por una tasa explícita", lo cual es correcto pero es una regla sobre la *firma*, no sobre el *origen*. El brief §5 usa la tasa de cambio en cinco puntos distintos: conversión de entrada (base de costo fiscal), tasa del año para gravar el dividendo distribuido (§5.4), tasa final de salida (§5.5), base gravable en pesos (§5.6) y la separación del impuesto atribuible a devaluación (AC §10.4). El spine no dice **quién construye la serie año a año** a partir de la "devaluación anual esperada" del input, ni si es un objeto único que circula, ni con qué convención de capitalización.

Si `motor/friccion/` la deriva por su cuenta para valorar el portafolio y `motor/fiscal/` la deriva por la suya para la base gravable, dos módulos pueden producir tasas distintas para el año N y el AC §10.4 —que es la tesis comercial del producto— saldría con una cifra que no cuadra con el resto del memorando. Es exactamente "dos módulos producen centavos distintos para el mismo cálculo", que es lo que AD-4 dice prevenir, en el eje que AD-4 no cubre.

**Remedio.** Media línea en AD-4 o un AD propio: la senda de cambio es un objeto único (`SendaCambiaria`, año → tasa) construido una sola vez en el borde a partir de los inputs, inyectado a todos los pasos, y sus valores por año quedan asentados en el `LibroDeAsientos`. Nadie deriva una tasa por su cuenta.

---

### F-6 — medium — El guard numérico no declara normalización ni ruta degradada

**Dos problemas distintos en la misma Rule.**

**(a) Sin contrato de normalización, el guard es inusable.** "Extrae todo token numérico de la prosa y falla el render si alguno no aparece en el JSON de origen" fallará contra: el año fiscal (`2027`), ordinales y conteos en prosa (`tres escenarios`, `los 15 vehículos`), separadores de miles colombianos (`1.234.567,89` frente al `Decimal` crudo), porcentajes presentados (`30 %` frente a `0.30` en el JSON, que es justo la convención que declara la sección de Consistency Conventions), puntos básicos redondeados y cifras citadas del propio bloque de disclaimers. Una regla que falsea-positivo en cada corrida es una regla que alguien va a desactivar en la segunda semana — y entonces §8 vuelve a ser un prompt. **La Rule necesita declarar la canonicalización** (comparar contra el JSON *ya formateado* para presentación, no contra el crudo) y una lista blanca cerrada (años, ordinales, numerales de norma citados desde configuración).

**(b) Sin ruta degradada, un fallo externo deja sin salida.** AD-9 dice "falla el render". Si la API está caída o el guard dispara, ¿no hay memorando? El waterfall, el desglose de fricción y los disclaimers son 100 % determinísticos y no dependen del LLM. No declarar la ruta degradada es un hueco operacional **y** un punto de divergencia: Streamlit y el CLI inventarán cada uno su propio comportamiento ante el fallo.

**Remedio.** Declarar que el `Renderer` puede emitir el memorando sin las secciones narrativas, marcado visiblemente como tal, y que esa es la única degradación permitida.

---

### F-7 — medium — El AC §10.5 (< 2 minutos) no tiene presupuesto

**El problema.** El AC 10.5 es un requisito de latencia: "genero un memorando en PDF, con disclaimers, en menos de dos minutos". El spine, en cambio, define implícitamente el perfil de ejecución en la sección de costos: **3 llamadas por memorando** a `claude-opus-5`, con **5k tokens de salida por llamada**. Encadenadas en serie, eso son 15k tokens de salida de un modelo de razonamiento; el cálculo determinístico y el PDF son despreciables al lado. El riesgo de exceder los dos minutos es real y estructural, no una micro-optimización.

Ningún AD aborda esto, y el documento no reconoce el AC.

**Remedio (barato y sin plataforma).** Basta una decisión declarada, cualquiera de estas: emitir las tres llamadas en paralelo (son independientes entre sí — redacción, abogado del diablo, auditoría de supuestos), usar streaming para que el tiempo percibido no sea el total, o declarar explícitamente que el presupuesto de 2 minutos se mide sobre la ruta determinística y que la narrativa se agrega de forma asíncrona. Lo que no puede quedar es el AC sin mención: si Streamlit y el CLI resuelven esto distinto, uno de los dos incumple.

---

### F-8 — medium — `motor/cumplimiento/` está fuera de los Binds de AD-7 y sus datos no tienen hogar

**El problema.** AD-7 (config tributaria fechada, falla ruidosa, sin defaults) declara `Binds: adaptadores/config/, motor/fiscal/, motor/escenarios/` — **`motor/cumplimiento/` no está**. Pero §7 está lleno de parámetros tributarios de la misma naturaleza: el umbral de US$60.000 de *situs* estadounidense, el umbral de declaración del Formulario 160, los umbrales de impuesto al patrimonio (que §6 declara explícitamente variables entre escenarios A/B/C). Son parámetros normativos con fecha de vigencia, cubiertos por el principio §2.3, y hoy nada impide que se codifiquen como constantes en Python.

Además, §7 exige que cada alerta **"cite la norma o el concepto aplicable como texto configurable"**. En AD-3 la enumeración de lo que vive en YAML es "configuración tributaria, catálogo de vehículos y tarifas de bróker" — los textos de alerta no están; y el árbol de directorios solo muestra `config/tributario/` y `config/brokers.yaml`. Sin hogar declarado, esos textos terminan hardcodeados, y entonces cambiar la cita de una norma exige tocar código, violando el AC §10.7 en el módulo de cumplimiento.

**Remedio.** Añadir `motor/cumplimiento/` a los Binds de AD-7, y un `config/cumplimiento.yaml` (umbrales + textos de norma + advertencia de validación profesional) al árbol y a la enumeración de AD-3.

---

### F-9 — low — `anthropic 1.0.0` contra el criterio de AD-15

Verificado como versión vigente, así que no es un error de vigencia. Es una inconsistencia de criterio: AD-15 descartó Python 3.14 con el argumento de que "en una herramienta financiera la ruleta de dependencias no compensa la novedad", y `anthropic 1.0.0` es el `.0` de un mayor recién salido con guía de migración activa. Basta con reconocerlo explícitamente (fijar la versión exacta en el lock, y anotar por qué se acepta aquí y no en el intérprete) o retroceder al último `0.125.x`. Es la única dependencia del stack cuya elección no está argumentada bajo el mismo estándar que el resto.

---

### F-10 — low — Falta el paso de aprovisionamiento de Chromium

AD-12 elige Chromium headless vía `playwright` y descarta WeasyPrint con un argumento excelente y verificado (la cadena GTK+ en Windows). Pero `pip install playwright` **no** trae el navegador: hace falta `playwright install chromium`, una descarga aparte. El brief §11 exige un README que explique cómo correr todo desde cero, y este es precisamente el paso que rompe ese "desde cero" en una máquina limpia de Windows. El memlog anota que el operador ya tiene `playwright` npm global — que es un binario distinto del de la librería Python. Una línea en la Structural Seed o en AD-12 lo cierra.

---

### F-11 — low — "No descartable" no está traducido a la pantalla

§9 exige disclaimers "de forma visible y **no descartable**", explícitamente incluyendo pantalla. AD-10 cubre bien la emisión, pero en Streamlit "no descartable" tiene un significado concreto: no puede estar dentro de un `expander` colapsado, ni en una `sidebar` que se pliega, ni en un `toast`. Una cláusula de una línea en AD-10 o en AD-14 lo vuelve revisable objetivamente en vez de dejarlo al criterio de quien maquete la pantalla.

---

### F-12 — low — Dos ambigüedades menores de convención

**(a) Determinismo frente a la prosa del LLM.** La convención dice "dos corridas con los mismos inputs producen bytes idénticos". Para el núcleo es cierto y valioso. Para el memorando **no lo es**: la prosa del LLM no es reproducible byte a byte, y el PDF de Chromium lleva metadatos con marca de tiempo. Como AD-11 promete "reproducir un memorando de hace seis meses", conviene precisar el alcance: se reproducen **las cifras y el libro de asientos**, no los bytes del documento.

**(b) Camino de la figura al HTML.** AD-12 declara el HTML como único formato intermedio y matplotlib está en el stack. Cómo llega la figura al HTML (PNG embebido como data-URI, o SVG en línea) no está fijado; es el "activo comercial del producto" según §5 y va a un PDF impreso, donde la elección importa. Una línea basta.

---

## Lo que está bien, y conviene dejarlo escrito

Para que la lista de hallazgos no se lea como un juicio global negativo:

- **El spine es correctamente aburrido.** Un proceso, un paquete, archivos, git. Cero infraestructura, cero costo recurrente, cero servicios. Es la respuesta correcta y el documento la defiende con alternativas descartadas y costo de reversión explícito por decisión, no por reflejo.
- **AD-1 con test de arquitectura** es el patrón que los otros AD deberían imitar: la regla trae su propio mecanismo.
- **AD-4 (`Money` con moneda explícita)** identifica el error más caro posible en este dominio específico —sumar COP con USD— y lo hace imposible en runtime, no por convención.
- **AD-7 sin defaults numéricos** implementa literalmente el §12 del brief ("un número inventado es peor que un campo vacío") como comportamiento del cargador.
- **AD-8 (libro de asientos)** es la decisión más elegante del documento: convierte la trazabilidad de §2.2 en una propiedad estructural y, de paso, mata la clase entera de bug "dos módulos reportan totales distintos".
- **AD-5 y AD-6 corrigen defectos reales del brief** en vez de implementarlo al pie de la letra. Eso es exactamente lo que §12 pide del agente.
- **AD-12 descarta WeasyPrint con evidencia concreta** (issues de Windows), no con preferencia.
- **La verificación de versiones se sostuvo al muestreo independiente**, incluido el precio del modelo que sustenta el cuadro de costos.
- **Los supuestos de costeo están declarados como supuestos**, con la advertencia de que es cota superior. El cuadro no se presenta como si fuera un dato.

## Cierre

Ninguno de los hallazgos pide una capa nueva, un servicio, una base de datos ni una herramienta de plataforma. F-1 son dos funciones y dos tests. F-2 es un AD más del molde de AD-10. F-3 es un hook o un workflow de quince líneas, ya nombrable con lo que hay instalado. F-4 es una frase que asigna dueño. Los `medium` son cláusulas dentro de AD existentes. El spine no está sobre-ingenierizado; está **bajo-especificado en cuatro puntos donde el brief es explícito y bloqueante**. Con F-1 a F-4 cerrados, este documento pasa.
