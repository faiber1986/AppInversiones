# Crítica al brief — Motor de Fricción Neta

> Entregable de Fase 0. La sección 12 del brief pide explícitamente: *"Cuestiona este brief. Si algo aquí está mal planteado, sobredimensionado o es innecesario para un MVP, dilo antes de construirlo."*
>
> Lo que sigue son defectos concretos, no observaciones de estilo. Cada uno indica qué hay que decidir y quién puede decidirlo.
> Fecha: 2026-08-24

---

## 1. El disparador del impuesto anual está mal condicionado — CORREGIDO EN ARQUITECTURA

**Dónde:** §5, paso 4. *"Impuesto anual (solo vehículos distributivos)"*.

**El problema:** la condición es falsa para la mitad del catálogo de §4. Un CDT, un TES y un FIC generan **rendimientos financieros gravables cada año** sin que exista dividendo alguno. Si el motor condiciona el hecho gravable anual a `es_distributivo`, esos tres vehículos aparecerán como si difirieran el impuesto hasta la venta, y su retorno neto saldrá sobrestimado.

Es un error que favorece sistemáticamente a la renta fija local frente a los ETF, justo la comparación que el producto existe para hacer bien.

**Estado:** resuelto en **AD-5**. Cada vehículo declara `evento_ingreso_anual: dividendo | rendimiento_financiero | ninguno_hasta_realizar`, y el paso anual despacha sobre ese campo. El brief no necesita cambiar; la corrección vive en el modelo de datos.

---

## 2. Riesgo de doble conteo del TER — CORREGIDO EN ARQUITECTURA

**Dónde:** §5, paso 3, tercer viñeta: *"Restar TER sobre el valor del portafolio"*.

**El problema:** el TER de un ETF **ya viene descontado del NAV**. El retorno que publica el fondo es neto de comisión de gestión. Si cargas ese número como "retorno total esperado anual" y además restas el TER, lo estás cobrando dos veces. Sobre 20 años y un TER de 0,20%, el error acumulado no es marginal.

La ambigüedad no está en el motor, está en el input: nada en el brief dice si el "retorno total esperado" que se carga es bruto o neto de TER.

**Estado:** resuelto en **AD-6**. Cada vehículo declara `retorno_esperado_base: bruto_de_ter | neto_de_ter`, campo obligatorio sin valor por defecto. El motor solo resta TER cuando la convención es `bruto_de_ter`.

---

## 3. El costo fiscal en COP tiene un matiz que afecta la tesis comercial del producto — REQUIERE TU TRIBUTARISTA

**Dónde:** §5, paso 6, el recuadro marcado *"Punto crítico"*.

**El argumento del brief:** la base gravable se calcula en pesos, así que la devaluación del COP genera ganancia gravable en Colombia aunque el retorno en dólares sea cero. El sistema debe mostrar cuánto del impuesto corresponde a devaluación y no a rentabilidad real.

**Por qué importa:** este es el activo comercial del producto. Es lo que lo diferencia de cualquier comparador de retornos brutos.

**El matiz:** el Estatuto Tributario contempla un **reajuste fiscal opcional del costo de los activos (art. 70 ET)**. En la medida en que aplique a esta clase de activo y el contribuyente lo tome, erosiona parte de esa ganancia por devaluación. Si el modelo la ignora, la cifra de "impuesto por devaluación" que le muestras al cliente sale **inflada**.

**No voy a asumir cómo aplica.** Es exactamente el tipo de interpretación normativa que §8 me prohíbe hacer por mi cuenta, y un número inflado aquí desacreditaría el argumento central del producto ante el primer cliente que lo consulte con su contador.

**Qué hay que hacer:** preguntarle a tu tributarista si el reajuste fiscal del costo aplica a (a) acciones y ETF del exterior, (b) TES, (c) FIC — y bajo qué condiciones. La respuesta entra como parámetro de configuración por escenario. **AD-7** ya deja el hueco listo para recibirla y hace que el sistema falle ruidosamente mientras esté vacío.

---

## 4. Hueco no especificado: qué pasa con el efectivo del dividendo — REQUIERE DECISIÓN TUYA

**Dónde:** §5, paso 3, cuarta viñeta.

**El problema:** el brief dice que el dividendo neto *"se reinvierte si el vehículo es acumulativo; si es distributivo, se registra como ingreso gravable"*. Registra el **hecho gravable**, pero no dice qué pasa con **el dinero**. Hay tres caminos y dan tres TIR distintas:

| Opción | Qué implica |
|---|---|
| Reinvertir en el mismo vehículo | El dividendo vuelve al portafolio; el distributivo se acerca al acumulativo salvo por el impuesto pagado en el camino |
| Acumular en caja sin rendimiento | El más conservador; penaliza al distributivo y probablemente exagera la penalización |
| Repatriar a COP cada año | Realiza spread cambiario cada año, no solo al entrar y salir |

Sin esta decisión, la comparación distributivo vs. acumulativo —que es el eje del catálogo de §4— no está definida.

**Recomendación:** modelarlo como un parámetro de la simulación, no como una constante. El operador elige por corrida y el memorando declara cuál se usó. El `LibroDeAsientos` de **AD-8** acomoda las tres sin cambio estructural.

---

## 5. Sobre-proceso — DECIDIDO, MITIGADO

**Dónde:** §0, fases 0 y 1.

**La observación:** correr la suite completa de BMAD *más* el ciclo completo de Spec-Driven Development de Spec Kit sobre un MVP de un solo operador es ceremonia cara. Dos frameworks de planeación producen dos juegos de artefactos que después hay que reconciliar a mano, y la reconciliación es trabajo que no produce software.

**Estado:** mitigado por decisión tuya del 2026-08-24. BMAD se usa **solo para la decisión de arquitectura** (que es el objetivo explícito de tu Fase 0); todo el qué/porqué/cómo lo lleva Spec Kit. No se generan PRD, epics ni stories con BMAD.

---

## 6. Riesgo técnico en el criterio de aceptación §10.5 — CORREGIDO EN ARQUITECTURA

**Dónde:** §10, criterio 5: *"Genero un memorando en PDF, con disclaimers, en menos de dos minutos"*.

**El problema:** la ruta obvia en Python para HTML→PDF es WeasyPrint, y WeasyPrint en Windows exige Pango, cairo y GDK-PixBuf instalados por separado como parte de GTK+. Hay fallos de instalación reportados en Windows 11 hasta 2025-2026. Eso choca de frente con §11: *"el repo debe quedar limpio, con README que explique cómo correr todo desde cero"*.

**Estado:** resuelto en **AD-12**. HTML como único formato intermedio, y de HTML a PDF con Chromium headless vía playwright, que ya corre en tu máquina. El HTML es el contrato; el motor de PDF queda intercambiable.

---

## 7. Refuerzo propuesto a §8: hacer la prohibición verificable, no solo declarada — INCORPORADO

**Dónde:** §8, *"Prohibido: generar cualquier cifra que no venga del motor de cálculo"*.

**La observación:** tal como está redactado, el cumplimiento de esa prohibición depende de que el modelo obedezca el prompt del sistema. Un prompt es una instrucción, no una garantía. Para el principio no negociable #1 del brief —*"cualquier diseño donde un modelo genere cifras es un defecto crítico"*— eso es una defensa débil.

**La propuesta:** un guard determinístico que, después de generar el memorando, extraiga **todo token numérico** de la prosa y falle el render si alguno no aparece en el JSON del motor. Convierte una instrucción de prompt en un test ejecutable.

**Estado:** incorporado como **AD-9**. El guard es código con test propio, no una línea de prompt.

---

## 8. Observación menor: §6 pide tres escenarios, el catálogo tiene ocho tipos, y §10.1 pide quince vehículos

No es un defecto, es una nota de volumen. La combinatoria del output es 15 vehículos × 3 escenarios = 45 resultados por corrida. El gráfico de cascada de §5 es por vehículo y por escenario.

**Implicación para la Fase 1:** hay que definir cómo se presentan 45 resultados sin que el cliente no técnico se pierda —el criterio §10.3 exige que entienda el desglose en 30 segundos. Probablemente significa una vista de comparación agregada más un waterfall por vehículo bajo demanda, pero es una decisión de producto, no de arquitectura, y le corresponde a Spec Kit.

---

## Resumen: qué necesita tu decisión

| # | Asunto | Quién decide | Bloquea |
|---|---|---|---|
| 3 | ¿Aplica el reajuste fiscal del costo (art. 70 ET)? | Tu contador tributarista | Fase 1 |
| 4 | ¿Qué se hace con el efectivo del dividendo distribuido? | Tú | Fase 1 |
| 8 | ¿Cómo se presentan 45 resultados sin perder al cliente? | Tú, vía Spec Kit | Fase 1 |

Ninguno bloquea el cierre de la Fase 0: la arquitectura acomoda las tres respuestas sin cambio estructural. Los tres sí bloquean la especificación.

Los defectos 1, 2, 6 y 7 quedan cerrados en la arquitectura y no requieren nada de ti.
