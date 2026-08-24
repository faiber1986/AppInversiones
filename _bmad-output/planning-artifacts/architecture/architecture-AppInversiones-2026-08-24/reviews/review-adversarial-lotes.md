---
tipo: reviewer-gate-review
lente: adversarial — búsqueda de huecos de divergencia
alcance: enmienda AD-29 … AD-37 (contabilidad por lotes, tri-modo de reajuste, procedencia) y sus choques con AD-1 … AD-28
artefacto: ARCHITECTURE-SPINE.md — Motor de Fricción Neta
fecha: 2026-08-24
veredicto: FAIL
pares_criticos: 7
pares_altos: 6
pares_medios: 2
---

# Revisión adversarial — La enmienda de lotes

## Veredicto

**FAIL.**

La enmienda arregla el defecto que la motivó: agregar la posición borraba la fragmentación por tenencia, y AD-29 lo corta de raíz. Ese diagnóstico es correcto y el remedio es el correcto.

Pero la enmienda introduce **una segunda dimensión ortogonal (el modo de reajuste) y una unidad más fina (el lote) en un spine cuyas claves, cuyo paso anual y cuyo libro estaban dimensionados para la dimensión anterior**, y no reindexó ninguno de los tres. El resultado es que tres de los AD más cargados del spine —AD-21 (clave de libro), AD-22 (paso anual canónico) y AD-8 (todo total se deriva sumando el libro)— quedaron **subespecificados respecto de la enmienda que los usa**, y dos de los AD nuevos son **contradicciones internas literales**, no huecos:

- **AD-37 exige emitir un porcentaje**, mientras **AD-20 lo declara un `Concepto`** del vocabulario cerrado con signo y grupo, y **AD-17 exige que todo asiento lleve `monto_origen: Money` y `monto_cop: Money`**. Un porcentaje no es un `Money`. Las tres reglas juntas no se pueden obedecer.
- **AD-29 prohíbe explícitamente calcular el impuesto sobre el agregado** ("ninguna ruta lo calcula sobre el agregado"), lo cual **prohíbe la forma correcta** de aplicar una tarifa progresiva de cédula general y la porción exenta anual de ganancia ocasional, que son por contribuyente-año, no por lote.

Y hay un hueco que, por sí solo, justifica el FAIL sin ningún otro: **el lote tiene un reloj de tenencia con granularidad de día (`fecha_adquisicion`) dentro de un motor cuyo paso anual (AD-22) tiene granularidad de año y cuyo núcleo tiene prohibido leer el reloj**. El spine no declara qué fecha recibe un lote creado por reinversión, ni en qué punto del año ocurre la venta —AD-22 no tiene paso de venta—. Dos implementaciones obedientes cruzan o no cruzan el umbral de ganancia ocasional sobre exactamente los mismos inputs: 15 % contra 39 % sobre la misma ganancia, que es literalmente la comparación que el producto existe para hacer bien.

Cuento **7 pares críticos, 6 altos y 2 medios**. Los remedios son AD nuevos (AD-38 … AD-50), tipos más precisos y vocabularios cerrados. Ninguno introduce infraestructura.

## Método

No critico en abstracto. Cada hallazgo es un **par**: dos módulos implementados un nivel más abajo del spine, ambos obedeciendo **todos** los AD al pie de la letra —los viejos y los nuevos—, que al integrarse producen cifras incompatibles o un artefacto imposible. Cada par se cierra con un AD nuevo o con uno existente endurecido.

Los AD-1 a AD-28 **no** se revisan de cero: solo se examinan donde la enmienda los toca. Los pares van ordenados por severidad.

---

# CRÍTICOS

## L-1 — El reloj de tenencia del lote tiene granularidad de día; el motor tiene granularidad de año y no puede leer el calendario

**Severidad: CRÍTICA. Es el par más barato de construir y el más caro de todos.**

AD-29 da al lote `fecha_adquisicion` y le da **"su propio reloj de tenencia"**. AD-34 paso (5) clasifica **"por tenencia del lote"** en ganancia ocasional o cédula general. Los umbrales de tenencia son de 2 o 4 años según escenario (AD-19, AD-7).

Ahora mírese de dónde salen los lotes. AD-22 fija el paso anual: apreciación → dividendo bruto → retención → **destino del dividendo neto** → TER → custodia, y ordena que cuando el destino es reinversión **el paso cree un lote nuevo**. El paso anual recibe `anio: int` — las Consistency Conventions son explícitas: *"el año fiscal es un entero explícito, nunca derivado de `today()` dentro del núcleo"*, y el determinismo prohíbe leer el reloj. **El paso anual no tiene una fecha. Tiene un entero.** Pero debe construir un `Lote` que exige `fecha_adquisicion`.

**Módulo A — `motor/friccion/paso_anual.py`.** El dividendo se percibe durante el año gravable; el módulo fecha el lote al inicio del año, porque el flujo de dividendos empieza a acumularse desde enero:

```python
Lote(fecha_adquisicion=date(anio, 1, 1), cantidad=..., trm_reconocimiento=...)
```

**Módulo B — `motor/friccion/paso_anual.py`, otra mano.** El lote se reconoce al cierre del ejercicio, que es cuando el valor patrimonial se declara (AD-36) y cuando la valoración del año es firme:

```python
Lote(fecha_adquisicion=date(anio, 12, 31), cantidad=..., trm_reconocimiento=...)
```

Ambos cumplen AD-29 (lote inmutable, lote nuevo por reinversión), AD-22 (crea lote, no incrementa), AD-30 (TRM congelada), el determinismo (no leyeron el reloj, derivaron la fecha del entero) y AD-1.

**Qué se rompe.** Escenario A, umbral de ganancia ocasional 2 años. Reinversión del dividendo de 2029; la posición se liquida el 2031-06-30.

| | `fecha_adquisicion` | Tenencia a la venta | Clasificación | Tarifa |
|---|---|---|---|---|
| Módulo A | 2029-01-01 | 2 años 5 meses | `ganancia_ocasional` | 15 % |
| Módulo B | 2029-12-31 | 1 año 6 meses | `cedula_general` | 39 % |

Sobre una ganancia del lote de COP 8.400.000: **COP 1.260.000 contra COP 3.276.000**. Con cinco lotes de reinversión en el horizonte, la divergencia acumulada ronda los COP 10.000.000 sobre una posición mediana — y **AD-37 reporta 0 % de fragmentación en el módulo A y ~100 % de la porción reinvertida en el módulo B**. La métrica que la enmienda creó para hacer visible el sobrecosto del vehículo distributivo depende enteramente de una convención que el spine no declara.

**El mismo hueco tiene un segundo brazo, peor: AD-22 no tiene paso de venta.** La secuencia canónica termina en custodia. AD-34 pasos (4) y (5) describen la venta, AD-29 describe ventas parciales que fragmentan la posición, y AD-37 describe la liquidación — pero **ningún AD dice en qué punto del año ocurre la realización**.

- **Módulo C** vende al inicio del año terminal: la porción vendida no paga TER ni custodia ese año.
- **Módulo D** vende después de custodia, al final de la secuencia canónica: la porción vendida paga un año completo de TER y custodia sobre unidades que ya no se tienen.

Sobre USD 500.000 a 0,22 % de TER más custodia, la diferencia es ~USD 1.100 por corrida — y, otra vez, **mueve el reloj de tenencia doce meses**, con lo cual el par C/D reproduce por sí solo la tabla de arriba.

**Cierra con: AD-49 (fecha declarada, venta con slot).**

---

## L-2 — La clave del libro (AD-21) quedó corta en dos dimensiones: el modo de reajuste y el mundo contrafactual

**Severidad: CRÍTICA. AD-21 existe precisamente para prevenir este fallo, y la enmienda lo reabrió por dos flancos.**

AD-21: *"todo `LibroDeAsientos` se crea con clave `(corrida_id, vehiculo_id, escenario)` y rechaza cualquier asiento cuya clave no coincida"*. Su `Prevents` es explícito: evitar que tres corridas escriban en un libro común y tripliquen la fricción, *"un fallo que sobrevive a la convención de determinismo porque produce siempre los mismos bytes equivocados"*.

AD-31 añade **tres modos de reajuste que corren siempre**. AD-28 añade **una segunda pasada contrafactual sobre la misma secuencia**. Ninguna de las dos dimensiones está en la clave.

### Flanco 1 — el modo de reajuste

**Módulo A — `motor/escenarios/corrida.py`.** Obedece AD-31 (*"los tres modos corren siempre, como corridas separadas e independientes"*) y AD-21 (una clave por escenario). Construye **un** libro por `(corrida, vehiculo, escenario)`, como AD-21 le manda, y corre los tres modos contra él:

```python
libro = LibroDeAsientos(clave=(corrida_id, veh, Escenario.A))
for modo in ModoReajuste:          # sin_reajuste, art_70, art_73
    fiscal.liquidar(posicion, modo, libro)   # cada modo escribe su asiento (AD-34)
```

Cada `append` pasa el guard de AD-21: la clave coincide. Cada paso escribe su asiento como exige AD-34 (*"un asiento por paso"*). Ninguna ruta compuso dos ajustes en un mismo cálculo, así que **`ReajusteDoble` (AD-31) nunca se levanta**: AD-31 vigila el *cálculo*, y aquí quien compone es la *suma*.

**Módulo B — `adaptadores/render/`.** Obedece AD-8 al pie de la letra: *"todo total se deriva sumando el libro. Ningún total se calcula por una vía paralela. El render lee el libro; no recalcula."* Suma `costo_fiscal_ajustado` del libro y obtiene la suma de los tres ajustes.

**Qué se rompe.** Sobre una base COP de 100.000.000, con un ajuste art. 70 de +4.400.000 y un art. 73 de +28.700.000, el costo fiscal derivado del libro es **133.100.000** en vez de 100.000.000, 104.400.000 o 128.700.000 — ninguno de los tres valores legítimos. La base gravable cae en 33.100.000 y el impuesto en ~5.000.000 COP a tarifa de ganancia ocasional. Es exactamente el *"costo fiscal inflado y una base gravable indefendible"* que AD-31 dice prevenir, producido sin violar AD-31.

**Segundo brazo del mismo flanco.** Aun si los tres modos usan tres objetos `LibroDeAsientos` distintos, **sus tres claves son idénticas**. `motor/comparacion/` debe indexar 9 celdas (AD-24) y el único identificador que el libro expone es su clave de AD-21. Un `dict[ClaveLibro, LibroDeAsientos]` colapsa 3 modos en 1 y **cuál sobrevive depende del orden de iteración** — con las Consistency Conventions garantizando que el resultado equivocado sea perfectamente reproducible.

### Flanco 2 — el mundo contrafactual

AD-28 ordena recorrer *"la misma secuencia lote por lote, sustituyendo la `trm_reconocimiento`"*, y que la función *"escriba su propio asiento"*.

**Módulo C — `motor/fiscal/devaluacion.py`, lectura mínima.** Corre el contrafactual en memoria y escribe **un solo** asiento: `porcion_devaluacion = impuesto_real − impuesto_contrafactual`.

**Módulo D — `motor/fiscal/devaluacion.py`, lectura ortodoxa.** AD-34 exige *"exactamente este orden, en una única función, con un asiento por paso"*, y AD-8 exige que *ningún* total se calcule por vía paralela. Un contrafactual que calcula un impuesto completo en memoria y solo publica la diferencia **es** una vía paralela: produce una cifra (el impuesto contrafactual) que no está en el libro y que nadie puede auditar sumando. Así que el módulo D reutiliza la misma función de AD-34 y **escribe los cinco asientos del contrafactual en el libro**.

**Qué se rompe.** El libro del módulo D contiene ahora dos juegos completos de `costo_fiscal_ajustado`, `base_gravable` e `impuesto_realizacion`. Derivar el impuesto sumando el libro devuelve **impuesto real + impuesto contrafactual**, cerca del doble. Y el `Renderer`, que por AD-14 y AD-8 no recalcula nada, publica ese número.

Y no hay forma de distinguirlos desde el asiento: AD-17 fija los campos (`monto_origen`, `monto_cop`, `tasa_aplicada`, `anio_tasa`), AD-20 fija concepto/dueño/signo/grupo, AD-21 fija la clave. **Ninguno de los tres tiene un campo que diga "esto pertenece al mundo contrafactual"**, y AD-20 prohíbe inventar conceptos duplicados con sufijo `_contrafactual` fuera del enum cerrado.

**Cierra con: AD-40 (clave de libro completa) + AD-46 (naturaleza del concepto).**

---

## L-3 — El `Lote` no tiene identidad, y el contrafactual de AD-28 colapsa la igualdad de lotes

**Severidad: CRÍTICA.**

AD-29 enumera los campos del lote: `fecha_adquisicion`, `cantidad`, `costo_origen`, `trm_reconocimiento`, `anio_gravable_reconocimiento`. **No hay identificador.** AD-21 ordena que todo tipo de `motor/dominio/` sea `frozen=True`, y un modelo pydantic congelado tiene igualdad **por valor** y es hasheable por valor. Dos lotes con los mismos cinco campos **son el mismo objeto** para `==`, `in`, `set`, `dict`, `list.index` y `list.remove`.

Sobre esa base, AD-29 ofrece `identificacion_especifica` como método de asignación en venta. **Es inimplementable: no hay nada que identificar.** El operador no puede señalar "vender el lote de junio de 2029" si ese lote no tiene nombre; solo puede señalar una posición ordinal, y la posición ordinal depende del orden que cada módulo imponga (ver L-4).

**Y entonces llega AD-28 y borra el único campo que solía desempatar.**

AD-28 recorre la secuencia *"sustituyendo la `trm_reconocimiento` de cada lote por la tasa de entrada de la posición"*. En el mundo contrafactual, **todos los lotes tienen la misma TRM**. Es decir: el contrafactual convierte lotes que en el mundo real eran distinguibles en lotes que son iguales por valor.

**El par.** Un plan de aportes mensuales iguales, o dos reinversiones de igual monto en el mismo año — el caso más común del vehículo distributivo, que es precisamente el vehículo que el producto existe para evaluar:

```
Mundo real:
  L_a = Lote(2030-01-01, 12.500000, Money(1000, USD), trm=4310, 2030)
  L_b = Lote(2030-01-01, 12.500000, Money(1000, USD), trm=4455, 2030)   # != L_a
Mundo contrafactual (AD-28 sustituye ambas TRM por la de entrada, 4100):
  L_a' = Lote(2030-01-01, 12.500000, Money(1000, USD), trm=4100, 2030)
  L_b' = Lote(2030-01-01, 12.500000, Money(1000, USD), trm=4100, 2030)   # == L_a'
```

**Módulo A — `motor/fiscal/asignacion.py`.** Asigna la venta por índice sobre la secuencia. En ambos mundos consume `posicion.lotes[0]` y `[1]`. Correcto.

**Módulo B — `motor/friccion/`, que también debe saber qué lotes salieron** para construir la posición del año siguiente (nadie más puede: `FRI --> FIS`, y la posición es estado de `friccion`). Recibe de `fiscal` la lista de lotes vendidos y la sustrae de la posición:

```python
restantes = [l for l in posicion.lotes if l not in vendidos]
```

En el mundo real esto funciona. **En el mundo contrafactual, `L_b' in vendidos` es `True` porque `L_b' == L_a'`, y el filtro elimina los dos lotes** cuando solo se vendió uno. La posición contrafactual del año siguiente pierde un lote entero.

**Qué se rompe.** AD-28 define la porción atribuible a devaluación como *"la diferencia entre ambos impuestos"*. Si las dos pasadas asignan lotes distintos, esa diferencia deja de medir el efecto cambiario y pasa a medir **el efecto cambiario más un artefacto de asignación**. Con USD 1.000 y una TRM de 4.100 el lote fantasma vale COP 4.100.000 de costo fiscal desaparecido; sobre una ganancia gravada al 39 %, la "porción atribuible a devaluación" del criterio §10.4 se infla en COP 1.599.000 — y la cifra sale rotulada como el resultado de un método declarado y único.

**Y hay un tercer brazo silencioso**: si algún módulo construye `set(lotes)` o `frozenset` para deduplicar, `Posicion` pierde lotes en el mundo real también, cada vez que dos aportes coincidan en los cinco campos.

**Cierra con: AD-38 (identidad de lote) + AD-41 (contrafactual de un solo campo).**

---

## L-4 — La asignación FIFO tiene dos dueños posibles y ningún criterio de orden declarado

**Severidad: CRÍTICA.**

AD-29 dice *"el método de asignación en venta es parámetro de la corrida: `fifo` por defecto"*. No dice **quién ejecuta la asignación**, ni **sobre qué orden**, ni **cómo desempata**.

El spine reparte las dos mitades del problema entre dos módulos:

- **`motor/fiscal/`** es, por AD-20, *"el único dueño de todo concepto tributario"* y por AD-29 quien calcula *"el impuesto de realización lote por lote"*. **Necesita** saber qué lotes se vendieron.
- **`motor/friccion/`** es, por la tabla de capas, *"la evolución año por año de una posición como secuencia de lotes"*. **También necesita** saber qué lotes se vendieron, para construir la posición del año siguiente sobre la que se cobran TER y custodia (AD-22). Y AD-20 se lo permite: solo le prohíbe *calcular y escribir* conceptos tributarios, no ordenar una lista.

**El par.**

**Módulo A — `motor/fiscal/asignacion.py`.** FIFO significa "primero el más antiguo". Ordena por la fecha:

```python
orden = sorted(posicion.lotes, key=lambda l: l.fecha_adquisicion)
```

**Módulo B — `motor/friccion/posicion.py`.** `Posicion` es *"una secuencia inmutable de lotes"* (AD-29). FIFO sobre una secuencia significa "en orden de la secuencia": el lote inicial, luego las reinversiones en el orden en que el paso anual las creó.

```python
orden = posicion.lotes            # el orden de inserción es el orden de llegada
```

Ambos son FIFO. Ambos cumplen AD-29, AD-1 y el determinismo.

**Qué se rompe.** Coinciden **salvo** cuando dos lotes comparten `fecha_adquisicion` — que, por L-1, es **el caso normal**: si el paso anual fecha las reinversiones a `date(anio, 1, 1)`, todas las reinversiones de un mismo año empatan, y en el año de entrada el lote inicial empata con la primera reinversión. El `sorted` de Python es estable, así que A y B coinciden *por accidente* mientras nadie toque el `key`; el día que alguien lo cambie a `(fecha_adquisicion, costo_origen)` —una desambiguación aparentemente inocua— el orden se invierte para los empatados.

Y cuando A y B difieren, **la posición que `friccion` lleva a 2032 no es la posición sobre la que `fiscal` liquidó en 2031**. Los lotes que quedan tienen TRM y relojes de tenencia distintos de los que el impuesto supuso. La divergencia no se detecta con ningún test unitario de módulo: cada módulo, aislado, es correcto.

**Brazo adicional — `identificacion_especifica` a través de las 9 celdas.** El método es parámetro de la corrida, y la corrida produce 9 celdas (AD-24). ¿La selección específica es la misma en las nueve?

- **Módulo C** aplica la lista fija del operador idénticamente a las 9 celdas.
- **Módulo D** lee "identificación específica" como una *estrategia*: reselecciona los lotes que minimizan el impuesto **en cada celda**, porque el costo fiscal ajustado difiere por modo y el óptimo también.

El módulo D produce 9 celdas que ya no son comparables entre sí: la comparación de AD-24 y el ordenamiento de AD-26 dejan de comparar modos y pasan a comparar selecciones de lote. El memorando afirma que el art. 73 rinde 12 % mejor que el art. 70 cuando la diferencia real es que se vendieron lotes distintos.

**Cierra con: AD-38 (asignación con dueño, orden total y congelada por corrida).**

---

## L-5 — La venta parcial parte un lote inmutable, y el spine no dice qué se crea

**Severidad: CRÍTICA.**

AD-29: *"`Lote` es inmutable"*, *"ninguna operación incrementa la cantidad de un lote existente"*. La enmienda prohíbe **aumentar** una cantidad y guarda silencio sobre **disminuirla**. Y una venta parcial que consume el 40 % de un lote es exactamente eso.

**Módulo A — `motor/fiscal/asignacion.py`.** Crea un lote remanente que hereda todo salvo la cantidad y el costo, prorrateados:

```python
remanente = Lote(
    fecha_adquisicion=l.fecha_adquisicion,        # hereda el reloj
    cantidad=l.cantidad * Decimal("0.6"),
    costo_origen=l.costo_origen * Decimal("0.6"),
    trm_reconocimiento=l.trm_reconocimiento,      # hereda (AD-30)
    anio_gravable_reconocimiento=l.anio_gravable_reconocimiento,
)
```

**Módulo B — `motor/friccion/posicion.py`.** Lee la enmienda literalmente: un `Lote` nace en un reconocimiento (AD-30: *"`Lote.trm_reconocimiento` se fija en el reconocimiento inicial"*), y este lote nace en 2031, no en 2027. Fecha el remanente el día de la venta parcial y le pone el año gravable en curso — que es, además, coherente con AD-36, que exige declarar el valor patrimonial del lote *por año gravable*, y con AD-30, que insiste en que la TRM se fija en *el* reconocimiento.

**Qué se rompe.** Lote adquirido 2027-01-10, 100 unidades. Venta de 40 unidades en 2031; las 60 restantes se venden en 2032. Umbral de ganancia ocasional 4 años (escenario B).

| | Reloj del remanente | Tenencia en 2032 | Clasificación | Impuesto sobre COP 100.000.000 |
|---|---|---|---|---|
| Módulo A | 2027-01-10 | 5 años | `ganancia_ocasional` | 15.000.000 |
| Módulo B | 2031-xx-xx | 1 año | `cedula_general` | 39.000.000 |

**COP 24.000.000 de diferencia sobre la misma venta**, y AD-37 reporta 0 % de fragmentación contra 100 %. El par no requiere ninguna reinversión de dividendo: basta con vender en dos tramos, que es el comportamiento por defecto de cualquier retiro programado.

**Brazo secundario.** El módulo B además rompe AD-30 en su propósito: si el remanente se "reconoce" en 2031, ¿con qué TRM? Si hereda la de 2027 (correcto por art. 269) el campo `anio_gravable_reconocimiento` contradice a `trm_reconocimiento` dentro del mismo objeto congelado; si toma la de 2031, la base COP se reexpresó — el defecto que AD-30 declara prohibido, cometido por una ruta que AD-30 no contempla.

**Brazo terciario — el residuo.** El prorrateo de `costo_origen` a 0,6 sobre `Money` exige una regla de cuantización que AD-4 solo define *"al presentar"*. Dentro del núcleo, `Money(1000, USD) * Decimal("0.6")` más `* Decimal("0.4")` no tiene por qué sumar `Money(1000, USD)` con cantidades no terminales. AD-8 exige que los totales se deriven del libro; si el costo del lote vendido más el del remanente no reconstituyen el costo original, **el libro deja de cuadrar por céntimos que se acumulan a lo largo de 10 fragmentaciones**.

**Cierra con: AD-39 (partición de lote como operación cerrada).**

---

## L-6 — Art. 70 no declara si el ajuste es de un año o acumulado; art. 73 tiene dos campos candidatos para "año de adquisición"

**Severidad: CRÍTICA. Es el par con la mayor magnitud absoluta de todos.**

AD-31: *"art. 70 un porcentaje por **año gravable**, art. 73 un factor por **año de adquisición**"*. Consistency Conventions repiten lo mismo. AD-34 paso (3): *"aplicación del modo de reajuste sobre esa base en COP"*.

### Flanco 1 — art. 70: ¿un año o la ventana completa?

"Un porcentaje por año gravable" describe la **forma de la serie**, no **cuántas entradas se aplican**. Y el art. 70 ET es, en la práctica, un ajuste que se declara **cada** año gravable sobre el costo del año anterior — pero el spine nunca lo dice, y AD-36 (*"el valor patrimonial por lote y por año gravable"*) admite las dos lecturas.

**Módulo A — `motor/fiscal/reajuste.py`.** El ajuste corresponde al año gravable de la enajenación: se aplica el porcentaje de ese año, una vez.

```python
factor = Decimal(1) + serie_art70[anio_venta]
```

**Módulo B — `motor/fiscal/costo.py`.** El ajuste es acumulativo desde el año siguiente al reconocimiento del lote hasta el año de la venta, uno por año gravable, que es lo que "una entrada por año, explícita" (Consistency Conventions) sugiere que las entradas son *para*:

```python
factor = prod(Decimal(1) + serie_art70[a] for a in range(anio_reconocimiento + 1, anio_venta + 1))
```

**Qué se rompe.** Lote adquirido en 2027, base COP 100.000.000, vendido en 2033. Porcentajes art. 70 (ilustrativos) 2028–2033: 4,0 % · 3,5 % · 5,2 % · 4,8 % · 3,9 % · 4,4 %.

| | Costo fiscal ajustado | Base gravable (venta 190.000.000) | Impuesto a 15 % | Impuesto a 39 % |
|---|---|---|---|---|
| Módulo A (un año) | 104.400.000 | 85.600.000 | 12.840.000 | 33.384.000 |
| Módulo B (acumulado) | 128.726.000 | 61.274.000 | 9.191.100 | 23.896.860 |

**Entre COP 3,6 y COP 9,5 millones** sobre una sola posición, con los dos módulos citando el mismo AD y la misma serie YAML. Y como AD-36 exige emitir el valor patrimonial por año, el módulo A publicará seis años con el mismo valor patrimonial y el módulo B una curva creciente: **dos declaraciones de renta distintas**, que es la salida que AD-36 existe para producir.

### Flanco 2 — art. 73: ¿qué campo es "el año de adquisición"?

AD-29 le da al lote **dos** campos de año: `fecha_adquisicion` (una fecha) y `anio_gravable_reconocimiento` (un entero). El spine no dice qué relación guardan ni cuál indexa el art. 73.

**El par.** Un ETF estadounidense declara dividendo con fecha de pago 2029-12-28; el bróker liquida y reinvierte el 2030-01-05. El ingreso se causa en el año gravable 2029; las unidades se adquieren en 2030.

```
Lote(fecha_adquisicion=2030-01-05, anio_gravable_reconocimiento=2029, ...)
```

- **Módulo A** indexa `serie_art73[lote.fecha_adquisicion.year]` → F(2030) = 1,2381
- **Módulo B** indexa `serie_art73[lote.anio_gravable_reconocimiento]` → F(2029) = 1,2874

Ambos "indexan por año de adquisición". Sobre una base de COP 18.000.000: **22.285.800 contra 23.173.200**. La divergencia por lote es modesta; con diez lotes de reinversión y el cruce de año que produce todo dividendo de diciembre, se vuelve estructural — y es **sistemática, no aleatoria**: siempre favorece al mismo módulo.

**Peor: el mismo par decide si `VigenciaNoCubierta` se levanta o no.** Consistency Conventions: *"Un año ausente levanta `VigenciaNoCubierta`; nunca se interpola"*. Si `art-73.yaml` solo tiene entradas hasta 2029, el módulo A falla la corrida entera y el módulo B produce un memorando. **Dos implementaciones obedientes, una emite y la otra no.**

### Flanco 3 — ¿puede el contrafactual de AD-28 cambiar qué factor del art. 73 aplica?

Sí, y el spine no lo impide. AD-28 sustituye la `trm_reconocimiento` *"por la tasa de entrada de la posición"*. Esa frase admite dos lecturas:

- **Módulo E — sustitución de campo.** "Como si la TRM del día de adquisición del lote hubiera sido la de entrada." El año del lote no cambia; el factor art. 73 sigue siendo F(2030) y la ventana art. 70 sigue siendo 2031–2033.
- **Módulo F — sustitución de reconocimiento.** "Como si la base COP se hubiera establecido en la entrada." Un lote de reinversión de 2030 en una posición abierta en 2027 pasa a reconocerse en 2027: factor **F(2027)** en lugar de F(2030), y ventana art. 70 de **2028–2033** (6 años) en lugar de 2031–2033 (3 años).

Con los porcentajes de arriba, la ventana de 6 años multiplica el costo por 1,28726 y la de 3 años por 1,13; sobre una base de 18.000.000 el módulo F entrega COP 2,8 millones más de costo fiscal contrafactual, es decir **un impuesto contrafactual menor y una "porción atribuible a devaluación" sobrestimada** en ~COP 1,1 millones al 39 %. La cifra del criterio §10.4 mide entonces el efecto cambiario **más un artefacto de reajuste**, bajo la etiqueta de un método único y declarado.

**Cierra con: AD-41 (ventana de reajuste declarada, contrafactual de un solo campo) + AD-38/AD-49 (año fiscal único del lote).**

---

## L-7 — AD-29 prohíbe la forma correcta de aplicar una tarifa progresiva y una exención anual

**Severidad: CRÍTICA. No es un hueco de divergencia: es una regla nueva que prohíbe el resultado correcto.**

AD-29: *"El impuesto de realización se calcula **lote por lote y se suma**; ninguna ruta lo calcula sobre el agregado."*

La cédula general del art. 241 ET es una **tabla progresiva por rangos de UVT sobre la renta líquida gravable del contribuyente en el año**. La ganancia ocasional tiene, según escenario, una **porción exenta anual**. Ninguna de las dos es una función lineal de la base, y por tanto **`f(a) + f(b) + f(c) != f(a+b+c)`**. AD-29 ordena la izquierda; la norma ordena la derecha.

**El par.** Tres lotes fragmentados, cada uno con ganancia de COP 30.000.000 clasificada a cédula general.

- **Módulo A — `motor/fiscal/liquidacion.py`.** Lee de la configuración la `tarifa_marginal` del `PerfilCliente` (39 %) y la aplica por lote: `3 × 11.700.000 = 35.100.000`. Cumple AD-29 y AD-19 (la tarifa vino de la config).
- **Módulo B — `motor/fiscal/liquidacion.py`, otra mano.** AD-19 prohíbe constantes normativas en el núcleo y ordena leer *"todo umbral, tarifa, UVT"* de la config; la config del escenario trae la **tabla** del art. 241. Aplicada por lote, cada 30.000.000 cae en un rango bajo: ~19 % → `3 × 5.700.000 = 17.100.000`.
- **La respuesta correcta**, tabla aplicada al agregado de 90.000.000, está en un rango superior: ~24.000.000.

**Tres cifras, ninguna coincidente, y AD-29 declara ilegal la única correcta.** El mismo defecto multiplica por tres la porción exenta de ganancia ocasional, si la hay: aplicada por lote se exime tres veces.

Y la enmienda entera se apoya en esto: AD-37 mide el sobrecosto de fragmentación como diferencia de tarifa, lo cual solo tiene sentido si la tarifa está bien aplicada.

**Cierra con: AD-47 (dos fases: clasificación por lote, tarificación por cédula-año).**

---

# ALTOS

## L-8 — AD-37 exige un porcentaje; AD-20 y AD-17 exigen que sea un `Money`. Y el denominador no está definido

**Severidad: ALTA, con un brazo que es contradicción literal.**

### Brazo 1 — la contradicción

AD-20 declara: *"`motor/fiscal/` es el único dueño de todo concepto tributario, incluidas la retención en origen, el ajuste por reajuste fiscal (AD-31) y **el sobrecosto por fragmentación de lotes (AD-37)**"*, y todo miembro de `Concepto` *"declara su módulo dueño, su signo (aporte o detracción) y su grupo de waterfall"*. AD-17 obliga a que **todo** `Asiento` lleve `monto_origen: Money`, `monto_cop: Money`, `tasa_aplicada: Decimal`, `anio_tasa: int`.

AD-37 dice que lo que se reporta es *"qué **porcentaje** de la ganancia quedó gravada a tarifa de cédula general"*, y que *"se deriva del libro (AD-8) con concepto propio del vocabulario cerrado (AD-20)"*.

Un porcentaje no tiene moneda, no tiene tasa aplicada, no tiene un `monto_origen` en USD y no pertenece a ningún grupo de waterfall porque **no es una barra**: no aporta ni detrae del retorno neto. Las tres reglas no se pueden obedecer a la vez. Y hay dos cantidades distintas escondidas bajo un solo nombre:

- **el sobrecosto en pesos** = Σ (tarifa_cedula_general − tarifa_ganancia_ocasional) × ganancia del lote fragmentado — esto **sí** es un `Money` y **sí** es un concepto del waterfall;
- **el porcentaje de ganancia fragmentada** — esto es un ratio y no puede ser un asiento.

AD-20 nombra la primera ("sobrecosto"); AD-37 define la segunda ("porcentaje"). Dos implementadores leyendo cada uno "su" AD producen dos métricas distintas con el mismo nombre.

### Brazo 2 — el denominador

Aun resuelta la contradicción, AD-37 dice "de la ganancia" sin decir de cuál. Hay al menos cuatro candidatos, y no son variantes cosméticas: **cambian el signo**.

**El par.** Venta de tres lotes:

| Lote | Tenencia | Ganancia COP | Clasificación |
|---|---|---|---|
| L1 | 5 años | +80.000.000 | `ganancia_ocasional` |
| L2 | 1 año (reinversión) | +20.000.000 | `cedula_general` |
| L3 | 1 año (reinversión) | −30.000.000 | `cedula_general` |

- **Módulo A — `motor/fiscal/`** usa la ganancia total con signo: `(20 − 30) / (80 + 20 − 30)` = **−14,3 %**
- **Módulo B — `adaptadores/render/`** usa solo ganancias positivas: `20 / (80 + 20)` = **20,0 %**
- **Módulo C** usa la base gravable neta de la porción exenta de ganancia ocasional: un tercer número, mayor que ambos
- **Módulo D** pondera por impuesto en vez de por ganancia: `(sobrecosto de tarifa) / (impuesto total)` = un cuarto número

**Qué se rompe.** AD-37 dice que la métrica *"se emite junto al desglose de fricción, no dentro de él"* — es decir, en la primera plana del memorando. **Un −14,3 % de "ganancia gravada a cédula general" en la primera plana de un documento entregado a un cliente es un defecto que mata el producto**, y el módulo A lo produce obedeciendo cada AD. Peor aún: cuando la ganancia total con signo se acerca a cero —el caso de un vehículo mediocre, que es justo el que hay que poder descartar— el módulo A produce un porcentaje que tiende a ±∞ o divide por cero.

**Cierra con: AD-42 (dos salidas nombradas, denominador fijo, indefinido explícito).**

---

## L-9 — `activo_fijo=false`: AD-33 dice 3 celdas, AD-24 dice 9, y nadie tipa "no disponible"

**Severidad: ALTA.**

AD-33: si `activo_fijo` es `false`, *"los modos `art_70` y `art_73` no están disponibles y **solo corre `sin_reajuste`**"*.
AD-24: *"las funciones públicas de `motor/comparacion/` devuelven **siempre** un resultado indexado por escenario × modo (3 × 3); no existe una firma que devuelva una sola celda"*, y *"el `Renderer` rechaza un payload incompleto"*.

**El par.**

- **Módulo A — `motor/escenarios/`.** Obedece AD-33 al pie de la letra: `modos = [ModoReajuste.sin_reajuste]`. Devuelve una rejilla de 3 celdas.
- **Módulo B — `motor/comparacion/` y `adaptadores/render/`.** Obedecen AD-24: esperan 9 y **rechazan el payload incompleto**.

**Qué se rompe.** Para **toda una clase de clientes** —los que no tienen el activo como activo fijo, que es exactamente el perfil de un comercializador o de quien opera con frecuencia— el sistema **no puede emitir memorando**. No falla ruidosamente en el núcleo con una excepción de dominio: falla en el borde, en el `Renderer`, con "payload incompleto", que es un mensaje que no dice nada sobre el perfil.

La salida obvia es peor: **`motor/comparacion/` rellena las 6 celdas faltantes** con un centinela que inventa por su cuenta. Ahora hay dos representaciones de "no disponible" —la de comparación y la que `motor/escenarios/` produciría por AD-32 para `elegibilidad_art_73 = no_aplica`— y el `Renderer` tiene que conocer las dos.

**Brazo 2 — el ordenamiento.** AD-26 exige que *"todo ordenamiento es total y determinístico"*. ¿Cuál es la métrica de una celda no disponible?

- `None` → `sorted` levanta `TypeError` y la corrida muere en `motor/comparacion/`.
- coerción a `Decimal(0)` → **el vehículo aparece rankeado**, y como el retorno neto de los demás es positivo, aparece último *por una razón que no es económica*. El memorando afirma que el ETF irlandés es el peor de la lista cuando lo que ocurre es que su casilla art. 73 no aplica.
- exclusión → el ranking deja de ser total sobre el conjunto declarado, contra AD-26.

**Brazo 3 — `sin_clasificar` mata el barrido entero.** AD-32: si `elegibilidad_art_73` es `sin_clasificar`, *"el modo `art_73` levanta `ElegibilidadNoClasificada` para ese vehículo"*. Pero ese modo corre **dentro** de un barrido de N vehículos × 9 celdas (AD-26). Una excepción de dominio propagada mata las otras 8 celdas de ese vehículo y las 9N−9 de los demás.

- **Módulo C — `motor/comparacion/`** la captura y la convierte en celda no disponible: un módulo del núcleo *tragándose* una excepción de dominio, que es exactamente lo que AD-7 y AD-32 querían evitar al hacerla ruidosa.
- **Módulo D** la deja propagar: `app/` muestra un traceback en vez de 8 celdas perfectamente válidas.

Dos implementaciones obedientes: una emite memorando, la otra no emite nada.

**Cierra con: AD-43 (`ResultadoCelda` como tipo suma cerrado).**

---

## L-10 — `cantidad` no tiene tipo, escala, redondeo ni prohibición de `float`

**Severidad: ALTA.**

AD-4 blinda el dinero: `Money(Decimal, Moneda)`, sin `float`, sin `Decimal` desnudo, cuantización solo al presentar. Las Consistency Conventions cubren "Dinero y tasas". **`cantidad` no está cubierta por ninguna de las dos.** AD-29 la enumera como campo del lote y no dice más.

Y la enmienda la hizo fraccionaria de manera esencial: *"toda reinversión de dividendo crea un lote nuevo"*, y una reinversión de USD 412,55 a USD 87,33 por unidad son 4,7240352… unidades.

**El par.**

- **Módulo A — `motor/friccion/paso_anual.py`.** Los brókers cotizan fracciones a tres decimales; redondea al crear el lote: `Decimal("4.724")`.
- **Módulo B — `motor/fiscal/`.** Necesita el costo por unidad para asignar la venta; recalcula la división a la precisión por defecto de `decimal` (28 dígitos): `4.724035269…`.

**Qué se rompe.**

1. **Σ cantidades de lotes ≠ cantidad de la posición.** Con diez reinversiones truncadas, la deriva llega a ~0,005 unidades. Una venta de "toda la posición" o deja un lote huérfano de 0,005 unidades —con su propio reloj de tenencia y **su propia fila en la salida por lote y por año de AD-36**— o sobregira el último lote y produce una `cantidad` negativa que ningún AD prohíbe. AD-8 exige que *ningún* total se calcule por vía paralela: la cantidad de la posición y la suma de las cantidades de los lotes son dos vías para el mismo total, y no coinciden.

2. **Nada prohíbe `float`.** AD-1 permite la stdlib; AD-4 prohíbe `float` **para dinero**. `cantidad: float` cumple todos los AD. La suma binaria es dependiente del orden, y FIFO e identificación específica recorren los lotes en órdenes distintos: la comprobación `cantidad_restante == 0` da `True` en un camino y `1.4e-15` en el otro. Se crea un lote fantasma de 10⁻¹⁵ unidades, con reloj propio, que AD-36 obliga a **listar en el memorando**, año por año. La convención de determinismo no lo detecta: los bytes son idénticos en cada corrida, e idénticamente equivocados.

3. **El prorrateo del costo al partir un lote (L-5) hereda el mismo hueco** por el lado del dinero: AD-4 solo define la cuantización *al presentar*, así que el residuo de una partición 60/40 no tiene destino declarado.

**Cierra con: AD-44 (`Cantidad` como tipo con escala y residuo declarados; prohibición de `float` en todo `motor/`).**

---

## L-11 — AD-35 no declara cómo se agrega la procedencia, y sin la lista de parámetros el guard de AD-23 es un no-op

**Severidad: ALTA.**

AD-35: *"Ningún resultado que consuma un parámetro `supuesto_no_verificado` se emite sin la marca correspondiente propagada hasta el artefacto final"*. Declara la obligación de propagar; **no declara la operación de agregación**. Una celda de la rejilla consume una docena de parámetros: umbral de tenencia, tarifa de ganancia ocasional, tabla de cédula general, retención en origen del domicilio, la serie art. 70, la serie art. 73, UVT del año.

**El par.**

- **Módulo A — `motor/fiscal/`.** Agrega por el peor: si algún insumo es `supuesto_no_verificado`, todo el resultado lo es. Adjunta **un booleano** al resultado.
- **Módulo B — `adaptadores/render/`.** Tiene que marcar líneas, no documentos: agrega por concepto. La línea "impuesto de dividendo" sale `verificado_profesional` porque su tarifa lo era; la línea "reajuste art. 70" sale `supuesto_no_verificado`.

**Qué se rompe.** El memorando muestra **todas** sus líneas visibles marcadas como verificadas salvo una, y el **total** marcado como no verificado (el booleano del módulo A). El cliente pregunta cuál de las dos marcas manda, y el operador no tiene respuesta — que es precisamente el modo de fallo que AD-35 dice prevenir: *"un parámetro sin procedencia visible es indistinguible de uno validado, y esa indistinguibilidad es el riesgo"*.

Y hay una tercera lectura que ningún AD excluye: **agregar por mayoría o por conteo** ("83 % de los parámetros verificados"). Es una cifra que suena responsable y no significa nada.

**Brazo 2 — el guard se vuelve un no-op.** AD-23 y AD-35 exigen que *"los guards fallen el render si la prosa presenta como establecido un parámetro no verificado"*. Para hacerlo, el guard necesita **la lista de parámetros no verificados por nombre**, para buscar sus menciones en la prosa. Si el módulo A colapsó la docena de procedencias en un booleano, el guard solo puede hacer dos cosas: fallar todo memorando que tenga el booleano encendido —lo cual, con dos series de reajuste marcadas `TODO` en el propio spine, es **todos**— o no fallar nunca. La segunda opción es un control de cumplimiento que silenciosamente no hace nada, que es la clase exacta de fallo que AD-9 y AD-23 existen para impedir.

**Cierra con: AD-45 (retículo de procedencia con conjunto de parámetros, propagado por asiento).**

---

## L-12 — AD-32 abre una segunda vía de procedencia que evade AD-35, justo en el juicio más consecuente del sistema

**Severidad: ALTA.**

AD-35 obliga a `procedencia: {fuente, fecha_vigencia, estado}` en *"cada **parámetro tributario**"*. AD-19 separa tajantemente las dos poblaciones: *"los valores por domicilio viven en la configuración tributaria, no en el catálogo; **el catálogo declara el domicilio, no su tarifa**"*.

AD-32 pone en el **catálogo** el campo `elegibilidad_art_73: defendible | no_aplica | sin_clasificar`, con `fuente_documental` como respaldo. **`fuente_documental` no tiene `estado`.**

**Qué se rompe.** `elegibilidad_art_73 = defendible` es el juicio normativo **más consecuente de todo el sistema**: es el que habilita el modo art. 73 y, según los números de L-6, un alza de costo fiscal del orden del 25–29 % — millones de pesos de impuesto menos. Y es un juicio sobre la forma jurídica de un emisor extranjero hecho a partir de un KID o un prospecto, es decir, **exactamente el tipo de material no firmado por un profesional que AD-35 nombra en su `Prevents`**.

- **Módulo A — `adaptadores/config/`** valida procedencia sobre `config/tributario/` y `config/reajuste/`, que es lo que AD-35 le bindea. El catálogo pasa con `fuente_documental` y sin `estado`.
- **Módulo B — `adaptadores/render/`** rechaza *"un payload cuyos parámetros no declaren procedencia"* (AD-35). El payload la declara: todos los parámetros **tributarios** la traen. La celda art. 73 sale **sin marca alguna de no verificación**.

El memorando presenta un ahorro de COP 24 millones como hecho establecido, sobre una clasificación que nadie firmó. AD-35 se cumple al pie de la letra y su `Prevents` queda derrotado por un campo de AD-32.

**El mismo hueco cubre a AD-5 y AD-6**: `evento_ingreso_anual` y `retorno_esperado_base` son juicios de clasificación con consecuencia fiscal directa (deciden si hay impuesto anual y si se resta TER) y viven en el catálogo, sin `estado`. Y `brokers.yaml`, que porta el spread FX —30 a 100 pb, la magnitud que el brief §1 declara *ser el producto*— tampoco.

**Cierra con: AD-50 (los juicios normativos del catálogo portan `procedencia`).**

---

## L-13 — AD-36 mete un **saldo** en un libro de **flujos**

**Severidad: ALTA.**

AD-8: el libro es append-only y *"todo total se deriva sumando el libro"*. AD-20: cada `Concepto` declara **signo** (aporte o detracción) y **grupo de waterfall**. Ambas reglas presuponen que un asiento es un **flujo**: algo que ocurre en un año y se acumula.

AD-36 exige emitir *"por lote y por año gravable, el valor patrimonial correspondiente al modo de reajuste elegido"*, y subraya que *"es una capacidad de salida declarada, **no un subproducto derivable** del cálculo de venta"*. El valor patrimonial es un **saldo**: el mismo activo, revaluado, declarado cada año.

**El par.**

- **Módulo A — `motor/fiscal/patrimonio.py`.** Obedece AD-8 y AD-36: emite un asiento `valor_patrimonial` por lote y por año, con su `monto_cop`. Le asigna un signo y un grupo porque AD-20 lo obliga.
- **Módulo B — `adaptadores/render/`.** Obedece AD-8 y AD-14: no recalcula nada; deriva el retorno neto acumulado sumando `monto_cop` del libro, agrupado por grupo de waterfall.

**Qué se rompe.** Seis años × 128.726.000 = **COP 772 millones** sumados al retorno neto de una posición de 100 millones. El waterfall del brief §5 crece una barra que es el activo contado seis veces. Y no hay forma de que el módulo B lo evite: AD-20 obliga a que el concepto tenga grupo, y **no existe la categoría "no sumable"** en el vocabulario.

La salida alternativa es igual de mala: si `valor_patrimonial` **no** entra al libro, entonces AD-36 produce una cifra que llega al memorando **sin pasar por el libro**, que es exactamente la "vía paralela" que AD-8 prohíbe — y que ningún test de arquitectura detecta, porque no es un `import`.

**Cierra con: AD-46 (`naturaleza: flujo | saldo` en `Concepto`; solo los flujos agregan).**

---

# MEDIOS

## L-14 — `tasa_aplicada` / `anio_tasa` del asiento de conversión de base no proviene de `CurvaDeCambio`, y la auditoría obvia fuerza la violación del art. 269

**Severidad: MEDIA-ALTA.**

AD-17 obliga a que todo asiento lleve `tasa_aplicada` y `anio_tasa`. AD-18 declara que `CurvaDeCambio` es *"el único origen de tasas del sistema"* y que *"ningún módulo construye una tasa aritméticamente"*. AD-34 paso (2) convierte la base con `Lote.trm_reconocimiento`, **que en el año de la venta no salió de ninguno de los dos métodos de la curva**.

**El par.** Lote adquirido 2027, TRM congelada 4.100. Venta en 2033.

- **Módulo A** escribe `anio_tasa = 2027`: el año al que pertenece la tasa.
- **Módulo B** escribe `anio_tasa = 2033`: el año del asiento, que es el año en el que ese paso se ejecutó.

Los dos reconcilian `monto_origen × tasa_aplicada = monto_cop`. Pero **la auditoría que AD-18 invita** —comprobar que `tasa_aplicada == CurvaDeCambio.tasa_valoracion(anio_tasa)`— pasa para A y falla para B. Y quien "arregle" el fallo de B llamará a `tasa_valoracion(2033)` para la base de costo: **reexpresará la base**, que AD-30 declara un defecto por el art. 269. El spine contiene un invariante de auditoría que, aplicado, fuerza la infracción que otro AD prohíbe.

**Cierra con: AD-48 (`origen_tasa` en el asiento).**

---

## L-15 — `corrida_id` contra 9 celdas: o la clave colisiona, o el barrido se muda a `app/`

**Severidad: MEDIA.**

Consistency Conventions: *"Corrida por UUID generado en `app/` e inyectado, nunca dentro del núcleo"*. AD-31: *"los tres modos corren siempre, como **corridas separadas e independientes**"*. AD-21: la clave del libro empieza por `corrida_id`.

- **Módulo A — `app/cli.py`.** Lee "corridas separadas" literalmente y genera N × 9 UUID, iterando escenarios y modos para inyectarlos. Ahora `app/` conoce la taxonomía de la rejilla y **construye el barrido** — contra AD-26 (*"`motor/comparacion/` es el único que corre N vehículos × 3 escenarios"*) y contra AD-14. Y `app/streamlit_app.py`, escrito por otra mano, itera en otro orden: los dos frontends producen rankings distintos ante empate, que es el defecto exacto que AD-26 existe para prevenir.
- **Módulo B — `app/streamlit_app.py`.** Genera un UUID por acción del usuario. Entonces las 9 celdas comparten `corrida_id` y la clave de AD-21 colisiona: **L-2**.

**Cierra con: AD-40** — la coordenada completa vive en la clave del libro; `corrida_id` sigue siendo uno por acción del usuario y nada sobre la generación de UUID se muda al núcleo.

---

# Remedios — AD nuevos, redactados para pegar

Quince pares se cierran con trece ADs. Ninguno introduce servicios, colas ni event sourcing: son identidad, tipos, vocabularios cerrados y claves más largas.

### AD-38 — El lote tiene identidad, y la asignación en venta tiene un solo dueño

- **Binds:** `motor/dominio/`, `motor/fiscal/`, `motor/friccion/`, `motor/escenarios/`
- **Prevents:** L-3 y L-4 — que `identificacion_especifica` no tenga nada que identificar, que el contrafactual de AD-28 colapse dos lotes en uno al igualarles la TRM, y que `motor/fiscal/` y `motor/friccion/` ordenen la cola FIFO por criterios distintos y lleven a 2032 una posición que no es la que se liquidó en 2031
- **Rule:** `Lote` lleva `lote_id: LoteId`, obligatorio, **primero** en el orden de campos y **único campo que define la igualdad y el hash** del lote (`__eq__` y `__hash__` se derivan solo de `lote_id`). El `lote_id` es determinístico: `f"{vehiculo_id}:{ordinal:04d}"`, donde el ordinal es el orden de creación dentro de la posición, empezando en 0 para el lote de entrada. No se genera con aleatoriedad ni con reloj. **`motor/fiscal/asignacion.py` es el único dueño de la asignación**; expone una única función que devuelve un `AsignacionDeVenta` inmutable —una secuencia de `(lote_id, cantidad_consumida)`— y `motor/friccion/` construye la posición del año siguiente **exclusivamente a partir de ese objeto**, nunca recomputando la cola ni filtrando por valor. El orden FIFO es **total y declarado**: `(anio_fiscal, momento, ordinal del lote_id)`; no existe otro criterio de desempate. `identificacion_especifica` se expresa como una secuencia de `lote_id` y es **entrada congelada de la corrida**: idéntica en las 9 celdas y en los dos mundos de AD-41. Ningún módulo la reoptimiza por celda; un test lo verifica comparando el `AsignacionDeVenta` de las 9 celdas.

### AD-39 — Partir un lote es una operación cerrada que hereda el reloj

- **Binds:** `motor/dominio/`, `motor/fiscal/`
- **Prevents:** L-5 — que una venta parcial produzca un remanente cuya fecha de adquisición se reinicie, moviendo el lote de ganancia ocasional a cédula general y cambiando el impuesto en COP 24 millones sobre la misma venta; y que el prorrateo del costo deje céntimos sin destino
- **Rule:** la única forma de derivar un lote de otro es `Lote.partir(cantidad) -> tuple[Lote, Lote]`. Los dos lotes resultantes **heredan sin excepción** `anio_fiscal`, `momento` (AD-49), `trm_reconocimiento` y un nuevo campo `lote_id_origen`; solo `cantidad` y `costo_origen` se reparten, a prorrata de la cantidad, con `ROUND_DOWN` a la escala de AD-44 y **el residuo asignado íntegro al lote remanente**, de modo que `partido.costo_origen + remanente.costo_origen == original.costo_origen` es exacto. Ningún constructor de `Lote` fuera de `partir` puede recibir un `lote_id_origen`, y ninguna ruta puede construir un remanente a mano. `Lote.partir` es la **única** excepción a "toda creación de lote es un reconocimiento nuevo": un lote partido **no** es un reconocimiento y no se le aplica AD-30.

### AD-40 — La clave del libro es la coordenada completa de la celda

- **Binds:** `motor/dominio/`, `motor/escenarios/`, `motor/comparacion/`, `motor/fiscal/`
- **Prevents:** L-2 y L-15 — que los tres modos de AD-31 escriban en un mismo libro y el costo fiscal derivado de AD-8 sume los tres ajustes (COP 133 millones donde debía haber 104 o 128); que el contrafactual de AD-28 duplique el impuesto derivable del libro; y que un índice por clave de libro colapse 3 modos en 1 con el resultado dependiendo del orden de iteración
- **Rule:** la clave del libro es `(corrida_id, vehiculo_id, escenario, modo_reajuste, mundo)`, donde `mundo` es un `StrEnum` cerrado `real | contrafactual_devaluacion`. `LibroDeAsientos` rechaza con `ClaveDeLibroAjena` cualquier asiento cuya clave completa no coincida. Una corrida produce **exactamente 3 × 3 × 2 = 18 libros por vehículo**, todos construidos por `motor/escenarios/`. `corrida_id` sigue siendo **uno solo por acción del usuario**, generado en `app/` e inyectado; `app/` **no** itera la rejilla ni genera un UUID por celda. Toda agregación entre celdas se hace leyendo libros, nunca escribiendo en uno común. `motor/comparacion/` indexa por la clave completa y nunca por una proyección de ella.

### AD-41 — El contrafactual sustituye exactamente un campo y no mueve ninguna ventana

- **Binds:** `motor/fiscal/`
- **Prevents:** L-3 y el flanco 3 de L-6 — que el contrafactual de AD-28 se lea como "como si el lote se hubiera adquirido a la entrada" y, al hacerlo, corra la ventana acumulada del art. 70 o cambie qué factor del art. 73 aplica, con lo cual la "porción atribuible a devaluación" del criterio §10.4 mediría el efecto cambiario **más** un artefacto de reajuste
- **Rule:** el contrafactual de AD-28 construye cada lote sustituyendo **exclusivamente** `trm_reconocimiento`. `lote_id`, `anio_fiscal`, `momento`, `cantidad` y `costo_origen` son **idénticos** al mundo real, y por tanto lo son también el factor del art. 73, la ventana de años gravables del art. 70, el reloj de tenencia, la clasificación por cédula y la asignación en venta (AD-38). Un test compara los lotes de los dos mundos campo por campo y falla si difiere alguno distinto de `trm_reconocimiento`. **La ventana del art. 70 se define de una sola manera y se declara aquí:** es acumulativa, un factor por año gravable, desde `anio_fiscal + 1` del lote hasta el año de realización inclusive; el art. 73 usa un único factor, indexado por `anio_fiscal` del lote. Ninguna ventana se deriva del año en que se estableció la base COP.

### AD-42 — La fragmentación tiene dos salidas con nombre, denominador fijo e indefinido explícito

- **Binds:** `motor/dominio/`, `motor/fiscal/`, `motor/comparacion/`, `adaptadores/render/`
- **Prevents:** L-8 — que AD-37 exija emitir un porcentaje mientras AD-20 lo declara un `Concepto` y AD-17 obliga a que todo asiento lleve `Money`; y que cuatro denominadores legítimos produzcan −14,3 %, 20,0 % y dos cifras más para la misma venta, con un porcentaje negativo en la primera plana del memorando
- **Rule:** AD-37 se desdobla en dos salidas distintas y separadamente nombradas:
  **(a) `sobrecosto_fragmentacion`** — un `Money` en COP, miembro del `Concepto` cerrado de AD-20 con signo de detracción, naturaleza `flujo` (AD-46) y grupo propio de waterfall, igual a `Σ sobre los lotes con ganancia positiva clasificados a cédula general de (tarifa_efectiva_cedula_general − tarifa_ganancia_ocasional) × ganancia_del_lote`, con las tarifas efectivas tomadas de la fase 2 de AD-47. Es un asiento y se deriva sumando el libro.
  **(b) `fragmentacion_pct`** — un `Decimal` en fracción, **no es un asiento**, con denominador fijado como **`Σ max(0, ganancia_del_lote)` sobre todos los lotes de la enajenación** y numerador `Σ max(0, ganancia_del_lote)` restringido a los lotes clasificados a cédula general. Es por construcción un valor en `[0, 1]`. Si el denominador es cero, la métrica se emite como **no disponible con razón `sin_ganancia_positiva`** (AD-43), nunca como cero ni como `NaN`.
  Ningún módulo deriva (b) de (a) ni al revés, y el memorando declara el denominador junto a la cifra.

### AD-43 — La celda es un tipo suma cerrado; la rejilla siempre tiene 9

- **Binds:** `motor/dominio/`, `motor/escenarios/`, `motor/comparacion/`, `adaptadores/render/`
- **Prevents:** L-9 — que AD-33 devuelva 3 celdas mientras AD-24 exige 9 y el `Renderer` rechace el payload, dejando sin memorando a toda una clase de perfiles; que `motor/comparacion/` invente su propio centinela de "no disponible"; que una celda no disponible se coaccione a cero y **rankee** en AD-26; y que la `ElegibilidadNoClasificada` de AD-32 mate las otras ocho celdas del barrido
- **Rule:** `ResultadoCelda` es un tipo suma cerrado: `Disponible(libro: LibroDeAsientos)` | `NoDisponible(razon: RazonNoDisponible)`. `RazonNoDisponible` es un `StrEnum` cerrado: `perfil_no_activo_fijo | elegibilidad_art_73_no_aplica | elegibilidad_art_73_sin_clasificar | vigencia_no_cubierta | sin_ganancia_positiva`. **`motor/escenarios/` construye siempre las 9 celdas**; nunca devuelve una rejilla parcial y `motor/comparacion/` nunca rellena. `sin_clasificar` produce una celda `NoDisponible`, no una excepción; `ElegibilidadNoClasificada` se reserva para el caso en que un llamador pida **explícitamente** ese único modo. El ordenamiento de AD-26 recorre **solo** celdas `Disponible`; una celda `NoDisponible` no tiene métrica, no se coacciona a ningún número y no participa del ranking, y el resultado reporta el conteo de celdas excluidas y sus razones.

### AD-44 — `Cantidad` es un tipo con escala, residuo declarado y sin `float`

- **Binds:** `motor/dominio/`, `motor/friccion/`, `motor/fiscal/`, todo el repositorio vía AD-25
- **Prevents:** L-10 — que la reinversión fraccionaria de dividendos produzca una deriva entre la cantidad de la posición y la suma de las cantidades de los lotes (dos vías para el mismo total, contra AD-8); que un `cantidad: float` deje un lote fantasma de 10⁻¹⁵ unidades con reloj de tenencia propio y fila propia en la salida de AD-36; y que una cantidad negativa por sobregiro no la rechace nadie
- **Rule:** `Cantidad` es un tipo del dominio que envuelve `Decimal` con **escala fija de 8 decimales** y `ROUND_DOWN` en toda construcción. Es no negativa: una `Cantidad` negativa levanta `CantidadInvalida`. Toda división que produzca unidades asigna el residuo por una regla declarada —**al último lote creado**— de modo que `Σ cantidad de los lotes == cantidad de la posición` es exacto; `Posicion` verifica ese invariante al construirse y levanta `PosicionDescuadrada`. Un lote con `cantidad == 0` nunca se crea. **`tests/test_arquitectura.py` prohíbe el tipo `float` y la función `float()` en todo `motor/`**, no solo para dinero, con el mismo mecanismo con el que hace cumplir AD-1 y AD-19.

### AD-45 — La procedencia se agrega por el peor y viaja con el conjunto de parámetros

- **Binds:** `motor/dominio/`, `motor/fiscal/`, `motor/cumplimiento/`, `adaptadores/config/`, `adaptadores/render/`, `adaptadores/llm/`
- **Prevents:** L-11 — que un módulo agregue por el peor y otro por concepto, y el memorando muestre todas sus líneas verificadas y el total no verificado; que aparezca una agregación "por mayoría" que suena responsable y no significa nada; y que el guard de AD-23, sin la lista de nombres, o falle todo memorando o no falle ninguno
- **Rule:** `EstadoProcedencia` es un retículo totalmente ordenado con `supuesto_no_verificado > verificado_profesional`, y **la única agregación admitida es el máximo (el peor gana)**. Además del estado agregado, todo resultado carga `parametros_no_verificados: frozenset[ParametroId]`, **no vacío si y solo si** el estado agregado es `supuesto_no_verificado`; la violación de ese bicondicional levanta `ProcedenciaInconsistente`. El conjunto se propaga **por asiento**: AD-17 gana el campo `procedencias: frozenset[ParametroId]`, y la unión de los asientos es la procedencia del total. `adaptadores/render/` marca la línea de cada asiento con su propio conjunto y el total con la unión; los guards de AD-23 reciben el conjunto de nombres y verifican mención por mención. Ninguna agregación por conteo, porcentaje ni mayoría existe en el sistema.

### AD-46 — Todo `Concepto` declara su naturaleza; solo los flujos agregan

- **Binds:** `motor/dominio/`, `motor/fiscal/`, `adaptadores/render/`
- **Prevents:** L-13 y el flanco 2 de L-2 — que el valor patrimonial de AD-36, que es un **saldo**, se sume seis veces al retorno neto acumulado y engorde el waterfall en COP 772 millones; y que la alternativa (emitirlo fuera del libro) abra la "vía paralela" que AD-8 prohíbe
- **Rule:** además de dueño, signo y grupo (AD-20), cada miembro de `Concepto` declara `naturaleza: flujo | saldo`. **Solo los conceptos `flujo` participan en cualquier total derivado**; `LibroDeAsientos.total(...)` levanta `AgregacionDeSaldo` si el filtro incluye un concepto `saldo`. `valor_patrimonial` es `saldo`, vive en el libro —de modo que sigue siendo auditable y no hay vía paralela— y se proyecta por `(lote_id, anio_gravable, modo_reajuste)` para la salida de AD-36. Ningún concepto `saldo` tiene grupo de waterfall.

### AD-47 — Clasificación por lote, tarificación por cédula-año

- **Binds:** `motor/fiscal/`
- **Prevents:** L-7 — que AD-29 (*"lote por lote y se suma; ninguna ruta lo calcula sobre el agregado"*) prohíba la única forma correcta de aplicar una tabla progresiva de cédula general y una porción exenta anual de ganancia ocasional, produciendo COP 35,1 o 17,1 millones donde la norma exige ~24
- **Rule:** la liquidación tiene **dos fases y el límite entre ellas es duro**.
  **Fase 1, estrictamente por lote y jamás agregada:** costo fiscal ajustado (AD-34 pasos 1–3), base gravable del lote (paso 4) y clasificación por tenencia del lote en `ganancia_ocasional | cedula_general` (paso 5). Nada de esta fase se agrega antes de clasificar; es lo que AD-29 protege.
  **Fase 2, por `(cedula, anio_gravable)` y exactamente una vez:** las bases de los lotes de la misma cédula y el mismo año se **suman**, y sobre esa suma se aplican una sola vez la tabla de tarifas, la porción exenta y cualquier umbral en UVT, todos leídos de la config (AD-19, AD-35).
  El texto de AD-29 se corrige: lo que nunca se agrega es **la clasificación y la base antes de clasificar**, no la aplicación de la tarifa. La imputación del impuesto de fase 2 de vuelta a cada lote —necesaria para AD-42— se hace a prorrata de la base gravable del lote dentro de su cédula-año, por una única función, y produce la `tarifa_efectiva` que AD-42 consume.

### AD-48 — El asiento declara el origen de su tasa

- **Binds:** `motor/dominio/`, `motor/fiscal/`, `motor/friccion/`
- **Prevents:** L-14 — que la conversión de la base de costo con la TRM congelada del lote (AD-34 paso 2) sea indistinguible de una conversión hecha con la curva, y que la auditoría obvia (`tasa_aplicada == CurvaDeCambio.tasa_valoracion(anio_tasa)`) empuje a alguien a recomputar la base con la tasa del año de venta — la reexpresión que AD-30 declara un defecto por el art. 269
- **Rule:** el `Asiento` de AD-17 gana `origen_tasa`, un `StrEnum` cerrado: `curva_valoracion | curva_transaccion | trm_lote_congelada`. `anio_tasa` significa **siempre el año al que pertenece la tasa**, nunca el año en que se escribió el asiento; para `trm_lote_congelada` es el `anio_fiscal` del lote. `tests/test_arquitectura.py` reconcilia contra `CurvaDeCambio` **solo** los asientos cuyo `origen_tasa` es una de las dos rutas de curva, y verifica que ningún asiento con `origen_tasa = trm_lote_congelada` tenga un `anio_tasa` posterior al `anio_fiscal` de su lote.

### AD-49 — El lote tiene año fiscal y momento declarados; la realización tiene su lugar en la secuencia

- **Binds:** `motor/dominio/`, `motor/friccion/`, `motor/fiscal/`, `app/`
- **Prevents:** L-1 — que un lote de reinversión fechado el 1 de enero cruce el umbral de ganancia ocasional y el mismo lote fechado el 31 de diciembre no lo cruce (15 % contra 39 % sobre la misma ganancia); y que AD-22 no tenga paso de venta, de modo que dos implementaciones cobren o no cobren un año entero de TER y custodia sobre unidades vendidas y muevan el reloj de tenencia doce meses
- **Rule:** el lote **no lleva una fecha libre**. Los campos `fecha_adquisicion` y `anio_gravable_reconocimiento` de AD-29 se sustituyen por **`anio_fiscal: int`** —el único año del lote— más **`momento: MomentoDelAnio`**, un `StrEnum` cerrado `apertura | cierre`. La convención es de la corrida, no del implementador, y se declara aquí: **toda adquisición por reinversión se reconoce en `cierre` del año en que el ingreso se causa**, y la entrada inicial en `apertura` del año de entrada. La tenencia se mide en **años fiscales completos** entre el `(anio_fiscal, momento)` del lote y el de la realización — nunca en días, nunca con `date`, nunca contra el calendario. **AD-22 gana un paso terminal explícito**: apreciación → dividendo bruto → retención → destino del dividendo neto → TER → custodia → **realización**. La realización es el último paso del año y se ejecuta **después** de TER y custodia: la porción vendida soporta el costo del año completo y el año de realización cuenta como año completo de tenencia. `anio_venta` es entrada explícita de la corrida, nunca derivada del horizonte dentro del núcleo.

### AD-50 — Los juicios normativos del catálogo portan procedencia

- **Binds:** catálogo, `adaptadores/config/`, `motor/fiscal/`, `adaptadores/render/`
- **Prevents:** L-12 — que `elegibilidad_art_73`, el juicio con mayor consecuencia fiscal de todo el sistema (un alza de costo fiscal del orden del 25–29 %, millones de pesos de impuesto), viva en el catálogo con un `fuente_documental` que **no tiene `estado`**, y llegue al memorando presentado como hecho establecido — derrotando el `Prevents` de AD-35 desde un campo de AD-32
- **Rule:** `elegibilidad_art_73` y `forma_juridica_emisor` (AD-32), `evento_ingreso_anual` (AD-5), `retorno_esperado_base` (AD-6) y el spread FX de `brokers.yaml` (AD-18) son **juicios normativos**, no datos descriptivos, y portan el mismo objeto `procedencia: {fuente, fecha_vigencia, estado}` que exige AD-35 para los parámetros tributarios. `fuente_documental` sobrevive como campo adicional, nunca como sustituto. Su ausencia levanta `ProcedenciaNoDeclarada`. Entran al retículo de AD-45 con el mismo peso que cualquier parámetro tributario: una celda art. 73 apoyada en una clasificación `supuesto_no_verificado` **no puede** emitirse sin la marca.

---

## Cambios de texto que la enmienda exige en AD ya existentes

Los remedios no solo añaden. Tres AD vigentes quedan con texto que ahora es incorrecto y hay que reescribir, no solo complementar:

| AD | Frase que cambia | Por qué |
|---|---|---|
| **AD-21** | la clave `(corrida_id, vehiculo_id, escenario)` | pasa a cinco componentes por AD-40 |
| **AD-22** | la secuencia termina en custodia | gana el paso `realización` por AD-49 |
| **AD-29** | `fecha_adquisicion` y `anio_gravable_reconocimiento` como campos; *"ninguna ruta lo calcula sobre el agregado"* | los dos campos se funden en `anio_fiscal` + `momento` (AD-49); la prohibición se acota a la clasificación y la base (AD-47) |
| **AD-37** | *"reporta qué porcentaje… con concepto propio del vocabulario cerrado"* | se desdobla en asiento `Money` + ratio no-asiento (AD-42) |
| **AD-33** | *"solo corre `sin_reajuste`"* | se corren las 9 celdas, 6 como `NoDisponible(perfil_no_activo_fijo)` (AD-43) |
| **AD-32** | `sin_clasificar` levanta `ElegibilidadNoClasificada` dentro del barrido | pasa a celda `NoDisponible` (AD-43) |

## Excepciones tipadas que la enmienda añade al núcleo

Las Consistency Conventions listan las excepciones del dominio. Los remedios añaden, y la lista debe actualizarse: `AgregacionDeSaldo`, `CantidadInvalida`, `PosicionDescuadrada`, `ProcedenciaInconsistente`. `ClaveDeLibroAjena` cambia de significado (clave de cinco componentes) y `ElegibilidadNoClasificada` cambia de disparador (solo ante petición explícita de un modo, nunca dentro del barrido).

## Lo que la enmienda hace bien y no debe tocarse

Para que el FAIL no se lea como un rechazo del cambio:

- **AD-29 en su núcleo es correcto y necesario.** El diagnóstico —agregar borra la fragmentación por tenencia y subestima sistemáticamente el impuesto del vehículo distributivo— es exacto, y ningún remedio de arriba lo debilita. AD-47 lo precisa; no lo revierte.
- **AD-30 está bien situado.** Congelar la TRM en el lote y declarar explícitamente que `tasa_valoracion` **no** se usa para recomputar la base es la lectura correcta del art. 269 y cierra por adelantado una divergencia que habría sido cara.
- **AD-31 acierta al hacer los modos excluyentes y al negarse a interpolar series.** El problema no es la regla; es que el resto del spine no ganó la dimensión.
- **AD-32 y AD-33 sitúan bien las propiedades**: la forma jurídica es del vehículo, el activo fijo es del contribuyente. Esa distinción es correcta y es fácil de equivocar.
- **AD-35 identifica el riesgo real** —la indistinguibilidad entre lo verificado y lo supuesto— y su `Prevents` está bien escrito. Le falta el álgebra, no el juicio.
- **AD-24 acierta al prohibir la omisión silenciosa.** Solo le faltaba el tipo que representa lo omitido.

## Condición de salida

El gate cierra cuando los quince pares tengan un AD que los rechace **por construcción**, no por convención, y cuando existan:

1. un test que corra las 18 combinaciones de AD-40 y verifique que ningún libro acepta un asiento de otra celda;
2. un test que compare campo por campo los lotes del mundo real y el contrafactual y falle si difiere alguno distinto de `trm_reconocimiento` (AD-41);
3. un caso calculado a mano en `tests/fiscal/` que liquide una venta parcial en dos tramos y verifique que el remanente conserva el reloj (AD-39);
4. un caso calculado a mano de tres lotes de cédula general que verifique la tarificación en fase 2 y no la suma de tarificaciones por lote (AD-47);
5. un caso calculado a mano del art. 70 acumulado sobre seis años que fije la ventana de AD-41 con su aritmética escrita en el propio test;
6. un test de la rejilla completa para un perfil con `activo_fijo=false`, que verifique 9 celdas con 6 `NoDisponible` y un ranking que las excluye (AD-43);
7. la extensión de `tests/test_arquitectura.py` a la prohibición de `float` en `motor/` (AD-44).
