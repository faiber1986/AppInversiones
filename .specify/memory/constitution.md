<!--
SYNC IMPACT REPORT
==================
Cambio de versión: (plantilla sin ratificar) → 1.0.0
Tipo: MAJOR — ratificación inicial. Se pasa de plantilla con placeholders a
      constitución concreta con 10 principios.

Principios añadidos (10):
  I.    El LLM no calcula nada
  II.   Todo cálculo es trazable
  III.  Los parámetros tributarios son configuración, no código
  IV.   Todo parámetro tributario declara su procedencia
  V.    La herramienta no recomienda
  VI.   Sin dependencias de datos de pago en el MVP
  VII.  Cobertura de tests con casos calculados a mano
  VIII. Los disclaimers legales son requisito funcional bloqueante
  IX.   Ninguna comparación se presenta en una sola celda
  X.    El núcleo es puro

Secciones añadidas:
  - Restricciones operativas (§11 del brief)
  - Flujo de desarrollo y compuertas de calidad
  - Gobernanza

Plantillas dependientes:
  ✅ .specify/templates/plan-template.md   — ACTUALIZADA: "Constitution Check" pasó de
                                              "[Gates determined based on constitution
                                              file]" a 11 compuertas concretas G1-G11
                                              mapeadas a los principios I-X
  ✅ .specify/templates/spec-template.md    — revisada; no exige cambios estructurales
  ✅ .specify/templates/tasks-template.md   — ACTUALIZADA: dos conflictos con la
                                              constitución. (1) los tests venían marcados
                                              "OPTIONAL - only if tests requested", que
                                              contradice el principio VII; ahora son
                                              obligatorios para motor/fiscal y
                                              motor/friccion, con el requisito de cálculo
                                              a mano. (2) las tareas de ejemplo asumían
                                              base de datos, autenticación y API HTTP,
                                              contra AD-3, AD-13 y el alcance del brief
  ⚠  README.md                              — pendiente: no existe todavía. Debe
                                              documentar `playwright install chromium`
                                              y cómo correr todo desde cero (§11)

TODOs diferidos: ninguno.
-->

# Constitución del Motor de Fricción Neta

Herramienta de **análisis** —no de asesoría— que compara vehículos de inversión por su
retorno neto real en COP para residentes fiscales colombianos, después de TER, retención
en origen, comisiones, spread cambiario e impuesto de renta colombiano.

Esta constitución gobierna toda decisión de diseño e implementación. Los 37 ADs del
*architecture spine* son el contrato técnico que la implementa.

## Core Principles

### I. El LLM no calcula nada (NO NEGOCIABLE)

Todo número sale de código Python determinístico, testeable y auditable. El LLM solo lee
resultados ya calculados y redacta narrativa. **Cualquier diseño donde un modelo genere
cifras es un defecto crítico.**

El adaptador LLM recibe un JSON de resultados ya calculados, nunca una petición de
cálculo. Un guard determinístico normaliza y extrae todo token numérico de la prosa
generada y **falla el render** si alguno no aparece en el JSON de origen. No hay ruta
degradada: si el guard falla, no hay memorando.

*Racional:* una instrucción de prompt es una petición, no una garantía. Un principio que
solo vive en el system prompt no es un principio, es una esperanza. Se hace cumplir con
código que tiene test propio. (`AD-9`)

### II. Todo cálculo es trazable (NO NEGOCIABLE)

Cada número del output debe poder descomponerse en sus inputs y en la fórmula aplicada.
**Si no se puede auditar, no se muestra.**

El motor no devuelve escalares: emite un libro de asientos append-only de entradas
tipadas. Todo total —retorno neto, desglose de fricción, porción atribuible a devaluación,
TIR, fragmentación por lotes— se **deriva sumando el libro**. Ningún total se calcula por
una vía paralela. El render lee el libro; no recalcula.

*Racional:* el producto se vende porque el cliente puede ver de dónde sale cada punto
básico. Un total que no se descompone es indistinguible de uno inventado. (`AD-8`, `AD-20`)

### III. Los parámetros tributarios son configuración, no código (NO NEGOCIABLE)

Tarifas, UVT, umbrales, plazos y series de reajuste viven en YAML versionado con fecha de
vigencia. Cambiar una tarifa MUST requerir editar un archivo de configuración, nunca
refactorizar código.

**Nunca existe un valor por defecto numérico.** Un parámetro ausente o marcado con el
centinela `TODO` levanta `ParametroTributarioFaltante`. Un año simulado sin configuración
vigente levanta `VigenciaNoCubierta`: nunca se interpola, extrapola ni hereda de otro
escenario. Ningún módulo del núcleo declara una constante numérica normativa.

*Racional:* la normativa colombiana cambia con cada reforma. Un número inventado en una
herramienta financiera es peor que un campo vacío, porque el campo vacío se ve.
(`AD-7`, `AD-19`, `AD-31`)

### IV. Todo parámetro tributario declara su procedencia (NO NEGOCIABLE)

Cada parámetro lleva `procedencia: {fuente, fecha_vigencia, estado}`, donde `estado` es
`verificado_profesional` o `supuesto_no_verificado`. Su ausencia levanta
`ProcedenciaNoDeclarada`.

**Estado actual del proyecto:** el material tributario disponible NO está firmado por un
profesional y tiene una contradicción interna conocida entre dos de sus documentos. Por
tanto todo parámetro derivado de él es `supuesto_no_verificado`, y **ningún output puede
presentarlo como hecho establecido**. El `Renderer` rechaza payloads sin procedencia
declarada; los guards del principio V fallan el render si la prosa presenta como
establecido un parámetro no verificado.

*Racional:* un parámetro sin procedencia visible es indistinguible de uno validado por un
tributarista, y esa indistinguibilidad es exactamente el riesgo. Este principio es tan
bloqueante como el I. (`AD-35`)

### V. La herramienta no recomienda (NO NEGOCIABLE)

Produce comparativos y escenarios. **Nunca emite un "compre X".** Es una restricción
regulatoria, no estética.

Se hace cumplir con guards determinísticos que fallan el render ante lenguaje
recomendatorio, estimación de retornos futuros no presente en el JSON, o interpretación
normativa fuera de los textos configurados. El pipeline verifica además **presencia
estructural**: un memorando sin sección de abogado del diablo no se emite.

*Racional:* el guard numérico del principio I dejaría pasar intacta la frase "recomiendo
el ETF irlandés", porque no contiene ninguna cifra. Una restricción regulatoria sostenida
solo por un prompt no está sostenida. (`AD-23`)

### VI. Sin dependencias de datos de pago en el MVP

El sistema MUST funcionar con fuentes gratuitas. El diseño MUST permitir cambiar de
proveedor de datos sin tocar lógica de negocio: el núcleo lee mercado solo por el puerto
`FuenteMercado` y nunca sabe quién es el proveedor. Un adaptador de pago exige reabrir
esta decisión explícitamente.

*Racional:* atarse a un proveedor de pago en el MVP compromete tanto el costo como la
capacidad de reemplazarlo cuando cambie sus términos. (`AD-11`)

### VII. Cobertura de tests con casos calculados a mano (NO NEGOCIABLE)

Ningún cálculo financiero se mergea sin test unitario con caso conocido **calculado a
mano**, con su aritmética documentada en el propio test. **Un caso derivado de correr el
código no cuenta como caso conocido** — solo confirma que el código hace lo que hace.

Aplica de forma bloqueante a `motor/fiscal/` y `motor/friccion/`. La compuerta es un
workflow de GitHub Actions sobre `push` y `pull_request` que ejecuta `ruff check`,
`pytest` y el test de arquitectura, y falla si la cobertura de esos dos módulos baja de la
línea acordada.

*Racional:* un test escrito contra el output actual del código convierte cualquier bug en
comportamiento esperado. (`AD-25`)

### VIII. Los disclaimers legales son requisito funcional bloqueante

Todo output —pantalla, PDF, memorando, exportación— MUST incluir de forma visible y no
descartable:

1. Que el documento es un **análisis informativo**, no una recomendación profesional de
   inversión ni asesoría en los términos de la regulación del mercado de valores
   colombiano.
2. Que quien lo emite **no es asesor inscrito en el RNPMV** ni entidad vigilada por la
   Superintendencia Financiera de Colombia.
3. Que los cálculos tributarios son **estimaciones que requieren validación** por contador
   o abogado tributarista, y que la normativa puede haber cambiado.
4. La **fecha de vigencia** de los parámetros tributarios usados, y su estado de
   procedencia (principio IV).

Los emite un único `Renderer`, nunca las plantillas ni el LLM. Ningún módulo escribe un
artefacto de salida sin pasar por él.

*Racional:* si cada plantilla es responsable de sus disclaimers, la primera ruta de salida
nueva se publica sin ellos. Centralizar la emisión hace que omitirlos requiera un acto
deliberado. Esto no es opcional ni negociable. (`AD-10`)

### IX. Ninguna comparación se presenta en una sola celda

Todo resultado corre en **3 escenarios normativos × 3 modos de reajuste fiscal**. Las
funciones públicas de comparación devuelven siempre la matriz completa; no existe una
firma que devuelva una sola celda.

Las celdas no disponibles —porque la elegibilidad del vehículo es `sin_clasificar`, o
porque el perfil no es activo fijo— se emiten **como no disponibles con su razón**, nunca
se omiten en silencio ni se sustituyen por otro modo. Si un vehículo gana en una celda y
pierde en otra, la divergencia se emite en la primera página del memorando.

*Racional:* presentar un solo escenario permite elegir el que favorece la conclusión
deseada. Y una degradación silenciosa de `art_73` a `art_70` produciría una cifra que el
operador no puede defender ante el contador del cliente.
(`AD-24`, `AD-31`, `AD-32`, `AD-33`)

### X. El núcleo es puro

Todo módulo bajo `motor/` importa únicamente la stdlib, `decimal` y `pydantic`. Tiene
prohibido importar `streamlit`, `anthropic`, `httpx2`, `pyyaml`, `pyarrow`, `matplotlib`,
`jinja2`, `playwright` o cualquier librería de E/S, red o UI.

El núcleo no lee reloj, ni entorno, ni aleatoriedad: fecha, semilla e identificador de
corrida entran como parámetros. Dos corridas con los mismos inputs producen bytes
idénticos. El núcleo nunca registra logs ni imprime: devuelve o levanta excepciones
tipadas del dominio.

`tests/test_arquitectura.py` recorre los `import` de `motor/` y falla la compuerta ante
cualquier violación.

*Racional:* es la condición que hace posibles los principios II y VII. Un dominio que hace
E/S no se puede testear con casos calculados a mano ni auditar. (`AD-1`)

## Restricciones operativas

- **Un solo desarrollador.** Optimizar para mantenibilidad, no para escala. La
  sobre-ingeniería es un defecto, no una virtud.
- **Cero costo recurrente de infraestructura en el MVP.** El sistema corre como un proceso
  Python local en Windows. Sin servidor, sin contenedor obligatorio, sin servicio
  gestionado. El único costo recurrente es la API de Anthropic.
- **Python como lenguaje principal.** Cualquier desviación MUST justificarse por escrito.
- **Repo limpio con README que explique cómo correr todo desde cero**, incluidos los pasos
  de setup que un gestor de dependencias no resuelve (`playwright install chromium`).
- **Fuera de alcance, explícitamente:** valoración de empresas, derivados, optimización de
  portafolio, multi-tenant, autenticación, facturación, y cualquier conexión con brókeres
  o ejecución de órdenes. El MVP se define tanto por lo que excluye como por lo que
  incluye.

## Flujo de desarrollo y compuertas de calidad

- **Compuerta automatizada:** workflow de GitHub Actions sobre `push` y `pull_request`
  ejecutando `ruff check`, `pytest`, `tests/test_arquitectura.py` y el umbral de cobertura
  de los módulos fiscal y de fricción. Un hook `pre-push` local ejecuta lo mismo.
- **Ningún merge con la compuerta en rojo.** Un fallo de compuerta se diagnostica; no se
  esquiva.
- **Trazabilidad de la configuración tributaria:** cada cambio de un parámetro se hace en
  un commit propio cuyo mensaje declara la fuente normativa y la fecha de vigencia. El
  historial de git ES el registro de auditoría de vigencias.
- **Los valores numéricos tributarios los carga el operador**, validados con su contador
  tributarista. Ningún agente ni desarrollador inventa una tarifa, un porcentaje o un
  factor. Un parámetro desconocido queda como `TODO` visible y se escala.

## Governance

Esta constitución **prevalece sobre cualquier otra práctica**. Ante conflicto entre esta
constitución y una decisión de implementación, gana la constitución.

**Procedimiento de enmienda.** Se enmienda por commit explícito que declare qué principio
cambia, por qué, y qué impacto tiene sobre los artefactos dependientes (spec, plan, tasks,
ADs). Los 37 ADs del *architecture spine* son el contrato técnico que implementa estos
principios: **un cambio que contradiga un AD exige enmendar el AD primero**, no eludirlo.

**Versionado.** Semántico. MAJOR: remoción o redefinición incompatible de un principio o
regla de gobernanza. MINOR: principio o sección nueva, o ampliación material de una guía
existente. PATCH: aclaración, redacción, corrección no semántica.

**Revisión de cumplimiento.** Toda revisión de código verifica el cumplimiento de los
principios I, II, IV, V, VII, VIII y X, que son los que tienen mecanismo de cumplimiento
automatizado. Los principios marcados NO NEGOCIABLE no admiten excepción por conveniencia,
plazo ni alcance: si uno estorba, se enmienda la constitución o se cambia el diseño, no se
ignora el principio.

**Complejidad.** Toda complejidad debe justificarse contra las restricciones operativas.
Ante dos diseños que satisfacen los principios, gana el más aburrido.

**Version**: 1.0.0 | **Ratified**: 2026-08-24 | **Last Amended**: 2026-08-24
