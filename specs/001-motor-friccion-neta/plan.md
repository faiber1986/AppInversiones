# Implementation Plan: Motor de Fricción Neta

**Branch**: `001-motor-friccion-neta` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: `specs/001-motor-friccion-neta/spec.md` — 59 FR, 7 historias, 12 criterios de éxito

---

## Summary

Herramienta local de análisis que compara vehículos de inversión por su **retorno neto real en COP**
para un residente fiscal colombiano, descomponiendo toda la fricción y generando un memorando
defendible.

**Enfoque técnico:** monolito Python de un proceso con **núcleo hexagonal puro**. `motor/` calcula y
no toca red, disco ni UI; los adaptadores hablan con el mundo tras protocolos que el núcleo declara.
El estado vive en YAML versionado en git (dato curado, con historia auditable de vigencias) y
snapshots Parquet inmutables (dato descargable). Sin base de datos, sin contenedor, sin nube.

**La arquitectura no se decide aquí.** Está fijada por los **51 ADs** del *architecture spine*, que
pasaron dos rondas de Reviewer Gate adversarial. Este plan los **consume**. Un diseño que contradiga
un AD exige enmendar el AD primero, no eludirlo.

---

## Technical Context

**Language/Version**: Python 3.12, fijado en `.python-version`, gestionado con uv (`AD-15`)

**Primary Dependencies**: pydantic 2.13.4 · pyyaml 6.0.3 · httpx2 ≥2,<3 · streamlit 1.62.0 ·
matplotlib 3.11.1 · pyarrow 25.0.1 · jinja2 3.1.6 · playwright 1.62.0 · anthropic 1.0.0 ·
pytest 9.1.1 · ruff 0.16.4 — todas verificadas en vivo contra PyPI el 2026-08-24

**Storage**: Sin DBMS (`AD-3`). YAML en git para configuración, catálogo, perfiles y textos
normativos; Parquet inmutable direccionado por contenido para el caché de TRM

**Testing**: pytest, con casos calculados a mano obligatorios en `motor/fiscal/` y `motor/friccion/`

**Target Platform**: Windows 11 local, proceso único, sin exposición a red (`AD-2`, `AD-13`)

**Project Type**: Aplicación de escritorio local — Streamlit en localhost + CLI

**Performance Goals**: Memorando en PDF completo en < 120 s (SC-006). Sin objetivos de throughput:
un operador, ejecución por lotes

**Constraints**: Cero costo recurrente de infraestructura. Determinismo byte a byte. `float`
prohibido en todo `motor/`. Datos fiscales de clientes no salen de la máquina

**Scale/Scope**: 15 vehículos × 9 celdas = 135 conjuntos de resultados por corrida.
18 libros de asientos por vehículo (`AD-41`)

---

## Constitution Check

*GATE: debe pasar antes de la Fase 0 y volver a evaluarse tras la Fase 1.*

Compuertas derivadas de `.specify/memory/constitution.md` v1.0.0.

| # | Compuerta | Principio | ¿Pasa? | Cómo |
|---|---|---|---|---|
| G1 | Ninguna cifra la genera el LLM; hay guard de código | I (NO NEG.) | **SÍ** | `G-NUM` con tabla de normalización y casos negativos ([guards-llm.md](./contracts/guards-llm.md)) |
| G2 | Todo total se deriva del libro | II (NO NEG.) | **SÍ** | `LibroDeAsientos` append-only; solo agregan los flujos (`AD-46`) |
| G3 | Ningún parámetro tributario en código | III (NO NEG.) | **SÍ** | `config/*.yaml` + `test_arquitectura.py` prohíbe literales normativos (`AD-19`) |
| G4 | Procedencia declarada y propagada | IV (NO NEG.) | **SÍ** | `Procedencia` obligatoria; agregación peor-gana; `frozenset[ParametroId]` por asiento (`AD-49`) |
| G5 | Ninguna ruta emite lenguaje recomendatorio | V (NO NEG.) | **SÍ** | `G-CUMP` + `G-EST` |
| G6 | Ninguna dependencia de datos es de pago | VI | **SÍ** | TRM de `datos.gov.co`, sin credencial, verificado (`research.md` R-1) |
| G7 | Cálculo nuevo trae caso calculado a mano | VII (NO NEG.) | **SÍ** | Compuerta `AD-25` construida **antes** del primer cálculo — ver Orden de construcción |
| G8 | Toda salida pasa por el `Renderer` único | VIII | **SÍ** | `AD-10`; `G-EST` verifica los cuatro elementos legales |
| G9 | Matriz completa; celdas no disponibles con su razón | IX | **SÍ** | `ResultadoCelda` como suma cerrada (`AD-48`) |
| G10 | `motor/` sin E/S, red ni UI | X (NO NEG.) | **SÍ** | `test_arquitectura.py` por recorrido de AST (`research.md` R-2) |
| G11 | La solución es la más aburrida que satisface los principios | Gobernanza | **SÍ** | Sin DBMS, sin contenedor, sin servicios, sin nube |

**Reevaluación post-Fase 1:** las once siguen pasando. El diseño no introdujo ninguna violación.
La tabla de *Complexity Tracking* queda vacía.

**Contrato técnico:** los **51 ADs** de
`_bmad-output/planning-artifacts/architecture/architecture-AppInversiones-2026-08-24/ARCHITECTURE-SPINE.md`.

---

## Project Structure

### Documentación de esta feature

```text
specs/001-motor-friccion-neta/
├── plan.md              # este archivo
├── spec.md              # 59 FR
├── research.md          # Fase 0
├── data-model.md        # Fase 1
├── quickstart.md        # Fase 1 — validación end-to-end
├── contracts/
│   ├── puertos.md       # protocolos del núcleo
│   ├── config-yaml.md   # esquema de configuración tributaria
│   └── guards-llm.md    # contrato de los guards
├── checklists/
│   └── requirements.md
└── tasks.md             # Fase 2 — lo produce /speckit-tasks
```

### Código fuente

```text
motor/                   # NÚCLEO PURO — stdlib + decimal + pydantic, nada más (AD-1)
├── dominio/             # Money, Cantidad, Lote, Asiento, Libro, CurvaDeCambio,
│                        #   ResultadoCelda, Procedencia, Concepto, excepciones
├── friccion/            # secuencia anual canónica + paso terminal de realización (AD-22)
├── fiscal/              # único dueño del cálculo tributario (AD-20, AD-29..AD-44)
├── escenarios/          # una posición → 18 libros (3 × 3 × 2) (AD-41)
├── comparacion/         # barrido de N vehículos, deltas, ordenamiento (AD-26)
├── cumplimiento/        # alertas §7 — emite ids, no textos (AD-27)
└── puertos/             # Protocol: RepositorioConfig, RepositorioCatalogo,
                         #   FuenteMercado, RedactorNarrativo

adaptadores/
├── config/              # YAML + pydantic; centinela TODO (AD-7, AD-35)
├── mercado/             # httpx2 → datos.gov.co; snapshots Parquet (AD-11, AD-16)
├── llm/                 # cliente anthropic; solo redacta (AD-9)
└── render/              # guards, waterfall, Jinja2 → HTML → PDF (AD-10, AD-12, AD-23)

app/                     # Streamlit + CLI — compone, no calcula (AD-14)

config/
├── tributario/          # escenario-{a,b,c}.yaml
├── reajuste/            # art-70.yaml, art-73.yaml
├── alertas.yaml
└── brokers.yaml
catalogo/vehiculos.yaml
perfiles/

tests/
├── test_arquitectura.py # hace cumplir AD-1, AD-19, AD-45
├── dominio/  fiscal/  friccion/  render/  adaptadores/

.github/workflows/       # compuerta de calidad (AD-25)
```

**Structure Decision:** monolito de un paquete con límites internos fuertes. Los límites se
sostienen con un test de arquitectura, no con la red (`AD-13`). No hay `src/` ni separación
frontend/backend: no hay segundo consumidor ni equipo que desacoplar.

---

## Orden de construcción

El orden **no es negociable en su primer tramo**, y es un hallazgo del Reviewer Gate, no una
preferencia.

### Tramo 0 — La compuerta, antes que nada

`pyproject.toml`, `.python-version`, `uv.lock`, `ruff`, `pytest`, `tests/test_arquitectura.py`,
`.github/workflows/`, hook `pre-push`, `README.md`.

> **Por qué primero.** `AD-1` dice «falla el build» y el principio VII dice «no se mergea sin test».
> Ambas reglas delegaban a un mecanismo que no existía — el propio Reviewer Gate lo marcó como
> hallazgo alto. Si el primer cálculo financiero se escribe antes que la compuerta, el principio VII
> nace sin quien lo haga cumplir, y el momento de mayor riesgo del proyecto pasa sin red.

**Criterio de salida:** V-1 del quickstart pasa, incluida la prueba de que la compuerta *muerde*.

### Tramo 1 — Tipos del dominio

`Money`, `Cantidad`, `Procedencia`, `Reconocimiento`, `Lote` (con `lote_id` y `partir`), `Posicion`,
`Concepto`, `Asiento`, `LibroDeAsientos`, `CurvaDeCambio`, `ResultadoCelda`, excepciones.

Sostienen todo lo demás y no dependen de nada. **Doce de los quince pares de divergencia del
Reviewer Gate nacían de tenerlos declarados pero no especificados** — por eso van con
[data-model.md](./data-model.md) en la mano y con tests de invariantes propios.

### Tramo 2 — Configuración y catálogo

`adaptadores/config/`, esquema pydantic, centinela `TODO`, `config/*.yaml` y `catalogo/` **todos con
valores `TODO`**. Criterio de salida: **V-2 pasa** — el sistema se niega a producir cifras.

> Probar que el sistema **se niega a inventar** antes de probar que calcula bien. Es el principio III
> y el compromiso de no rellenar con datos inventados.

### Tramo 3 — Motor fiscal

`motor/fiscal/`: costo fiscal por lote (`AD-34`), reajuste excluyente (`AD-31`), asignación de venta
(`AD-39`), impuesto en dos fases (`AD-44`), contrafactual de devaluación (`AD-28`), fragmentación
(`AD-37`, `AD-47`), valor patrimonial (`AD-36`).

Es el módulo de mayor riesgo. Cada función entra con su caso calculado a mano.

### Tramo 4 — Motor de fricción

Secuencia anual canónica con paso terminal de realización (`AD-22`), TER condicionado (`AD-6`),
evento gravable declarado (`AD-5`), spread por modo de dividendo (`AD-51`).

### Tramo 5 — Escenarios, comparación y cumplimiento

18 libros por vehículo (`AD-41`), barrido y ordenamiento sobre celdas `Disponible` (`AD-26`,
`AD-48`), alertas por identificador (`AD-27`).

### Tramo 6 — Mercado

Adaptador `datos.gov.co` con `httpx2`, snapshots Parquet inmutables.

### Tramo 7 — Render y guards

Guards **antes** que las plantillas: son la condición de emisión, no un filtro posterior.
Luego waterfall en matplotlib, Jinja2 → HTML → PDF por Chromium.

### Tramo 8 — Interfaz

CLI primero (guionizable, testeable), Streamlit después. Celda de referencia + banda de
sensibilidad (FR-056…FR-059).

---

## Mapa: historias → módulos → ADs

| Historia | FR | Módulos | ADs que la gobiernan |
|---|---|---|---|
| **US1** — Comparar por retorno neto real (P1) | FR-005…FR-011 | `friccion`, `fiscal`, `comparacion` | `AD-4`, `AD-6`, `AD-8`, `AD-17`, `AD-22`, `AD-51` |
| **US2** — Impuesto por devaluación (P1) | FR-025 | `fiscal` | `AD-18`, `AD-28`, `AD-30`, `AD-43` |
| **US3** — Costo de fragmentación (P1) | FR-012…FR-017 | `dominio`, `fiscal` | `AD-29`, `AD-38`…`AD-40`, `AD-42`, `AD-44`, `AD-47` |
| **US4** — Nueve celdas (P1) | FR-018…FR-029 | `escenarios`, `comparacion` | `AD-24`, `AD-31`…`AD-33`, `AD-41`, `AD-48` |
| **US5** — Alertas de cumplimiento (P2) | FR-035…FR-038 | `cumplimiento` | `AD-19`, `AD-27` |
| **US6** — Memorando y cascada (P2) | FR-039…FR-048 | `adaptadores/llm`, `adaptadores/render` | `AD-9`, `AD-10`, `AD-12`, `AD-20`, `AD-23`, `AD-49` |
| **US7** — Valor patrimonial anual (P3) | FR-023 | `fiscal` | `AD-36`, `AD-46` |

**Transversales:** FR-030…FR-034 (configuración y procedencia) → `adaptadores/config` +
`AD-3`, `AD-7`, `AD-35`, `AD-43`, `AD-50`. FR-049…FR-051 (requisitos negativos) → se verifican por
ausencia y por el test de arquitectura.

---

## Riesgos

| Riesgo | Mitigación |
|---|---|
| `anthropic` 1.0.0 es un major de hace días que migró a `httpx2` | Está tras un puerto (`AD-9`); el radio de daño es un adaptador. `AD-16` fija un solo stack HTTP para que no convivan dos |
| `datos.gov.co` es un endpoint público sin SLA | El snapshot en caché es la fuente de verdad de una corrida; la red solo se toca para refrescar |
| `playwright install chromium` no lo resuelve `uv sync` | Paso explícito en el README y en el Tramo 0 |
| SC-006 (PDF < 2 min) contra 3 llamadas al LLM | `effort` bajo o medio para redacción; paralelizar las tres llamadas si no basta. Es afinamiento, no estructura |
| El `uv` del operador está seis meses desactualizado (0.10.5 vs 0.12.5) | Actualizar en el Tramo 0 |

---

## Complexity Tracking

Sin violaciones que justificar. Las once compuertas constitucionales pasan.

---

## Lo que este plan NO resuelve

Cuatro preguntas **normativas**, escaladas al tributarista del operador (`research.md` R-4). No
bloquean escribir código; **sí bloquean que el sistema produzca una sola cifra**, por diseño:

1. ¿La ventana del art. 70 es anual o acumulada?
2. ¿Qué campo del lote indexa el factor del art. 73?
3. ¿Aplica el reajuste a ETF del exterior, TES y FIC, y bajo qué condiciones?
4. Los valores de todas las series y tarifas.

`AD-43` las recibe como **campos declarados en configuración**, no como comportamiento cableado.
Mientras no estén, el cargador levanta `ParametroTributarioFaltante`.

> **Procedencia.** El material tributario disponible no está firmado por un profesional y contiene
> una contradicción interna entre dos de sus documentos. Todo parámetro derivado nace
> `supuesto_no_verificado` y esa marca se propaga hasta el artefacto final.
