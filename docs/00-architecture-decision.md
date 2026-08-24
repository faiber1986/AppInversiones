# Decisión de arquitectura — Motor de Fricción Neta

> Entregable de la **Fase 0** del brief (§0.2). Responde las cinco preguntas que pediste decidir explícitamente, con sus trade-offs.
>
> El contrato técnico ejecutable vive en `_bmad-output/planning-artifacts/architecture/architecture-AppInversiones-2026-08-24/ARCHITECTURE-SPINE.md` (51 decisiones arquitectónicas). Este documento es el porqué; el spine es el qué.
>
> Fecha: 2026-08-24 · Producido con BMAD-METHOD v6.11.0, workflow `bmad-architecture` en modo headless, con Reviewer Gate completo.

---

## Resumen en una línea

**Un proceso Python local en tu máquina, sin base de datos, sin contenedor, sin nube. YAML en git para lo que debe tener historia auditable, Parquet en disco para lo que se puede volver a descargar. Costo de infraestructura: cero, en los tres escenarios.**

La respuesta es más aburrida que la que tu brief ofrecía como piso ("un contenedor Docker, Postgres y archivos Parquet"). Ni siquiera Postgres se justifica. Abajo está el porqué.

---

## 1. Local-first vs. cloud

**Decisión: local-first. Ni AWS, ni Azure, ni VPS.**

El sistema corre como un proceso Python en tu Windows 11. No hay servidor, ni contenedor obligatorio, ni servicio gestionado. (`AD-2`)

### Por qué

Miré qué carga de trabajo hay realmente detrás del brief y no encontré nada que necesite un servidor:

| Lo que el sistema hace | Lo que exigiría un servidor |
|---|---|
| Un operador ejecutando análisis por lotes | Concurrencia — no hay |
| 15 vehículos curados a mano | Volumen — son kilobytes |
| Cálculo determinístico de segundos | Cómputo sostenido — no hay |
| Memorandos generados a demanda | Disponibilidad 24/7 — el brief no la pide |

Y hay una razón que no es de costo: el sistema procesa **datos fiscales de terceros**. Que no salgan de tu máquina es una propiedad que se pierde el día que subes esto a un VPS, y recuperarla después cuesta mucho más que mantenerla desde el principio.

### Qué descarté y qué costaría revertirlo

**Descartado: VPS simple con Docker.** Es la opción defendible que más cerca estuvo. La rechacé porque introduce un costo recurrente, una superficie de administración (parches, TLS, respaldos) y un lugar más donde viven datos de clientes, todo a cambio de cero capacidad nueva para un solo operador.

**Costo de revertir en 6 meses: bajo-moderado — 1 a 2 días de empaquetado, no una reescritura.** Y esto no es casualidad: es consecuencia directa de `AD-1` (el núcleo no toca red ni UI) y `AD-3` (el estado son archivos). Un núcleo puro con estado en archivos se mete en un contenedor sin tocarse. Si hubiera aceptado que la lógica de negocio hablara con una base de datos, revertir costaría semanas.

**Descartado: AWS o Azure gestionados.** No se evaluaron en serio. Para un usuario, la pregunta no es cuál nube sino si alguna, y la respuesta es no.

---

## 2. Base de datos: relacional vs. time-series vs. Parquet

**Decisión: ninguna base de datos. Dos niveles de almacenamiento en archivos.** (`AD-3`)

| Nivel | Qué guarda | Formato | Versionado |
|---|---|---|---|
| **(a) Dato curado** | Config tributaria, catálogo de vehículos, tarifas de bróker, textos normativos de alertas | YAML | git |
| **(b) Dato descargado** | Series de mercado, TRM | Parquet inmutable en `data/cache/` | Direccionado por contenido, `.gitignore` |

### Por qué no Postgres

Este es el punto donde vale la pena detenerse, porque tu brief lo ofrecía explícitamente y yo lo estoy rechazando.

Tu §2.3 exige que los parámetros tributarios sean **configuración versionada con fecha de vigencia**, y tu §2.2 exige que todo cálculo sea auditable. Traducido: necesitas saber, seis meses después, exactamente qué tarifa estaba vigente cuando produjiste un memorando, y poder demostrarlo.

Una base de datos relacional **no te da eso de fábrica**. Un `UPDATE` sobre una tarifa destruye el valor anterior. Para conseguir historia auditable tendrías que construir a mano tablas temporales o de auditoría, con sus triggers, y mantenerlas.

Git ya es exactamente ese mecanismo, y es mejor: cada cambio de tarifa queda con autor, fecha, mensaje explicando la fuente normativa, y diff. Es el sistema de control de versiones haciendo aquello para lo que existe.

Además, el dato es **texto curado a mano por un humano**, en la escala de kilobytes. Un archivo YAML es directamente legible y editable por tu contador tributarista; una fila de Postgres no.

| | Base de datos | YAML en git |
|---|---|---|
| Historia por fecha de vigencia | Hay que construirla | Nativa |
| Quién cambió qué y por qué | Hay que construirlo | Nativo |
| Editable por tu tributarista | No | Sí |
| Proceso extra corriendo | Sí | No |
| Respaldo | Hay que montarlo | El remoto de git |

### Por qué Parquet sí, pero solo para el nivel (b)

Los datos de mercado son series temporales columnares, se descargan en volumen y son regenerables. Parquet les da compresión, esquema tipado e inmutabilidad con una sola dependencia (`pyarrow`). Cada descarga se guarda como snapshot direccionado por contenido y **nunca se sobrescribe**, así que un memorando de hace seis meses se puede reproducir con los datos de entonces. (`AD-11`)

**Descartado: base de datos time-series (InfluxDB, TimescaleDB).** Existen para ingesta de alta frecuencia y consultas por ventanas móviles. Aquí hay series anuales de 15 instrumentos.

**Costo de revertir en 6 meses: bajo.** Las lecturas ya pasan por puertos (`RepositorioConfig`, `FuenteMercado`), así que meter un DBMS es escribir un adaptador nuevo, no tocar el motor.

**Cuándo reabrir esta decisión:** si el catálogo pasa de decenas a cientos de vehículos, o si aparece un segundo operador escribiendo config al mismo tiempo. Ninguna de las dos está en el alcance del MVP.

---

## 3. Monolito vs. servicios

**Decisión: monolito. Un paquete Python, un proceso.** (`AD-13`)

No hay segundo consumidor, no hay límite de escalado que separar, no hay dos equipos que desacoplar. Partir esto en servicios compraría serialización, versionado de contrato entre servicios y un modo de fallo distribuido nuevo, a cambio de nada.

**Lo que sí hay son límites internos fuertes.** El paradigma es **hexagonal (puertos y adaptadores) con núcleo funcional puro**:

- `motor/` calcula y **no puede importar** `streamlit`, `anthropic`, `httpx2`, `pyarrow`, `matplotlib` ni ninguna librería de E/S. Un test recorre los `import` del núcleo y falla el build ante cualquier violación. (`AD-1`)
- Los adaptadores implementan protocolos que declara el núcleo; `app/` los inyecta.

Esto es lo que hace testeable el motor con casos calculados a mano (tu §2.6) y lo que mantiene barato revertir la decisión 1.

**Descartado: separar motor y UI en servicios.** Un límite de módulo que un test hace cumplir da la misma disciplina que un límite de red, sin latencia, sin serialización y sin un segundo proceso que operar.

---

## 4. Cómo se cachean y versionan los datos de mercado

**Decisión: puerto + snapshots inmutables direccionados por contenido.** (`AD-11`)

1. El núcleo lee mercado **solo** por el puerto `FuenteMercado`. Nunca sabe quién es el proveedor.
2. Cada descarga se persiste como snapshot Parquet nombrado por el hash de su contenido.
3. **Un snapshot nunca se sobrescribe.** Datos nuevos son un snapshot nuevo.
4. Toda corrida registra en su libro de asientos el id del snapshot que usó.

Esto compra dos cosas que el brief pide por separado:

- **§2.5 — cambiar de proveedor sin tocar lógica de negocio.** Cambiar de proveedor es escribir un adaptador nuevo.
- **§2.2 — trazabilidad.** Un memorando de marzo se puede reproducir byte por byte, porque los datos que usó siguen existiendo sin cambios y él sabe cuáles fueron.

El proveedor concreto queda **diferido a la Fase 1**. Es una elección que el puerto aísla, y el brief §2.5 solo impone que sea gratuito.

---

## 5. Costo mensual en tres escenarios

**Infraestructura: 0 USD/mes en los tres.** El único costo recurrente es la API de Anthropic.

| Escenario | Memorandos/mes | LLM | Infraestructura | Total |
|---|---|---|---|---|
| 1 usuario (tú) | 10 | 6–9 USD | 0 USD | **~6–9 USD/mes** |
| 10 clientes | 30 | 18–25 USD | 0 USD | **~18–25 USD/mes** |
| 100 clientes | 300 | 180–250 USD | 0 USD | **~180–250 USD/mes** |

### Cómo se calculó, y qué asumí

Base: 3 llamadas por memorando —redacción, abogado del diablo y auditoría de supuestos, los tres usos permitidos de tu §8—, con ~15k tokens de entrada y 5–8k de salida cada una, a **5 USD / 25 USD por millón de tokens** (`claude-opus-5`, precio verificado, no citado de memoria).

**Es un rango y no una cifra única por una razón concreta:** en `claude-opus-5` el razonamiento adaptativo está encendido por defecto y **factura como tokens de salida**. El extremo bajo asume `effort` bajo o medio para la redacción, que no es una tarea de razonamiento duro; el extremo alto asume el valor por defecto. El *prompt caching* sobre el system prompt de reglas y disclaimers, que es estable entre llamadas, baja aún más el extremo bajo y no está descontado aquí.

**Dos supuestos que debes validar, porque son míos y no tuyos:**

1. **El volumen de memorandos me lo inventé.** Tu brief no da ninguno. Todo el cuadro escala linealmente con ese número: si haces el triple de memorandos, paga el triple.
2. **"10 clientes" y "100 clientes" los interpreté como clientes *atendidos por ti***, no como usuarios con login propio. Tu §3 deja multi-tenant, autenticación y facturación explícitamente fuera de alcance. **Si lo que quieres es que los clientes entren solos, este cuadro no aplica** y la infraestructura deja de ser cero.

### Por qué los tres escenarios cuestan lo mismo en infraestructura

Porque son el mismo despliegue. Atender 100 clientes no cambia dónde corre el software; cambia cuántas veces lo corres. Lo que escala aquí no es la infraestructura: **es tu tiempo**.

---

## Stack fijado

Versiones verificadas en vivo contra la API JSON de PyPI el 2026-08-24.

| Componente | Elección | Versión |
|---|---|---|
| Lenguaje | Python (fijado en `.python-version`) | 3.12 |
| Gestor de entorno | uv | 0.12.5 |
| Validación de config | pydantic + pyyaml | 2.13.4 / 6.0.3 |
| Interfaz | Streamlit local, sin exponer a red | 1.62.0 |
| Gráfico de cascada | matplotlib | 3.11.1 |
| Cache de mercado | pyarrow (Parquet) | 25.0.1 |
| Memorando | Jinja2 → HTML → PDF | 3.1.6 |
| Motor de PDF | playwright (Chromium headless) | 1.62.0 |
| LLM | anthropic / `claude-opus-5` | 1.0.0 |
| Tests / lint | pytest / ruff | 9.1.1 / 0.16.4 |

### Dos notas de riesgo que salieron de la revisión

1. **Tu `uv` está seis meses desactualizado.** Tienes 0.10.5 (febrero 2026); el vigente es 0.12.5. Conviene actualizar antes de empezar.
2. **`anthropic` 1.0.0 es un major publicado el 2026-08-20**, cuatro días antes de este documento, y migró a `httpx2`. Por eso el adaptador de mercado usa `httpx2` y no `httpx` ni `requests`: un solo stack HTTP en el proceso (`AD-16`). El SDK está detrás de un puerto, así que si su migración da problemas, el radio de daño es un adaptador.

### Por qué no WeasyPrint

Es la ruta obvia en Python para HTML→PDF y la descarté. WeasyPrint exige Pango, cairo y GDK-PixBuf de GTK+ instalados aparte; en Windows la versión 69 sigue pidiendo MSYS2 y `pacman`. Eso choca de frente con tu §11: *"un README que explique cómo correr todo desde cero"*. Chromium headless vía playwright ya funciona en tu máquina. (`AD-12`)

**Un paso de setup que debe quedar en el README:** `playwright install chromium` descarga ~100–150 MB una sola vez y `uv sync` no lo resuelve.

---

## Qué encontró el Reviewer Gate

Corrieron tres revisores adversariales en paralelo contra el spine. **Ninguno lo aprobó a la primera**, y el más duro lo reprobó. Todas sus correcciones están aplicadas.

| Lente | Veredicto | Hallazgo más valioso |
|---|---|---|
| Rúbrica de buen spine | CONCERNS | "La herramienta no recomienda" (§2.4, restricción regulatoria) no tenía mecanismo de cumplimiento: el guard numérico deja pasar *"recomiendo el ETF irlandés"* porque no contiene ninguna cifra |
| Verificación de tecnología | CONCERNS | La tabla de versiones afirmaba estar verificada contra PyPI y una fila no lo estaba; faltaban dos paquetes que las propias reglas exigían |
| Adversarial de divergencia | **FAIL** | Dos reglas del spine se contradecían entre sí — ver abajo |

### El defecto que valió la pena todo el ejercicio

El revisor adversarial construyó pares de módulos que cumplen **todas** las reglas al pie de la letra y aun así producen resultados incompatibles. Encontró 15. El peor:

> `motor/friccion/` opera en dólares, así que escribe sus asientos en USD. `motor/fiscal/` causa impuesto en pesos, así que escribe los suyos en COP. Ambos cumplen las dos reglas. Pero una regla exigía derivar el retorno neto en COP **sumando el libro**, y otra exigía que sumar monedas distintas levantara una excepción.
>
> **El entregable central de tu §5.7 era imposible por construcción.**

Se cerró con `AD-17`: todo asiento es bimonetario y se convierte al escribirse, registrando qué tasa usó y de qué año.

Doce de los quince pares nacían del mismo sitio: yo había *declarado* el libro de asientos sin *especificarlo*. Los otros cierres importantes:

- **`AD-18`** — La tasa de cambio no tenía dueño. El spine no nombraba la TRM ni una vez. `motor/friccion/` podía valorar un dividendo a tasa de cierre y `motor/fiscal/` gravarlo a tasa de apertura, y la razón impuesto/dividendo dejaría de coincidir con ninguna tarifa de la configuración. Peor: nadie decía si el spread FX entra en la valoración — entre 30 y 100 puntos básicos, que es *la magnitud que tu §1 declara ser el producto*.
- **`AD-19`** — `motor/cumplimiento/` podía tener el umbral de ganancia ocasional como constante (2 años) mientras `motor/fiscal/` leía 4 años de la config del escenario B. El memorando se contradiría a sí mismo en la misma página, exactamente sobre el "requisito de honestidad" de tu §6.
- **`AD-20`** — La retención en origen estaba asignada a la vez a dos módulos. Con `concepto` como texto libre, uno la escribiría como magnitud positiva y otro como decremento negativo: doble conteo, dos barras en el waterfall para una sola fricción y —si los signos se cancelan— **un total limpio y equivocado que ningún test unitario de módulo detecta**.
- **`AD-21`** — Nada impedía compartir un libro entre los tres escenarios y triplicar toda la fricción. El fallo habría sobrevivido a la regla de determinismo, porque produce siempre los mismos bytes equivocados.
- **`AD-28`** — La cifra de tu criterio de aceptación §10.4 (qué porción del impuesto se debe a devaluación) tenía dos métodos legítimos y ningún dueño. Ahora es contrafactual, en una sola función, declarado en el memorando.

Las revisiones completas están en `_bmad-output/planning-artifacts/architecture/architecture-AppInversiones-2026-08-24/reviews/`.

---

## Preguntas abiertas

### Bloquean la Fase 1, no la Fase 0

La arquitectura acomoda cualquiera de las respuestas sin cambio estructural, así que no impiden cerrar esta fase. Sí impiden escribir la especificación.

1. **¿Qué se hace con el efectivo del dividendo distribuido?** ¿Se reinvierte, se acumula en caja, se repatría? Cambia la TIR y define la comparación distributivo vs. acumulativo, que es el eje de tu catálogo. **Requiere decisión tuya.**
2. **¿Aplica el reajuste fiscal del costo (art. 70 ET)?** Erosiona parte de la ganancia por devaluación, que es tu argumento comercial central. Si el modelo lo ignora, la cifra que le muestras al cliente sale inflada. **Requiere a tu tributarista.** No lo interpreto yo.
3. **¿Cómo se presentan 45 resultados** (15 vehículos × 3 escenarios) sin perder al cliente no técnico en 30 segundos, como pide tu §10.3? Decisión de producto, le corresponde a Spec Kit.

### No bloquean nada

4. **Cuál proveedor gratuito de datos de mercado y TRM.** `AD-11` lo aísla tras un puerto.
5. **Los valores numéricos de los escenarios A, B y C.** Tu §6 dice explícitamente que los cargas tú, validados con tu contador. `AD-7` hace que el sistema **falle ruidosamente** mientras estén marcados `TODO`: nunca hay un cero silencioso.

---

## Todos los supuestos que hice

El modo headless corre el riesgo de decidir en silencio. Estos son todos los supuestos que tomé, sin excepción:

| # | Supuesto | Si es falso |
|---|---|---|
| 1 | "10/100 clientes" = atendidos por ti, no usuarios con login | El costeo deja de ser cero y cambia la decisión 1 |
| 2 | Volumen: 10/30/300 memorandos al mes | El cuadro de costos escala lineal con esto |
| 3 | 3 llamadas al LLM por memorando (los 3 usos de §8) | Costo proporcional |
| 4 | `claude-opus-5`, 15k entrada / 5–8k salida por llamada | Costo proporcional |
| 5 | Costo sin descontar *prompt caching* | El real será menor |
| 6 | Existe una fuente gratuita de mercado adecuada; **no verifiqué cuál ni su licencia** | `AD-11` aísla la elección; la arquitectura no cambia |
| 7 | Es aceptable que los datos fiscales de clientes vivan en texto plano en un repo git privado local, **sin cifrado en reposo** | Se reabre si un cliente lo exige |
| 8 | El catálogo se cura a mano y no crece a cientos | `AD-3` (sin base de datos) es lo primero a reabrir |
| 9 | Corres esto en tu máquina; sin acceso remoto ni requisito de disponibilidad | Cambia la decisión 1 |
| 10 | El cuadro de costos **no es una cota superior** — el *thinking* de Opus 5 factura como salida | Corregido: por eso es un rango |

---

## Lo que este documento no decide

Nada del qué/porqué funcional. Ni el flujo de la interfaz, ni la forma del memorando, ni cómo se presentan los 45 resultados, ni el proveedor de datos. Eso es Fase 1, con GitHub Spec Kit, alimentado por estas decisiones.

---

# Addendum — 2026-08-24, tras resolución parcial del bloqueante 2

El operador entregó el marco de reajuste fiscal. No fue solo la respuesta a una pregunta abierta: introdujo cuatro cambios estructurales al modelo de dominio. El spine pasó de 28 a **37 decisiones arquitectónicas** (`AD-29` a `AD-37`); ningún AD previo se renumeró.

## Advertencia de procedencia — condiciona todo lo demás

El material tributario recibido **no está firmado por un profesional** y tiene una **contradicción interna entre dos de sus documentos**. Instrucción del operador: tratar todo parámetro derivado de él como supuesto no verificado, marcado como tal en configuración, pendiente de concepto profesional, y **no presentarlo como hecho en ningún output**.

Esto no quedó como nota al pie. Quedó como invariante ejecutable (`AD-35`): cada parámetro tributario declara `procedencia: {fuente, fecha_vigencia, estado}`, su ausencia levanta `ProcedenciaNoDeclarada`, el `Renderer` rechaza payloads sin ella, y los guards del LLM fallan el render si la prosa presenta como establecido un parámetro no verificado.

## Cambio 1 — El lote es la unidad fiscal, no la posición (`AD-29`, `AD-30`)

Cada adquisición es un lote inmutable con su propio reloj de tenencia y su TRM de reconocimiento **congelada** (art. 269 ET: la base en COP se fija al reconocimiento inicial y no se reexpresa anualmente). Toda reinversión de dividendo crea un lote **nuevo**.

**Por qué es estructural y no un refinamiento:** al vender, la posición se fragmenta entre lotes que superan el umbral de ganancia ocasional y lotes que no. Un cálculo sobre la posición agregada los trata a todos igual y **subestima sistemáticamente el impuesto del vehículo distributivo** — justo la comparación que el producto existe para hacer bien. `AD-37` lo convierte en métrica propia: qué porcentaje de la ganancia quedó gravada a cédula general por fragmentación.

## Cambio 2 — Modos de reajuste excluyentes (`AD-31`)

Art. 70 y art. 73 son alternativos y excluyentes sobre el mismo activo para la misma enajenación. Se trata con el mismo rigor que la mezcla de monedas: componer ambos ajustes levanta `ReajusteDoble`, igual que sumar COP con USD levanta `MonedaIncompatible`.

Tres modos corren siempre como corridas separadas: `sin_reajuste`, `art_70` (porcentaje por **año gravable**), `art_73` (factor por **año de adquisición**). Ambas series quedan como `TODO` en `config/reajuste/`, una entrada por año, con fuente normativa y fecha de vigencia obligatorias. **Ningún porcentaje ni factor fue inventado.**

`AD-36` recoge la consecuencia operativa que el operador señaló: adoptar art. 73 obliga a declarar el costo ajustado como valor patrimonial en **cada** declaración anual. Es obligación recurrente, así que el valor patrimonial por lote y por año es una salida de primera clase, no un subproducto de la liquidación de venta.

## Cambio 3 — Campos nuevos de catálogo (`AD-32`)

`forma_juridica_emisor`, `nombre_legal_completo`, `isin`, `fuente_documental` y `elegibilidad_art_73`. Regla dura: `sin_clasificar` **bloquea** el modo art_73 para ese vehículo con `ElegibilidadNoClasificada`. Nunca hay degradación silenciosa a art_70.

## Cambio 4 — Activo fijo (`AD-33`)

Parámetro del **perfil del cliente**, no del vehículo — es propiedad de quién tiene el activo, no de qué activo es. Si es `false`, solo corre `sin_reajuste`. El motor nunca lo infiere.

## Secuencia canónica del costo fiscal (`AD-34`)

Orden obligatorio, una sola función, un asiento por paso: costo en moneda origen + comisiones → conversión a COP con la TRM de reconocimiento del lote → reajuste sobre esa base COP → base gravable al vender → clasificación por tenencia **del lote**.

Fijar el orden importa porque reajustar en moneda de origen y luego convertir da un número distinto que convertir y luego reajustar, y ambas lecturas parecen razonables por separado.

## Consecuencia que hay que mirar de frente

`AD-24` ya exigía todo output en los tres escenarios normativos. Los tres modos de reajuste son **ortogonales** a esos escenarios: son **9 celdas por vehículo**, y con 15 vehículos, **135 conjuntos de resultados** por corrida.

El criterio de aceptación §10.3 —que un cliente no técnico entienda el desglose en 30 segundos— se vuelve sustancialmente más difícil. `AD-24` fija que las nueve celdas **existan** y que las no disponibles (`AD-32`, `AD-33`) se emitan con su razón en vez de omitirse en silencio. **Cómo se presentan es diseño de producto y le corresponde a la Fase 1.**

## Qué sigue abierto

El **bloqueante 1 sigue sin resolver**: qué pasa con el efectivo del dividendo distribuido. El cambio 1 dice que toda reinversión crea un lote nuevo y que la fragmentación es un costo del vehículo distributivo, lo que sugiere reinversión — pero no lo afirma, y no lo doy por supuesto. Va a `/speckit.clarify`.

---

# Addendum 2 — Segunda pasada del Reviewer Gate sobre la contabilidad por lotes

El revisor adversarial corrió contra los `AD-29`–`AD-37` y dio **FAIL**: 7 pares críticos y 6 altos. Acepté el veredicto — **dos de los ADs que yo había agregado eran contradicciones literales, no huecos**. El spine pasó de 37 a **51 ADs**: 6 enmendados en sitio, 14 nuevos. Ninguno renumerado.

## Los tres hallazgos que valían el ejercicio

**1. La clave del libro quedó incompleta.** Al introducir el modo de reajuste y el mundo contrafactual creé dos dimensiones ortogonales, pero dejé la clave del libro en `(corrida, vehículo, escenario)`. Los tres modos pasaban el guard y escribían **en el mismo libro**, y `AD-8` obliga a derivar los totales sumándolo: el costo fiscal salía sumando los tres ajustes. `ReajusteDoble` no lo atrapaba, porque vigila el cálculo y no la suma. Cerrado con `AD-41`: clave de cinco campos, **18 libros por vehículo**.

**2. Prohibí la forma correcta de liquidar.** `AD-29` decía "impuesto lote por lote, **nunca** sobre el agregado". Pero una tabla progresiva y una exención anual se aplican precisamente sobre la base agregada de la cédula en el año. Con mi regla, tres lotes pequeños habrían pagado una tarifa marginal muy inferior a la que corresponde a su suma. Cerrado con `AD-44`: **dos fases** — clasificación por lote, tarificación por `(cédula, año)`.

**3. El reloj del lote no se podía leer.** Le puse `fecha_adquisicion` día a día a un motor que avanza por años. Un lote de reinversión fechado en enero y otro en diciembre del mismo año habrían cruzado el umbral de tenencia en sentidos opuestos, saltando entre la tarifa de ganancia ocasional y la marginal sobre la misma operación. Y `AD-22` **no tenía paso de venta**, así que vender antes o después de la custodia movía el reloj doce meses. Cerrado con `AD-42` y un paso terminal de realización.

## Otros cierres

| AD | Cierra |
|---|---|
| `AD-38` | El lote no tenía identidad: con `frozen=True` la igualdad era por valor, y el contrafactual —que iguala la TRM de todos los lotes— hacía que dos lotes distintos se volvieran idénticos |
| `AD-39` | FIFO tenía dos dueños, y con la tenencia medida en años los empates son el caso normal |
| `AD-40` | Venta parcial de un lote inmutable: nadie decía qué representa el remanente |
| `AD-43` | El art. 70 no dice si su porcentaje es anual o acumulado, y el art. 73 tenía dos campos candidatos para "año de adquisición" |
| `AD-45` | `cantidad` no tenía tipo — `AD-4` cubre dinero, no participaciones |
| `AD-46` | El valor patrimonial anual es un **saldo** entrando a un libro de **flujos**: sumarlo por año inflaba el retorno en un múltiplo del capital |
| `AD-47` | "Porcentaje de la ganancia" admitía varios denominadores razonables con resultados de signo contrario |
| `AD-48` | "No disponible" no tenía tipo: los perfiles no-activo-fijo se quedaban **sin memorando** |
| `AD-49` | La procedencia no declaraba cómo se agrega: un total podía quedar marcado como verificado con una línea que no lo está |
| `AD-50` | `elegibilidad_art_73` evadía el invariante de procedencia |
| `AD-51` | El spread cambiario según el modo de destino del dividendo (aporte del operador) |

## Lo que deliberadamente NO decidí

`AD-43` deja la **ventana del art. 70** (¿anual o acumulada sobre los años de tenencia?) como parámetro declarado en configuración, no como lectura normativa mía. Mueve el costo fiscal en decenas de millones sobre un caso realista. Es exactamente el tipo de interpretación que el brief §8 me prohíbe hacer por mi cuenta: **va al tributarista**, y mientras no esté declarada el sistema falla.
