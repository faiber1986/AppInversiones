---
tipo: reviewer-gate-review
lente: adversarial — búsqueda de huecos de divergencia
artefacto: ARCHITECTURE-SPINE.md — Motor de Fricción Neta
brief: MVP_MOTOR_FRICCION_BRIEF.md
fecha: 2026-08-24
veredicto: FAIL
---

# Revisión adversarial — Huecos de divergencia

## Veredicto

**FAIL** — el spine blinda bien los *límites* del sistema (AD-1, AD-2, AD-13 son sólidos y no encontré forma de divergir contra ellos), pero deja sin cerrar el contrato de su propio artefacto central: el `Asiento` de AD-8 es una tupla de cinco campos sobre la que dos implementadores obedientes producen libros incompatibles, sumables a cifras distintas o directamente no sumables — y en un dominio donde el brief §2.1 y §12 definen "cifra inventada" como el defecto crítico, un total silenciosamente equivocado es peor que una excepción.

## Método

No critico en abstracto. Cada hallazgo es un **par**: dos módulos implementados un nivel más abajo del spine, ambos cumpliendo **todos** los AD al pie de la letra, que al integrarse producen un resultado incorrecto. Cada par se cierra con un AD nuevo o con uno más estricto. Los remedios respetan el sesgo del proyecto: nada de servicios, colas ni event sourcing — solo ADs más estrictos, tipos más precisos y vocabularios cerrados.

Los pares están ordenados por severidad. Los remedios se consolidan al final en ocho ADs redactados y listos para pegar en el spine.

---

## D-1 — El libro mezcla monedas: AD-4 y AD-8 se contradicen entre sí

**Severidad: CRÍTICA (contradicción interna, no solo hueco).**

**Módulo A — `motor/friccion/`.** Evoluciona un portafolio denominado en USD (brief §5.1: la primera operación es COP→USD). Escribe sus asientos en la moneda en que ocurre el hecho:

- `Asiento(2028, "ter", Money(Decimal("-412.55"), USD), ...)`
- `Asiento(2028, "custodia", Money(Decimal("-35.00"), USD), ...)`
- `Asiento(2028, "retencion_origen", Money(Decimal("-186.20"), USD), ...)`

Esto cumple AD-4 al pie de la letra: cada monto lleva moneda explícita, no hay `float`, no hay `Decimal` desnudo, no hubo conversión sin tasa explícita. Y cumple AD-8: el motor emite asientos tipados, append-only.

**Módulo B — `motor/fiscal/`.** El impuesto colombiano se causa y se paga en pesos. Escribe:

- `Asiento(2028, "impuesto_dividendo", Money(Decimal("-1420880"), COP), ...)`
- `Asiento(2033, "impuesto_realizacion", Money(Decimal("-58314200"), COP), ...)`

También cumple AD-4 y AD-8 sin objeción.

**Qué se rompe.** AD-8 ordena: *"todo total —retorno neto, desglose de fricción, porción del impuesto atribuible a devaluación, TIR— se **deriva sumando el libro**. Ningún total se calcula por una vía paralela."* AD-4 ordena que `Money` *"rechaza en tiempo de ejecución cualquier operación aritmética entre monedas distintas"*. El total obligatorio del brief §5.7 es **"retorno neto acumulado en COP"**. Derivarlo sumando este libro levanta `MonedaIncompatible` en el primer asiento cruzado. Los dos ADs, obedecidos literalmente, hacen imposible el entregable.

Y la salida obvia es peor que el problema: quien intente arreglarlo en el sitio equivocado convertirá los asientos USD a COP **en el render**, es decir, en un adaptador, con una tasa que el adaptador tuvo que elegir por su cuenta — y ahí se ha filtrado cálculo financiero fuera de `motor/`, contra AD-1 y AD-14, sin que ningún test de arquitectura lo detecte (convertir monedas no es un `import` prohibido).

**Cierra con:** AD-15 (asiento bimonetario) — todo `Asiento` lleva obligatoriamente `monto_origen: Money`, `monto_cop: Money`, `tasa_aplicada: Decimal` y `anio_tasa: int`. La conversión ocurre **en el momento de escribir el asiento**, dentro del núcleo, con la tasa que dicta el dueño único de la curva (D-2). Los totales se derivan sumando `monto_cop`; `monto_origen` existe para auditar y para el desglose en USD.

---

## D-2 — Dos dueños de la tasa de cambio del año *t*

**Severidad: CRÍTICA.**

El spine nombra la tasa de cambio **cero veces**. No hay AD que diga de quién es. El brief §5 la exige en cuatro puntos distintos: conversión de entrada (§5.1), conversión del dividendo del año (§5.4), conversión de salida (§5.5) y base de costo fiscal en COP (§5.6). Y `motor/friccion/` y `motor/fiscal/` necesitan las dos primeras y la cuarta.

**Módulo A — `motor/friccion/`.** Recibe `trm_inicial` y `devaluacion_anual` como supuestos (brief §5 Inputs). Para convertir el dividendo del año 3 usa la tasa de **cierre** del año:

```
tasa(t) = trm_inicial * (1 + devaluacion) ** t     # t = 3 → 4000 * 1.05**3 = 4630.50
```

Es la convención natural para quien evoluciona un saldo: el dividendo se recibe a lo largo del año y se valora al cierre, igual que el saldo del portafolio. Cumple AD-4 (tasa explícita), AD-8, AD-7 (la devaluación no es parámetro tributario), y el determinismo de las Consistency Conventions.

**Módulo B — `motor/fiscal/`.** El impuesto sobre el dividendo se causa al momento de la percepción; el módulo fiscal usa la tasa de **apertura** del año gravable, que es la que corresponde a un ingreso causado durante el ejercicio:

```
tasa(t) = trm_inicial * (1 + devaluacion) ** (t - 1)   # t = 3 → 4410.00
```

Cumple exactamente los mismos ADs. Ninguna regla del spine prefiere una convención sobre la otra.

**Qué se rompe.** Con 5% de devaluación, un dividendo bruto de USD 1.000 queda registrado en el libro por COP 4.630.500 y gravado sobre una base de COP 4.410.000. La razón `impuesto / dividendo` que el memorando muestra ya no coincide con ninguna tarifa de la configuración. Un cliente con calculadora encuentra la inconsistencia en el primer minuto — y la trazabilidad exigida por el brief §2.2 la delata en el propio bloque de `inputs`, donde el auditor verá dos tasas distintas para el mismo año en dos asientos contiguos.

**Variante del mismo hueco, más cara.** Hay **dos tipos de tasa**, no uno: la tasa de mercado y la tasa efectiva con spread cambiario. `motor/friccion/` aplica el spread de compra en la entrada (brief §5.1) y el de venta en la salida (§5.5). ¿Y en la valoración del dividendo del año? Un implementador aplica el spread de venta porque "el dividendo hay que repatriarlo"; otro no, porque la valoración fiscal es a TRM oficial y el spread solo existe si hay transacción. Ambos cumplen todos los AD. La diferencia es de 30 a 100 puntos básicos sobre la porción del dividendo — exactamente la magnitud que el brief §1 declara *ser el producto*.

**Cierra con:** AD-16 (curva de cambio de dueño único) — `motor/escenarios/` construye una única `CurvaDeCambio` inmutable al inicio de la corrida, indexada por año fiscal, con convención de cierre de año declarada; ningún otro módulo deriva, compone ni desplaza tasas. La curva expone **dos métodos distintos y no intercambiables**: `tasa_valoracion(anio)` (sin spread — valoración y base gravable) y `tasa_transaccion(anio, sentido)` (con spread — solo entrada y salida). El spread se aplica exactamente dos veces por corrida y el motor lo verifica contando los asientos de concepto `spread_fx_entrada` y `spread_fx_salida`.

---

## D-3 — `motor/cumplimiento/` y `motor/fiscal/` leen el umbral de ganancia ocasional de fuentes distintas

**Severidad: CRÍTICA. El brief lo prohíbe explícitamente y el spine no lo impide.**

AD-7 declara `Binds: adaptadores/config/, motor/fiscal/, motor/escenarios/`. **`motor/cumplimiento/` no está en esa lista.** Y en el diagrama de invariantes, `CUM` tiene una sola flecha: `CUM --> DOM`. No hay ninguna ruta arquitectónica por la que el módulo de cumplimiento reciba configuración tributaria.

**Módulo A — `motor/fiscal/`.** Lee `umbral_ganancia_ocasional_anios` de la configuración fechada del escenario B, obtiene 4 (el proyecto de reforma eleva el umbral de 2 a 4 años, brief §6), y para un horizonte de 3 años clasifica la realización como **cédula general** a la tarifa marginal del contribuyente (39%). Cumple AD-7 al milímetro: parámetro de config, versión registrada en el resultado.

**Módulo B — `motor/cumplimiento/`.** Implementa la alerta del brief §7: *"¿El horizonte planeado deja la posición por debajo del umbral de ganancia ocasional en alguno de los tres escenarios?"*. Como no está bindeado por AD-7 y su única dependencia permitida es `motor/dominio/`, declara el umbral como constante de dominio en `motor/cumplimiento/umbrales.py`:

```python
UMBRAL_GANANCIA_OCASIONAL_ANIOS = 2   # art. 300 ET
```

Con horizonte 3 años → 3 ≥ 2 → **no levanta la bandera**. También cumple todos los AD: no importó nada prohibido (AD-1), no calculó impuestos (su mandato en el spine es *"levanta banderas, no calcula"*), no usó reloj ni entorno.

**Qué se rompe.** El memorando dice, en la misma página, que la posición tributa a 39% por no alcanzar el umbral de ganancia ocasional (motor fiscal, escenario B) y que **no hay alerta** por el umbral de ganancia ocasional (módulo de cumplimiento). El documento se contradice a sí mismo sobre el punto que el brief §6 eleva a *"requisito de honestidad del sistema"*. Es el peor modo de fallo posible del producto: un memorando internamente inconsistente, firmado, entregado a un cliente.

**El mismo hueco tiene un segundo brazo.** El umbral de US$60.000 de *estate tax* y el umbral de declaración del Formulario 160 (denominado en UVT) también son parámetros normativos que el brief §2.3 exige en configuración — y el spine no les asigna archivo. Si viven como constantes en `motor/cumplimiento/`, se viola el brief §2.3 directamente; si viven en `config/tributario/escenario-*.yaml`, ver D-9.

**Cierra con:** AD-17 (parámetro normativo único) — se amplía el `Binds` de AD-7 a `motor/cumplimiento/`; **todo** umbral, tarifa, UVT y plazo de origen normativo es campo de un objeto `ParametrosVigentes` validado; `motor/cumplimiento/` recibe **la misma instancia** que usó el paso fiscal y el libro registra un solo `config_version_id` por año. Ninguna constante numérica de origen normativo puede existir bajo `motor/` — el test de arquitectura de AD-1 se amplía para rechazar literales numéricos distintos de `0` y `1` en `motor/fiscal/` y `motor/cumplimiento/`.

---

## D-4 — El vocabulario de `concepto` está abierto: dos módulos nombran la misma fricción distinto

**Severidad: CRÍTICA.**

AD-8 define el asiento como `(anio, concepto, monto, formula, inputs)` y no dice **qué tipo es `concepto`**, **quién puede escribir cada uno**, ni **qué signo lleva `monto`**. Todo el desglose del brief §5.7 y el waterfall de §5 "Output visual requerido" se construyen agrupando por `concepto`.

**Módulo A — `motor/friccion/`.** Brief §5.3 pone la retención en origen **dentro** del paso anual de evolución. Así que `motor/friccion/` la calcula ahí mismo y escribe:

```python
Asiento(anio=2028, concepto="retencion_origen", monto=Money(Decimal("186.20"), USD), ...)
```

Signo **positivo**, porque el módulo trata los asientos como *magnitudes de fricción* — la barra del waterfall mide cuánto se perdió, y perder un número negativo confunde.

**Módulo B — `motor/fiscal/`.** El spine le asigna a `motor/fiscal/` *"retención en origen e impuesto colombiano"* (fila de la tabla de capas y comentario del árbol de directorios). Así que `motor/fiscal/`, que es el módulo que conoce el domicilio y la tarifa, también la escribe:

```python
Asiento(anio=2028, concepto="retencion_en_la_fuente_origen", monto=Money(Decimal("-186.20"), USD), ...)
```

Signo **negativo**, porque el módulo trata los asientos como *decrementos del saldo* — sumar el libro debe dar el neto, que es literalmente lo que ordena AD-8.

Ambos cumplen AD-8, AD-4, AD-1 y AD-5. El spine **le asigna la retención a los dos**: la tabla de capas se la da a `motor/fiscal/`, y el brief §5.3 la pone dentro del bucle anual que vive en `motor/friccion/`. No hay AD que resuelva el empate.

**Qué se rompe, en tres capas a la vez:**

1. **Doble conteo.** La retención aparece dos veces en el libro. El "retorno neto derivado sumando el libro" resta la retención dos veces.
2. **Waterfall con dos barras para una sola fricción.** El render agrupa por `concepto` y produce `retencion_origen` y `retencion_en_la_fuente_origen` como capas separadas. El gráfico que el brief §5 llama *"el activo comercial del producto"* muestra una fricción inventada.
3. **Signos mezclados.** Con `+186.20` de A y `-186.20` de B, la suma del libro es **cero** para ese concepto. El total sale limpio y equivocado — el modo de fallo silencioso.

Y nada de esto lo detecta un test unitario de `motor/fiscal/` ni uno de `motor/friccion/`: cada módulo pasa sus propios casos calculados a mano (brief §2.6). Solo un test de integración que nadie pidió lo vería.

**Cierra con:** AD-18 (vocabulario cerrado y dueño único por concepto) — `Concepto` es un `StrEnum` cerrado en `motor/dominio/`. Cada miembro declara, en una tabla del dominio: su **módulo dueño** (el único autorizado a escribir asientos con ese concepto), su **signo** (todo asiento de costo se escribe negativo, sin excepción), su **grupo del waterfall** y su **moneda de causación**. `LibroDeAsientos.append` rechaza en tiempo de ejecución un asiento cuyo concepto no pertenezca al módulo llamante, y rechaza un monto cuyo signo contradiga la declaración. Añadir un concepto es un cambio de dominio con test — no un string nuevo en un módulo.

---

## D-5 — El `Asiento` no lleva escenario: A, B y C escriben en el mismo libro

**Severidad: CRÍTICA.**

El spine dice que `motor/escenarios/` *"ejecuta todo cálculo en A, B y C"* y *"corre A, B y C sobre el mismo input"*. AD-8 dice que el libro es **append-only**. La tupla del asiento **no tiene campo de escenario**. Nada dice cuántos libros hay por corrida.

**Módulo A — `motor/escenarios/`.** Construye **un** `LibroDeAsientos` por corrida y lo pasa a las tres ejecuciones. Es la lectura natural de "el render lee el libro" (AD-8, singular) y de AD-10, que exige un único `Renderer` para todo el memorando. Cumple AD-8: solo hace `append`, nunca borra ni sobreescribe.

**Módulo B — `motor/friccion/`.** Recibe un libro y le agrega sus asientos. También cumple AD-8 sin objeción — es exactamente lo que el AD le manda hacer.

**Qué se rompe.** Los tres escenarios comparten el mismo bucle de evolución del portafolio (la fricción de mercado no depende del régimen tributario). El libro termina con **tres asientos `ter` para el año 2028**, idénticos, indistinguibles. El total derivado sumando el libro triplica el TER, la custodia, el spread y las comisiones. El waterfall muestra fricción de mercado 3× y fricción tributaria mezclada entre tres regímenes. Y el memorando no puede cumplir el criterio de aceptación 2 del brief ("cada comparación corre en los tres escenarios") porque no hay forma de separar las cifras.

**Corolario del mismo hueco: mutación de estado compartido entre escenarios.** El spine no exige inmutabilidad en ninguna parte. Las Consistency Conventions exigen determinismo (*"dos corridas con los mismos inputs producen bytes idénticos"*) — y una corrida que muta estado compartido entre A, B y C **sigue siendo determinista**: produce los mismos bytes equivocados siempre. Par concreto:

- `motor/escenarios/` memoiza la `EvolucionAnual` porque es pura y no depende del escenario. Optimización legítima, ningún AD la prohíbe.
- `motor/fiscal/` adjunta `impuesto_anual` sobre los objetos `EstadoAnual` que recibe, en vez de devolver objetos nuevos. Ningún AD lo prohíbe: no hizo E/S, no importó nada vetado, no registró logs.

Escenario A escribe su impuesto sobre los estados memoizados. Escenario B los sobreescribe. Escenario C los sobreescribe otra vez. Los tres reportes muestran las cifras de C. Determinista, reproducible, y equivocado.

**Cierra con:** AD-19 (aislamiento de escenarios por tipo) — `Asiento` lleva `escenario: Escenario` obligatorio; `LibroDeAsientos` se construye con su clave de identidad `(corrida_id, vehiculo_id, escenario)` y **rechaza** cualquier asiento cuya clave no coincida; el comparativo de tres escenarios es una colección de tres libros, no un libro con tres regímenes dentro. Complementariamente: todo tipo de `motor/dominio/` es `frozen=True` (pydantic `ConfigDict(frozen=True)`); el cálculo devuelve valores nuevos, nunca muta los recibidos.

---

## D-6 — Dos dueños del costo fiscal: ¿pre-spread o post-spread?

**Severidad: ALTA.**

Brief §5.1: *"COP → USD aplicando spread cambiario de compra. Registrar el tipo de cambio de entrada: **es la base de costo fiscal**."* Brief §5.6: *"Base gravable = valor de venta en COP − costo fiscal en COP."* El spine no nombra `CostoFiscal` en ninguna parte ni le asigna dueño.

**Módulo A — `motor/friccion/`.** Ejecuta el paso de entrada. Con COP 400.000.000, TRM 4.000 y spread de compra de 0,4%, compra USD 99.601,59 y registra `spread_fx_entrada` como fricción. Reporta el costo fiscal como el desembolso real del cliente: **COP 400.000.000**. Es lo que salió del bolsillo; es lo que un contador defendería.

**Módulo B — `motor/fiscal/`.** Toma la lectura literal del brief §5.1 — "el tipo de cambio de entrada es la base de costo fiscal" — y reconstruye: `USD 99.601,59 × 4.000 = COP 398.406.360`. Los USD efectivamente adquiridos, valorados a la TRM de entrada. También defendible: el spread es un costo de intermediación, no parte del costo del activo.

Ambos cumplen AD-4, AD-7 y AD-8. Ninguna regla del spine elige.

**Qué se rompe.** La base gravable difiere en COP 1.593.640. A tarifa de ganancia ocasional del 15%, son COP 239.046 de impuesto que aparecen o desaparecen según qué módulo ganó. Peor: **la descomposición del criterio de aceptación 4** ("qué porción del impuesto se debe a devaluación") se desplaza entera, porque la porción atribuible a devaluación se mide contra el costo fiscal. Y peor todavía: si `motor/friccion/` escribe el asiento del spread como fricción **y** `motor/fiscal/` lo excluye del costo fiscal, el spread queda contado dos veces — una como barra del waterfall, otra como mayor base gravable.

**Cierra con:** AD-20 (`CostoFiscal` es un tipo del dominio con un solo constructor) — `motor/fiscal/` lo calcula exactamente una vez, a partir de entradas declaradas, y lo escribe en el libro como asiento con `formula` e `inputs` completos. Ningún otro módulo lo reconstruye. La convención (`incluye_spread: bool`) es un parámetro **declarado en la configuración tributaria por escenario**, no una decisión de código — porque es precisamente el tipo de punto que un contador tributarista debe poder cambiar sin refactor (brief §2.3).

---

## D-7 — El escenario B no tiene vigencia para los años previos a 2027

**Severidad: ALTA. Bloquea el criterio de aceptación 2.**

AD-7 indexa la config por `(escenario, vigencia_desde)` y ordena: *"El cargador levanta `ParametroTributarioFaltante` ante cualquier parámetro ausente o marcado `TODO`. **Nunca existe un valor por defecto numérico.**"* El brief §6 dice que las medidas del escenario B *"aplicarían desde el año gravable 2027"*. El spine no dice si la config se resuelve **una vez por corrida** o **por año gravable**.

**Módulo A — `motor/fiscal/`.** Resuelve por año gravable, que es lo correcto para un horizonte de 20 años que cruza reformas. Para el año 2026 en escenario B no encuentra ninguna vigencia con `vigencia_desde <= 2026` y levanta `ParametroTributarioFaltante`, obedeciendo AD-7 literal y ruidosamente. **El escenario B queda inutilizable para cualquier corrida que empiece antes de 2027** — es decir, todas.

**Módulo B — `motor/escenarios/`.** Resuelve una sola vez al inicio con `anio_inicial`, o cae a la vigencia de escenario A para los años previos a la reforma (que es económicamente correcto: antes de 2027 rige el régimen actual). AD-7 prohíbe *"un valor por defecto numérico"* — pero heredar el conjunto de parámetros de otro escenario no es un valor por defecto numérico, así que el módulo no viola nada.

**Qué se rompe.** O el escenario B siempre truena y el criterio de aceptación 2 no se puede cumplir, o el escenario B hereda silenciosamente parámetros de A sin que el memorando registre de qué vigencia salió cada año — rompiendo AD-7 en su parte final (*"todo resultado calculado carga el identificador de la versión de config que usó"*, en singular, cuando en realidad hay una por año).

**Cierra con:** AD-21 (el escenario es una línea de tiempo completa y explícita) — cada escenario declara en YAML la **secuencia completa** de vigencias que cubre el horizonte, incluidas las pre-reforma; el escenario B declara explícitamente sus años pre-2027, no los hereda. La resolución es una función pura `(escenario, anio_gravable) -> ParametrosVigentes` invocada **por año gravable**, y el libro registra el `config_version_id` **por año**, no por corrida. Así el fallo ruidoso de AD-7 recupera su significado: si truena, es porque falta un parámetro de verdad.

---

## D-8 — La porción del impuesto atribuible a devaluación tiene dos definiciones legítimas

**Severidad: ALTA. Es el criterio de aceptación 4 del brief.**

El spine menciona la descomposición **una sola vez**, dentro de la enumeración de AD-8, como uno de los totales que se derivan sumando el libro. Pero **no es una suma**: es una atribución, y hay al menos dos métodos.

**Módulo A — `motor/fiscal/`, método contrafactual.** Recalcula todo el impuesto de realización con una curva de cambio plana (`trm_salida = trm_entrada`) y atribuye la diferencia a devaluación:

```
impuesto_por_devaluacion = impuesto_real − impuesto_con_trm_constante
```

**Módulo B — `adaptadores/render/`, método proporcional.** Descompone la ganancia y prorratea el impuesto:

```
ganancia_fx      = usd_invertido * (trm_salida − trm_entrada)
ganancia_total   = base_gravable_cop
impuesto_por_dev = impuesto_total * (ganancia_fx / ganancia_total)
```

Ambos leen del libro. Ambos son defendibles ante un tributarista. Ninguno viola AD-8, porque AD-8 prohíbe *recalcular totales*, y esta cifra no es un total del libro: no existe como asiento.

**Qué se rompe.** Los dos métodos coinciden solo si la tarifa es plana y única. En el momento en que la clasificación cruza el umbral de ganancia ocasional, o en que la cédula general aplica una tabla progresiva (brief §6 la lista como variable de escenario), divergen — y pueden divergir mucho: el método contrafactual captura el salto de tarifa completo, el proporcional lo diluye. En un caso donde la devaluación es lo único que empuja la ganancia por encima de un rango, el contrafactual atribuye a devaluación el 100% del salto y el proporcional una fracción.

El brief eleva esta cifra a criterio de aceptación y a *"punto crítico que el modelo debe hacer visible"*. Que dos rutas del sistema puedan producirla distinta, y que el render pueda producirla sin pasar por el motor, es un hueco de primer orden. Además, el método proporcional viviendo en `adaptadores/render/` es lógica financiera en un adaptador — otra fuga que el test de arquitectura de AD-1 no ve, porque no involucra ningún `import` prohibido.

**Cierra con:** AD-22 (toda cifra contrafactual es un asiento con método declarado) — la atribución a devaluación es una función nombrada de `motor/fiscal/`, de método **contrafactual** fijado en el spine, que escribe su resultado al libro como asiento de `naturaleza = CONTRAFACTUAL` con el supuesto contrafactual completo en `inputs`. Los totales reales suman solo asientos `REAL`; los contrafactuales nunca entran a una suma. `adaptadores/render/` no puede producir una cifra que no esté en el libro — y esto ahora sí es verificable: un test recorre los números del HTML renderizado y los busca en el libro, reutilizando el mismo guard numérico que AD-9 ya exige para la prosa del LLM.

---

## D-9 — Dos fuentes para la retención en origen: el catálogo y la config tributaria

**Severidad: ALTA.**

Brief §4: *"Cada vehículo del catálogo se define en un archivo de datos con: ticker, ISIN, domicilio, TER, dividend yield histórico, tipo de distribución, moneda, mercado de negociación, y bróker(s)"* — y la tabla de la misma sección lista *"Retención 30% sobre dividendos"* como la **fricción característica del tipo de vehículo**. Brief §2.3: *"Tarifas, UVT, umbrales y plazos viven en un archivo de configuración versionado con fecha de vigencia."* AD-3 mete ambos archivos en el mismo cajón ("texto YAML versionado en git"). AD-7 no dice cuál gana.

**Módulo A — `catalogo/vehiculos.yaml` + `motor/friccion/`.** El ETF estadounidense declara `retencion_origen: 0.30` como parte de su perfil de fricción, junto al TER y al dividend yield. `motor/friccion/`, que ejecuta el paso de retención dentro del bucle anual (brief §5.3), lo lee del vehículo que ya tiene en la mano.

**Módulo B — `config/tributario/escenario-a.yaml` + `motor/fiscal/`.** La retención es una tarifa de origen normativo con vigencia, así que vive en la config: `retenciones_por_domicilio: {US: 0.30, IE: 0.15}`. `motor/fiscal/` la resuelve por `(domicilio, tipo_ingreso, vigencia)`.

Ambos archivos son YAML en git (AD-3), ambos validados con pydantic en el borde (Consistency Conventions), ambos fallan ruidoso ante un `TODO` (AD-7). Ambos módulos cumplen todo.

**Qué se rompe.** Dos fuentes de verdad para el mismo número. Mientras coincidan, nadie lo nota. El día que el tributarista actualice la tabla de la config y no el catálogo (o al revés), el memorando aplica una tarifa en el bucle de fricción y otra en el paso fiscal — y AD-7 no se queja, porque ningún parámetro falta. La divergencia es invisible por construcción. Y el criterio de aceptación 7 ("cambiar una tarifa tributaria requiere editar un archivo de configuración") queda a medias: requiere editar *dos*, y nadie lo dice.

**Segundo brazo — la UVT triplicada.** Los umbrales del brief §7 (Formulario 160, impuesto al patrimonio) están denominados en UVT. Si la UVT vive dentro de cada `escenario-*.yaml`, tres archivos pueden llevar tres UVT distintas para el mismo año — y de hecho lo harán, porque quien edite `escenario-b.yaml` para la reforma de 2027 actualizará la UVT allí y no en `escenario-a.yaml`. Resultado: el mismo portafolio dispara la alerta de Formulario 160 en A y no en B, por una razón que **no tiene nada que ver con la reforma**. El memorando presenta como diferencia normativa lo que es un artefacto de configuración.

**Cierra con:** parte de AD-17 — separación estricta de responsabilidad entre archivos: el **catálogo declara hechos del instrumento**; la **config declara tarifas, umbrales y plazos normativos**. La retención en origen se resuelve exclusivamente por `(domicilio, tipo_ingreso, vigencia)` desde la config; el cargador rechaza el arranque si una clave normativa aparece en el catálogo. La UVT y los umbrales en USD viven en `config/tributario/base-comun.yaml` que los tres escenarios comparten, y cada `escenario-*.yaml` solo puede **sobreescribir** una lista declarada de claves.

---

## D-10 — El orden del paso anual no está fijado: sobre qué saldo se resta el TER

**Severidad: ALTA.**

El brief §5.3 enumera un orden (*"debe implementarse en este orden"*), pero el spine **no lo eleva a AD**. La única descripción es *"evolución año por año"* en la tabla de capas. Un implementador que trabaje desde el spine — que es lo que el spine dice ser: el sustrato del que se construye todo lo demás — no tiene la restricción.

**Módulo A — `motor/friccion/`, orden literal del brief.** Sobre el saldo corriente, en secuencia: apreciación → dividendo bruto → retención → reinversión del dividendo neto → TER → custodia. El TER se resta sobre el saldo **ya apreciado y ya con el dividendo reinvertido**.

**Módulo B — `motor/friccion/` reescrito por otro implementador, convención actuarial.** El TER se cobra sobre el patrimonio promedio, o sobre el saldo de apertura del año — que es la convención con la que se publica un TER y la única que reproduce el número del *factsheet* del fondo.

Ambos cumplen AD-6 (restan TER solo cuando la convención es `bruto_de_ter`), AD-4, AD-8. Ninguno viola nada.

**Qué se rompe.** Con 8% de retorno y 0,20% de TER, la diferencia anual es de ~1,6 bp; a 20 años compuestos, decenas de puntos básicos sobre el resultado final. El brief §1 fija la magnitud del producto en 200 bp. Un error de método que consume un porcentaje visible del valor que el producto pretende medir no es ruido.

**Y el brazo peligroso del mismo par:** si `motor/friccion/` calcula el dividendo bruto sobre el saldo de cierre y `motor/fiscal/` reconstruye la base del impuesto anual sobre el saldo de apertura (porque necesita el dividendo y no recibió el asiento, sino el estado), la retención en origen y el impuesto colombiano se calculan **sobre bases distintas para el mismo dividendo**. La traza de `inputs` muestra dos dividendos brutos distintos en el mismo año.

**Cierra con:** parte de AD-19 y una regla nueva dentro de AD-18 — el paso anual se declara como una tubería ordenada de fases nombradas (`FaseAnual` como enum) sobre un `EstadoAnual` inmutable; cada fase declara sobre qué base opera (`saldo_apertura` vs `saldo_corriente`), y el motor no permite reordenarlas. `motor/fiscal/` **nunca recalcula un monto que ya es un asiento**: recibe el dividendo bruto leyéndolo del libro por `(anio, Concepto.DIVIDENDO_BRUTO)`.

---

## D-11 — AD-6 hace que el waterfall mienta al comparar vehículos

**Severidad: MEDIA-ALTA. Este es un par entre un módulo y el objetivo comercial del producto.**

AD-6 dice: *"El motor resta TER únicamente cuando la convención es `bruto_de_ter`."*

**Módulo A — `motor/friccion/`, vehículo con `retorno_esperado_base: bruto_de_ter`.** Resta el TER y escribe el asiento. En el waterfall aparece una barra de TER de 42 bp.

**Módulo B — `motor/friccion/`, vehículo con `retorno_esperado_base: neto_de_ter`.** No resta nada, porque el TER ya viene descontado del NAV. Y por tanto **no escribe ningún asiento de TER**. Obediencia perfecta a AD-6.

**Qué se rompe.** El render agrupa por concepto (AD-8). El segundo vehículo aparece en el gráfico **sin barra de TER**. El brief §5.7 exige el desglose comparado de *"cuánto se perdió por TER"*, y el brief §5 llama al waterfall *"el activo comercial del producto: es lo que se le muestra al cliente"*. Un cliente no técnico, en los 30 segundos que exige el criterio de aceptación 3, concluye que el segundo fondo no cobra comisión de gestión. Es exactamente la conclusión falsa que el producto existe para prevenir.

**Cierra con:** parte de AD-15 y AD-22 — el `Asiento` lleva `naturaleza: Naturaleza` (`REAL | INFORMATIVO | CONTRAFACTUAL`). Con `neto_de_ter`, el motor escribe igualmente un asiento de TER de naturaleza `INFORMATIVO`, con el monto derivado del TER declarado del vehículo, para que el waterfall sea comparable entre vehículos. Los totales suman solo `REAL` — nunca hay riesgo de doble resta. El render muestra las barras informativas con una marca visual distinta y una nota al pie: "ya descontado del NAV".

---

## D-12 — Cuantizar "al presentar" no dice cuándo: las partes no suman al total

**Severidad: MEDIA.**

AD-4 cierra con: *"La cuantización a centavos ocurre solo al presentar, con `ROUND_HALF_UP`."* No define **presentar**, ni **quién**, ni **una sola vez**. Y `adaptadores/render/` tiene al menos tres consumidores del libro: el waterfall de matplotlib, la plantilla Jinja del memorando, y el JSON que se le entrega al LLM (AD-9).

**Módulo A — el waterfall.** Cuantiza cada barra al construirla: seis capas de fricción, cada una redondeada a la unidad de presentación. Cumple AD-4: cuantizó al presentar.

**Módulo B — la plantilla del memorando.** Cuantiza el total `retorno_neto` derivado del libro **sin cuantizar**. También cumple AD-4.

**Qué se rompe.** La suma de las seis barras mostradas no es igual al total mostrado. En COP a centavos el residuo es invisible; pero el brief §5.7 exige el desglose *"en COP **y en puntos básicos anualizados**"*, y en bp la unidad de presentación es 1 bp: seis capas redondeadas individualmente pueden desviarse hasta 3 bp del total, sobre un producto cuya tesis es que la diferencia total es de 200 bp. El cliente ve un waterfall que no cuadra — y en un documento cuya premisa es la auditabilidad (brief §2.2), un gráfico que no cuadra destruye la credibilidad más que un decimal de más.

**Segundo brazo:** la anualización en bp no está definida en ninguna parte. `((1+r)^(1/n) − 1)` vs `r/n`; denominador = monto inicial vs saldo promedio. Cuatro combinaciones, todas conformes a los ADs, todas produciendo un desglose distinto.

**Tercer brazo, y este sí rompe el build:** el guard numérico de AD-9 compara los tokens numéricos de la prosa del LLM contra "el JSON de origen". Si el JSON lleva montos sin cuantizar y la prosa cita las cifras cuantizadas que el LLM vio en el contexto, el guard **falla el render** por un desajuste de redondeo, no por una alucinación. AD-9, que es una de las invariantes mejor diseñadas del spine, se vuelve inoperante por un hueco de AD-4.

**Cierra con:** parte de AD-15 — la cuantización ocurre **exactamente una vez**, en un `Presentador` del dominio que recibe el libro completo y devuelve un `LibroPresentado` cuyas partes suman al total **por construcción** (reparto por mayor residuo; la última partida absorbe el remanente). El waterfall, el memorando y el JSON del adaptador LLM consumen el mismo `LibroPresentado`. La fórmula de anualización en bp se declara en el spine, una vez.

---

## D-13 — `formula` e `inputs` no tienen tipo: la traza es inconsumible

**Severidad: MEDIA.**

AD-8 exige `formula` e `inputs` en cada asiento y no les da tipo. Brief §2.2: *"Cada número del output debe poder descomponerse en sus inputs y en la fórmula aplicada. Si no se puede auditar, no se muestra."*

**Módulo A — `motor/friccion/`.** `formula="saldo_corriente * ter_anual"`, `inputs={"saldo_corriente": "412550.18", "ter_anual": "0.0020"}` — cadenas legibles, decimales serializados a texto.

**Módulo B — `motor/fiscal/`.** `formula=Formula.CEDULA_GENERAL_PROGRESIVA` (enum del dominio), `inputs=BaseCedulaGeneral(base=Money(...), tabla_id="...", rango_aplicado=3)` — un modelo pydantic anidado.

Ambos cumplen AD-8. El render, que por AD-8 *"lee el libro; no recalcula"*, tiene que mostrar la traza de los dos — y termina con dos rutas de renderizado, o descartando una. La consecuencia práctica es que la mitad de los asientos no se pueden auditar, y por la regla del brief §2.2, no se pueden mostrar.

**Cierra con:** parte de AD-15 — `formula: Formula` es un enum cerrado del dominio, cada miembro con su plantilla de renderizado; `inputs: Mapping[str, Money | Decimal | int | str]` es plano y homogéneo. Un test verifica que toda `Formula` tenga plantilla y que toda clave de `inputs` aparezca en ella.

---

## D-14 — La procedencia se cuela como asiento

**Severidad: MEDIA.**

AD-11 ordena: *"Toda corrida registra el id del snapshot que usó en el `LibroDeAsientos`."* AD-7 ordena: *"Todo resultado calculado carga el identificador de la versión de config que usó."* Y AD-8 define el asiento como una entrada monetaria de cinco campos, sin sitio para ninguno de los dos.

**Módulo A — `motor/escenarios/`, lectura literal de AD-11.** Cumple escribiendo un asiento de metadatos:

```python
Asiento(anio=0, concepto="snapshot_mercado", monto=Money(Decimal("0"), COP),
        formula="n/a", inputs={"snapshot_id": "sha256:9f3c…"})
```

**Módulo B — `motor/escenarios/` en otra implementación.** Cumple poniéndolo en un objeto de cabecera del libro, porque un asiento de monto cero no es un hecho contable.

**Qué se rompe.** Si gana A, el libro contiene pseudo-asientos que el waterfall tiene que filtrar por nombre — y el filtro es una lista de strings que nadie mantiene, el mismo hueco de D-4. El `anio=0` inventado contamina toda agrupación por año. Si ambos ocurren (uno cumpliendo AD-11, otro cumpliendo AD-7), la procedencia queda en dos sitios y puede contradecirse: el memorando cita una versión de config en el pie y otra en la traza.

**Cierra con:** parte de AD-15 — el libro lleva una cabecera `Procedencia(corrida_id, vehiculo_id, escenario, snapshot_id, config_version_por_anio, fecha_corrida)` separada de los asientos. Los asientos son **solo hechos monetarios**. AD-11 y AD-7 se reformulan para apuntar a la cabecera, no al cuerpo.

---

## D-15 — La alerta de cumplimiento: dos textos y una cifra fuera del libro

**Severidad: MEDIA.**

Brief §7: *"Cada alerta debe citar la norma o el concepto aplicable **como texto configurable**"*. AD-14: *"Toda cifra que muestre [`app/`] viene de un `LibroDeAsientos`."* AD-10 le da al `Renderer` los disclaimers, pero no las alertas.

**Módulo A — `motor/cumplimiento/`.** Devuelve `Alerta(codigo, severidad, cita_norma)` con el texto leído de la config, y la cifra de contraste ("exposición situs: USD 78.400 > USD 60.000") como campos `Money` del propio objeto `Alerta` — **no como asientos**, porque el módulo "levanta banderas, no calcula".

**Módulo B — `adaptadores/render/`.** Tiene un parcial Jinja con el copy de cada alerta escrito a mano, porque queda mejor maquetado y porque el `codigo` de la alerta basta para seleccionarlo.

**Qué se rompe.** Dos textos para una alerta: el configurable que exige el brief §7 queda muerto en una de las rutas de salida y nadie lo nota, porque la otra ruta se ve bien. Y la cifra de contraste viaja fuera del libro, contra AD-14 — con lo cual el guard numérico de AD-9 **fallará el render** si el LLM cita la exposición situs, porque ese número no está en el JSON de origen. Un requisito del brief §7 rompiendo un requisito del brief §8.

**Cierra con:** parte de AD-22 — `Alerta` es tipo del dominio con `codigo: CodigoAlerta` (enum cerrado); su texto normativo se resuelve por `codigo` desde la config, en un solo sitio; **toda cifra que la alerta muestre es un asiento del libro** (de naturaleza `INFORMATIVO`), y el JSON que recibe el adaptador LLM se construye desde el libro, no desde una estructura paralela.

---

## Tabla resumen

| # | Par de divergencia | Severidad | AD que lo cierra |
|---|---|---|---|
| D-1 | El libro mezcla COP y USD; AD-4 prohíbe la suma que AD-8 exige | Crítica | AD-15 asiento bimonetario |
| D-2 | Dos dueños de la tasa de cambio del año *t*; tasa con/sin spread indistinguibles | Crítica | AD-16 curva de cambio de dueño único |
| D-3 | `motor/cumplimiento/` no está bindeado por AD-7 y hardcodea umbrales | Crítica | AD-17 parámetro normativo único |
| D-4 | `concepto` es un string abierto, sin dueño ni signo declarado | Crítica | AD-18 vocabulario cerrado y dueño único |
| D-5 | El asiento no lleva escenario; A/B/C escriben en un libro y mutan estado compartido | Crítica | AD-19 aislamiento por tipo + inmutabilidad |
| D-6 | Dos dueños del costo fiscal (pre-spread vs post-spread) | Alta | AD-20 `CostoFiscal` de constructor único |
| D-7 | El escenario B no tiene vigencia antes de 2027: truena o hereda en silencio | Alta | AD-21 escenario = línea de tiempo completa |
| D-8 | La atribución a devaluación tiene dos métodos legítimos y ningún dueño | Alta | AD-22 contrafactual con método declarado |
| D-9 | Retención en origen (y UVT) en el catálogo y en la config a la vez | Alta | AD-17 (separación catálogo/config/base común) |
| D-10 | El orden del paso anual no está fijado; TER y dividendo sobre bases distintas | Alta | AD-18/AD-19 (tubería de fases nombradas) |
| D-11 | Con `neto_de_ter` no hay barra de TER: el waterfall miente al comparar | Media-alta | AD-15 (`naturaleza = INFORMATIVO`) |
| D-12 | "Cuantizar al presentar" no dice cuándo ni cuántas veces; las partes no suman y el guard de AD-9 se rompe | Media | AD-15 (`Presentador` único) |
| D-13 | `formula` e `inputs` sin tipo: la traza es inconsumible | Media | AD-15 (tipos cerrados) |
| D-14 | La procedencia se cuela como pseudo-asiento | Media | AD-15 (cabecera `Procedencia`) |
| D-15 | Alertas con dos textos y cifras fuera del libro | Media | AD-22 (`Alerta` tipada, cifras al libro) |

**Concentración del daño:** doce de los quince pares nacen del mismo sitio — el contrato del `Asiento` y del `LibroDeAsientos`. AD-8 declaró el artefacto correcto y le dejó la forma abierta. Cerrar esa forma cierra casi toda la superficie de divergencia del sistema.

---

## Los ocho AD que cierran los huecos

Redactados en el formato del spine, listos para pegar.

### AD-15 — El asiento es bimonetario, tipado y con procedencia en la cabecera

- **Binds:** `motor/dominio/`, todo módulo que escriba o lea el libro
- **Prevents:** que el "retorno neto en COP" sea imposible de derivar (AD-4 contra AD-8), que dos módulos produzcan trazas de formas distintas, y que las partes mostradas no sumen al total mostrado
- **Rule:** el `Asiento` es un modelo congelado con campos obligatorios: `escenario`, `anio`, `concepto: Concepto`, `naturaleza: Naturaleza`, `monto_origen: Money`, `monto_cop: Money`, `tasa_aplicada: Decimal`, `anio_tasa: int`, `formula: Formula`, `inputs: Mapping[str, Money | Decimal | int | str]`. La conversión a COP ocurre al **escribir** el asiento, dentro del núcleo, con la tasa de la `CurvaDeCambio` (AD-16). `Naturaleza` es `REAL | INFORMATIVO | CONTRAFACTUAL`; **todo total suma solo asientos `REAL`**. El libro lleva una cabecera `Procedencia` con `corrida_id`, `vehiculo_id`, `escenario`, `snapshot_id`, `config_version_por_anio` y `fecha_corrida`: la metadata **nunca** viaja como asiento. La cuantización ocurre exactamente una vez, en un `Presentador` que recibe el libro y devuelve un `LibroPresentado` cuyas partes suman al total por construcción (reparto por mayor residuo). El waterfall, el memorando y el JSON del adaptador LLM consumen ese mismo `LibroPresentado`. La anualización en puntos básicos es `((1 + r) ** (1 / n) − 1) * 10000` sobre el monto inicial en COP, y no admite otra forma.

### AD-16 — Una sola curva de cambio, con valoración y transacción separadas

- **Binds:** `motor/escenarios/`, `motor/friccion/`, `motor/fiscal/`
- **Prevents:** que dos módulos apliquen tasas distintas al mismo año, y que el spread cambiario se aplique en una valoración o se cuente dos veces
- **Rule:** `motor/escenarios/` construye al inicio de la corrida una única `CurvaDeCambio` inmutable, indexada por año fiscal, con **convención de cierre de año** declarada: `tasa(t) = trm_inicial * (1 + devaluacion) ** t`. Ningún otro módulo deriva, compone ni desplaza tasas. La curva expone dos métodos no intercambiables: `tasa_valoracion(anio)` —sin spread, para valoración y base gravable— y `tasa_transaccion(anio, sentido)` —con spread, exclusivamente para la conversión de entrada y la de salida—. El motor verifica al cerrar el libro que existan exactamente un asiento `SPREAD_FX_ENTRADA` y uno `SPREAD_FX_SALIDA` por corrida.

### AD-17 — Un parámetro normativo, un archivo dueño, un objeto vigente

- **Binds:** `adaptadores/config/`, `motor/fiscal/`, `motor/escenarios/`, **`motor/cumplimiento/`**, `catalogo/`
- **Prevents:** que el memorando se contradiga a sí mismo entre el impuesto calculado y la alerta levantada, y que la misma tarifa viva en dos archivos
- **Rule:** se amplía el `Binds` de AD-7 a `motor/cumplimiento/`. **Todo** umbral, tarifa, UVT y plazo de origen normativo —incluidos el umbral de *estate tax*, el del Formulario 160 y el de ganancia ocasional— es campo de un objeto `ParametrosVigentes` validado. `motor/cumplimiento/` recibe **la misma instancia** que usó el paso fiscal para ese año gravable. Ninguna constante numérica de origen normativo puede existir bajo `motor/`: el test de arquitectura de AD-1 se amplía para rechazar literales numéricos distintos de `0` y `1` en `motor/fiscal/` y `motor/cumplimiento/`. El **catálogo declara hechos del instrumento** (ISIN, domicilio, moneda, TER, dividend yield, convención de TER, `evento_ingreso_anual`); la **config declara tarifas, umbrales y plazos**. La retención en origen se resuelve exclusivamente por `(domicilio, tipo_ingreso, vigencia)` desde la config, nunca desde el catálogo, y el cargador rechaza el arranque si una clave normativa aparece en el catálogo. La UVT y los umbrales en USD viven en `config/tributario/base-comun.yaml`; cada `escenario-*.yaml` solo puede sobreescribir una lista declarada de claves.

### AD-18 — `Concepto` es un enum cerrado con dueño, signo y grupo

- **Binds:** `motor/dominio/`, `motor/friccion/`, `motor/fiscal/`, `motor/cumplimiento/`, `adaptadores/render/`
- **Prevents:** que la misma fricción se registre dos veces con nombres distintos, que dos módulos usen signos opuestos y el total salga limpio y equivocado, y que el waterfall muestre una capa inventada
- **Rule:** `Concepto` es un `StrEnum` cerrado en `motor/dominio/`. Una tabla del dominio declara, por cada miembro: **módulo dueño** (el único autorizado a escribir ese concepto), **signo** (todo costo se escribe negativo, sin excepción), **grupo del waterfall** y **moneda de causación**. `LibroDeAsientos.append` rechaza en tiempo de ejecución un asiento cuyo concepto no pertenezca al módulo llamante o cuyo signo contradiga la declaración. La retención en origen pertenece a `motor/fiscal/`; `motor/friccion/` la obtiene invocándolo, no recalculándola. Añadir un concepto es un cambio de `motor/dominio/` con test — nunca un string nuevo en un módulo.

### AD-19 — Un libro por `(corrida, vehículo, escenario)`; el dominio es inmutable

- **Binds:** `motor/escenarios/`, `motor/dominio/`, todo el núcleo
- **Prevents:** que los tres escenarios se sumen entre sí en un libro compartido, y que un escenario mute estado que otro va a leer
- **Rule:** `LibroDeAsientos` se construye con su clave de identidad `(corrida_id, vehiculo_id, escenario)` y **rechaza** todo asiento cuya clave no coincida. El comparativo de tres escenarios es una colección de tres libros, nunca un libro con tres regímenes dentro. Todo tipo de `motor/dominio/` es `frozen=True`; el cálculo devuelve valores nuevos y jamás muta los que recibe. El paso anual se declara como una tubería ordenada de fases nombradas (`FaseAnual` como enum, en el orden del brief §5.3), cada una con su base declarada (`saldo_apertura` o `saldo_corriente`); el motor no permite reordenarlas. Ningún módulo recalcula un monto que ya es un asiento: lo lee del libro por `(anio, concepto)`.

### AD-20 — `CostoFiscal` tiene un solo constructor y su convención es configuración

- **Binds:** `motor/fiscal/`, `motor/friccion/`, `config/tributario/`
- **Prevents:** que la base gravable —y con ella la porción atribuible a devaluación— dependa de qué módulo la reconstruyó, y que el spread de entrada se cuente como fricción y como mayor base gravable a la vez
- **Rule:** `CostoFiscal` es un tipo del dominio construido exactamente una vez, en `motor/fiscal/`, a partir de entradas declaradas, y escrito al libro con `formula` e `inputs` completos. Ningún otro módulo lo reconstruye ni lo deriva. Si incluye o no el spread cambiario de entrada es un parámetro declarado en la configuración tributaria por escenario (`costo_fiscal_incluye_spread: bool`), no una decisión de código.

### AD-21 — Un escenario es una línea de tiempo completa y explícita

- **Binds:** `adaptadores/config/`, `motor/fiscal/`, `motor/escenarios/`
- **Prevents:** que el escenario B sea inutilizable para todo horizonte que empiece antes de 2027, o que herede parámetros de A sin dejar rastro
- **Rule:** cada escenario declara en YAML la **secuencia completa** de vigencias que cubre el horizonte, incluidas las anteriores a la reforma; el escenario B declara explícitamente sus años pre-2027, no los hereda de A. La resolución es una función pura `(escenario, anio_gravable) -> ParametrosVigentes`, invocada **por año gravable**, nunca una sola vez por corrida. La cabecera `Procedencia` registra el `config_version_id` **por año**. Así el fallo ruidoso de AD-7 recupera su significado: si truena, es porque falta un parámetro de verdad.

### AD-22 — Toda cifra contrafactual y toda alerta nacen del libro, con método declarado

- **Binds:** `motor/fiscal/`, `motor/cumplimiento/`, `adaptadores/render/`, `adaptadores/llm/`
- **Prevents:** que la porción del impuesto atribuible a devaluación —criterio de aceptación 4 del brief— tenga dos definiciones, y que la lógica financiera se filtre a los adaptadores sin que el test de AD-1 lo vea
- **Rule:** la atribución del impuesto a devaluación es una función nombrada de `motor/fiscal/`, de método **contrafactual** (`impuesto_real − impuesto_recalculado_con_curva_de_cambio_plana`), que escribe su resultado al libro como asiento de `naturaleza = CONTRAFACTUAL` con el supuesto completo en `inputs`. Los asientos contrafactuales nunca entran a una suma de totales. `Alerta` es un tipo del dominio con `codigo: CodigoAlerta` (enum cerrado); su texto normativo se resuelve por código desde la config, en un solo sitio, y **toda cifra que la alerta muestre es un asiento del libro** de naturaleza `INFORMATIVO`. `adaptadores/render/` no produce ninguna cifra: un test recorre los números del HTML renderizado y los busca en el `LibroPresentado`, reutilizando el mismo guard numérico que AD-9 exige para la prosa del LLM.

---

## Lo que sí quedó blindado

Para calibrar el veredicto: hay tres zonas donde intenté construir un par de divergencia y **no pude**.

- **AD-1 + el test de arquitectura.** Dos módulos del núcleo no pueden divergir sobre si tienen E/S: el test recorre los `import` y falla el build. Es una invariante con dientes, no una intención. (Su límite queda expuesto en D-2 y D-8: no detecta lógica financiera que se filtra a un adaptador sin importar nada prohibido — por eso AD-22 le agrega el segundo diente.)
- **AD-9.** El guard numérico determinístico sobre la prosa del LLM es la respuesta correcta a un requisito que muchos diseños dejan en el prompt. No hay dos formas de implementarlo que produzcan resultados incompatibles — aunque D-12 muestra que un hueco de AD-4 lo puede volver inoperante desde afuera.
- **AD-10 + AD-12.** Un solo `Renderer`, un solo intermedio HTML. Cerrado: no hay ruta de salida que pueda nacer sin disclaimers, y no hay dos motores de PDF que puedan producir dos maquetaciones.

El problema del spine no es la altitud ni el paradigma — ambos son correctos para este MVP. El problema es que su artefacto central, el `LibroDeAsientos`, quedó **declarado y no especificado**, y todo lo que se construya sobre él va a divergir. Los ocho ADs propuestos no agregan infraestructura ni cambian el despliegue: son tipos más precisos, enums cerrados y reglas de dueño único. Caben en la misma arquitectura local, monolítica y de un solo proceso que el brief exige.
