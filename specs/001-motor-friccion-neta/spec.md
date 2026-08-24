# Feature Specification: Motor de Fricción Neta

**Feature Branch**: `001-motor-friccion-neta`

**Created**: 2026-08-24

**Status**: Draft

**Input**: Brief `MVP_MOTOR_FRICCION_BRIEF.md` §3–§10, más los cambios estructurales de contabilidad por lotes y reajuste fiscal aportados por el operador el 2026-08-24.

---

## Por qué existe esto

Todo el mundo compara vehículos de inversión por su **retorno bruto**. Nadie los compara por su **retorno neto real en COP** después de comisión de gestión, retención en la fuente en el país de origen, comisiones de bróker y custodia, spread cambiario de entrada y salida, e impuesto de renta colombiano al momento de realizar.

Esa diferencia puede superar los **200 puntos básicos anuales**. Ese es el producto.

El destinatario es un residente fiscal colombiano, y el emisor es un ingeniero financiero que produce memorandos de inversión defendibles para él. La herramienta es de **análisis**, no de asesoría: produce comparativos y escenarios, y nunca emite un "compre X".

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Comparar vehículos por retorno neto real (Priority: P1)

El operador carga un monto, un horizonte, un perfil fiscal y un conjunto de vehículos del catálogo. El sistema calcula el retorno neto en COP de cada uno, descomponiendo toda la fricción, y los presenta comparados.

**Why this priority**: es el producto. Sin esto no hay nada. Todo lo demás —memorando, alertas, narrativa— explica o envuelve este cálculo.

**Independent Test**: se prueba cargando 15 vehículos con supuestos conocidos y verificando que el retorno neto, el desglose de fricción y la TIR neta de cada uno coinciden con casos calculados a mano.

**Acceptance Scenarios**:

1. **Given** un catálogo con 15 vehículos, un monto en COP, un horizonte en años y un perfil fiscal, **When** el operador ejecuta la comparación, **Then** el sistema devuelve para cada vehículo el retorno bruto acumulado, el retorno neto acumulado en COP, el desglose de fricción en COP y en puntos básicos anualizados, y la TIR neta anualizada.
2. **Given** una comparación ejecutada, **When** el operador inspecciona cualquier cifra del resultado, **Then** puede descomponerla hasta los inputs y la fórmula que la produjeron.
3. **Given** un vehículo cuyo retorno esperado se declara neto de TER, **When** corre la evolución anual, **Then** el sistema **no** resta TER por segunda vez.
4. **Given** un CDT o un TES, **When** corre la evolución anual, **Then** el sistema causa impuesto anual sobre rendimientos financieros aunque el vehículo no distribuya dividendos.
5. **Given** cualquier corrida, **When** se emite el resultado, **Then** ningún parámetro tributario aparece sin su estado de procedencia.
6. **Given** un vehículo distributivo, **When** se corre en modo `acumular_caja`, **Then** el sistema **no** aplica spread cambiario al dividendo hasta la salida; **When** se corre en modo `repatriar`, **Then** lo aplica cada año; **When** se corre en modo `reinvertir`, **Then** lo aplica en cada reinversión.

---

### User Story 2 — Ver el impuesto que se paga por devaluación, no por rentabilidad (Priority: P1)

La base gravable se calcula en pesos. Si el COP se devalúa, se genera ganancia gravable en Colombia aunque el retorno en dólares haya sido cero o negativo. El operador necesita mostrarle al cliente **cuánto del impuesto corresponde a devaluación y no a rentabilidad real**.

**Why this priority**: es el argumento que distingue este análisis de cualquier comparador de retornos. Es P1 junto con la historia 1 porque sin él el producto no dice nada que el cliente no pueda encontrar gratis.

**Independent Test**: se prueba con un caso donde el retorno en USD es exactamente cero y la devaluación del COP es positiva; el sistema debe reportar impuesto pagado mayor que cero y atribuir el 100% a devaluación.

**Acceptance Scenarios**:

1. **Given** una posición con retorno cero en moneda de origen y devaluación positiva del COP, **When** se liquida la venta, **Then** el sistema reporta una base gravable positiva y atribuye la totalidad del impuesto a devaluación.
2. **Given** cualquier liquidación, **When** se emite el resultado, **Then** la porción del impuesto atribuible a devaluación se calcula por un único método declarado, y el resultado dice cuál fue.

---

### User Story 3 — Ver el costo oculto de la fragmentación por lotes (Priority: P1)

Cada reinversión de dividendo crea un lote nuevo con su propio reloj de tenencia. Al vender, la posición se fragmenta: unos lotes superan el umbral de ganancia ocasional y otros no, y estos últimos tributan a la tarifa marginal de cédula general. **Es un costo del vehículo distributivo que hoy nadie mide.**

**Why this priority**: sin contabilidad por lotes, el sistema calcula sobre la posición agregada y **subestima sistemáticamente el impuesto del vehículo distributivo** — justo la comparación distributivo vs. acumulativo que es el eje del catálogo. No es un refinamiento posterior: hacerlo después obligaría a rehacer el motor fiscal.

**Independent Test**: se prueba con un vehículo distributivo que reinvierte dividendos durante un horizonte apenas superior al umbral de tenencia; el sistema debe reportar que una fracción no nula de la ganancia quedó gravada a cédula general.

**Acceptance Scenarios**:

1. **Given** una posición con lotes adquiridos en años distintos, **When** se liquida la venta, **Then** el impuesto se calcula lote por lote y se suma, nunca sobre la posición agregada.
2. **Given** una venta parcial, **When** se asignan los lotes, **Then** se usa el método declarado en la corrida (FIFO por defecto, identificación específica como opción) y el resultado dice cuál se usó.
3. **Given** cualquier liquidación, **When** se emite el resultado, **Then** el sistema reporta qué porcentaje de la ganancia quedó gravada a tarifa de cédula general por fragmentación de lotes.
4. **Given** un lote reconocido en un año, **When** transcurren años posteriores, **Then** su base de costo en COP **no** se reexpresa: la TRM de reconocimiento queda fija.

---

### User Story 4 — Correr todo en las nueve celdas y ver dónde se invierte el resultado (Priority: P1)

La normativa está en movimiento y el tratamiento del reajuste fiscal admite tres modos excluyentes. Toda comparación corre en **3 escenarios normativos × 3 modos de reajuste = 9 celdas**. Si un vehículo gana en una celda y pierde en otra, el memorando lo dice en la primera página.

**Why this priority**: es el "requisito de honestidad" del sistema. Presentar una sola celda permite elegir la que favorece la conclusión deseada, y eso destruye la defendibilidad del memorando.

**Independent Test**: se prueba ejecutando una comparación y verificando que devuelve las nueve celdas, que ninguna se omite, y que las no disponibles vienen con su razón.

**Acceptance Scenarios**:

1. **Given** cualquier comparación, **When** se ejecuta, **Then** devuelve resultados para las nueve celdas.
2. **Given** un vehículo con elegibilidad art. 73 `sin_clasificar`, **When** se solicita el modo `art_73`, **Then** el sistema **falla con mensaje explícito** y **nunca** cae en silencio a `art_70`.
3. **Given** un perfil de cliente con activo fijo en falso, **When** se ejecuta la comparación, **Then** los modos `art_70` y `art_73` se emiten como no disponibles con su razón, y solo `sin_reajuste` produce cifras.
4. **Given** una ruta de cálculo que intente componer el ajuste del art. 70 con el del art. 73 sobre el mismo activo para la misma enajenación, **When** se ejecuta, **Then** el sistema levanta una excepción.
5. **Given** un vehículo que gana en una celda y pierde en otra, **When** se genera el memorando, **Then** la divergencia aparece en la primera página.
6. **Given** una celda de referencia designada por el operador, **When** se presenta el resultado, **Then** el cliente ve la cascada de esa celda y las otras ocho como banda de sensibilidad, y las nueve siguen disponibles.

---

### User Story 5 — Recibir alertas de cumplimiento (Priority: P2)

Sobre cada celda simulada, el sistema verifica automáticamente umbrales que disparan obligaciones formales. **Levanta banderas; no calcula impuestos.**

**Why this priority**: alto valor para el cliente y bajo costo de construcción una vez existe el motor, pero el producto es defendible sin él. Es P2, no P1.

**Independent Test**: se prueba con posiciones construidas para cruzar cada umbral y verificando que cada bandera se levanta con su texto normativo y su advertencia de validación profesional.

**Acceptance Scenarios**:

1. **Given** una posición cuyo total de activos en el exterior supera el umbral de declaración obligatoria, **When** se ejecuta, **Then** se levanta la alerta de Formulario 160.
2. **Given** una posición en activos con *situs* estadounidense superior a US$60.000, **When** se ejecuta, **Then** se levanta la alerta de exposición a *estate tax*.
3. **Given** un horizonte que deja la posición bajo el umbral de ganancia ocasional en alguna celda, **When** se ejecuta, **Then** se levanta la alerta indicando en cuáles.
4. **Given** cualquier alerta levantada, **When** se emite, **Then** cita la norma o concepto aplicable desde texto configurable e incluye la advertencia de que requiere validación profesional.
5. **Given** una alerta cuyo texto normativo no está configurado, **When** se emite, **Then** el sistema falla en vez de mostrar un identificador crudo.

---

### User Story 6 — Generar el memorando con su gráfico de cascada (Priority: P2)

El operador genera un memorando profesional en español, con el gráfico de cascada que va del retorno bruto al neto mostrando cada capa de fricción como una barra, y lo exporta a PDF con sus disclaimers.

**Why this priority**: es el entregable comercial, pero depende de que las historias 1 a 4 estén completas. El gráfico de cascada es el activo que se le muestra al cliente.

**Independent Test**: se prueba generando un memorando desde un conjunto de resultados conocido y verificando el contenido, los disclaimers, la presencia de la sección de abogado del diablo y el tiempo total.

**Acceptance Scenarios**:

1. **Given** un conjunto de resultados calculados, **When** se genera el memorando, **Then** toda cifra de la prosa existe en los resultados del motor; si alguna no existe, **no se emite memorando**.
2. **Given** un memorando generado, **When** se inspecciona, **Then** contiene la sección de abogado del diablo; si falta, **no se emite**.
3. **Given** cualquier artefacto de salida, **When** se emite, **Then** incluye de forma visible y no descartable: que es análisis informativo y no recomendación ni asesoría en los términos de la regulación del mercado de valores colombiano; que el emisor no es asesor inscrito en el RNPMV ni entidad vigilada por la Superintendencia Financiera de Colombia; que los cálculos tributarios son estimaciones que requieren validación por contador o abogado tributarista; y la fecha de vigencia y el estado de procedencia de los parámetros usados.
4. **Given** una prosa generada que contiene lenguaje recomendatorio, una estimación de retorno futuro no presente en los resultados, o interpretación normativa fuera de los textos configurados, **When** se valida, **Then** **no se emite memorando**.
5. **Given** un gráfico de cascada emitido, **When** lo ve un cliente no técnico, **Then** identifica en 30 segundos cuánto se pierde por cada capa de fricción.

---

### User Story 7 — Emitir el valor patrimonial anual (Priority: P3)

Adoptar el reajuste del art. 73 obliga a declarar el costo ajustado como valor patrimonial en **cada** declaración anual. Es una obligación recurrente, no un cálculo puntual de la venta.

**Why this priority**: necesario para que el modo `art_73` sea utilizable en la práctica, pero no bloquea la comparación ni el memorando. Es la última pieza del alcance.

**Independent Test**: se prueba solicitando el valor patrimonial de una posición para cada año del horizonte y verificando que coincide con el costo ajustado del modo elegido.

**Acceptance Scenarios**:

1. **Given** una posición con varios lotes y un modo de reajuste, **When** se solicita el valor patrimonial, **Then** el sistema lo emite por lote y por año gravable.

---

### Edge Cases

- **Venta parcial que parte un lote.** El lote es inmutable: ¿qué representa el remanente? El sistema debe producir un resultado determinístico y trazable, sin mutar el lote original.
- **Año simulado sin configuración vigente en algún escenario.** El escenario B aplica desde el año gravable 2027; una simulación que empiece antes no tiene configuración B vigente. El sistema falla explícitamente en vez de heredar de otro escenario.
- **Serie de reajuste con un año faltante.** No se interpola, ni se extrapola, ni se hereda del año vecino: falla.
- **Retorno negativo en moneda de origen con devaluación positiva.** Debe producir base gravable positiva y atribución del 100% del impuesto a devaluación.
- **Horizonte exactamente igual al umbral de tenencia.** La clasificación entre ganancia ocasional y cédula general en el borde debe ser determinística y estar documentada en un test.
- **Vehículo cuya convención de TER no está declarada.** Falla; no asume ninguna de las dos.
- **Cantidad fraccionaria de participaciones por reinversión de dividendos.** Debe tener regla de precisión declarada, y el residuo del redondeo debe tener destino explícito.
- **Modo `repatriar` con dividendo pequeño.** El spread cambiario anual puede superar el dividendo neto. El sistema debe producir el resultado real (negativo) y no truncarlo a cero.
- **Cambio de la celda de referencia.** Designar otra celda no debe alterar ninguna cifra calculada, solo qué se muestra primero.
- **Todos los vehículos con elegibilidad `sin_clasificar`.** La comparación devuelve las nueve celdas con seis marcadas no disponibles y su razón; no falla la corrida completa.

---

## Requirements *(mandatory)*

### Catálogo y perfil

- **FR-001**: El sistema MUST soportar un catálogo de 10 a 15 vehículos cargados manualmente. NO existe descubrimiento automático de universo.
- **FR-002**: El sistema MUST modelar 8 tipos de vehículo, cada uno con su perfil de fricción propio: ETF de renta variable de EE.UU.; ETF UCITS distributivo de Irlanda; ETF UCITS acumulativo de Irlanda; acción individual local; FIC; CDT o renta fija local; TES; y Mercado Global Colombiano.
- **FR-003**: Cada vehículo MUST declarar: `ticker`, `isin`, `nombre_legal_completo`, `domicilio`, `forma_juridica_emisor` (`icav | plc | unit_trust | fondo_contractual | partnership | sociedad | otro`), TER, dividend yield histórico, tipo de distribución, `evento_ingreso_anual` (`dividendo | rendimiento_financiero | ninguno_hasta_realizar`), `retorno_esperado_base` (`bruto_de_ter | neto_de_ter`), moneda, mercado de negociación, brókers accesibles desde Colombia, `fuente_documental` y `elegibilidad_art_73` (`defendible | no_aplica | sin_clasificar`). Todos obligatorios, sin valor por defecto.
- **FR-004**: El perfil del cliente MUST declarar `activo_fijo` y la tarifa marginal en cédula general. El sistema MUST NOT inferir `activo_fijo` a partir del vehículo, del horizonte ni de la frecuencia de operación.

### Motor de fricción

- **FR-005**: El sistema MUST convertir el monto inicial de COP a la moneda de destino aplicando el spread cambiario de compra, y MUST registrar el tipo de cambio de entrada.
- **FR-006**: El sistema MUST restar la comisión de compra del bróker al entrar.
- **FR-007**: El sistema MUST ejecutar la evolución **año por año, inspeccionable**, sin fórmula cerrada, en este orden exacto: apreciación de capital (retorno total esperado − dividend yield) → dividendo bruto → retención en origen según domicilio → destino del dividendo neto → TER → custodia del bróker.
- **FR-008**: El sistema MUST restar TER **únicamente** cuando el vehículo declara `retorno_esperado_base: bruto_de_ter`.
- **FR-009**: El sistema MUST causar el impuesto anual según el `evento_ingreso_anual` declarado por el vehículo, y MUST NOT derivarlo de si el vehículo distribuye.
- **FR-010**: Al salir, el sistema MUST convertir el valor final a COP, restar la comisión de venta y aplicar el spread cambiario de salida.
- **FR-011**: El sistema MUST reportar por separado: retorno bruto acumulado, retorno neto acumulado en COP, desglose de fricción (TER, retención en origen, comisiones, spread FX, impuesto colombiano) en COP y en puntos básicos anualizados, y TIR neta anualizada.

### Contabilidad por lotes

- **FR-012**: Cada adquisición MUST constituir un lote independiente con fecha, cantidad, costo en moneda de origen, TRM de reconocimiento y su propio reloj de tenencia.
- **FR-013**: Cada reinversión de dividendo MUST crear un **lote nuevo**. Ninguna operación incrementa la cantidad de un lote existente.
- **FR-014**: La base de costo en COP de un lote MUST fijarse en el reconocimiento inicial y MUST NOT reexpresarse anualmente.
- **FR-015**: El impuesto de realización MUST calcularse **lote por lote y sumarse**. Ninguna ruta lo calcula sobre la posición agregada.
- **FR-016**: El método de asignación de lotes en venta MUST ser parámetro de la corrida: FIFO por defecto, identificación específica como opción. El resultado MUST declarar cuál se usó.
- **FR-017**: El sistema MUST reportar qué porcentaje de la ganancia quedó gravada a tarifa de cédula general por fragmentación de lotes.

### Costo fiscal y reajuste

- **FR-018**: El costo fiscal MUST calcularse en este orden exacto: (1) costo de adquisición en moneda de origen + comisiones de compra; (2) conversión a COP con la TRM de reconocimiento del lote; (3) aplicación del modo de reajuste sobre esa base en COP; (4) al vender, precio de venta convertido a COP − costo fiscal ajustado = base gravable; (5) clasificación por tenencia **del lote**.
- **FR-019**: El sistema MUST soportar tres modos de reajuste: `sin_reajuste`, `art_70` (porcentaje por año gravable) y `art_73` (factor por año de adquisición), y MUST correrlos los tres siempre.
- **FR-020**: Los modos `art_70` y `art_73` MUST ser mutuamente excluyentes sobre el mismo activo para la misma enajenación. Cualquier ruta que intente componer ambos ajustes MUST levantar excepción.
- **FR-021**: Si `elegibilidad_art_73` es `sin_clasificar`, el modo `art_73` MUST fallar con mensaje explícito para ese vehículo, y MUST NOT degradarse en silencio a `art_70` ni a ningún otro modo.
- **FR-022**: Si el perfil declara `activo_fijo` en falso, los modos `art_70` y `art_73` MUST emitirse como no disponibles con su razón, y solo `sin_reajuste` MUST producir cifras.
- **FR-023**: El sistema MUST poder emitir el valor patrimonial por lote y por año gravable correspondiente al modo de reajuste elegido.
- **FR-024**: El sistema MUST clasificar cada lote por su tenencia: ganancia ocasional si alcanza el umbral vigente en la celda, cédula general a la tarifa marginal del contribuyente si no.
- **FR-025**: El sistema MUST mostrar explícitamente qué porción del impuesto pagado corresponde a devaluación del COP y no a rentabilidad real, por un único método declarado en el resultado.

### Escenarios y modos

- **FR-026**: El sistema MUST correr todo cálculo en 3 escenarios normativos (A régimen vigente, B reforma aprobada íntegra, C reforma parcial o hundida) × 3 modos de reajuste = **9 celdas**.
- **FR-027**: Las variables que cambian entre escenarios MUST estar parametrizadas: tarifa y umbral de tenencia para ganancia ocasional; tratamiento del componente inflacionario de rendimientos financieros; tarifas y rangos de la tabla progresiva; tratamiento de dividendos de residentes; umbrales y tarifas del impuesto al patrimonio.
- **FR-028**: Ninguna comparación MUST presentarse en una sola celda. Las celdas no disponibles MUST emitirse con su razón, nunca omitirse en silencio ni sustituirse por otro modo.
- **FR-029**: Si un vehículo gana en una celda y pierde en otra, el memorando MUST decirlo en la primera página.

### Configuración y procedencia

- **FR-030**: Todo parámetro tributario MUST vivir en configuración versionada con fecha de vigencia. Cambiar una tarifa MUST requerir editar configuración, nunca código.
- **FR-031**: El sistema MUST NOT tener ningún valor por defecto numérico para un parámetro tributario. Un parámetro ausente o marcado `TODO` MUST hacer fallar la corrida.
- **FR-032**: Un año simulado sin configuración vigente en algún escenario MUST hacer fallar la corrida. El sistema MUST NOT heredar de otro escenario, interpolar ni extrapolar.
- **FR-033**: Todo parámetro tributario MUST declarar su procedencia: fuente, fecha de vigencia y estado (`verificado_profesional | supuesto_no_verificado`). Su ausencia MUST hacer fallar la corrida.
- **FR-034**: Ningún resultado que consuma un parámetro `supuesto_no_verificado` MUST emitirse sin propagar esa marca hasta el artefacto final. Ningún output MUST presentar tal parámetro como hecho establecido.

### Cumplimiento

- **FR-035**: El sistema MUST verificar sobre cada celda: si el total de activos en el exterior supera el umbral de declaración obligatoria (Formulario 160); si la operación requiere registro cambiario ante el Banco de la República; si la posición en activos con *situs* estadounidense supera US$60.000 (*estate tax*); si el horizonte deja la posición bajo el umbral de ganancia ocasional; y si el patrimonio proyectado cruza el umbral de impuesto al patrimonio.
- **FR-036**: El módulo de cumplimiento MUST levantar banderas y MUST NOT calcular impuestos.
- **FR-037**: Cada alerta MUST citar la norma o concepto aplicable desde texto configurable e incluir la advertencia de que requiere validación profesional. Una alerta sin texto configurado MUST fallar, no degradarse a un identificador crudo.
- **FR-038**: El módulo de cumplimiento MUST leer sus umbrales de la misma configuración que el motor fiscal.

### Capa narrativa

- **FR-039**: El sistema MUST generar el memorando a partir de un conjunto de resultados ya calculados, y MUST NOT permitir que la capa narrativa realice cálculo alguno.
- **FR-040**: El sistema MUST verificar que toda cifra de la prosa generada existe en los resultados del motor, y MUST NO emitir el memorando si alguna no existe.
- **FR-041**: Todo memorando MUST incluir una sección de **abogado del diablo** por vehículo analizado: qué tendría que pasar para que fuera una mala decisión, qué supuestos son frágiles, qué riesgo no está en el modelo. Un memorando sin ella MUST NO emitirse.
- **FR-042**: El sistema MUST señalar supuestos internamente inconsistentes o fuera de rangos históricos razonables.
- **FR-043**: El sistema MUST NO emitir recomendaciones de compra o venta, MUST NO estimar retornos futuros no presentes en los resultados, y MUST NO interpretar normativa fuera de los textos configurados. Estas prohibiciones MUST verificarse sobre la prosa producida, no solo instruirse.

### Salidas

- **FR-044**: El sistema MUST producir un gráfico de cascada que vaya del retorno bruto al retorno neto mostrando cada capa de fricción como una barra.
- **FR-045**: El sistema MUST generar el memorando en Markdown y exportarlo a PDF.
- **FR-046**: Todo artefacto de salida MUST incluir, de forma visible y no descartable, los cuatro elementos legales: naturaleza de análisis informativo y no de recomendación ni asesoría en los términos de la regulación del mercado de valores colombiano; que el emisor no es asesor inscrito en el RNPMV ni entidad vigilada por la Superintendencia Financiera de Colombia; que los cálculos tributarios son estimaciones que requieren validación por contador o abogado tributarista y que la normativa puede haber cambiado; y la fecha de vigencia y el estado de procedencia de los parámetros usados.
- **FR-047**: Toda cifra mostrada MUST poder descomponerse hasta sus inputs y la fórmula aplicada. Lo que no se pueda auditar MUST NO mostrarse.
- **FR-048**: El operador MUST poder ajustar los supuestos de la simulación y volver a ejecutar.

### Fuera de alcance (requisitos negativos)

- **FR-049**: El sistema MUST NO incluir valoración de empresas (DCF, múltiplos), opciones, futuros ni derivados, optimización de portafolio ni frontera eficiente, multi-tenant, autenticación ni facturación.
- **FR-050**: El sistema MUST NO conectarse con brókeres ni ejecutar órdenes. Nunca.
- **FR-051**: El sistema MUST NO incluir ninguna funcionalidad de recomendación automática.

### Destino del dividendo distribuido

*Resuelto por el operador el 2026-08-24.*

- **FR-052**: El destino del efectivo del dividendo neto MUST ser un parámetro de la corrida con tres valores: `reinvertir` (por defecto), `acumular_caja`, `repatriar`. Los tres MUST poder correrse para comparar. El resultado MUST declarar cuál se usó.
- **FR-053**: En modo `reinvertir`, el dividendo neto MUST adquirir participaciones del mismo vehículo, creando un lote nuevo con su propio reloj de tenencia y su propia TRM de reconocimiento.
- **FR-054**: El spread cambiario del dividendo MUST depender del modo, porque cada modo mueve el efectivo de forma distinta:
  - `reinvertir`: se aplica spread en la reinversión, porque un lote nuevo implica una conversión nueva.
  - `acumular_caja`: **no** se aplica spread hasta la salida; el efectivo permanece en la moneda de origen.
  - `repatriar`: se aplica spread **cada año**, porque el efectivo vuelve a COP anualmente.
- **FR-055**: El memorando MUST declarar siempre qué modo de destino del dividendo se usó, y MUST mostrar el costo de fragmentación de lotes que resulta de ese modo (FR-017).

### Presentación de la matriz de resultados

*Resuelto por el operador el 2026-08-24.*

- **FR-056**: El operador MUST poder designar una **celda de referencia** (un escenario normativo + un modo de reajuste) como caso base de la presentación.
- **FR-057**: La presentación al cliente MUST mostrar el gráfico de cascada de la celda de referencia, y MUST mostrar las otras ocho celdas como **banda de sensibilidad** en torno a ella.
- **FR-058**: La divergencia entre celdas MUST destacarse cuando **invierte el ordenamiento** entre vehículos; una diferencia de magnitud que no cambia el orden no MUST competir por la atención del cliente.
- **FR-059**: Designar una celda de referencia MUST NO omitir ninguna de las nueve. La celda de referencia es un recurso de presentación, no un filtro: las nueve siguen calculándose y estando disponibles (FR-028).

---

### Key Entities

- **Vehículo**: instrumento del catálogo. Lleva su identificación legal (ISIN, nombre legal, forma jurídica del emisor), su perfil de fricción (TER, dividend yield, tipo de distribución, convención de TER), su domicilio, su clasificación fiscal (`evento_ingreso_anual`, `elegibilidad_art_73`) y la fuente documental que sustenta esa clasificación.
- **Lote**: unidad fiscal. Fecha de adquisición, cantidad, costo en moneda de origen, TRM de reconocimiento y reloj de tenencia propio. Inmutable.
- **Posición**: secuencia de lotes de un mismo vehículo para un mismo cliente.
- **Perfil del cliente**: tarifa marginal en cédula general y `activo_fijo`. Determina qué modos de reajuste están disponibles.
- **Escenario normativo**: A, B o C. Agrupa las variables tributarias que cambian entre marcos.
- **Modo de reajuste**: `sin_reajuste`, `art_70` o `art_73`. Excluyentes entre sí.
- **Celda de resultado**: intersección de un escenario normativo y un modo de reajuste. Nueve por vehículo.
- **Parámetro tributario**: valor normativo con fecha de vigencia y procedencia (fuente + estado).
- **Asiento**: registro atómico y trazable de un movimiento del cálculo, con su año, concepto, monto, fórmula e inputs.
- **Alerta de cumplimiento**: bandera con identificador, celdas donde aplica, texto normativo configurable y advertencia de validación profesional.
- **Memorando**: artefacto narrativo derivado de un conjunto de resultados, con su cascada, su sección de abogado del diablo y sus disclaimers.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El operador carga 15 vehículos y los compara en un solo comando o una sola pantalla.
- **SC-002**: Cada comparación devuelve las 9 celdas (3 escenarios × 3 modos) sin que el operador tenga que solicitarlas por separado; las no disponibles vienen con su razón.
- **SC-003**: Un cliente no técnico identifica, en 30 segundos frente al gráfico de cascada, cuánto se pierde por cada capa de fricción.
- **SC-004**: El sistema muestra qué porción del impuesto se debe a devaluación del COP y no a rentabilidad real, en toda liquidación.
- **SC-005**: El sistema reporta qué porcentaje de la ganancia quedó gravada a cédula general por fragmentación de lotes, en toda liquidación.
- **SC-006**: El operador genera un memorando en PDF con todos sus disclaimers en menos de dos minutos desde que dispara la generación.
- **SC-007**: Toda cifra del output se descompone hasta sus inputs y su fórmula, verificable por inspección.
- **SC-008**: Cambiar una tarifa tributaria requiere editar exactamente un archivo de configuración y ninguna línea de código, verificado por una prueba que altera la configuración y observa el cambio en el resultado.
- **SC-009**: Los cálculos fiscales y de fricción se validan contra casos calculados a mano, con la aritmética documentada junto al caso.
- **SC-010**: Un intento de emitir un memorando cuya prosa contenga una cifra ausente de los resultados, lenguaje recomendatorio, o sin sección de abogado del diablo, no produce memorando.
- **SC-011**: Una corrida con un parámetro tributario ausente, sin procedencia declarada, o sin vigencia para el año simulado, falla de forma explícita en lugar de producir una cifra.
- **SC-012**: Ningún artefacto de salida presenta un parámetro `supuesto_no_verificado` como hecho establecido.

---

## Assumptions

- **Procedencia del material tributario.** El material disponible **no está firmado por un profesional** y tiene una **contradicción interna conocida entre dos de sus documentos**. En consecuencia, todo parámetro derivado de él se trata como `supuesto_no_verificado`, se marca como tal en configuración, y queda pendiente de concepto profesional. Esto no es una nota al pie: condiciona qué puede afirmar cualquier output del sistema.
- **Los valores numéricos los carga el operador.** Tarifas, umbrales, UVT, porcentajes del art. 70 y factores del art. 73 los carga el operador validados con su contador tributarista. Quedan como `TODO` en configuración, con campo obligatorio de fuente normativa y fecha de vigencia. **Ningún valor se inventa.**
- El catálogo se cura a mano y no crece a cientos de vehículos.
- El operador ejecuta la herramienta él mismo; no hay usuarios finales con acceso propio.
- Existe al menos una fuente gratuita de datos de mercado y de TRM adecuada; su elección concreta y su licencia se resuelven en la fase de planeación.
- El sistema modela un vehículo por corrida de comparación contra un mismo perfil de cliente; no modela un portafolio combinado ni su interacción fiscal.
- Los supuestos de mercado (retorno total esperado, dividend yield, devaluación anual esperada del COP) los aporta el operador; el sistema no los estima ni los proyecta.
- El modo por defecto del destino del dividendo es `reinvertir`, resuelto por el operador el 2026-08-24. No es un supuesto del especificador.
- La celda de referencia por defecto no está fijada: la designa el operador por corrida.
