# Research — Motor de Fricción Neta

**Fase 0 de `/speckit-plan`** · 2026-08-24

La mayor parte del espacio de decisión técnica ya está cerrado por la Fase 0 de BMAD: los 51 ADs del *architecture spine* fijan paradigma, almacenamiento, despliegue y stack, y el stack quedó verificado en vivo contra PyPI. Este documento resuelve **lo único que quedó diferido** y registra lo que deliberadamente no se resuelve aquí.

---

## R-1 — Proveedor de datos de mercado

**Decisión: `datos.gov.co`, dataset `32sa-8pi3` (Tasa de Cambio Representativa del Mercado), vía su API Socrata, sin credencial.**

### Qué necesita realmente el sistema

Antes de elegir proveedor hay que acotar la necesidad, porque es mucho más estrecha de lo que parece. Repasando qué consume cada input del motor:

| Input | De dónde viene | ¿Necesita proveedor? |
|---|---|---|
| TER, dividend yield histórico, ISIN, forma jurídica | Catálogo curado a mano (FR-003) | **No** |
| Retorno total esperado, dividend yield esperado | Supuesto del operador (Assumptions) | **No** |
| Devaluación anual esperada del COP | Supuesto del operador | **No** |
| Tarifas de bróker, custodia, spread FX | `config/brokers.yaml` | **No** |
| Parámetros tributarios | `config/tributario/`, los carga el tributarista | **No** |
| **TRM histórica** | Fuente externa | **Sí** |

El MVP necesita **una sola serie externa: la TRM**. Es la que fija la `trm_reconocimiento` de cada lote (`AD-30`) y alimenta `CurvaDeCambio` (`AD-18`). Todo lo demás es dato curado o supuesto declarado.

Esto es una conclusión útil, no una obviedad: descarta de entrada a los proveedores de datos de mercado (precios, fundamentales, series de fondos), que son justamente los que tienen problemas de licencia y de credencial.

### Verificación

No se aceptó la documentación: se ejecutó contra el endpoint el 2026-08-24.

| Prueba | Resultado |
|---|---|
| `GET /resource/32sa-8pi3.json` sin API key | **HTTP 200**, 0,39 s |
| Esquema devuelto | `valor`, `unidad`, `vigenciadesde`, `vigenciahasta` |
| Registro más antiguo | **1991-12-02** (TRM 643,42) |
| Registros totales | **8 332** |
| Filtro por rango (`$where ... between`) | Funciona |
| Ordenamiento (`$order`) | Funciona |

### Rationale

1. **Es la fuente de la autoridad correcta.** La TRM la certifica la Superintendencia Financiera; este dataset es su publicación oficial en el portal de datos abiertos del Estado. Para un memorando que debe ser defendible ante el contador de un cliente, la procedencia del tipo de cambio importa tanto como su valor.
2. **Gratuita y sin credencial**, lo que satisface el principio VI y `AD-11` sin excepciones ni cuentas que administrar.
3. **Su esquema ya es de vigencias.** `vigenciadesde` / `vigenciahasta` mapean directamente al modelo de vigencias del proyecto (`AD-7`), y resuelven de forma nativa fines de semana y festivos: un registro cubre un rango de días, no un día suelto.
4. **Profundidad suficiente.** Desde 1991 cubre cualquier año de adquisición que el art. 73 pueda indexar (`AD-43`).

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| Portal de Estadísticas Económicas del Banco de la República | Es la misma cifra, pero orientado a descarga manual e interactiva; no expone una API de rango tan simple. Queda como **fuente de contraste** para verificar el adaptador, no como origen de datos. |
| Web service SOAP de la Superintendencia Financiera | La cifra es la misma. SOAP añade una dependencia de parseo y un modo de fallo más, a cambio de nada. |
| Proveedores de datos de mercado (Yahoo, Alpha Vantage, etc.) | **Innecesarios**: el sistema no consume precios ni fundamentales. Además arrastran credencial y restricciones de licencia que chocan con el principio VI. |

### Implicaciones para el diseño

- El puerto `FuenteMercado` (`AD-11`) queda con una superficie mínima: dada una ventana de fechas, devolver la serie TRM. Cambiar de proveedor es reimplementar un método.
- Cada descarga se persiste como snapshot Parquet inmutable direccionado por contenido (`AD-11`), así que un memorando de hace seis meses se reproduce con la TRM de entonces.
- El adaptador usa `httpx2` (`AD-16`), el mismo cliente que arrastra el SDK `anthropic`.
- **Riesgo registrado:** es un endpoint público de un portal estatal, sin SLA. Mitigación: el snapshot en caché es la fuente de verdad de una corrida; la red solo se toca para refrescar. Una corrida nunca depende de que el portal esté arriba.

---

## R-2 — Cómo se verifican los invariantes del núcleo sin ejecutar el código

**Decisión: un único `tests/test_arquitectura.py` que recorre el AST de `motor/` con el módulo `ast` de la stdlib.**

Tres ADs delegan su cumplimiento a este test, así que su diseño es parte del plan y no un detalle de implementación:

| Verificación | AD | Qué recorre el AST |
|---|---|---|
| Imports prohibidos en `motor/` | `AD-1` | Nodos `Import` / `ImportFrom`; falla si el módulo raíz está fuera de la lista blanca (stdlib, `decimal`, `pydantic`) |
| `float` prohibido en todo `motor/` | `AD-45` | Nodos `Name`/`Attribute` con `float`, y `Constant` de tipo `float` |
| Sin literales numéricos normativos en `motor/fiscal/` y `motor/cumplimiento/` | `AD-19` | Nodos `Constant` numéricos fuera de una lista blanca declarada de constantes de unidad (0, 1, −1, 100) |

**Rationale:** el análisis estático es determinístico, no requiere ejecutar nada, y falla en la compuerta antes de que un import indebido llegue a producción. La alternativa —comprobar en tiempo de ejecución con introspección— solo detecta lo que la suite ejercita, y depende de la cobertura para ser fiable.

**Alternativa considerada:** `import-linter` como dependencia. Descartada: añade una dependencia para lo que la stdlib resuelve en decenas de líneas, y no cubre las verificaciones 2 y 3, que son específicas de este dominio.

---

## R-3 — Contrato de normalización del guard numérico

**Decisión: el guard normaliza antes de comparar, y su tabla de normalización es parte del contrato con test propio.**

`AD-9` exige que el guard falle el render si una cifra de la prosa no existe en el JSON del motor. El punto fino es que "existe" requiere normalizar, porque el mismo número se escribe de muchas formas en un memorando en español colombiano:

| Forma en la prosa | Canoniza a |
|---|---|
| `1.234.567,89` (separadores colombianos) | `1234567.89` |
| `COP 1.234.567` / `$1.234.567` | `1234567` |
| `1,2 millones` / `1,2 M` | `1200000` |
| `3,45 %` | `0.0345` |
| `345 pb` / `345 puntos básicos` | `0.0345` |
| `USD 1,234.56` (separadores anglosajones) | `1234.56` |

**Rationale:** sin este contrato el guard tiene dos modos de fallo, ambos inaceptables. Si normaliza de menos, marca como inventada toda cifra correctamente formateada y **no se emite ningún memorando**. Si normaliza de más, dos cifras distintas colapsan a la misma y **deja pasar una cifra inventada** — que es exactamente lo que el principio I existe para impedir.

**Consecuencia de diseño:** el guard tiene tests con casos positivos *y negativos* declarados. Un caso negativo es una cifra que **debe** hacer fallar el render.

---

## R-4 — Lo que deliberadamente NO se resuelve en este documento

Estas preguntas son normativas, no técnicas. Resolverlas por investigación propia violaría el §8 del brief y el principio IV de la constitución.

| Pregunta abierta | Efecto | Quién responde |
|---|---|---|
| ¿El porcentaje del art. 70 se aplica al año en curso o de forma **acumulada** sobre los años de tenencia? | Mueve el costo fiscal en decenas de millones de COP sobre un caso realista | Tributarista del operador |
| ¿Qué campo del lote indexa el factor del art. 73 — año fiscal de reconocimiento o año de la operación? | Determina qué factor aplica y si salta `VigenciaNoCubierta` | Tributarista del operador |
| ¿Aplica el reajuste fiscal a ETF del exterior, TES y FIC, y bajo qué condiciones? | Determina qué modos están disponibles por vehículo | Tributarista del operador |
| Valores de todas las series y tarifas | El sistema no produce ninguna cifra sin ellos | Tributarista del operador |

`AD-43` recibe las dos primeras como **campos declarados en configuración** (`ventana`, `campo_indice`), no como comportamiento cableado. Mientras no estén declaradas, el cargador levanta `ParametroTributarioFaltante` y **la corrida falla en vez de producir una cifra**.

> **Procedencia.** El material tributario del que se derivó el marco no está firmado por un profesional y contiene una contradicción interna entre dos de sus documentos. Todo parámetro derivado nace con `estado: supuesto_no_verificado` (`AD-35`), y esa marca se propaga por peor-gana hasta el artefacto final (`AD-49`).

---

## Resumen de decisiones

| # | Decisión | Estado |
|---|---|---|
| R-1 | TRM desde `datos.gov.co/resource/32sa-8pi3`, sin credencial, tras el puerto `FuenteMercado` | Verificado contra el endpoint |
| R-2 | Invariantes del núcleo por recorrido de AST con la stdlib | Decidido |
| R-3 | Contrato de normalización del guard numérico, con casos negativos | Decidido |
| R-4 | Cuatro preguntas normativas escaladas al tributarista | Escalado, no resuelto |

**No queda ningún `NEEDS CLARIFICATION` técnico.** Lo abierto es normativo y está escalado por diseño.
